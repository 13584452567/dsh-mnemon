---
id: "2ab89c06-182a-44ff-aee3-fb5457f6055e"
title: "HDC 投屏不出画真因：设备侧遗留 UI Test 扩展占用固定 @scrcpy_grpc_socket + 待机断流放宽（2026-10-25）"
description: "设备侧遗留 uitest 扩展进程占住固定抽象 socket 导致 onStart 永不返回的真因取证、启动前按前缀清理与待机断流重开设计、验证数字与 HDC server 转发未实测边界。"
status: "active"
created_at: "2026-10-24T23:38:07.546Z"
updated_at: "2026-10-24T23:38:07.546Z"
content_hash: "a2e2c9f47fdb5e717dde8ea07d7acf3e1a2d80a35dcc42889bc6bbd1e286cfc1"
source_paths:
  - "UotanStudio.Mirroring/Ohscrcpy/OhscrcpyServerManager.cs"
  - "UotanStudio.Mirroring/Ohscrcpy/OhscrcpyClient.cs"
  - "VERIFICATION.md"
session_ids:
  - "f3287c18-94d4-4a71-a9ff-ea356c62ae0a"
memory_body_ids:
  []
---

# HDC 投屏不出画：设备侧遗留扩展占用固定 gRPC socket（2026-10-25）

本轮以 DevEco 自带投屏插件的实际行为为金标准，排查「HDC 冷启动投屏不出画 / 视频 gRPC 未响应」。结论是**设备端固定抽象 socket 被历史遗留扩展进程占住**，不是采集库 ABI 不兼容；同时按用户要求放宽设备待机导致的流中断判定。逐字实测记录在仓库 `VERIFICATION.md`（自 L251 起）。

相邻但不同：宿主侧遗留投屏会话占用 device endpoint 见 `6f754ead-5fb1-4813-8e9c-cdc70dcbc7cb`；gRPC 缺单次尝试超时导致卡「正在连接」、取消「多久没帧即断联」见 `10a41837-82f0-43ce-8705-7353e1e215b7`；ohscrcpy 整体设计与所有权模型见 `fe4bd5e7-924b-45ec-aa91-975d63f3b737`。本文只记录**设备侧 socket 占用**这一类持有者与待机断流重开。

## 取证链（可复用）

用 DevEco 自带 `hdc.exe` 只做诊断，不参与产品运行：

1. `ps -ef` 中堆着 3 个历史失败尝试遗留的 `uitest start-daemon singleness --extension-name libuotan_hdc_mirror_<hash>.z.so` 进程。
2. 设备侧 gRPC 抽象 socket 名是**固定**的 `@scrcpy_grpc_socket`，`/proc/net/unix` 只有一个条目，被最早那个遗留扩展占着。
3. `hdc fport tcp:15555 localabstract:scrcpy_grpc_socket` 后直接探测：
   - `POST /ScrcpyService/onRequestIDRFrame` → HTTP 200 + `application/grpc` + 5 字节，说明扩展进程、h2 与转发链路都活着；
   - `POST /ScrcpyService/onStart` → **永不返回响应头**，即旧扩展的 onStart 处理器已卡死。
4. `kill -9` 掉占用 socket 的遗留扩展后，`onStart` 立即返回 200 + 32 字节视频数据，应用冷启动随即出画。

要点：抽象 socket 名不可协商，因此「旧扩展活着」与「新扩展就绪」在客户端看来都是连接成功但无响应。此前把「视频 gRPC 未响应 / 可重试失败 / Capture log (empty)」判为设备 ABI 能力不足（B 类快速失败）是**误分类**；DevEco 能投屏而本产品不能，差别只在于 DevEco 自己管理其扩展生命周期。诊断抓手是 `onRequestIDRFrame` 可用而 `onStart` 无响应头的这种**同 socket 内部分端点可用**形态。

## 修复

1. **启动前清理本产品遗留扩展**（`OhscrcpyServerManager.StartVideoDaemon`）：按精确前缀 `libuotan_hdc_mirror_` 找出遗留进程，`kill -9` 后最多等 2 s 确认消失；只处理本产品前缀，绝不按进程名批量杀，日志写中英双语摘要。`ps -ef` 解析沿用 `ParseProcessIds` / `ExcludeProcessLines`，过滤 `--extension-name ` 与 shell wrapper 行以免误杀自身命令行。
2. **待机断流不再判联**（`OhscrcpyClient`）：设备进入待机会主动关闭 gRPC 视频流；`ReadFrame` 遇到流结束改为「标记待机 + 限速重开 `onStart` + 重新应用码流配置」，只有 ReplyMessage 报文格式错误才是致命错误。重开预算 `ReopenVideoBudgetMs` 3 s、重试间隔 `ReopenRetryDelayMs` 2 s；payload 通道改为每次重连新建（`CreatePayloadChannel`，容量 8、`SingleReader/SingleWriter`、`FullMode=Wait`），旧 reader 的 `_readerFailure` 与 `ThrowStreamEnded` 语义保留，避免把上一代流的异常带进新一代。
3. **单次尝试上限**（与本轮诊断互补）：每次视频 gRPC 请求 4 s、关键帧请求 3 s 上限，让上层重试循环真正生效；失败分类维持 A（未就绪可重试）/ B（能力不足不可重试）/ C（已启动但视频失败），B 类引用设备采集日志。
4. **推送可见性门**：`WaitForRemoteFile` 以 `RemoteFileVisibleBudgetMs` 2 s 轮询确认推送文件对设备 shell 可见，避免慢速目标上「刚 push 完就 start」；扩展就绪预算 `ExtensionReadyBudgetMs` 6 s、轮询 200 ms。

## 验证（同一台华为 5XM0225C18001159，HDC 直连）

- 清理遗留扩展后冷启动出画：视频区均值 50.31、max 255（修复前该指标长期为 43.37 / 0 附近的黑场）。
- `hdc fport ls` 可见应用注册的 `tcp:…→localabstract:scrcpy_grpc_socket` 与 `tcp:…→localabstract:uitest_socket` 两条转发。
- 门禁：`dotnet build UotanStudio.slnx --no-restore -c Debug` 0 错误；Mirroring 127/127（1 硬件跳过）、Media 202/202（1 Linux 跳过）、Devices 249/249、Architecture 16/16；whitespace 与 style 零报告。

## 未验证边界

- 待机断流的自动重开只有代码路径与门禁验证，**未在真机等屏幕超时后复现**整条重开序列。
- **HDC server 转发路线**：`fport` 到两个抽象 socket 手工验证可用，但应用本轮跑的是直连路线（外部 hdc server 不在应用配置端口），端到端 server 转发投屏未实测；用户要求的「HDC USB 直写与 HDC Server 转发都能投屏」中后者仍是待验证项。
- 手工 `kill -9` 在本机 shell 下曾报 `Operation not permitted`，产品路径改用设备侧 `kill` 语义后正常；跨设备权限差异未穷举。

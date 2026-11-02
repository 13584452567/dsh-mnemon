---
id: "a8f55ea5-bea4-4769-840d-683303866320"
title: "ADB 直连被外部读者共享与设备扫描锁卡顿：机制、已实施修复、判定路径"
description: "WinUSB 多句柄共享 ADB 接口导致直连读到他人帧、路由层 fail-open 漏洞及 fail-closed/实时重探测修复、Scan() 扫描锁 250ms 上限与慢扫描日志、logcat 捕获 5s 退避与超时报文增强，含验证数字与未验证边界。"
status: "active"
created_at: "2026-10-24T20:20:39.699Z"
updated_at: "2026-10-24T20:20:39.699Z"
content_hash: "40285054e8a6643f0974a3a4181840bce9f95db789156793adedc580609a35d6"
source_paths:
  - "UotanStudio.Comm/Devices/Services/DebugBridgeSessionOpener.cs"
  - "UotanStudio.Comm/Routing/DebugBridgeRoute.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Discovery.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Logs.cs"
  - "UotanStudio/ViewModels/Devices/DeviceViewModel.cs"
  - "UotanStudio/ViewModels/Devices/DeviceViewModel.Connection.cs"
  - "VERIFICATION.md"
session_ids:
  - "82580b7c-4ddf-4e91-94ac-3fe96ec36bae"
memory_body_ids:
  []
---

# ADB 直连被外部读者共享 + 设备扫描锁导致刷新卡顿：机制、已实施修复与判定路径

本轮排查把三条看似无关的现象收敛到同一个结构问题：**ADB USB 接口在 Windows 上不独占**，以及**宿主把完整 USB 枚举放在可被 UI 路径递归进入的锁里**。结论与实现均已在仓库内落地。

## 一、WinUSB 多句柄共享接口是“会话没释放”的真实机制

排除掉的假设：本进程内的 ADB/logcat 流随进程退出而终止，WinUSB 句柄关闭后不再拉数据，无法解释“每次 24 字节都不同、0 ms 到达”的持续满速入站流量。

真实成因：
- 持续输出方在 **adb server 侧**——遗留的终端 `adb logcat`（Ctrl+C 未生效/窗口被关）、Android Studio、或被孤儿化的 `adb.exe` 一直附着同一接口全速读取，设备端会话因此常开。
- Windows WinUSB **允许同一接口被多个文件句柄同时打开**，驱动不做独占检查。所以直连会话仍能“打开成功”，但两个 reader 分割同一条 ADB 字节流；落到我方帧头位置的是别人帧中间的载荷（logcat 文本/二进制混合，高熵、每次不同、0 ms 到达）。

四条现象因此自洽：0 ms 全速、每次不同、全新启动同样复现、`adb devices` 确实能看到设备。

判定（不改代码、不重编译）：`adb kill-server`（或结束遗留 `adb.exe` / 关掉跑 logcat 的终端）后直连能握手 ⇒ 共用接口确认；`adb devices` 已空仍乱码 ⇒ 设备复合/端点问题。

## 二、路由层 fail-open 漏洞（为什么直连没被拦住）

1. `DebugBridgeSessionOpener.AnyServerListening` 在探测**抛异常**（超时/DNS/连接重置）时返回 `false`——把“无法判定”当成“没有 server”，直接放行直写。改为 **fail-closed**：无结论按“存在”处理，并记一条 `route` 级日志写明原因。
2. `AdbServerOwns`（Auto 模式）只读启动快照；app 先启动、server 后起来的时间窗里同样放行。现在快照显示“无 server”时**实时再探测一次**。
3. 握手失败结论日志附带实时 server 探测结果，日志本身即可区分两种成因：
   - `An ADB server is listening on 127.0.0.1:5037; … a server that already attached this device splits every frame with this session.`
   - `No ADB server is listening, so the endpoint itself is streaming non-ADB data …`

已知取舍：fail-closed 后，“server 在听但并未接管这台设备”时直连也会被拒，是否误伤需真机确认。

## 三、“点刷新卡死几秒”的根因：扫描锁

`DeviceMonitorService.Scan()` 在 `_scanLock` 内执行**完整 USB 枚举**（server 探测 + fastboot 序列号控制传输，端点异常时可拖到数秒），期间任何界面路径调用 `Scan()` 都被阻塞，点击刷新正好命中。

修复：
- 锁等待上限 **250 ms**；超时不阻塞，直接返回上一份已完成快照并记一条 Debug。
- 扫描 ≥400 ms 记 Info（`DeviceMonitor - Full scan took … ms`），控制器刷新单独计时——下次日志直接指出秒数花在哪一段。

并发注意：`ScanCore()` 用 `_controller.RefreshAsync().GetAwaiter().GetResult()` 同步阻塞等待；曾因日志插值写成 `{endpoints.Count}` 对方法组取值而触发 `CS1503`，此处改日志需保持编译期可解析的表达式。

## 四、logcat 抖动与超时报文

三簇日志的判读（与“有人在抢流量”属不同故障形态）：

| 日志 | 含义 |
| --- | --- |
| `Clearing logcat buffers timed out` | ADB shell `logcat -b all -c` 在 10 s 预算内无任何应答——链路执行不了命令，不是 logcat 本身坏 |
| `TaskCanceledException` @ `AdbConnection.SendAsync` | 调用方令牌在等待发送闸门时触发（10 s 预算到期），属“被取消”而非传输错误 |
| 每 ~11 s 一簇、共 11 次 `OperationCanceledException` | 日志视图捕获被反复拆掉重启，每次重启顺带取消一批在途 ADB 操作 |

修复：
- 同一设备捕获失败后，任意一次设备列表刷新（状态抖动）都会重启捕获——现在同设备重启带 **5 s 退避**，成功启动即清零。
- `ClearLogcatBuffersAsync` 的 `TimeoutException` 原来丢弃内部异常且无上下文；现在带上耗时、预算、实际路由（server/direct）、是否看到成功标记、设备侧诊断输出，并保留内部异常。

## 五、验证与未验证边界

- `UotanStudio` 0 警告 0 错误；`UotanStudio.Tests.Devices` 247/247、`Protocols` 38/38；改动文件 whitespace/style 零报告；结论同步写入 `VERIFICATION.md`。
- 某轮测试因并发会话在途改动（`AdbShutdownFarewellTests.cs` 引用不存在的 `IAdbStream.LocalId`）无法编译，改用 `--no-build` 跑上一份 dll——工作树存在其他会话的并发 Comm 改动时需注意这一点。
- 未验证：未在现场做 `kill-server` 前后对比；卡顿时长与 logcat 恢复只能靠下一次现场日志（`Full scan took … ms`）确认；无真机。

## 相关文档

- 外来帧证据与接口清单/多读者日志：`44373da3-33a5-4818-8a9d-fa21d5bc2e75`
- `Invalid message magic` 帧错位与启动时序竞态：`78613e8c-9182-4f03-8607-b11ce652e00d`

## 源码入口

- `UotanStudio.Comm/Devices/Services/DebugBridgeSessionOpener.cs`（`AnyServerListening`/`AdbServerOwns`）
- `UotanStudio.Comm/Routing/DebugBridgeRoute.cs`
- `UotanStudio/Services/Devices/DeviceMonitorService.Discovery.cs`（`Scan`/`ScanCore`）
- `UotanStudio/Services/Devices/DeviceMonitorService.Logs.cs`（捕获注册与重启）
- `UotanStudio/ViewModels/Devices/DeviceViewModel.cs`、`DeviceViewModel.Connection.cs`

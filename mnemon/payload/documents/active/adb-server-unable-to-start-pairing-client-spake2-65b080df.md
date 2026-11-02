---
id: "65b080df-c163-4577-8543-ecbb049d88ac"
title: "ADB server 模式扫码配对失败：“Unable to start pairing client” 根因与进程内 SPAKE2 回退（2026-10-25）"
description: "host:pair 结果帧两种形状的解析容错、AdbServerPairingClientException 失败分类、配对端点 5 秒实时 mDNS 刷新与进程内 SPAKE2 回退（ServerOnly 除外）、验证数字与真机未验证边界。"
status: "active"
created_at: "2026-10-25T02:50:30.702Z"
updated_at: "2026-10-25T02:50:30.702Z"
content_hash: "7b9d3f95a43fb9cea21384ce748efc13b9b607f463a476f40d306fcb7bed082b"
source_paths:
  - "UotanStudio.Comm/Protocols/Adb/AdbServerServices.cs"
  - "UotanStudio.Comm/Protocols/Adb/Pairing/AdbQrPairingService.cs"
  - "UotanStudio.Tests/Protocols/AdbRemoteProtocolTests.cs"
  - "UotanStudio.Tests/Devices/AdbPairingDiscoveryTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "a5b4fbd8-9d1a-4133-af18-c01117cc6cf9"
memory_body_ids:
  []
---

# ADB server 模式扫码配对失败：根因与回退设计

相关文档：`a1040553-a8fc-487d-948f-157183fd1cfa`（同一天的配对后连接端口自动发现）、`b1898c91-0104-446e-9ebf-b27f6dcb61c6`（ADB/HDC 远端连接与真机矩阵）。

## 症状

有 adb server 时扫码配对必然以「读写操作失败…」结束。用本机 platform-tools 36.0.0 的 server 做探针，向 `host:pair:<密码>:<ip>:<port>` 得到的报文等价于：

- server 先回 `OKAY`（请求确认），随后再回**一个结果帧**：`FAIL` + 长度 + 文本，或 `OKAY` + 长度 + 文本；
- 指向无监听端口（如 `127.0.0.1:37777`）→ `Failed: Unable to start pairing client.`；端口有监听时 server 不再报此错，继续做 TLS 握手。

结论：这句话的含义是「**adb server 连不上设备刚广播出来的配对端口**」，与密码或 SPAKE2 协议无关，是可恢复的失败。

## 根因

1. 原实现把该文本当成不可恢复的 `IOException` 直接抛出，于是只要 server 在监听，扫码配对就没有重试路径。
2. 原读取逻辑只认「确认 + 裸长度」，遇到「确认 + `OKAY`/`FAIL` + 长度 + 文本」的状态式结果帧会误判为 `Invalid ADB server response length.`，成功的 `host:pair` 也可能被当成失败。

## 设计要点

- **`AdbServerServices`：结果帧容错 + 失败分类**
  - `RunPairCommandAsync`（`AdbServerServices.cs:355`）同时接受两种帧形状（确认+状态+长度+文本、确认+长度+文本）；入口 `PairAsync` 在 `AdbServerServices.cs:236`。
  - `Unable to start/create pairing client` 分类为内部异常 `AdbServerPairingClientException`（`AdbServerServices.cs:694`），保留 server 原文，与「密码错误/连接中断」区分开。
- **`AdbQrPairingService`：server 配不上时改用工具箱自己的 SPAKE2**
  - 捕获该异常后（`ServerOnly` 模式除外，见 `AllowsInProcessPairing`，`AdbQrPairingService.cs:204`；捕获点 `:170`），先用 **5 秒实时 mDNS 重新解析配对端点**（`RefreshPairingEndpointAsync`，`:221`）——server 的 mDNS 表是缓存，常指向已关闭的旧端口，这是原失败的实际来源。
  - 再用本机 `~/.android/adbkey`（adb server 自身使用的同一把用户身份密钥）在进程内完成 TLS1.3 + SPAKE2 配对；配对成功后仍由 server `host:connect:` 建连，路由保持 `AdbServer`（连接后半段抽成 `ConnectThroughServerAsync`，`:390`）。
  - `ServerOnly` 路由模式保持原错误抛出，不违反「仅 server」约束；入口 `PairAsync` 在 `AdbQrPairingService.cs:135`。
- **测试与记录**：`AdbRemoteProtocolTests` 增加状态帧成功、启动失败分类两个用例；`AdbQrPairingRouteTests` 增加路由模式门控 theory；`AdbPairingDiscoveryTests` 增补连接端口选择与地址族匹配断言；`VERIFICATION.md` 的「扫码配对」条目改为实测结论（含 `AdbServerPairingClientException` 分类与结果帧两种形状）。

## 验证（Windows x64，.NET 10 Debug，/9w 环境）

- 编译 Comm/Tests 依赖链：0 警告 0 错误。
- 定向：`UotanStudio.Tests.Protocols` 45/45；`UotanStudio.Tests.Devices` 260/260；受影响两类 12/12、5/5。
- 全量：7347 总数 / 7340 通过 / 0 失败 / 7 跳过（多出的 1 个跳过是「本机有 adb server」的环境跳过）。
- `dotnet format` whitespace 与 style 均零报告（`STYLE_EXIT=0`）。
- NativeAOT publish exit 0（`UotanStudio.Desktop.exe` 154,700,800 B），只有第三方库既有 IL2026/IL2104/IL3053 警告（YamlDotNet、Common.Logging、Iciclecreek.Avalonia.Terminal、LTRData.Extensions 等），无本项目新增。
- 端到端探针：调用编译出的 `AdbServerServices.PairAsync` 打真实 server，返回 `AdbServerPairingClientException: ADB server: Failed: Unable to start pairing client.`，证明新分类生效。

## 未验证的边界

- 真实扫码端到端：本机没有可控的 Android 11+ 扫码环境，「回退后能否在真机连通」仍未验证；单元测试只覆盖帧形状、失败分类与路由门控。
- 手机在配对后是否撤回 `_adb-tls-connect` 广播、多网卡与 IPv6-only 链路同样未实测。
- 全解构建当时被正在运行的 `UotanStudio.Desktop`（设备管理器窗口）锁住 `bin/Debug` DLL 而未跑完，属环境限制而非编译错误；改动要生效需先关闭该应用再重建/运行。

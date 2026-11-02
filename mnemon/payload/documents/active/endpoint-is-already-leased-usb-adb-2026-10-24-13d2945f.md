---
id: "13d2945f-9a23-4bc7-8aa9-ac82e702b5c8"
title: "“Endpoint is already leased.”根因与投屏启动有界等待（USB ADB 直连，2026-10-24）"
description: "直连 USB ADB 端点被详情补全/日志抓取/投屏并发抢占导致投屏启动失败：DeviceEndpointBusyException 结构化占用 + AcquireForMirror 15s 有界退避的设计、验证数字与未验证边界。"
status: "active"
created_at: "2026-10-24T16:29:26.629Z"
updated_at: "2026-10-24T16:29:26.629Z"
content_hash: "f21f6b5bb6f59e3474be00a7050935d80575f18b07173140bcbfe0b880d628e9"
source_paths:
  - "UotanStudio.Comm/Devices/Services/DeviceEndpointBusyException.cs"
  - "UotanStudio.Comm/Devices/Services/DeviceController.cs"
  - "UotanStudio.Comm/Devices/Services/DeviceController.Sessions.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.ControlledSessions.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Operations.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Ohscrcpy.cs"
  - "UotanStudio.Tests/Devices/DeviceEndpointBusyTests.cs"
session_ids:
  - "267c8b4c-936f-434e-b6b5-83cbf726189e"
memory_body_ids:
  []
---

# 直连 USB ADB 设备的“Endpoint is already leased.”根因与投屏启动有界等待（2026-10-24）

## 现象与归因

投屏启动（`ScreenMirrorViewModel.StartForDevice` → `DeviceMonitorService.StartMirror`）在直连 USB ADB 设备上报错 `Endpoint is already leased.`，该文本是 `DeviceController` 内部独占端点表（`_leased`）的不变量原文，被直接当成启动失败原因写进日志。

根因不是租约泄漏，而是**同一端点的多个宿主操作竞争**：

- 设备出现/列表刷新后，`DeviceViewModel` 自动做详情补全 → `QueryViaAdbSession` → `RunAdb`，独占租用该 USB ADB 端点跑 `getprop`/电池/存储批量；随后 `FlashSuService.HasSuAsync` 再租一次。
- 日志抓取、`su` 探测、文件传输同样借用这条独占租约。
- 投屏启动既不等待也不重试，直接撞上被占用的端点。

为什么只在“USB ADB 端点直连路由”下出现：走 adb server 路由时详情补全用 `QueryViaAdbServer`，不占独占 USB 端点。`DeviceLease` 释放顺序本身正确（“释放后同一端点可立即重开”已写成回归测试）。

同类误报的另一来源（陈旧 USB 清单缓存导致 HDC 直连误判占用）见既有文档 `64a77aaf-9665-44d1-8b37-0bfd4194e9e2` 与 `HdcDirectConnection.ResolveUsbWithFreshCache`：停止 HDC server 会重绑定/重新枚举 USB 接口，缓存 key 变陈旧，需要失效缓存后重解析一次再判定“被其它宿主占有”。

## 修复设计

1. **结构化占用异常（Comm 层）**：新增 `UotanStudio.Comm/Devices/Services/DeviceEndpointBusyException.cs`，派生自 `InvalidOperationException`，保留原文消息 `Endpoint is already leased.` 并携带 `EndpointId`/`Endpoint`。三处占用拒绝改为抛出它：`DeviceController.cs` 的 `_leased.Add(endpoint.Id)` / `_leased.Add(id)` 两处，以及 `DeviceController.Sessions.cs` 的 `_leased.Add(endpoint.Id)`。独占语义、fail-closed 与跨路由拒绝逻辑不变。
2. **投屏入口做有界占用等待（宿主层）**：`DeviceMonitorService.ControlledSessions.cs` 新增 `AcquireForMirror`：**只对“端点被占用”重试**，15 s 预算、100→500 ms 退避；其它失败照常立即抛出。ADB 与 HDC 两条投屏入口都改用它（`DeviceMonitorService.Operations.cs`、`DeviceMonitorService.Ohscrcpy.cs`）。预算耗尽后抛可操作的双语 `IOException`（提示等待其它操作、关闭日志抓取后重试），不再泄露控制器内部原文。
3. **回归测试**：`UotanStudio.Tests/Devices/DeviceEndpointBusyTests.cs`——占用时抛类型化异常、释放后同一端点可立即重开、忙碌后等待成功、持续占用转为可操作错误。

设计取舍：把“等待/退避”放在宿主门的投屏入口而不是 `DeviceController` 内部，保持 Comm 的独占语义仍是 fail-closed 快失败；需要容忍短时占用的调用方显式选择有界等待，避免全局把真实冲突拖成超时。

## 验证（Windows x64，Debug，主树工作区）

- `dotnet build UotanStudio.slnx --no-restore -c Debug`：0 错误，仅既有 `UotanStudio.Tests/Protocols/SpacemiTFlashTests.cs` 的 xUnit1051 警告。
- 全量测试：**7250 总数 / 7244 通过 / 6 跳过 / 0 失败**（约 2m41s）。跳过项均为既有环境门控（`UOTAN_EROFS_CORPUS`、`UOTAN_HDC_MIRROR_DIRECT`、随包 scrcpy-server.jar、Project 打包已知缺陷等）。
- 定向：`DeviceEndpointBusyTests` 4/4、`UotanStudio.Tests.Devices` 226/226、`MirrorSessionFacadeTests` 12/12、`McpBootProtocolToolTests` 9/9、`HeadlessControllerGatewayTests` 10/10。
- `dotnet format … whitespace|style --verify-no-changes` 两道零报告。
- NativeAOT：`dotnet publish UotanStudio.Desktop/UotanStudio.Desktop.csproj -c Release -r win-x64 --self-contained -p:PublishAot=true` exit 0，仅有第三方既有 `IL2104/IL3053` 裁剪警告（LTRData.Extensions、YamlDotNet、Common.Logging）。

## 未验证边界

- 真实设备上“详情补全”和“投屏启动”的实际重叠窗口未在硬件上复现；15 s 预算是否覆盖最慢补全路径没有实测。
- 日志抓取长期占用端点时，GUI 上的等待提示未逐项目视。
- 结论来自假件与控制器单测；真实 USB 设备上的并发抢占（多宿主进程、WinUSB/libusb 后端差异）未验证。

## 源码入口

- `UotanStudio.Comm/Devices/Services/DeviceController.cs`、`DeviceController.Sessions.cs`、`DeviceEndpointBusyException.cs`
- `UotanStudio/Services/Devices/DeviceMonitorService.ControlledSessions.cs`、`DeviceMonitorService.Operations.cs`、`DeviceMonitorService.Ohscrcpy.cs`、`HdcDirectConnection.cs`
- `UotanStudio.Tests/Devices/DeviceEndpointBusyTests.cs`
- 变更记录已写入仓库 `VERIFICATION.md`。

## 相关文档

- `b1898c91-0104-446e-9ebf-b27f6dcb61c6`：ADB/HDC 远端连接修复与真机验证（路由矩阵、fail-closed 前提）。
- `64a77aaf-9665-44d1-8b37-0bfd4194e9e2`：投屏断流与 HDC 三类故障隔离（解码线程、USB 清单缓存、server 客户端生命周期）。

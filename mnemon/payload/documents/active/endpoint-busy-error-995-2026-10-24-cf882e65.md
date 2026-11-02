---
id: "cf882e65-ccd3-495b-88c1-257498f9f5b5"
title: "投屏 endpoint busy 第二类持有者：无界设备详情补全 + ERROR 995 折叠成零字节读（宿主抢占设计，2026-10-24）"
description: "在遗留投屏会话回收之外的新归因：刷新触发的设备详情补全无界占用端点（7 条 shell 命令 + HasSuAsync 无取消），且 WinUSB 异步 995=ERROR_OPERATION_ABORTED 被折叠成零字节非超时读使协议循环空转。BeginBackgroundEndpointWork/CancelBackgroundEndpointWork/ReleaseMirrorEndpointHolders 抢占 + 补全 20s/8s 有界 + 995 改致命的设计、实测数字（10/10、2/2、237/237）与真机未验证边界。"
status: "active"
created_at: "2026-10-24T18:55:38.300Z"
updated_at: "2026-10-24T18:55:38.300Z"
content_hash: "78b09c12799ae78104df6584eae6df4a44cf759230736d33984c7e2d811af396"
source_paths:
  - "UotanStudio/Services/Devices/DeviceMonitorService.ControlledSessions.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Operations.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Ohscrcpy.cs"
  - "UotanStudio/ViewModels/Devices/DeviceViewModel.Details.cs"
  - "UotanStudio.Comm/Transports/Usb/Backend/Windows/WinUSBDevice.AsyncIO.cs"
  - "UotanStudio.Comm/Transports/Usb/Backend/Windows/WinUSBDevice.cs"
  - "UotanStudio.Tests/Devices/DeviceEndpointBusyTests.cs"
  - "UotanStudio.Tests/Devices/WinUsbAbortedTransferTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "b92c19e2-e219-480c-afe3-bba3b6e103ef"
memory_body_ids:
  []
---

# 投屏启动 endpoint busy 第二类持有者：无界的设备详情补全 + ERROR 995 被折叠成零字节读

前两轮见既有文档 `13d2945f-9a23-4bc7-8aa9-ac82e702b5c8`（`DeviceEndpointBusyException` + `AcquireForMirror` 15 s 有界退避）与 `6f754ead-5fb1-4813-8e9c-cdc70dcbc7cb`（回收本宿主遗留投屏会话）。本文记录这两轮都管不到的一类持有者：**刷新触发的设备详情补全**，以及让它"看不见也松不掉"的 USB 异步错误归类缺陷。

## 现象与因果链

日志同时出现 `native error 995` 与投屏启动 `endpoint is busy`，但两者不是"USB 链路坏了"：

1. **995 = `ERROR_OPERATION_ABORTED`，不是链路故障。** `UotanStudio.Comm/Transports/Usb/Backend/Windows/WinUSBDevice.AsyncIO.cs` 的异步批量 IN 把 995 归入"非断连错误 → 可重试"，折叠成 `UsbChunkResult.Error`；`ReadPacket` 再把它变成**零字节、非超时**的读，`AdbConnection.ReadExactIntoAsync` 在 `n == 0 && allowIdle` 分支里直接 `continue` —— 协议循环以为只是空闲，既不报错也不释放端点。同步 `ReadChunk` 对同一错误是 `Fatal`，只有异步路径漏了。
2. **真正占着端点的持有者是详情补全，且整段无界。** `UotanStudio/ViewModels/Devices/DeviceViewModel.Details.cs` 一次补全最多 7 条 shell 命令（`getprop` / `wm size` / `wm density` / `dumpsys battery` / `/proc/meminfo` / `df`），每条走 `ExecuteCommand` 的 30 s 默认期限且不接取消；随后 `HasSuAsync` 又传 `CancellationToken.None`，再探测 2×6 s。刷新触发的两次扫描正好点燃补全，投屏启动只等 15 s（100→500 ms 退避），预算耗尽即 `endpoint is busy`。旧回收逻辑只回收遗留投屏会话，管不到后台补全。

结论：容忍（有界等待）与回收（遗留投屏会话）之外，还需要**本宿主后台工作的抢占**与**补全本身有界**两条正交修复。

## 修复设计

**宿主抢占（主修复）** — `UotanStudio/Services/Devices/DeviceMonitorService.ControlledSessions.cs`
- 新增 `BeginBackgroundEndpointWork` / `CancelBackgroundEndpointWork` / `ReleaseMirrorEndpointHolders`：后台工作按设备登记，投屏启动首次遇到占用时**先取消本宿主为该设备登记的后台工作**，再立即重试。
- ADB 与 HDC 两种接口按 `PhysicalId` 归并（`IsSamePhysicalTarget`），遗留投屏会话回收也改用这套匹配。
- 接线：`DeviceMonitorService.Operations.cs`（`StartMirror` 路径）与 `DeviceMonitorService.Ohscrcpy.cs`（`StartOhscrcpyMirror`）把 `releaseStaleOwner` 换成 `ReleaseMirrorEndpointHolders`。

**补全有界** — `UotanStudio/ViewModels/Devices/DeviceViewModel.Details.cs`
- 整段 20 s 预算 + 单条命令 8 s；令牌传给 `RunAdb`、`HasSuAsync` 与 shell 客户端。
- 取消按"暂无详情"处理，不抛到界面。

**Comm 侧同类错误不再静默** — `WinUSBDevice.AsyncIO.cs` 与 `WinUSBDevice.cs`
- 995 归为致命批量传输错误（读写对称；超时自中止路径仍返回 0），把"传输被中止"以故障上报，让会话及时松开端点；双语 XML 注释同步改写。

取舍：抢占只针对**本宿主自建**的后台工作（登记表内），不触碰其它进程的合法占用；Comm 层继续保持 fail-closed 快失败，容忍、回收与抢占都在宿主投屏入口一侧。

## 验证（本次实测）

- `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 错误、2 条既有 `xUnit1051` 警告。
- `DeviceEndpointBusyTests` 10/10（新增物理身份匹配 2 例、后台工作取消 2 例；位于 `UotanStudio.Tests/Devices/DeviceEndpointBusyTests.cs`）、`WinUsbAbortedTransferTests` 2/2（`UotanStudio.Tests/Devices/WinUsbAbortedTransferTests.cs`）、`UotanStudio.Tests.Devices` 237/237。
- 改动文件 `dotnet format` whitespace 与 style 两道门零报告；本轮记录写入仓库 `VERIFICATION.md`。
- 过程中一次 `CS0051`（`EndpointWorkScope` 构造函数参数 `BackgroundEndpointWork` 可访问性低于方法）修正后清零。

## 未验证边界

- 无可用硬件：真机上"刷新 → 补全 → 投屏"的重叠窗口、抢占后 15 s 是否够、995 改致命后各协议在真机上的重试表现均未复现；GUI 未目视。

## 相关文件

- `UotanStudio/Services/Devices/DeviceMonitorService.ControlledSessions.cs`
- `UotanStudio/Services/Devices/DeviceMonitorService.Operations.cs`
- `UotanStudio/Services/Devices/DeviceMonitorService.Ohscrcpy.cs`
- `UotanStudio/ViewModels/Devices/DeviceViewModel.Details.cs`
- `UotanStudio.Comm/Transports/Usb/Backend/Windows/WinUSBDevice.AsyncIO.cs`
- `UotanStudio.Comm/Transports/Usb/Backend/Windows/WinUSBDevice.cs`
- `UotanStudio.Tests/Devices/DeviceEndpointBusyTests.cs`
- `UotanStudio.Tests/Devices/WinUsbAbortedTransferTests.cs`
- `VERIFICATION.md`

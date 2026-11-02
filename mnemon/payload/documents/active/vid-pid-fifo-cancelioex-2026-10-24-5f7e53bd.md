---
id: "5f7e53bd-684f-407a-a6e6-1dc10fa79536"
title: "序列号回退 VID:PID 与端点永不归还：FIFO 窥视闸门、粘性序列号缓存与 CancelIoEx 中止确认隔离（2026-10-24）"
description: "在既有端点占用三篇之外的新归因与修复：Windows 序列号窥视需独占接口导致回退 VID:PID（FifoGate 到达顺序闸门 + 按实例 ID 的粘性序列号缓存，成功 30 分钟/失败抑制 2 秒）；CancelIoEx 后无界等待驱动确认使 DeviceLease 卡在会话步、端点永不归还（5 秒有界 + QuarantinedTransfers 保留 UAF 防护并抛 ERROR_OPERATION_ABORTED）；详情补全 30s/45s 并接取消与 Debug 日志。含 245/245 等实测数字与真机未验证边界。"
status: "active"
created_at: "2026-10-24T19:07:56.032Z"
updated_at: "2026-10-24T19:07:56.032Z"
content_hash: "82ae75d857aba1fa254047cb530fac08ce4882993d53d051d26d9c8229244aa9"
source_paths:
  - "UotanStudio.Comm/Transports/Usb/Core/FifoGate.cs"
  - "UotanStudio.Comm/Transports/Usb/Backend/Windows/UsbSerialCache.cs"
  - "UotanStudio.Comm/Transports/Usb/Backend/Windows/WinUSBFinder.cs"
  - "UotanStudio.Comm/Transports/Usb/Backend/Windows/WinUSBFinder.Metadata.cs"
  - "UotanStudio.Comm/Transports/Usb/Backend/Windows/WinUSBDevice.AsyncIO.cs"
  - "UotanStudio.Comm/Devices/Services/DeviceCommandDispatcher.cs"
  - "UotanStudio/ViewModels/Devices/DeviceViewModel.Details.cs"
  - "UotanStudio.Tests/Devices/UsbSerialCacheTests.cs"
  - "UotanStudio.Tests/Devices/FifoGateTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "a6cecb73-74c0-4ed7-ae24-2ce1dd5a10a5"
memory_body_ids:
  []
---

# 序列号回退 VID:PID 与端点永不归还：粘性序列号缓存 / FIFO 闸门 / CancelIoEx 中止确认有界化（2026-10-24）

本文承接既有投屏 endpoint busy 三篇文档：`13d2945f-9a23-4bc7-8aa9-ac82e702b5c8`（结构化占用 + 15 s 有界退避）、`6f754ead-5fb1-4813-8e9c-cdc70dcbc7cb`（回收本宿主遗留投屏会话）、`cf882e65-ccd3-495b-88c1-257498f9f5b5`（详情补全无界 + ERROR 995 折叠）。本文记录它们都管不到的两件事：**序列号在重枚举后回退成 VID:PID**，以及**取消/超时后端点永不归还**的真正卡点。

## 症状一：启动时有序列号，随即回退成 VID:PID

Windows 上序列号只有两条来源：

1. 设备实例 ID 的最后一段——只有 Windows 首次插入时读到 iSerial 才是序列号，否则是 `6&1a2b&0&1` 这类含 `&` 的定位串；
2. 一次 `WinUsb_GetDescriptor` 窥视（`PeekSerialNumber`）——需临时打开接口，**接口被本程序或其它进程占着时返回 null**。

因此时序是：启动时接口空闲 → 窥视成功 → 显示序列号；随后会话/扫描占住接口 → 窥视失败 → `DeviceMonitorService.Discovery` 回退 `{VID:X4}:{PID:X4}`。副作用不止显示：`BuildDeviceKey` 从 `serial:` 落到 `path:`，同一设备条目身份会抖动。

关于"指令池化、先进先出"：**协议指令本就是每设备 FIFO**——`UotanStudio.Comm/Devices/Services/DeviceCommandDispatcher.cs` 用 `Channel` + 单读者，一次一条按 FIFO 执行，取消某次提交只影响该指令。落在池外的只有这次枚举期的裸句柄窥视，所以再池化一次协议指令不会改动该显示问题，真正要做的是把窥视纳入同一套先来后到。

## 修复设计（症状一）

- **`UotanStudio.Comm/Transports/Usb/Core/FifoGate.cs`**：严格到达顺序的串行闸门（票号链 + `TaskCompletionSource`），`PendingCount` 供诊断与判定；排队者一定在持有者之后放行，不会被后来的扫描插队。
- **`UotanStudio.Comm/Transports/Usb/Backend/Windows/UsbSerialCache.cs`**：按设备实例 ID 限定的"粘性序列号"——成功条目 30 分钟有效，失败条目只抑制 2 秒（避免一次扫描对同一占用接口连开两次句柄）；重新插入时实例 ID 变化故不串号。
- **`WinUSBFinder.cs` / `WinUSBFinder.Metadata.cs`**：新增 `ResolveEnumeratedSerial`（实例 ID 后缀 → 缓存 → 闸门内的单次窥视），枚举完成后再把 `SerialNumber` 回写缓存；窥视失败时**沿用上次读到的序列号**，不再降级成 VID:PID。
- 测试：`UotanStudio.Tests/Devices/UsbSerialCacheTests.cs`（6 例）、`FifoGateTests.cs`（2 例）。

## 症状二：投屏仍报 endpoint busy，且设备管理器没有详细信息

两个症状同时出现说明**端点压根没回到池里**——详情补全要的也是同一条独占端点。

卡点在 `UotanStudio.Comm/Transports/Usb/Backend/Windows/WinUSBDevice.AsyncIO.cs` 的取消/超时路径：`CancelIoEx` 之后是**无界等待**驱动确认中止（为防 UAF）。驱动一旦不确认：

1. 该批量 IN 永不完成 → ADB 消息循环永不结束；
2. `DeviceLease` 清理顺序是 流 → 会话 → 传输 → 端点，卡在会话一步 → **端点占用永不释放**；
3. 投屏等满 15 s 报 `endpoint is busy`，详情补全被同一条端点拒绝，于是界面没有设备详情。

上一轮把 995 归为致命只解决了"错被吞掉"，跨不过这步无界等待，所以现象未变。

## 修复设计（症状二）

1. **中止确认有界 + 隔离（核心）**：5 s 内驱动不确认中止，就把固定的 `OVERLAPPED` 与事件移入 `QuarantinedTransfers` 隔离区——**不释放**内核可能仍引用的内存（保留 UAF 防护），同时以 `ERROR_OPERATION_ABORTED` 抛错，让会话失败、租约走完清理、端点回到池里。代价：每次卡死泄漏一块固定 `OVERLAPPED` + 一个事件，换来的是一条坏管道不会永久锁死设备。
2. **详情查询预算**：`UotanStudio/ViewModels/Devices/DeviceViewModel.Details.cs` 单条命令 30 s（恢复原值）、总预算 45 s，改用 `CreateShellClient(...).ExecuteAsync(timeoutMs, token)`，命令仍可被投屏启动抢占取消；查询失败不再静默，写 `DeviceManager` Debug 日志，下次可直接区分超时 / 取消 / 传输故障。
3. 上一轮的抢占与遗留会话回收保持不变：首次占用先取消本宿主登记的后台工作与遗留投屏会话，再立即重试。

## 验证（本次实测，Windows x64 / Debug）

- `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 错误、2 条既有 `xUnit1051` 警告。
- `UotanStudio.Tests.Devices` 245/245；定向 `DeviceEndpointBusyTests` 10/10、`WinUsbAbortedTransferTests` 2/2、`UsbSerialCacheTests` 6/6、`FifoGateTests` 2/2。
- 改动文件 `dotnet format` whitespace 与 style 两道门零报告（过程中一次 `IDE0330` 提示改用 `System.Threading.Lock` 已修正）；本轮记录写入仓库 `VERIFICATION.md`。

## 未验证边界

- 没有真实 USB 设备：未复现"启动 → 有序列号 → 回退"的实际时序；窥视失败频率、重枚举间隔与 2 秒失败抑制是否匹配真机未实测。
- `CancelIoEx` 不被驱动确认这条路只能靠代码复核，硬件上无法复现；隔离区在长期反复卡死下的内存占用没有压测。
- 现场定位提示：日志出现 `WinUSB: the driver did not acknowledge the aborted overlapped transfer …` 表示管道/设备真卡死；出现 `DeviceManager … ADB detail query for … stopped` 表示详情因超时或取消中断。

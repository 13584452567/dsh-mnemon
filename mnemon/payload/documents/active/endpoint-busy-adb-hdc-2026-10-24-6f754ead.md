---
id: "6f754ead-5fb1-4813-8e9c-cdc70dcbc7cb"
title: "投屏启动 endpoint busy 真因：本宿主遗留投屏会话持有端点与回收设计（ADB/HDC，2026-10-24）"
description: "在既有 15s 有界等待之上，新归因：本宿主上一次投屏会话遗留持有独占端点。TrackMirrorSession 按物理序列号登记 + AcquireForMirror 先回收再重试的设计、验证数字（6/6、231/231、12/12）与真机未验证边界。"
status: "active"
created_at: "2026-10-24T17:24:41.211Z"
updated_at: "2026-10-24T17:24:41.211Z"
content_hash: "f9e51d4711284e3e5c67d05407b72fdded518c37212ed6750be187f42fffd623"
source_paths:
  - "UotanStudio/Services/Devices/DeviceMonitorService.ControlledSessions.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Operations.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Ohscrcpy.cs"
  - "UotanStudio.Tests/Devices/DeviceEndpointBusyTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "059c739f-f24b-42af-a6ce-2f1e44076c62"
memory_body_ids:
  []
---

# 投屏启动 "The device endpoint is busy" 真因：本宿主遗留投屏会话仍持有端点（回收设计）

上一轮有界等待设计见既有文档 `13d2945f-9a23-4bc7-8aa9-ac82e702b5c8`（`DeviceEndpointBusyException` + `AcquireForMirror` 15 s 退避）。本文只记录它未覆盖的一类持有者：**本宿主自己上一次投屏会话**。

## 现象与归因

日志序列：`00:43:27 Input injection failed … exit code 2 … "please press the 2017 key first"`（设备侧拒绝 uinput 注入，`OhscrcpyShell.Run` 如实上报退出码与原始输出，宿主无 bug，只标志该 HDC 投屏会话已不正常）→ `00:43:55` 会话清理走到最后一步（恢复 UI Test 控制代理）→ `00:44:15` 对同一台物理设备发起 USB ADB 直连投屏 → `00:44:30 Start failed: The device endpoint is busy`，正好耗尽 15 s 预算。

结论："投屏会话结束 → 独占端点归还"这条链**没有和下一次启动串行化**。仅靠有界等待，会把「本宿主上一次投屏会话仍持有端点」误判成详情刷新 / root 探测 / 日志抓取 / 文件传输抢占；端点只有一个持有者，等待不会自愈。已被关闭或被替换的投屏文档遗留的会话，正是启动方无法区分的那一类持有者。

## 修复设计（宿主层，未改动 Comm 独占语义）

文件：`UotanStudio/Services/Devices/DeviceMonitorService.ControlledSessions.cs`

1. **本宿主投屏会话登记表**：新增 `TrackMirrorSession` / `ReleaseTrackedMirrorSession`。按**物理设备序列号**登记并同时记下发现键，使同一设备的 ADB 与 HDC 两种接口发起的后续启动都能命中同一条遗留会话；登记新会话时先释放被替换的旧会话。
2. **`AcquireForMirror` 增加"回收遗留持有者"步骤**：首次 `DeviceEndpointBusyException` 时先停掉本宿主为该设备遗留的投屏会话，立即重试一次；只有仍占用才进入原有 100→500 ms 有界等待，超时依旧抛可操作的双语 `IOException`。
3. **两个启动入口接线**：`StartMirror`（ADB 直连/server，`UotanStudio/Services/Devices/DeviceMonitorService.Operations.cs`）与 `StartOhscrcpyMirror`（HDC，`UotanStudio/Services/Devices/DeviceMonitorService.Ohscrcpy.cs`）都执行回收 + 登记。回收时写一条 Info 日志说明"停了哪台设备的旧会话、为什么"，使下次日志能直接看到持有者，而不是继续猜成日志抓取。

取舍：回收只针对**本宿主自建**的会话（登记表内），不触碰其它进程或其它路由的合法占用；Comm 层仍是 fail-closed 快失败，容忍与回收都放在宿主投屏入口。

## 验证（本次实测数字）

- `DeviceEndpointBusyTests` **6/6**，含新增 `AcquireForMirror_LeftoverOwnSessionReclaimed_RetriesWithoutDelay`、`AcquireForMirror_ReclaimNeverFreesEndpoint_ReclaimsOnceThenThrowsActionableIOException`。
- `UotanStudio.Tests.Devices` **231/231**；`MirrorSessionFacadeTests` **12/12**。
- 改动文件 `dotnet build` 0 错误；`dotnet format whitespace|style --verify-no-changes` 零报告。
- 构建与定向测试在 `%TEMP%` 下的隔离副本中完成（工作区当时被并行会话的 `CompositionFramePresented` 删除改动卡住，未触碰无关文件），副本用完删除；实测记录写入仓库 `VERIFICATION.md`。

## 未验证边界

- 真实设备上"遗留会话 → 新启动"的时序与回收耗时（本轮无硬件），GUI 表现未目视。
- 工作区整体编译需等并行会话的 UI 改动收尾后，用 `dotnet build UotanStudio.slnx --no-restore -c Debug` 复核。

## 相关文件

- `UotanStudio/Services/Devices/DeviceMonitorService.ControlledSessions.cs`
- `UotanStudio/Services/Devices/DeviceMonitorService.Operations.cs`
- `UotanStudio/Services/Devices/DeviceMonitorService.Ohscrcpy.cs`
- `UotanStudio.Tests/Devices/DeviceEndpointBusyTests.cs`

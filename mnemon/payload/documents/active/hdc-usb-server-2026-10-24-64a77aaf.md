---
id: "64a77aaf-9665-44d1-8b37-0bfd4194e9e2"
title: "投屏断流与 HDC 连接三类故障隔离：解码线程、USB 清单缓存、server 客户端生命周期（2026-10-24）"
description: "三类故障的根因与修复边界：scrcpy 解码线程被单帧呈现异常杀死、USB 清单缓存陈旧导致 HDC 直连误报占用、HDC server 客户端 disposed 级联与 fail-fast 策略。"
status: "active"
created_at: "2026-10-23T20:16:07.390Z"
updated_at: "2026-10-23T20:16:07.390Z"
content_hash: "ed5d38202673a93b43890cf6859fc309b9d18014d7490aece84be915b96a94a6"
source_paths:
  - "UotanStudio/ViewModels/Editors/ScreenMirrorViewModel.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Transactions.cs"
  - "UotanStudio/Services/Devices/HdcDirectConnection.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcServerSyncClient.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcServerAppClient.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcSession.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcServerRemoteCommand.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcServerClient.cs"
  - "UotanStudio.Mirroring/Ohscrcpy/OhscrcpyServerManager.cs"
session_ids:
  - "3f1f093d-8607-406b-bd29-1921ddbd6f6d"
memory_body_ids:
  []
---

# 投屏断流与 HDC 连接三类故障隔离（2026-10-24）

一条修复轮次同时处理三个独立故障：ADB scrcpy 投屏一段时间后解码中断、无 server 时 USB HDC 误报“被其它主机占用”、HDC Server 投屏出现 disposed 客户端级联。前置架构见 `15613614-f9e3-4daa-b1c0-a2aa9cd9528d`（HDC server 路由会话化）与 `b1898c91-0104-446e-9ebf-b27f6dcb61c6`（ADB/HDC 远端连接）；ohscrcpy 链路与清理顺序以 `fe4bd5e7-924b-45ec-aa91-975d63f3b737` 与 `8f130951-1500-4159-8e8f-8e106efe5232` 为准。本文只记录本轮新增的故障语义与修复理由。

## 1. 解码线程被单帧呈现异常杀死（`Decode failed: Object reference not set...`）

现象：投屏运行一段时间后视频停住并报 `Decode failed`，但触摸/控制仍然正常。

根因链路：解码线程在 `decoder.Decode` 内部回调 `OnDecodedFrame`；帧呈现路径抛出的空引用异常直接冒泡出解码回调，被 `DecodeLoop` 外层 catch 兜住后**终止整个解码线程**。控制 socket 独立于解码循环，因此触摸照常工作——这是关键的诊断特征：控制可用 + 视频静止 = 解码线程已死，而不是网络或设备端问题。

修复（`UotanStudio/ViewModels/Editors/ScreenMirrorViewModel.cs`）：

- 对四条帧回调做异常隔离：`OnDecodedFrame`、dma-buf 帧、Windows 共享纹理帧、IOSurface 帧。单帧呈现失败只丢弃该帧，不再终止解码线程。
- 失败日志只记录一次 Warn，避免坏帧刷屏。
- 异常路径仍执行配对的资源收尾：共享纹理/IOSurface 帧调用 `MarkPresented()`，dma-buf 帧调用 `Dispose()`，防止槽位泄漏最终演变成别处的新故障。
- `Decode failed` 日志改为输出完整异常（含堆栈），使后续失败能定位到具体帧路径或解码器。

原则：帧呈现属于可丢帧操作，任何单帧异常都不得越过解码循环边界。

## 2. 无 HDC server 时 USB 直连误报“another host may own it”

现象：停止 HDC server 后，USB HDC 直连失败并统一报“接口被其它主机占用”。

根因：`ResolveDirectUsbDevice` 使用 500ms 的 USB 清单缓存。停止 server 后 USB 设备可能重绑定/重枚举，缓存中的旧 `DeviceKey` 已失效，`OpenLeaseByKey` 因此返回 `null`，最终被统一归类成“占用”，掩盖真实原因。

修复：

- `UotanStudio/Services/Devices/DeviceMonitorService.Transactions.cs` 新增 `InvalidateUsbInventoryCache()`。
- `UotanStudio/Services/Devices/HdcDirectConnection.cs` 在首次解析/打开失败时清空缓存、重新解析 USB 身份并重试一次。
- 错误信息包含序列号与具体设备键，用于区分“真的被其它驱动/进程持有”与“旧键失效”。

原则：短期清单缓存必须在打开失败时失效重试，否则陈旧键的典型表现会被误报为独占冲突。

## 3. HDC Server 投屏的 disposed 客户端级联

根因：`HdcServerSyncClient` / `HdcServerAppClient` 复用了会话持有的同一个 `HdcServerClient`。server 路由的会话/租约一旦被监视循环释放，后续每个采集库候选都会在已释放的客户端上继续重试，形成日志里的 disposed 级联；传输错误还会跑完全部 5 个候选库并反复清理，进一步掩盖真正原因。

修复：

- `UotanStudio.Comm/Protocols/Hdc/HdcServerSyncClient.cs`、`HdcServerAppClient.cs`：构造时捕获 host/port，每次文件/应用任务创建独立的 `HdcServerClient`，不再依赖会话内可被释放的实例。
- `UotanStudio.Comm/Protocols/Hdc/HdcSession.cs`：server shell 命令同样改为每次操作创建独立客户端，会话持有的客户端被释放后 shell/文件任务仍可继续。
- `UotanStudio.Comm/Protocols/Hdc/HdcServerRemoteCommand.cs`：参数顺序对齐上游客户端 `file send <local> <remote> remote -cwd "<cwd>"`，减少 server 端解析差异。
- `UotanStudio.Mirroring/Ohscrcpy/OhscrcpyServerManager.cs`：遇 `ObjectDisposedException` / `SocketException` 立即终止候选循环并抛出“HDC server 或会话已丢失”，不再对 5 个库做无意义重试。
- 新增 `HdcServerClient.IsDisposed` 诊断属性。

原则分层：候选库回退只用于“协议不兼容”，传输层死亡（disposed/socket 断开）必须立即 fail-fast；可被上层会话释放的实例不能被下层长期借用。

## 验证

- `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 错误（仅既有 SpacemiT 测试分析器警告）；whitespace / style `dotnet format --verify-no-changes` 通过。
- 定向：Devices 222 通过；Mirroring 110 通过 / 1 真机门控跳过；`HdcServerTransferTests` 10；`MirrorDocumentTests` 4；`ScreenMirrorCompositionRecoveryTests` 7。
- 全量：7229 总数 / 7222 通过 / 7 跳过 / 0 失败。
- NativeAOT：`dotnet publish UotanStudio.Desktop/UotanStudio.Desktop.csproj -c Release -r win-x64 --self-contained -p:PublishAot=true` 成功；仅既有第三方 trim/AOT 警告（Ursa、Dock.Avalonia、Lingua、DataGrid、YamlDotNet、Common.Logging 等）与 `Makaretu.Dns.LinuxHelper` 的 `Tmds.LibC` 加载提示（win-x64 不适用），未新增本次改动的裁剪警告。

## 未验证边界

- 真实 ADB 设备长时间投屏：NRE 是否不再断流未经实测；若仍失败，新日志会带完整堆栈可直接定位到具体解码器/帧路径。
- 停止 HDC server 后的 USB 直连：重枚举后的自动重试成功与否未在真机确认；仍报占用时需按新错误里的设备键判断是否真被其它驱动/进程持有。
- HDC Server 投屏：首次推送不再出现 disposed 级联未经真实 server 复核；server 进程自身退出时的 fail-fast 报错路径也只有回环 fake server 覆盖。

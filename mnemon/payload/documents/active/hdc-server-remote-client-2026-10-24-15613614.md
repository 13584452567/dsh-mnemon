---
id: "15613614-f9e3-4daa-b1c0-a2aa9cd9528d"
title: "HDC Server 路由会话化：remote-client 双工任务帧、统一文件/安装/卸载与投屏租约（2026-10-24）"
description: "HDC server 路由补齐文件/安装卸载/投屏会话的设计与语义（含 RecvDirectory 根目录名、fport 回退、清理顺序）、压缩架构门禁修法、验证数字与真机未验证边界。"
status: "active"
created_at: "2026-10-23T19:41:48.094Z"
updated_at: "2026-10-23T19:41:48.094Z"
content_hash: "4d6ef7df6a58074f57407f77b2117d6e8ec3908f456b383a93268e3bb7554b7b"
source_paths:
  - "UotanStudio.Comm/Protocols/Hdc/HdcServerTaskChannel.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcTaskChannel.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcServerSyncClient.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcServerAppClient.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcTransferEngine.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcServerClient.cs"
  - "UotanStudio.Comm/Protocols/Hdc/HdcDeviceServices.cs"
  - "UotanStudio/Services/Devices/HdcSessionRouting.cs"
  - "UotanStudio/Services/Devices/HdcDeviceService.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Ohscrcpy.cs"
  - "UotanStudio/ViewModels/Workspace/WorkspaceViewModel.cs"
  - "UotanStudio.Comm/Protocols/SpacemiT/SpacemiTFlasher.cs"
  - "UotanStudio.Tests/Devices/HdcServerTransferTests.cs"
  - "UotanStudio.Tests/Devices/HdcSessionRoutingTests.cs"
session_ids:
  - "724e5979-fa5d-4965-9fd5-7a6a35b7afbe"
memory_body_ids:
  []
---

# HDC Server 路由会话化：文件、安装卸载、投屏统一走 server-backed 会话

## 背景与取代范围

此前 `UotanStudio.Mirroring` 文档 `fe4bd5e7-924b-45ec-aa91-975d63f3b737` 记录的前提是“HDC server 路由没有文件传输通道”，因此 `DeviceMonitorService.Ohscrcpy` 只打开直连租约、`ServerOnly` 模式拒绝直连。本次把 HDC server 路由补齐为完整会话能力，**该前提在本仓库已失效**：server 拥有目标设备时同样可传文件、装/卸应用与投屏。

同一批改动还包含投屏侧的两项修复，其设计仍以 `8f130951-1500-4159-8e8f-8e106efe5232`（合成表面恢复状态机、APP_SWITCH 和弦）为准，本文不重复，只记录与 server 路由的接缝。

## server 端通道设计

- 新增 `HdcServerTaskChannel` / `HdcTaskChannel`：在 remote-client 连接上收发双工任务帧，作为 server 路由下 sync、app 与外设命令的统一承载层。
- `HdcServerSyncClient` 对应直连 `HdcSyncClient` 的文件通道语义；`HdcServerAppClient` 对应直连 `HdcAppClient` 的安装/卸载命令；`HdcServerClient` 负责 server 路由的会话建立与命令派发。
- `HdcTransferEngine` 抽出直连与 server 两条链路共用的传输编排（暂存、进度、校验、原子收尾），避免两套行为漂移。
- `HdcSession` 在 server 路由下返回可用的 sync/app 客户端，使上层无需分支判断“这条路由能不能传文件”。
- `HdcDeviceService` 统一直连与 server 的发送、拉取、安装、卸载入口。

## 目录接收语义（易错点）

目录条目的 `optionalName` 携带远端根目录名，例如 `tree/one.txt`，与直连 `HdcSyncClient.RecvDirectory` 的行为一致；实现把整棵树落到 `destination/tree/...`。新增测试最初期望扁平落盘 `destination/one.txt` 而失败，结论是**测试期望错、实现与既有直连行为一致**，已修正测试断言而不是改接收逻辑。同时保留“整棵暂存树完成后才发布”的语义，失败时不对外暴露半成品目录。

## 投屏与租约接缝

- `DeviceMonitorService.Ohscrcpy` 改走 `OpenHdcLease`：server 拥有目标时使用 server-backed `HdcSession`，`HdcForwardTunnel` 在 server 路由回退 `fport` 建立转发。
- 清理顺序固定为 **转发 → 会话 → 租约**，与直连路径一致，避免监听器或租约泄漏。
- `HdcSessionRouting`（宿主侧）封装路由判定；`WorkspaceViewModel` 移除了设备选中变化触发的自动重定向/聚焦，保证多设备各自独立文档。

## 架构门禁：压缩边界

全量测试唯一失败是 `CompressionImplementationBoundaryTests` 命中 `SpacemiTFlasher` 直接 `new GZipStream(...)`。处理方式是把超大数据镜像的分块压缩改走 `UotanStudio.Compress` 的公共入口（`CompressionService` + `CompressionOptions.Level`），**不扩大架构测试的例外白名单**。这是门禁的既定意图：生产代码的压缩/解压只能经 `UotanStudio.Compress`，`System.IO.Compression` 仅允许出现在该库内部。

## 验证

- 构建：`dotnet build UotanStudio.slnx --no-restore --nologo -c Debug` → 0 警告 / 0 错误；whitespace 与 style `dotnet format` 门禁通过。
- 定向：Devices 222/222；Mirroring 110 通过 + 1 真机门控跳过；`MirrorDocumentTests` 4；`ScreenMirrorCompositionRecoveryTests` 7；架构门禁 16。
- 全量：7229 总数 / 7222 通过 / 7 跳过 / **0 失败**。
- NativeAOT：`dotnet publish UotanStudio.Desktop/UotanStudio.Desktop.csproj -c Release -r win-x64 --self-contained -p:PublishAot=true` 成功，仅剩既有第三方 trim/AOT 警告。
- 新增测试的 xUnit 生命周期警告（`xUnit1051`）已消除；6 个文件补齐 `insert_final_newline=false` 所需的末尾换行。当时工作区含并行 SpacemiT 改动，仅编辑本任务相关文件。

## 未验证边界

- 真实 HDC server（DevEco）下的设备探测、sync 推送、`fport` 转发、gRPC 控制与视频解码：目前只有回环 fake server 与协议单测，**未在真实 server 上跑通**。
- 真实 HarmonyOS 设备上的多任务组合键（`META_LEFT`+`TAB`）是否打开最近任务未经观察，键码依据来自 DevEco 插件。
- 浮窗切换/拖拽的真实界面表现与 250ms 去抖时序仍需人工复核，自动测试只覆盖状态机。
- HarmonyOS 7.0.0.109 采集库 `dlopen` 的 protobuf/gRPC ABI 兼容问题属既有环境边界（见 `2c8758fd-cdbb-4569-8533-645110211a65`）。
- 改动未提交 Git。

## 源码入口

- `UotanStudio.Comm/Protocols/Hdc/HdcServerTaskChannel.cs`、`HdcTaskChannel.cs`、`HdcServerSyncClient.cs`、`HdcServerAppClient.cs`、`HdcTransferEngine.cs`、`HdcServerClient.cs`、`HdcDeviceServices.cs`
- `UotanStudio/Services/Devices/HdcSessionRouting.cs`、`HdcDeviceService.cs`、`DeviceMonitorService.Ohscrcpy.cs`
- `UotanStudio/ViewModels/Workspace/WorkspaceViewModel.cs`
- `UotanStudio.Comm/Protocols/SpacemiT/SpacemiTFlasher.cs`
- `UotanStudio.Tests/Devices/HdcServerTransferTests.cs`、`HdcSessionRoutingTests.cs`

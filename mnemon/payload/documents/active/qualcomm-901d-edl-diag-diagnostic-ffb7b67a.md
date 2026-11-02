---
id: "ffb7b67a-4d8f-4215-bc91-340492cba6b2"
title: "Qualcomm 901D 诊断端口：从 EDL 误判到 diag 协议名与 Diagnostic 会话工厂"
description: "0x901D 被误判为 EDL 的证据与修正：IsDiagPort 判定、diag 协议名、DiagnosticDeviceSessionFactory、宿主侧两处穷举映射补齐，含 273/273 协议测试、Devices 唯一无关失败与真机未验证边界。"
status: "active"
created_at: "2026-11-02T01:12:55.783Z"
updated_at: "2026-11-02T01:12:55.783Z"
content_hash: "de6900d4a35c56fa01832a09986b2053002223d0041c1ba4e778d3949705211d"
source_paths:
  - "UotanStudio.Comm/Devices/Classifiers/QualcommDiagPortClassifier.cs"
  - "UotanStudio.Comm/Devices/Services/Sessions/DefaultDeviceSessionFactories.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.ControlledSessions.cs"
  - "UotanStudio.Tests/Devices/QualcommDiagPortClassifierTests.cs"
  - "UotanStudio.Tests/Devices/DeviceProtocolMappingTests.cs"
  - "reference/edl-bkerler/edl-master/edlclient/Config/usb_ids.py"
session_ids:
  - "ab9cd58c-0b41-4cd9-8861-ed663e11ca29"
memory_body_ids:
  []
---

# Qualcomm 901D 诊断端口：从 EDL 误判到 `diag` 协议名与 Diagnostic 会话工厂

## 背景与结论

原实现把 Qualcomm `0x901D`（`setprop sys.usb.config diag,adb` 枚举出的基带调试端口）归入 EDL 协议，属于**协议误判**。本次将其从 EDL 分类表中移除，新增独立的诊断端口判定与 `diag` 协议名，并把后端直通到既有的 `DiagSession`（NV 读写、QCN 备份/还原、SPC 解锁、EFS2 均已具备）。界面未接线：`Diagnostic` 目前只到后端，界面上不可见。

## 决定性证据

`reference/edl-bkerler/edl-master/edlclient/Config/usb_ids.py`

- EDL 表 `default_ids` **不含** `0x901D`；
- DIAG 表 `default_diag_vid_pid` 第 35 行明确列出：

```
[0x05c6, 0x901d, 0],   # QC Android "setprop sys.usb.config diag,adb"
```

## 改动要点

- **分类器**：从 EDL 列表移除 `0x901D`；新增 `IsDiagPort` 判定，覆盖 Sierra `0x9071`/`0x9091`、Quectel `0x0125`、Netgear `0x68E2`、ZTE `0x0016`。
- **协议与会话类型**：新增 `diag` 协议名 → `DeviceSessionKind.Diagnostic` → `DiagnosticDeviceSessionFactory`（USB 诊断接口或串口均可；先 `LeasedTransport.RequireProtocol(endpoint, "diag")` 校验端点观测到的协议，再按 `SerialTransport` / `IUsbDeviceSession` 分流）。仅端点观测到的协议可授权会话，绝不按序列号重新推断。
- **宿主侧**：补 `DeviceProtocolKind.Diagnostic` 及两处穷举映射与状态标签——`DeviceMonitorService.ExpectedEndpointProtocol`（映射为 `"diag"`）与 `DeviceMonitorService` 的 `ProtocolLabel`。漏补会让设备被映射成 `null` 从而从列表中消失。
- **刻意不改**：`WaitForUsbReconnectAsync` 的重连映射是严格白名单（mtk/edl/fastboot/rkdl），DIAG 不纳入重连，保持原样。

## 回归守卫的实证

新增 `Diagnostic` 枚举值当场触发两处既有测试失败：

- `DeviceProtocolMappingTests.ExpectedEndpointProtocol_EveryDefinedProtocol_MapsToADiscoveryProtocolText`
- `DeviceProtocolMappingTests.ProtocolLabel_EveryDefinedProtocol_ProducesANonEmptyLabel`

均以 `ArgumentOutOfRangeException: Unexpected protocol kind. (Actual value was Diagnostic)` 失败，补上分支后转为通过。结论：这两个穷举测试确实能发现新增枚举值造成的宿主映射缺口，新增协议/会话类型时必须同步补齐。

## 验证（Windows x64，.NET 10，Debug）

- `dotnet build UotanStudio.slnx -c Debug`：0 错误 / 0 警告。
- `dotnet format ... whitespace --verify-no-changes` 与 `style --verify-no-changes`：均零报告。
- `UotanStudio.Tests.Protocols`：273 / 273 通过。
- `UotanStudio.Tests.Devices`：423 总数，419 通过、3 跳过、1 失败。
- 唯一失败 `AdbTlsTransportTests.UpgradeToTls_DeviceRebuildsItsCertificateEachHandshake`（`IOException: unexpected EOF`）与本次改动无关：ADB 源码与本会话均无相关修改（`git status` 无相关条目）。

## 环境坑（并行会话）

同一工作区存在并行改动时，`dotnet build` 可能报出与本任务无关的临时错误，重试即消失：

- `Views/**` 的 `InitializeComponent`/`DockControl` 缺失（XAML 生成中间产物冲突）；
- MSBuild 复制 `UotanStudio.dll` 失败（`MSB3021`/`MSB3027`，被正在运行的 `UotanStudio.Tests` 进程锁定）——等测试进程退出后重建即恢复；
- 增量构建出现的瞬时警告在下一轮构建中归零。

格式校验只处理本任务涉及的文件；`FlashViewModel.RockchipMaintenance.cs` 的 `FINALNEWLINE` 属并行改动，未触碰。

## 未验证边界

- **901D 真机未验证**：分类与工厂接线由直测覆盖，`0x901D` 端口上的真实 DIAG 交互（NV 读取、QCN 回写）未在硬件执行。
- 无 UI 接线，界面上看不到 `Diagnostic` 设备与相关入口。

## 关联文件

- 分类器：[QualcommDiagPortClassifier.cs](UotanStudio.Comm/Devices/Classifiers/QualcommDiagPortClassifier.cs)
- 会话工厂：[DefaultDeviceSessionFactories.cs](UotanStudio.Comm/Devices/Services/Sessions/DefaultDeviceSessionFactories.cs)
- 宿主映射：[DeviceMonitorService.ControlledSessions.cs](UotanStudio/Services/Devices/DeviceMonitorService.ControlledSessions.cs)、[DeviceMonitorService.cs](UotanStudio/Services/Devices/DeviceMonitorService.cs)
- 测试：[QualcommDiagPortClassifierTests.cs](UotanStudio.Tests/Devices/QualcommDiagPortClassifierTests.cs)、[DeviceProtocolMappingTests.cs](UotanStudio.Tests/Devices/DeviceProtocolMappingTests.cs)
- 证据源：`reference/edl-bkerler/edl-master/edlclient/Config/usb_ids.py`

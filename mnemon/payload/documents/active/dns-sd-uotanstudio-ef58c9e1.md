---
id: "ef58c9e1-8522-4442-a366-c48602055d04"
title: "顶部设备选择框变更通知链路与远程 DNS-SD 显示名裁短（UotanStudio）"
description: "顶部选择框不刷新槽位的两处根因（SelectedDeviceName 未转发、下拉项 Header 主题绑定）、DeviceViewModel/MainWindow 的订阅与退订结构，以及 DeviceInfo.DisplayIdentity 对 adb DNS-SD 实例串号的裁短优先级规则与验证结果。"
status: "active"
created_at: "2026-10-27T13:43:01.813Z"
updated_at: "2026-10-27T13:43:01.813Z"
content_hash: "a19e39d8e19abcf29c890c9ff95ce39ee20adbb6495092fd96c4604490e93b42"
source_paths:
  - "UotanStudio/ViewModels/Devices/DeviceViewModel.cs"
  - "UotanStudio/Views/Shell/MainWindow.axaml"
  - "UotanStudio/Views/Shell/MainWindow.axaml.cs"
  - "UotanStudio/ViewModels/Devices/DeviceInfo.cs"
  - "UotanStudio.Comm/Protocols/Adb/AdbTargetInfo.cs"
  - "UotanStudio.Tests/Devices/DeviceLinkDisplayTests.cs"
  - "UotanStudio.Tests/Protocols/AdbTargetInfoTests.cs"
session_ids:
  - "969a9a0a-3d11-4aea-a515-87ce66a51494"
memory_body_ids:
  []
---

# 顶部设备选择框的变更通知链路与远程链接显示名裁短（UotanStudio）

本轮修复两件事：顶部选择框（当前设备文本 + 下拉项）不跟随 `DeviceInfo` 的补充信息刷新；adb server 中转的 DNS-SD 实例名使显示名过长。显示名组装、槽位拼接与链路推断的基础规则见既有文档 `896d5b33-3b4c-4126-bb54-75943ef8bc61`，本文只记录本轮新增的结论。

## 1. 症状与根因（槽位不刷新）

症状：设备管理器显示 `(Android Slot A)`，顶部选择框的当前文本与下拉项仍只显示 `(Android)`。

根因是两个独立的断点，而不是 `DisplayName` 本身缺槽位：

- 当前设备文本：`MainWindow` 绑定 `DeviceViewModel.SelectedDeviceName`（即 `SelectedDevice.DisplayName`）。该属性只在选中设备/切换端点时由 `NotifySelectedDeviceProperties` 重发；之后补上的 `Slot` 只触发 `DeviceInfo.DisplayName` 的通知，ViewModel 没有转发，绑定值停在选中瞬间。
- 下拉项：条目 `Header` 原先来自 `ControlTheme` 里的 Setter 绑定，条目在扫描时创建，槽位是之后补充的，标题不再更新。

## 2. 修复结构（三处协同，不要只改其中一处）

| 位置 | 做法 |
| --- | --- |
| `UotanStudio/ViewModels/Devices/DeviceViewModel.cs` | 选中设备时订阅其 `PropertyChanged`，任何补充字段变化都重发 `SelectedDeviceName` 与 `DeviceDetail`；切换选中/释放时退订 |
| `UotanStudio/Views/Shell/MainWindow.axaml.cs` | 选择器条目创建时写入 `Header = device.DisplayName` 并订阅 `DisplayName` 变更实时刷新；条目移除、窗口关闭时退订清理 |
| `UotanStudio/Views/Shell/MainWindow.axaml` | 删除主题中 `Header` 的 Setter 绑定，标题只保留代码维护这一个来源 |

要点：`ObservableCollection` 的条目通知不会穿透到集合本身，因此「集合变化」与「条目属性变化」必须分别订阅；两套标题来源（主题 Setter + 代码赋值）会互相覆盖，必须只留一套。

## 3. 远程链接显示名裁短

症状：`adb-6a082bda-IZgxlP.adb-tls-connect._tcp-24129PN74C (Android Slot B)` 这类 DNS-SD 实例串号把设备名推到看不见的位置。期望 `24129PN74C (Android Slot B)`。

`DeviceInfo.DisplayName` 改为 `DisplayIdentity + " (" + StateLabel + ")"`，`DisplayIdentity` 优先级：

| 场景 | 显示身份 |
| --- | --- |
| 普通硬件序列号（USB 等） | 保持原样，如 `R58M12345 - SM_G9730` |
| DNS-SD 实例串号 + 已读到型号 | 型号，如 `24129PN74C` |
| 实例串号、型号未知、server 已解析出地址 | 地址，如 `192.168.1.144:40443` |
| 实例串号、型号与地址都缺失 | 去掉 `._adb-tls-connect._tcp` 服务后缀后的实例名 |

约束：只对 DNS-SD 实例串号生效；`host:port` 与硬件序列号的显示规则不变。

## 4. 边界归属

- 服务后缀判定公开为 `AdbTargetInfo.IsMdnsInstanceName`（`UotanStudio.Comm/Protocols/Adb/AdbTargetInfo.cs`），adb 发现的链路分类与宿主显示共用同一处事实；不要在界面层另写后缀判断，否则两处会漂移。
- 显示名仍是顶部选择框（`SelectedDeviceName`、下拉项）与设备管理器共用的单一字符串，改一处两处同时生效。

## 5. 验证与未验证边界

| 项目 | 结果 |
| --- | --- |
| Debug 全量构建 | 0 警告 0 错误 |
| `DeviceLinkDisplayTests` | 23/23（新增 4 项：带型号、地址回退、后缀裁剪、硬件串号不变；另含 `SelectedDeviceName_SlotEnrichmentAfterSelection_UpdatesTopSelectorText`） |
| `AdbTargetInfoTests` | 6/6 |
| `DeviceProtocolMappingTests` | 29/29 |
| `dotnet format` whitespace / style | 通过 |
| Release `win-x64` NativeAOT publish | 成功；仅既有第三方裁剪/AOT 警告（Avalonia/Dock/YamlDotNet/Common.Logging/LTRData） |

未验证：本机无真实 adb/hdc 设备，`24129PN74C (Android Slot B)` 一类字符串与下拉项实时刷新仍以真机界面确认。排查期间工作树另有并行会话的未提交改动（Firmware `ContainerEntryExtractor` 曾引用不存在的 `ExtractUpdateBinAsync` 使全量构建失败），与本任务无关，未触碰。

---
id: "98d253d8-8f3c-4dfc-b71d-a7f2902eeaee"
title: "子窗体打开集合与几何持久化缺口分析，及烧录侧边菜单展开状态持久化（UotanStudio）"
description: "烧录界面侧边二级菜单（高通/联发科/全志/镜像操作）分组展开状态经 SettingsPersistenceService 持久化的实现与验证；以及子窗体（工具窗口/浮动 Dock 窗口）打开集合与大小位置未持久化的缺口分析、WindowPlacementValidator 复用与 Dock 布局序列化的实施拆解。"
status: "active"
created_at: "2026-10-28T12:51:38.478Z"
updated_at: "2026-10-28T12:51:38.478Z"
content_hash: "997ab1031e0cae1553ede1c8e4ed3429c80ea9feb24575d5dc0bfb5c9285764a"
source_paths:
  - "UotanStudio/Services/SettingsPersistenceService.cs"
  - "UotanStudio/ViewModels/Flash/FlashSectionState.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Sections.cs"
  - "UotanStudio/Views/Shell/FlashView.axaml"
  - "UotanStudio/Views/UiWindowService.cs"
  - "UotanStudio/Services/Ui/Workspace/WorkspaceSessionSnapshot.cs"
  - "UotanStudio/Views/Workspace/WorkspaceDockFactory.cs"
  - "UotanStudio/Views/Shell/MainWindow.axaml.cs"
  - "UotanStudio/Services/Ui/Workspace/WindowPlacementValidator.cs"
  - "UotanStudio.Tests/Services/FlashSectionStateTests.cs"
session_ids:
  - "a88a4ea6-3350-4577-a4ff-52a4f32f52c7"
memory_body_ids:
  []
---

# 子窗体打开集合/几何持久化缺口分析，与烧录侧边菜单展开状态持久化（UotanStudio）

本轮延续“程序配置持久化范围”扩展：先把烧录界面（高通、联发科、全志、镜像操作等）侧边二级菜单的展开状态纳入设置持久化；随后用户追问“现在的窗口大小记忆能否记忆打开了哪些子窗体以及它们的大小和位置”，得到的是**现有实现的缺口分析 + 落地拆解**。主窗口几何与资源管理器展开状态的既有实现见文档 `dcf19c6b-5892-46ea-9bf2-e2111d5bfa41`，本文不重复。

## 1. 烧录界面侧边二级菜单展开状态（已实现）

- 新增 `UotanStudio/ViewModels/Flash/FlashSectionState.cs`：侧边二级菜单（高通 / 联发科 / 全志 / 镜像操作等）的分组展开状态模型，随程序设置持久化，不进入解决方案文件。
- 新增 `UotanStudio/ViewModels/Flash/FlashViewModel.Sections.cs`：分组状态的捕获与恢复；`UotanStudio/ViewModels/Flash/FlashViewModel.cs` 只加一行入口。
- `UotanStudio/Services/SettingsPersistenceService.cs`（+42 行）：读写该状态，沿用与 `mainWindowPlacement` 一致的“以磁盘最新设置合并写入”方式，不整体覆盖设置文件。
- `UotanStudio/ViewModels/Shell/SettingsViewModel.cs`（+1 行）：设置页保存前同样从磁盘合并，避免用陈旧快照覆盖刚写入的分组状态。
- `UotanStudio/Views/Shell/FlashView.axaml`：绑定改为分组状态，展开/收起即时反映到设置。
- 测试：新增 `UotanStudio.Tests/Services/FlashSectionStateTests.cs`。定向测试分批跑完 5 / 15 / 6 / 16 项，全部成功、0 失败；后续校验命令退出码 0。

结论性模式：设置类 UI 状态统一走 `SettingsPersistenceService` + 设置页合并写回，不要写进 `.uotan`，也不要在 ViewModel 里直接落盘。

## 2. 现状：三类窗口数据的持久化口径

| 数据 | 存储位置 | 实际内容 |
| --- | --- | --- |
| 主窗口 | `settings.json` → `mainWindowPlacement` | 宽高、坐标、窗口状态；含 ≥48px 可视校验与默认几何回退 |
| 工作区会话 | `workspace-session.json`（`WorkspaceSessionSnapshot`） | 打开的**文档**（类型 / 相对路径 / 打开方式 / 活动项）、`Docks`、面板可见性、草稿工作流、节点缩放与 minimap |
| 解决方案 | `.uotan` → `State` | 交付计划、面板可见性、资源管理器展开状态（`explorer`） |

`WorkspaceSessionSnapshot` 字段为 `Version` / `ProjectPath` / `Panels` / `Documents` / `Docks` / `DraftWorkflow` / `DraftPackingSettings` / `NodeZoom` / `NodeMinimap`；`WorkspaceDockState` 只有 `Id` / `Proportion` / `ActiveId`。

## 3. 缺口：子窗体既无“打开集合”也无几何

- 工具窗口（设备、日志、USB 树、密钥库、可视插件管理、APK 签名、BootPatch、AVB 工具）由 `UotanStudio/Views/UiWindowService.cs` 的 `ShowToolWindow(ref holder, factory)` 懒创建并缓存在私有字段中；`Show()` 后每次都用 XAML 默认尺寸与 `WindowStartupLocation`（多为 `CenterOwner` / `CenterScreen`）。
- 退出时 `UotanStudio/Views/Shell/MainWindow.axaml.cs` 的 `CloseChildWindows()` 全部关闭，重启后不会重开任何工具窗口。
- 方案编辑器 / 方案运算符 / 校验和窗口同样只做实例复用（`_schemeEditorWindow`、`_schemeOperatorWindow`、`_checksumWindow`），不记录几何；二维码配对、关于、新建工程、危险确认等是瞬时 `ShowDialog`。
- 浮动停靠窗口由 `UotanStudio/Views/Workspace/WorkspaceDockFactory.cs` 的 `CreateHostWindow` / `CreateManagedHostWindow` 创建；`WorkspaceSessionCapture.Enumerate` 会遍历 `IRootDock.Windows` 以便收集其中的文档，但 `Docks` 只保存 `Id`/`Proportion`/`ActiveId`——**浮动窗口是否打开、位置与大小都不保存，Dock 布局本身未序列化**。

因此“窗口大小记忆”目前只覆盖主窗口。

## 4. 落地拆解（三部分，可分批）

1. **窗口位置表**：把 `MainWindowPlacement` 泛化为按稳定窗口标识（`HostWindowKind`、插件编辑器用插件 id）索引的位置表，复用 `WindowPlacementValidator` 的屏内可视校验与默认回退，避免恢复后窗口落到看不见的地方。
2. **打开集合**：在会话文件或设置文件中记录已打开的子窗体集合，由 `UiWindowService` 在打开 / 关闭 / 移动时更新，主窗口恢复后再按记录重开，每个窗口注册自己的几何回调。
3. **浮动 Dock 窗口**：二选一——序列化 Dock 布局（`Dock.Model` 的布局持久化能力），或把浮动窗口并入同一张窗口位置表并在恢复时重建对应 dockable。

范围取舍：只做“工具窗口开关 + 各自大小位置”可以不碰 Dock 布局序列化，工作量明显更小；要求浮动面板一起恢复则必须同时做 Dock 布局持久化。

## 5. 未验证边界

- 子窗体持久化尚未实现，无构建 / 测试证据。
- 本轮只做了代码勘察（事件与文件路径来自仓库读取），未在 GUI 中实测重启后的窗口恢复行为。

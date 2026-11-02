---
id: "dcf19c6b-5892-46ea-9bf2-e2111d5bfa41"
title: "窗口几何持久化与越界保护、资源管理器展开状态持久化（UotanStudio）"
description: "主窗口大小/位置改存 settings.json 的 mainWindowPlacement、越界可视部分校验（≥48px）与提示后回落默认几何、解决方案 State 的 explorer 展开状态捕获/恢复结构，以及构建/测试/NativeAOT 发布实测结果与未验证的 GUI 边界。"
status: "active"
created_at: "2026-10-28T05:22:53.019Z"
updated_at: "2026-10-28T05:22:53.019Z"
content_hash: "b8f3703f8b4de768ac16a6a27b5de28a60b03c0ea469ae33c7264e6f7506567d"
source_paths:
  - "UotanStudio/Services/SettingsPersistenceService.cs"
  - "UotanStudio/Services/Ui/Workspace/WorkspaceAutoSaveService.cs"
  - "UotanStudio/Services/Ui/Workspace/WindowPlacementValidator.cs"
  - "UotanStudio/Services/Ui/Workspace/WorkspaceSessionSnapshot.cs"
  - "UotanStudio/Services/Ui/Projects/ProjectOperationsService.cs"
  - "UotanStudio/ViewModels/Workspace/ExplorerExpansionState.cs"
  - "UotanStudio/ViewModels/Workspace/ProjectExplorerViewModel.cs"
  - "UotanStudio/ViewModels/Shell/SettingsViewModel.cs"
session_ids:
  - "a91e204e-b85d-4d08-8802-333b24166c9f"
memory_body_ids:
  []
---

# 窗口几何持久化与越界保护、资源管理器展开状态持久化（UotanStudio）

本轮把主窗口大小/位置从“随解决方案保存”改为“随程序设置保存”，并新增越界可视部分校验与提示；同时把解决方案资源管理器的展开目录集合保存进 `.uotan` 解决方案文件。以下为用户明确要求、实现结构、关键取舍与验证结果。

## 0. 用户要求（原始约束）

窗口打开大小与位置还原持久化数据时，需要确保窗口内存在可视部分；超出屏幕边界则弹窗提示，并使用默认窗口大小与位置。恢复不得把窗口放到用户看不见的地方。

## 1. 主窗口几何 → 程序设置文件

存储位置改为 `%AppData%/UotanStudio/settings.json`，键 `mainWindowPlacement`（`AppSettings.MainWindowPlacement`，公共类型 `MainWindowPlacement`：`width`/`height`/`x`/`y`/`state`，`WindowState` 以名称序列化；`AppSettingsJsonContext` 增加 `List<string>` 与 `WindowState` 转换器，保持 NativeAOT 的源生成 JSON 路径）。

- `UotanStudio/Services/Ui/Workspace/WorkspaceAutoSaveService.cs`：启动时从设置文件恢复窗口，定时保存、显式保存与关闭时写回。**关闭自动保存时仍每 15 s 与退出前落盘**，避免用户关掉自动保存后几何永不记录。
- `UotanStudio/Services/SettingsPersistenceService.cs`：`SaveMainWindowPlacement` 以磁盘最新设置做**合并写入**（不整体覆盖），值未变化时跳过写盘。
- `UotanStudio/ViewModels/Shell/SettingsViewModel.cs`：保存设置页前先从磁盘合并 `MainWindowPlacement`，否则设置页写回会用陈旧快照覆盖刚记录的窗口位置。
- `WorkspaceSessionSnapshot` 的 5 个窗口字段已删除，`workspace-session.json` 不再记录窗口几何，因此窗口位置**对所有解决方案统一**，不再随工程切换跳动。

## 2. 越界保护（校验 → 不应用 → 提示 → 自愈）

`UotanStudio/Services/Ui/Workspace/WindowPlacementValidator.cs`：按像素工作区与主屏缩放换算窗口矩形，要求窗口与**某块**屏幕工作区的交集在两个方向都 ≥ 48 px 才算可见；屏幕列表为空或尺寸尚未测量时**不做判定**（不误报、不阻塞启动）。

RestoreWindowPlacement 判定失败时：不应用持久化几何，保持 XAML 默认大小与系统默认位置；恢复流程结束后再用 `DialogService` 提示。文案键 `Window_RestoreOutOfScreenTitle` / `Window_RestoreOutOfScreenMessage` 同时加入 `UotanStudio/Locale/Common.resx`、`Common.zh-CN.resx` 与 `LocaleKeySets.cs`（语言键集合校验测试会拦住漏项）。**下一次正常退出会把实际默认几何写回设置**，记录自愈，无需用户手动清理配置文件。

## 3. 资源管理器展开状态 → 解决方案文件

- `UotanStudio/ViewModels/Workspace/ExplorerExpansionState.cs`：捕获已展开目录为“工程根相对、`/` 分隔、序数排序”的路径集合；恢复时逐条路径展开，并沿途 `await` 惰性加载子节点；拒绝逃出根目录或不存在的路径（配置按不可信输入处理）。
- `UotanStudio/ViewModels/Workspace/ProjectExplorerViewModel.cs`：`CaptureExpandedPaths` / `RestoreExpandedPathsAsync`，并为 `ExplorerNode` 增加可等待的 `EnsureChildrenLoadedAsync`；捕获时无解决方案根则返回 `null`，恢复异常只记 Warn 日志，不影响打开工程。
- `UotanStudio/Services/Ui/Projects/ProjectOperationsService.cs`：解决方案 `State` 新增 `explorer` 键（与 `delivery`/`panels` 同层）；**捕获失败时保留磁盘上的旧值**，不要写成空集合；恢复在 `ProjectContextChangedMessage` 之后执行，保证树已重建。
- 文档同步：`SolutionSnapshot.State` XML 注释与 `UotanStudio/AGENTS.md` 已更新；公共/内部 API 注释保持中英双语 `<summary>`+`<para>` 格式。

## 4. 验证结果

| 检查 | 结果 |
| --- | --- |
| `dotnet build UotanStudio.slnx --no-restore -c Debug` | 成功，0 警告 0 错误 |
| `dotnet format --verify-no-changes` whitespace / style | 通过（首轮修了 4 处尾部换行、1 处 using 顺序） |
| 定向测试 | `WindowPlacementSettingsTests` 4、`WindowPlacementValidatorTests` 7、`ExplorerExpansionStateTests` 4、`WorkspaceAutoSaveTests` 15、`LocaleDictionaryTests` 6、`HostLayeringGuardTests` 4 全通过 |
| `--filter-namespace UotanStudio.Tests.Projects` | 123 通过，1 跳过（环境门控） |
| Release win-x64 NativeAOT publish | exit 0；39 条 IL 警告全部来自既有第三方（Avalonia XAML/Dock trampoline、YamlDotNet、LTRData、Common.Logging），日志中 0 次出现新类型名 |

易踩点：仓库 `.editorconfig` 为 `insert_final_newline = false`，新增文件带尾部换行会被 `dotnet format` 判为 FINALNEWLINE 错误（构建不报，只有 format 验证报）。

未验证边界：真实桌面窗口恢复、屏外几何弹窗、展开动画属 GUI 行为，需人工确认——先关后启应回到上次位置；手改 `mainWindowPlacement` 为屏幕外坐标后重启应出现提示并使用默认几何；展开若干目录后保存解决方案并重开应恢复展开。

## 5. 相关文件

- `UotanStudio/Services/SettingsPersistenceService.cs`
- `UotanStudio/Services/Ui/Workspace/WorkspaceAutoSaveService.cs`
- `UotanStudio/Services/Ui/Workspace/WindowPlacementValidator.cs`
- `UotanStudio/Services/Ui/Workspace/WorkspaceSessionSnapshot.cs`
- `UotanStudio/Services/Ui/Projects/ProjectOperationsService.cs`
- `UotanStudio/ViewModels/Workspace/ExplorerExpansionState.cs`
- `UotanStudio/ViewModels/Workspace/ProjectExplorerViewModel.cs`
- `UotanStudio/ViewModels/Shell/SettingsViewModel.cs`

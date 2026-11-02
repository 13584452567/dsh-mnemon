---
id: "57625aaf-c24d-44d0-813b-1a5348c53631"
title: "UotanStudio 本地化资源键登记约定与 Node_Family_* 动态拼键缺口"
description: "本地化键三处同步（resx EN/ZH + LocaleKeySets）约定、字面量扫描对前缀拼键的盲区、15 个 Node_Family_* 族键补齐与回归测试落点、实测验证数字、未登记的 Flash_* 键缺口。"
status: "active"
created_at: "2026-10-31T18:49:38.941Z"
updated_at: "2026-10-31T18:49:38.941Z"
content_hash: "66b5ab7f3f0382e1f5f9ea8347ea58f8f8163ce6139678cab2bbad71821ead5a"
source_paths:
  - "UotanStudio/Locale/Node.resx"
  - "UotanStudio/Locale/Node.zh-CN.resx"
  - "UotanStudio/Locale/LocaleKeySets.cs"
  - "UotanStudio/ViewModels/Nodes/NodeEditorViewModel.Toolbox.cs"
  - "UotanStudio/Workflow/Registry/NodeFactoryRegistrar.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs"
  - "UotanStudio.Tests/Services/NodeFactoryRegistrarTests.cs"
session_ids:
  - "a5c672b5-4643-4855-a1f3-0750907a65ee"
memory_body_ids:
  []
---

# UotanStudio 本地化资源键登记约定与 Node_Family_* 动态拼键缺口

UotanStudio 的本地化键必须同时存在于三处（英文 resx、zh-CN resx、`LocaleKeySets.cs` 索引数组），但**运行时动态拼接的键**（如节点工具箱的族名）不在任何静态键扫描的可见范围内。本文记录该约定、由此产生的漏登记缺口、修复落点与回归防护。

## 1. 根因：族名键是拼出来的，不是写出来的

- 节点工具箱按 `"Node_Family_" + Family` 取族显示名：`UotanStudio/ViewModels/Nodes/NodeEditorViewModel.Toolbox.cs`（约 67 行与 216 行）。
- 族名段来自节点类型键的第二段：`UotanStudio/Workflow/Registry/NodeFactoryRegistrar.cs`（约 85 行）。
- 节点目录扩张后新增 15 个族，`UotanStudio/Locale/` 里没有对应资源键，于是**每次构建工具箱都会为每个未知族打一条调试日志**；键缺失时 `GetString` 返回键名本身，UI 上会直接显示 `Node_Family_xxx`。

只有 8 个族出现在启动日志里（`CommandLine`、`Config`、`List`、`Search`、`Shell`、`Slot`、`Table`、`Vbmeta`）；另外 7 个是同根因的传输族（`Adb`、`Edl`、`Fastboot`、`Hdc`、`Mtk`、`Rkdl`、`Wait`），由 `DeviceProtocolNodeSpecs`/`MtkHdcNodeSpecs` 注册，日志未覆盖但键同样缺失。

## 2. 键扫描的盲区（重要）

仓库里用于找漏登记键的扫描是**字面量匹配**：一次全量运行报告 declared keys 3423、missing literal keys 19。拼接写法的键只能以裸前缀 `Node_Family_` 的形式出现在结果里，扫描无法枚举具体族名——即使把 `Node_Family_` 删掉，仍无法回答「哪些族缺键」。因此前缀拼键必须靠**针对性断言**兜底，不能靠扫描。

## 3. 修复落点

- `UotanStudio/Locale/Node.resx` 与 `Node.zh-CN.resx`：在 `Node_Family_Result` 之后补齐 15 个族键（英文/中文 key 集合保持一致）。
- `UotanStudio/Locale/LocaleKeySets.cs`：同步索引数组（约 2939 行起，15 条），`LocaleDictionaryTests` 会逐键比对 resx 与索引数组。
- `UotanStudio.Tests/Services/NodeFactoryRegistrarTests.cs`：新增 `Toolbox_ObjectFamilies_ResolveLocalizedNames`——对 `NodeFactoryRegistrar.Descriptors` 中所有 `BuiltInTypeKeys` 拼出 `"Node_Family_" + Family`，断言 `LanguageService.Instance.GetString(key) != key`，并顺带断言 `editor.ToolboxItems` 非空。以后新增节点族而漏登记键会在测试里直接失败。

## 4. 验证证据

- `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 警告 0 错误。
- 定向（`LocaleDictionaryTests` + `NodeFactoryRegistrarTests` + `NodeCatalogReductionTests`）：73 通过 / 0 失败。
- 全量 `dotnet test --solution UotanStudio.slnx --no-build -c Debug`：8143 总计、8123 通过、20 跳过、0 失败（约 2m40s）。首轮曾出现 3 个与本改动无关的偶发失败（`Firmware.RepackMetadataTests.FailedExtraction_DoesNotCreateMetadata`、`Flash.FlashGlobalLogRoutingTests.ImportRkConfigAsync_LoaderRow_IsEffectiveOnlyWhileItsFileExists`、`Firmware.ErofsRebuildFeatureReplayTests.Build_TargetDeclaredToDefineTheAdviseBit_...`，错误输出含 `adb-sync: pull v2 failed ... gone!`）；单独重跑与第二轮全量均通过，属偶发而非回归。
- `dotnet format ... whitespace/style --verify-no-changes`：本次改动的 4 个文件干净；仓库另有 `UotanStudio.Tests/Devices/RkdlDeviceE2ETests.cs`、`UotanStudio/Views/UiWindowService.cs` 的 FINALNEWLINE 报错和其它并行改动，未触碰。
- `dotnet publish UotanStudio.Desktop -c Release -r win-x64 --self-contained`：成功无 error；IL2026/IL2104/IL3053 警告来自 Dock、FluentAvalonia、YamlDotNet、Common.Logging、LTRData.Extensions 等既有第三方包。

## 5. 未验证边界

- 未做界面人工核对：工具箱族名在真实窗口中的显示没有截图确认。
- 全量测试运行依赖构建产物（`--no-build`），未验证其它配置（Release 测试）下的表现。

## 6. 观察到但未处理的缺口（属并行改动）

`Flash_DeviceNotConnected`、`Flash_OperationSucceeded`、`Flash_ProtocolMismatch` 在 `UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs`（`:791`、`:825`、`:849`、`:869`、`:889`、`:913`、`:936`、`:1436`、`:1449`、`:1812`）与 `FlashViewModel.Rockchip.cs`（`:435`、`:458`、`:485`、`:512`）被使用，但 `Locale/` 中完全不存在，运行时取值会退回显示键名。这与烧录页的并行未提交改动相关（参见文档 `19fd2e72-2815-4c5f-a4ba-1ec06dc3ebea` 中提到的 `Flash_*` 键并行改动），本次未修。

## 7. 关键文件

- `UotanStudio/ViewModels/Nodes/NodeEditorViewModel.Toolbox.cs`（族名拼键）
- `UotanStudio/Workflow/Registry/NodeFactoryRegistrar.cs`（`Family` 推导、`BuiltInTypeKeys`）
- `UotanStudio/Locale/Node.resx`、`Node.zh-CN.resx`、`Locale/LocaleKeySets.cs`
- `UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs`、`FlashViewModel.Rockchip.cs`（`Flash_*` 未登记键）
- `UotanStudio.Tests/Services/NodeFactoryRegistrarTests.cs`

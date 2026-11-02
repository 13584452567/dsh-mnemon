---
id: "8c616b6b-70e5-4e10-82a9-0abdca53f5c4"
title: "UotanStudio 烧录页日志重复根治：loader 结论单点归属与行内选文件抑制重查"
description: "烧录页每次动作写两行相同日志的根因（结论被两条路径各写一遍）与修复不变量：loader 有效性单点报告、行内选文件抑制属性变化重查、行状态与日志同源；含有意行为变化、回归用例与实测/未验证边界。"
status: "active"
created_at: "2026-10-31T16:49:33.141Z"
updated_at: "2026-10-31T16:49:33.141Z"
content_hash: "d1fe110b795b18c3da2e306042255ab450c31415c42e45a796380d27487e2ade"
source_paths:
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.cs"
  - "UotanStudio/ViewModels/Flash/FlashDeviceUiPolicy.cs"
  - "UotanStudio.Tests/Flash/FlashRockchipFileRowTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "1e6397a7-24b9-4b6c-a508-0b85c40b551a"
memory_body_ids:
  []
---

# 烧录页日志重复：loader 结论单点归属与行内选文件的抑制重查

> 关联文档：烧录页售后包识别与刷写接入（id `c16e6fe5-2b4b-470f-aa26-fe3efceb4f9b`），同属 `FlashViewModel` 的落点约定。

## 1. 现象与根因

烧录页每次动作写出两行完全相同的日志。共享的 `AppendLog` 与 `LogText` 绑定本身没有问题，重复来自**同一个结论被两条独立路径各写一遍**：

1. `ApplyRkLoaderPath`（`UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs`）自己在指向 loader 后写结论：文件存在写 `Flash_RkLoaderDeclared`，不存在写 `Flash_RkLoaderMissing`；而它的两个调用方随后又各跑一次真正的 loader 检查（`ValidateRkLoaderFileAsync` → `Flash_RkLoaderMissing` / `Flash_RkLoaderChecked`）。结果：导入配置时"该配置引用的 Loader 不存在"出现两遍；用「加载引导」选文件时 `Flash_RkLoaderDeclared` 与"瑞芯微 loader 有效"各出现两遍。
2. 行内选文件对同一次点击检查两次（`FlashViewModel.Operations.cs`）：先写 `entry.FilePath`，属性变化经 `OnEntryPropertyChanged` → `UpdateRowValidityText` 触发一整次 loader 检查，方法末尾又直接调一次 `ValidateRkLoaderFileAsync`。

## 2. 设计不变量（修复后的约束）

- **一个结论只有一个写者**：loader 是否合法只由 `ValidateRkLoaderFileAsync` 报告，设置路径（`ApplyRkLoaderPath`）只写 `_rkLoaderPath` 并重发门控，不写结论日志。
- **一次动作只查一次**：行内选择器写入行字段期间置 `_suppressSpecialFileRowCheck`，`FlashViewModel.cs` 的 `OnEntryPropertyChanged` 见到该标志直接返回，不再由属性变化二次触发检查；loader 行由选择器自己检查一次，parameter 行沿用 `UpdateSpecialFileRowStatus` 的同步检查。
- **行状态与日志同源**：新增 `PublishRkLoaderStatus(row, valid)`，行内 `StatusText` 与日志用同一处判断，避免"行说有效、日志说无效"式分叉。
- 选择语义简化：`FlashPartitionEntry.CanSelect` 移除，可见行一律可勾选，全选判定不再过滤 `CanSelect`。

## 3. 有意行为变化

`ApplyRkLoaderPath` 不再输出"加载 Loader：<路径>"这一行（"读取芯片信息"路径里的同名提示仍保留）；loader 有效性只由"瑞芯微 loader 有效/无效"一行表述。

## 4. 回归用例

`UotanStudio.Tests/Flash/FlashRockchipFileRowTests.cs` 新增两条按 `LogText` 行计数断言的用例：缺失 loader 只报一行 `Flash_RkLoaderMissing`、有效 loader 只报一行 `Flash_RkLoaderChecked`。

## 5. 实测状态（2026-11-01 前后，Windows x64，Debug）

- `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 错误 0 警告。
- 全量测试：8140 总数 / 8120 通过 / 20 跳过 / 0 失败（2m 41s）；`FlashRockchipFileRowTests` 9/9，`UotanStudio.Tests.Flash` 57/57。
- 触碰的四个文件 `dotnet format` 的 `whitespace`、`style` `--verify-no-changes` 零报告；全解校验仍被未触碰的 `UotanStudio.Tests/Devices/RkdlDeviceE2ETests.cs`、`UotanStudio/Views/UiWindowService.cs` 的 FINALNEWLINE 挡住。
- 未验证边界：界面日志框的实际显示未人工目视，去重结论只到视图模型层 `LogText` 断言；NativeAOT 发布本轮未执行。

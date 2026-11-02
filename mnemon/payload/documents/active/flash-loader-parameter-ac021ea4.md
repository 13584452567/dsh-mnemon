---
id: "ac021ea4-1790-442d-9260-1e7ddf79ce40"
title: "Flash 瑞芯微 loader/parameter 行的勾选校验与写入排除（实现记录）"
description: "Flash 面板 loader 行可点选文件、勾选即 CRC 校验、parameter 行分区数校验，以及 IsWritableEntry 把特殊行排除出通用刷写路径（含 parameter 行空引用修复）与门禁实测、未验证边界。"
status: "active"
created_at: "2026-10-30T18:59:02.667Z"
updated_at: "2026-10-30T18:59:02.667Z"
content_hash: "a28b801724164d831a88c9d985f58d4aa8d86e6fdfd0cdd56e7e190f27c18123"
source_paths:
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.cs"
  - "UotanStudio.Tests/Flash/FlashRockchipFileRowTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "adb14ccf-cf5e-4f43-a246-8f6d2f1a498d"
memory_body_ids:
  []
---

# Flash 瑞芯微 loader / parameter 行的勾选校验与写入排除（实现记录）

面向 `UotanStudio` 宿主 Flash 面板的文件行语义：哪些行可以点选文件、勾选后如何校验、以及为什么 `loader`/`parameter` 行必须被排除在通用刷写路径之外。loaders 两阶段读取与门控维度见 `4cf0b689-be12-4ab6-b574-db9d331f500a`，本文只覆盖文件行本身。

## 1. 行内「选择文件」不再拦截 loader

- 去掉 `entry.Command == LoaderCommand` 这个拦截后，`Loader` 行与普通行一样会打开 `StorageProvider` 文件选择器；选完经 `ApplyRkLoaderPath` 落到当前行，`EffectiveRkLoaderPath`/`HasRockchipLoader` 立即重算并重发门控（`UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs`）。
- 理由：loader 路径既可能来自配置导入，也可能由用户现场指定，选择器是唯一自然的入口；门控仍只看 `EffectiveRkLoaderPath`，不引入第二套状态。

## 2. 勾选即校验

- `IsSelected` 变真时触发 `OnEntrySelected`；`Command`/`FilePath` 被编辑时同样重查（`UotanStudio/ViewModels/Flash/FlashViewModel.cs`、`FlashViewModel.Rockchip.cs`）。
- loader 行用 `FormatDetector.Detect` 取注册名 `Rockchip Loader Image` 的处理器，`ReadMetadataAsync` 读 102 字节头部、三张条目表与末尾 CRC-32；只有 `ChecksumStatus=Verified` 才算有效。文件缺失、非 loader、结构非法、校验和不符分别落日志，行内状态显示「loader 有效 / loader 无效」。
- 因为 CRC 覆盖全文件，检查放到 `Task.Run`，回到调用线程才写日志与行状态；未完成的检查挂在 `SpecialFileRowCheck` 上供测试等待。
- `parameter` 行走 `RockchipParameterParser.Parse`，行内显示解析出的分区数。
- 只重查导入的 loader 行：该行文件由配置决定，用户无法当场确认；用户在行内点选的文件已由选择器接受，不重复读取。

## 3. 特殊行排除出镜像写入路径

- 新增 `IsWritableEntry`（排除 `loader`/`parameter`），`RunSelectedAsync`、`RunRkEraseSelectedAsync`、`OnEdlEraseClick`、`OnEdlReadbackClick`、`HasSelectedEdlEntries` 全部经它过滤。
- 这同时修掉一个真实缺陷：`parameter` 行此前未被排除，勾选后会走到 `FastbootStepService.ExecuteAsync(device, entry.Step!)` 抛空引用。
- 末段分支由 `entry.Step!` 改为 `entry.Step is { } step`；编辑后没有步骤的行报 `Flash_CommandNotRunnable` 并标失败，不再崩溃。loader 仍只经 `EffectiveRkLoaderPath` 交给 RKDL 刷写路径。
- 本地化：中英双份 resx 与 `LocaleKeySets` 同步新增 12 个键。

## 4. 门禁实测（Windows x64，Debug）

- `dotnet build UotanStudio.slnx -c Debug`：0 错误 0 警告。
- `dotnet format` 的 whitespace 与 style `--verify-no-changes`：均零报告。
- `UotanStudio.Tests.Flash` 53/53；`RockchipLoaderTests` 15 通过 / 1 跳过（`UOTAN_RK_LOADER_SAMPLE` 门控）；`LocaleDictionaryTests` 6/6。
- 全量：8121 总数 / 8098 通过 / 21 跳过 / 2 失败，两条失败均不在本次改动文件上：`HostLayeringGuardTests.Views_DoNotImportApplicationServiceNamespacesOutsideTheFrozenAllowList`（`AvbToolWindow.axaml.cs` 的冻结豁免已不匹配任何 import）与 `RockusbPipeSelectionTests.SelectPipes_OutEndpointDeclaredFirst_StillMapsInToZeroAndOutToOne`（并行会话新增的未跟踪用例）。
- NativeAOT：`dotnet publish UotanStudio.Desktop -c Release -r win-x64 --self-contained` 成功，产物 172,645,376 B，无指向 `UotanStudio.*` 的裁剪/AOT 警告。

## 5. 未验证边界

- 无 GUI 目视：行内链接的真实点击热区、`StorageProvider` 选择器在 loader 行的实际行为、`StatusText` 在表格列宽下的换行截断，都只有视图模型层断言与编译通过。
- 校验读的是勾选瞬间的磁盘内容；文件之后被替换不会自动重查，需重新勾选或重新导入。
- 真机 loader 加载与 RKDL 写入未执行。

## 6. 关键文件

- `UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs`（选择文件入口、写入路径过滤）
- `UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs`（`UpdateRkLoaderRowStatusAsync`、`ValidateRkParameterRow`、`IsWritableEntry`）
- `UotanStudio/ViewModels/Flash/FlashViewModel.cs`（`OnEntrySelected`、`SpecialFileRowCheck`）
- `UotanStudio.Tests/Flash/FlashRockchipFileRowTests.cs`（7 条行级用例）、`UotanStudio.Tests/Flash/FlashGlobalLogRoutingTests.cs`
- `UotanStudio/Locale/Common.resx`、`Common.zh-CN.resx`、`LocaleKeySets.cs`

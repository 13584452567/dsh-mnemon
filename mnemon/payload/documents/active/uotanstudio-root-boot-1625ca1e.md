---
id: "1625ca1e-b4e4-4731-9cdf-933a93898b1b"
title: "UotanStudio 烧录页 Root 修补接线（复用 Boot 补丁链路）"
description: "烧录页 Root 修补接入 Boot 补丁后端链路：boot 分区自动检测、勾选后异步拉清单并测速、选版本即预下载校验、起刷在 prepared 字典上修补，含实测验证数字与未验证边界。"
status: "active"
created_at: "2026-10-28T14:11:35.317Z"
updated_at: "2026-10-28T14:11:35.317Z"
content_hash: "cd8d0d116d51083620a862ff24e27b236cf497816d521758f10bd5c3d155032a"
source_paths:
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Root.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs"
  - "UotanStudio/Locale/Common.resx"
  - "UotanStudio/Locale/LocaleKeySets.cs"
  - "UotanStudio.Tests/Flash/FlashRootPartitionTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "2100d91b-0e58-4dae-93fc-6e4897edd35f"
memory_body_ids:
  []
---

# UotanStudio 烧录页 Root 修补接线（复用 Boot 补丁链路）

关联文档：`4824dc1f-7c5c-4c92-b4df-7c87c0e0bcef`（下载源测速与最优源固定）、`6e1bc9bf-cdf8-477b-83dc-aab69e6bbd6d`（Boot 补丁窗口在线版本与补丁包链路）。本文只记录「烧录页」这一侧的接线与取舍，测速/下载契约、清单为何仍走源站等约束以上述两篇为准。

## 1. 落点

烧录页的 Root 修补不再自己实现一套，而是与「修补 Boot」窗口共用同一条后端链路（`RootPatchPackageService` + `FlashRootImage`）。改动集中在：

- `UotanStudio/ViewModels/Flash/FlashViewModel.Root.cs`：分区名单、清单拉取、测速、版本选择与预准备、包解析校验。
- `UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs`：起刷时在预备字典上做修补。
- `UotanStudio/Views/.../Flash` 页与 `Locale/Common.resx`、`Common.zh-CN.resx`、`Locale/LocaleKeySets.cs`。

## 2. 分区下拉框改为自动检测

删除了硬编码的 `boot / init_boot / vendor_boot`。改为从烧录视图已加载的条目里筛出分区名包含 `boot` 的去重名单（`RootBootPartitionNames`），随分区列表变化重建并保留原有选择。

取舍：名单不变时不重建 `ItemsSource`，避免导入几百行整包时反复重置下拉框、丢失当前选择。

## 3. 勾选「添加 Root」才异步拉清单 + 测速

- 勾选 `AddRootBox` 触发 `ListPackagesAsync` 拉版本清单，再对最新资产的源站与 27 个镜像各取 256 KiB（单源 5 s 上限）测速，固定最快源，回填 `RootVersionBox` 并选中首项。
- 条目带「（已下载）」后缀；测速、下载、就绪、空清单都写进页面日志。
- 选中版本即预下载并校验（长度 + 公布 SHA-256），原子发布到共享下载目录 `<magisk|magisk-alpha|kernelsu|sukisu|apatch>/<version>/<asset>`，因此点「开始烧录」时通常已是缓存命中。
- 「自选文件」分支改走 `RootPatchPackageService.ValidateAsync`，与窗口判定一致；切换来源会取消在途请求并清空版本缓存。

设计理由：把「用户勾选/选版本」当成唯一触发点，避免打开页面就为未使用的功能付出网络与测速代价；预下载把耗时挪到用户还在配其它选项的时段。

## 4. 起刷时才修补，且不改原始条目

- 起刷校验换成「所选分区必须在本次勾选列表里」，否则提示新增的 `Flash_RootPartitionNotSelected`。
- 修补结果覆盖在 `prepared` 字典上，原始的 `FilePath`、行状态与列表展示完全不变；补丁包在修补前用 `ResolveRootPackageAsync` 再解析一次（缓存命中或按需下载）。
- 新增本地化键：`Flash_RootPartitionNotSelected`、`Flash_RootPackageMissing`（中英 resx + `LocaleKeySets` 三处同步）。

## 5. 验证（实测）

- `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 错误 0 警告。
- `dotnet format` `whitespace` / `style` 对本轮文件零报告。
- 定向测试：新增 `UotanStudio.Tests/Flash/FlashRootPartitionTests.cs` 6/6、`UotanStudio.Tests.Flash` 命名空间 22/22、`RootPatchPackageTests` 21/21、`FlashRootImageTests` 6/6、`LocaleDictionaryTests` 6/6、`HostLayeringGuardTests` 4/4。
- 全量 `dotnet test --solution UotanStudio.slnx --no-build -c Debug`：7804 总数 / 7796 通过 / 8 跳过 / 0 失败（1m42s）。
- NativeAOT：`dotnet publish UotanStudio.Desktop ... -r win-x64 --self-contained` 成功，`UotanStudio.Desktop.exe` 156,161,024 B；只有第三方既有的 IL2104/IL3053，没有指向本轮代码的裁剪/AOT 警告。结果记入 `VERIFICATION.md`。

## 6. 未验证边界

- GUI 实际渲染、勾选后的真实测速耗时与镜像命中率、真机 boot 修补＋烧录均无自动化覆盖，需要在界面里走一遍。
- APatch 仍不能在烧录流程中修补（需要 SuperKey）；选中会明确报「不受支持的包」。

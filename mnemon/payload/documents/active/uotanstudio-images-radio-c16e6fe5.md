---
id: "c16e6fe5-2b4b-470f-aa26-fe3efceb4f9b"
title: "UotanStudio 烧录页售后包（IMAGES/RADIO）识别与刷写接入"
description: "烧录页识别售后包目录（IMAGES+RADIO）并按分区列出镜像的规则与落点：照搬紫罗兰工具箱语义、RADIO 合并不重复、BLANK_GPT 清单过滤、fastboot 路由修正，含实测与未验证边界。"
status: "active"
created_at: "2026-10-28T17:09:07.689Z"
updated_at: "2026-10-28T17:09:07.689Z"
content_hash: "8481c1991927b982d7f8403a05f7519a4347eeef734850fbbbd1aa419c8646ac"
source_paths:
  - "UotanStudio/Services/Flash/FlashAfterSalesPackage.cs"
  - "UotanStudio/Services/Flash/FlashManifestSelection.cs"
  - "UotanStudio/Services/Flash/FlashBinPackage.cs"
  - "UotanStudio/Services/Flash/FlashEdlBundleService.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs"
  - "UotanStudio.Tests/Flash/FlashAfterSalesPackageTests.cs"
  - "reference/VioletToolBox/VioletToolBox/oujiaflash.cs"
  - "reference/VioletToolBox/VioletToolBox/SuperMaker.cs"
session_ids:
  - "347b4b96-3d6c-4134-8175-f1d45020691c"
memory_body_ids:
  []
---

# UotanStudio 烧录页售后包（IMAGES/RADIO）识别与刷写接入

参考实现：`reference/VioletToolBox`（紫罗兰工具箱）。相关参考点：售后散包扫描 `oujiaflash.cs` 的 `ParseAfterSalesImages`（约 1088–1214 行）、售后包 FB 模式（合并 super → 过滤 XML → 按表刷写，约 9470–9685 行）、super 合并 `SuperMaker.cs`（`MakeSuperFromDirectoryAsync`，56–105 行，产出 `ProcessedPartitions`）。

相关文档：`1625ca1e-b4e4-4731-9cdf-933a93898b1b`（烧录页 Root 修补接线，同一页面的另一条分支）。

## 1. 售后包结构判定

目录 = `IMAGES` + `RADIO`，再以 `META` / `MY_MANIFEST` 等标记目录辅助判定。判定规则：`IMAGES` 必须存在且含镜像或清单，外加任一标记目录。普通镜像目录不应被误判（有测试覆盖）。

## 2. 分区枚举规则（照搬紫罗兰语义）

| 规则 | 紫罗兰来源 | 本项目实现 |
| --- | --- | --- |
| 先处理 `IMAGES` 下的按分区子目录（`my_bigball/my_bigball.<版本>.img`），分区名取子目录名 | 8 个写死的 `specialSubFolders` | 泛化到任意子目录，命名取子目录名，忽略 `.violet-super-*` 类点目录 |
| 再扫 `IMAGES`、`RADIO` 顶层 `.img`/`.iso`，分区名取小写文件名（尾部数字版本段剥离） | 同函数 | 一致，另支持 `.img.zst` |
| 跳过 `userdata`、`metadata` | 同函数 | 一致（清单标签路径同样跳过） |
| `IMAGES` 顶层文件与高优先级子目录分区同名或前缀相同时跳过 | 冲突检测 | 一致（前缀匹配，比较前剥离版本号） |
| `RADIO` 与 `IMAGES` 同分区 | 紫罗兰直接追加（产生重复行） | 保留 `IMAGES`，重复的 RADIO 计数并写入日志（更安全） |
| `super.img` 已合并时不再刷散包 `my_*` | 合并后清理 `my*` 文件/目录 | `super.img` 存在时隐藏 `my*` 动态分区并计数 |
| `rawprogram*_BLANK_GPT.xml`、`rawprogram*_WIPE_PARTITIONS.xml` 不参与刷写 | 售后包模式的 XML 过滤步骤 | `FlashManifestSelection` 只选主 `rawprogram<N>.xml` |

最后一条顺带修掉了既有 EDL 包会执行 BLANK_GPT 拆 GPT 的隐患，因此改成了 `FlashBinPackage` 与 `FlashEdlBundleService` 共用同一枚举器。

## 3. 代码落点

- `UotanStudio/Services/Flash/FlashAfterSalesPackage.cs`（新增）：结构识别 + 分区读取，产出 `AfterSalesPartitionImage` / `AfterSalesPackageContents`（含 `Programmer`、`Programmers`、`MergedRadioImages`、`HasMergedSuper`、`HiddenDynamicImages`）。能被主 rawprogram 清单编程的镜像回填 `RawProgramEntry`，所以仍可 EDL 回读与整包刷写。
- `UotanStudio/Services/Flash/FlashManifestSelection.cs`（新增）：只枚举主编号清单；`FlashBinPackage.cs`（原 31–37 行）与 `FlashEdlBundleService.cs`（原 76–82 行）改为共用它。
- `UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs`：选择目录后先走售后包分支（约 458–466 行），`ApplyAfterSalesPackage` 建立行、`_edlEntries`、programmer 候选与日志。
- 路由修正 `OnFlashSelectedClick`（约 1811–1825 行）：只有 EDL 目标（或未选设备）才走 EDL 完整包；接到 fastboot 设备时按勾选逐行刷写。否则售后包在 fastboot 上「刷入选中分区」会静默无动作。
- 本地化：`Flash_AfterSalesPackage`、`Flash_AfterSalesMerged` 同步 `Common.resx` / `Common.zh-CN.resx` / `LocaleKeySets.cs` 三处。
- 测试：`UotanStudio.Tests/Flash/FlashAfterSalesPackageTests.cs`——分区映射与 RADIO 合并、已合并 super 隐藏散包、普通镜像目录不误判、清单过滤。

## 4. 设计取舍

- 行默认不勾选（紫罗兰默认全选），沿用该页面其他包的约定以避免误刷；「全选」可一键勾上。
- RADIO 重复分区不产生重复行，改为计数 + 日志，属于相对参考实现的有意偏离。
- 未实现 `SuperMaker` 式合并（用 `META/super_def*.json` + `lpmake` 把 `my_*` 打成 `IMAGES/super.img`），也不写 `violet` 标记文件——属构建/写盘动作，不在「读取」范围。若要接，可走 `UotanStudio.Delivery` 的 super 构建或单独服务。

## 5. 验证（实测）

- `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 警告 / 0 错误。
- 定向测试合计 36 通过 / 0 失败 / 0 跳过：`FlashAfterSalesPackageTests` 4、`FlashPackageParsingTests` 5、`FlashBinPackageTests` 3、`EdlSerialBundleServiceTests` 18、`LocaleDictionaryTests` 6。
- `dotnet format`（whitespace + style）`--verify-no-changes` 无输出、退出码 0。
- 开发中曾出现一次测试失败（`Read_AfterSalesPackage_MapsPartitionsAndMergesRadioDuplicates` 期望 `my_bigball` 排首、实得 `modem`），修正枚举顺序后通过。

## 6. 未验证边界

- 真实售后包目录的文件夹选择器、界面渲染、真机刷写均未验证（无可用设备；FolderPicker 属人工验证范围）。
- 该轮改动未包含 NativeAOT 发布验证（交付要求最终验证需补一次 `dotnet publish ... -r win-x64 --self-contained`）。
- EDL 回读路径只在单元测试层面覆盖，未对真机 rawprogram 清单做端到端确认。

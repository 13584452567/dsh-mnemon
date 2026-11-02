---
id: "2d0db777-3b1d-4d91-a144-cb2bfb14f7c9"
title: "UotanStudio 并发写入下的提交前门禁与分批提交（2026-10-27）"
description: "外部脚本并发改写工作区时的静止快照门禁程序、whitespace 失败修复方式、提交顺序约束、NativeAOT 第三方警告基线与 AVB 未验证边界。"
status: "active"
created_at: "2026-10-27T01:20:00.254Z"
updated_at: "2026-10-27T01:20:00.254Z"
content_hash: "23e629b49df523bc5b98d1afad04d03c4c0bad0b015a10ec5bb8cb9bf76aed4f"
source_paths:
  - "AGENTS.md"
  - "UotanStudio.Tests/AGENTS.md"
  - "Directory.Packages.props"
  - "UotanStudio.Desktop/UotanStudio.Desktop.csproj"
session_ids:
  - "a6687988-7cb3-4ff2-9d4d-a6018d2fac4f"
memory_body_ids:
  []
---

# UotanStudio 并发写入下的提交前门禁与分批提交（2026-10-27）

本轮在**另一路会话用自动化脚本持续改写同一工作区**的情况下，把一批面向“当前契约”的注释规范化（235 个文件）与功能改动（FAT 打包、AVB 签名/工具窗、Payload 分区补丁、依赖升级）分批提交。本文只记录可复用的操作程序与实测基线，供后续在脏工作区里跑门禁、拆提交时参考。

## 1. 门禁必须在“静止快照”上运行

工作区被外部脚本整批改写时（单轮最高 404 个改动文件，含 `Directory.Packages.props` 版本升级与未跟踪的临时打补丁脚本），边写边跑门禁的结果不可信，而且并行后台任务会被对方终止。

可用程序：

1. 监视**全仓库任意文件**变动（不是只盯自己改的文件），统计改动数与最近变动时间。
2. 要求静默约 8 分钟无变化后再取快照；依赖升级后必须重跑 `dotnet restore`，不能沿用升级前的还原结果。
3. 在快照上顺序执行：`dotnet build UotanStudio.slnx --no-restore -c Release` → `dotnet format ... whitespace --verify-no-changes` → `dotnet format ... style --verify-no-changes` → `dotnet test --solution UotanStudio.slnx --no-restore -c Release --minimum-expected-tests 6000`。
4. 全量门禁耗时较长，放后台容易被外部脚本的清理动作连带杀掉；改为前台分步执行并在每步后校验退出码。

实测：注释批量改写后 `whitespace` 门禁会因缩进/段落合并失败（本轮即如此）。处置是先跑 `dotnet format UotanStudio.slnx whitespace`（不带 `--verify-no-changes`）自动修复，再复检 whitespace/style，然后构建与测试；不要手工逐文件改缩进。

门禁通过后的基线数字：构建 0 警告 0 错误；whitespace 与 style 均无报告；测试 7662 总数 / 7655 通过 / 7 跳过 / 0 失败。

## 2. 提交拆分与顺序约束

- 按项目/主题分块提交（Comm、Firmware、宿主、Tests、Mirroring、Media、Compress、Hash、Mcp、其余叶子库），每块一个英文 Conventional Commit，trailer 只保留 `Signed-off-by`。
- **权威守卫测试要先看它约束哪些代码**：本轮 `test(firmware): guard the codec, hash, sparse-magic and ZIP authorities` 会扫描随后两个提交才落地的 FNV/稀疏魔数规则；若先提交守卫，中间提交会红。处置是把相关提交退回索引，按“实现 → 守卫”重排，使每个提交自身自洽。
- 依赖升级单独成提交（`build(deps): ...`），并在它之后重跑 restore、构建、测试与 AOT 发布，因为升级后的还原图与警告集合都可能变化。

## 3. NativeAOT 发布基线（依赖升级后复验）

`dotnet publish UotanStudio.Desktop/UotanStudio.Desktop.csproj -c Release -r win-x64 --self-contained`（artifacts 指向 %TEMP%）退出码 0，产物 `UotanStudio.Desktop.exe` = 156,844,032 B，**不产出** `UotanStudio.Desktop.pdb`。

裁剪/AOT 警告共 12 条（IL2104、IL3053），全部来自第三方程序集：Avalonia.Controls.DataGrid、Semi.Avalonia.Dock、Ursa、Ursa.Themes.Semi、Dock.Avalonia、Iciclecreek.Avalonia.Terminal、Common.Logging、LTRData.Extensions、YamlDotNet；**0 条**指向 `UotanStudio.*`。后续判断“是否引入新裁剪警告”时以此集合为基线，新增的 `UotanStudio.*` 警告即为回归。

PDB 不产出的机制与发布耗时的历史实测见 `b4c0eb13-4c71-4066-b8eb-d5e87d096ba1`（NativeAOT 发布瘦身与本地构建加速）。

## 4. 本轮未验证边界

AVB/OTAPatch 只对本地构建的 Payload 与最小 OTA ZIP 验证；未覆盖真实厂商 OTA、差分/source 操作、`super` 动态分区与真机启动链；AVB 工具窗口与原生文件选择器未做人工 GUI 验证；未与 Android `avbtool` 逐字节对照。这些不能写成“已通过”，后续接真实样本时需要单独补证。

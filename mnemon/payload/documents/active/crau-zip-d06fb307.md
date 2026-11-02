---
id: "d06fb307-a8f3-47c5-8055-6ac1f1ebb5a9"
title: "烧录页刷机包类型分发：内容识别（CrAU 魔数/ZIP 载荷）与不支持类型显式报错"
description: "UotanStudio 烧录页选包分发的判定顺序与依据：payload 改按 CrAU 魔数或 ZIP payload.bin 条目识别、保留 update.zip、未知类型报 Flash_UnsupportedPackage 并关闭刷写门控；含文件落点、设计取舍、实测与未验证边界。"
status: "active"
created_at: "2026-10-28T18:24:06.310Z"
updated_at: "2026-10-28T18:24:06.310Z"
content_hash: "4c160e8f48b4b98e3345e09a7de6dc4af2aeebb25df39f5e73280fc877330513"
source_paths:
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs"
  - "UotanStudio/Services/Flash/FlashPayloadService.cs"
  - "UotanStudio.Firmware/OtaDelta/Payload/PayloadTool.cs"
  - "UotanStudio.Tests/Services/FlashPayloadServiceTests.cs"
  - "UotanStudio.Tests/Firmware/PayloadToolTests.cs"
  - "UotanStudio/Locale/Common.resx"
  - "UotanStudio/Locale/Common.zh-CN.resx"
  - "UotanStudio/Locale/LocaleKeySets.cs"
session_ids:
  - "471b7295-13ef-498b-a8ae-2106f1e1c802"
memory_body_ids:
  []
---

# 烧录页刷机包类型分发：内容识别与不支持类型显式报错

> 同一提交（cfac2cb1 `feat(flash): read after-sales packages and detect payloads by content`）的售后包识别/EDL 清单过滤部分见文档 `c16e6fe5-2b4b-470f-aa26-fe3efceb4f9b`；本文只覆盖「选中文件的类型判定」这一层。

## 1. 目标与问题

此前 `FlashViewModel.Operations` 的选包分发按扩展名判定，最后有一条「任意扩展名 → 当 payload 解析」的兜底：任何未知文件都会被硬当作 payload 读，失败时只留下解析异常，用户看不出「这个文件不是支持的刷机包」。要求改为**按文件内容识别 + 对未知类型明确报错**。

## 2. 现行分发顺序（`FlashViewModel.Operations.cs` 约 198 行起）

| 顺序 | 判定 | 行为 |
| --- | --- | --- |
| 前置 | 二进制 `CFG\0` 头 | 瑞芯微配置导入 |
| 1 | `http(s)://` | 远程 Payload |
| 2 | `.img` / `.img.zst` | super_empty / LP super / 单镜像 |
| 3 | `.ofp` / `.ops` / `.ozip` | OPPO 容器 |
| 4 | `.nb0` / `.ntpi` | 厂商容器 |
| 5 | `.xml` 且为 scatter | MTK 散包 |
| 6 | `.bat` / `.sh` | 脚本清单 |
| 7 | `.txt` | 文本清单 |
| 8 | payload 头魔数 `CrAU` 或 ZIP 内含 `payload.bin` 条目 | Payload 分区列表 |
| 9 | 目录 | 售后包 / EDL 散包 / 普通目录 |
| 10 | 其余已存在的文件 | 记日志 `Flash_UnsupportedPackage`（`Unsupported package type: {0}`）并清空 `_loadedPath` |

要点：payload 分支**不再看扩展名**，且必须排在脚本/文本之后（`.txt`、`.bat` 内容不会被误判）；未知类型兜底放在目录分支之后、只对 `File.Exists` 生效，目录仍走售后包/EDL/普通目录三条路。

## 3. 内容判定实现

- `UotanStudio.Firmware/OtaDelta/Payload/PayloadTool.cs`：新增 `IsPayload`（读前 4 字节按大端比对 `CrAU`）与 `IsPayloadArchive`（用 `ZipContainer` 打开，按条目名 `payload.bin` 不区分大小写匹配，异常吞掉、只读条目名不读数据）。
- `UotanStudio/Services/Flash/FlashPayloadService.cs`：新增 `LooksLikePayload(path)` = 原始载荷 **或** 含载荷条目的 ZIP。
- 本地化 key `Flash_UnsupportedPackage` 同步 `Common.resx` / `Common.zh-CN.resx` / `LocaleKeySets.cs` 三处。

## 4. 设计取舍

- **魔数优先于扩展名**：`payload_20260101.bin`、无扩展名的 `ota` 仍被识别；反过来名为 `payload.bin` 但头不是 `CrAU` 的文件不再被强解。
- **保留 ZIP 包裹的 OTA**：原来整包 `update.zip`（`PayloadParser` 支持 ZIP 内 `payload.bin`）只靠那条「任意扩展名」兜底才能打开；改成魔数判定后会掉进「不支持」，因此补 `IsPayloadArchive` 把它显式纳入，而不是降级成不支持。
- **未知类型不静默失败**：报出本地化文案并清空 `_loadedPath`，刷写门控随之关闭，避免「选了包却没有可刷内容」的错觉。

## 5. 验证（实测）

- `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 警告 / 0 错误（过程中出现过一次 `CS0103: CultureInfo`，补 `using System.Globalization;` 后消失）。
- `dotnet format` whitespace + style `--verify-no-changes`：无差异。
- 定向测试：`PayloadToolTests` 11（新增 ZIP 有/无 `payload.bin`、原始载荷、缺失文件四种判定）、`FlashPayloadServiceTests` 2（`payload.bin`、无扩展名、ZIP、纯文本、缺失文件五种内容识别）、`FlashPackageParsingTests` 5、`LocaleDictionaryTests` 6、`FlashViewModelCommandTests` 2、`FlashAfterSalesPackageTests` 4、`FlashRootPartitionTests` 6，全部通过。

## 6. 未验证边界

- 真实界面里用选择器点开各类文件后的行为（Avalonia 文件选择器与日志渲染无自动化覆盖），需人工各点一次 `payload.bin`、`update.zip`、`.bat`、`.txt` 与一个不支持文件。
- 该提交未包含 NativeAOT 发布验证（项目要求最终验证补一次 `dotnet publish UotanStudio.Desktop/UotanStudio.Desktop.csproj -c Release -r win-x64 --self-contained`）。

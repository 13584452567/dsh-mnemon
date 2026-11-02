---
id: "1ded92c8-51ea-4402-b7f3-22f6b828ab17"
title: "HarmonyOS update.bin 容器格式解析与 UotanStudio 只读处理器实现（2026-10-26）"
description: "HarmonyOS update.bin 的头部/组件表 L1-L2 布局、元数据 TLV、输出命名算法与 UotanStudio 只读处理器的接入点、真实样例实测哈希、测试与未验证边界。"
status: "active"
created_at: "2026-10-26T07:53:17.356Z"
updated_at: "2026-10-26T07:53:17.356Z"
content_hash: "05f161f11cce45ec2346cec18aac54d43b22e0812350ce941c56e1bb88c2c4cd"
source_paths:
  - "UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinReader.cs"
  - "UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinFormatHandler.cs"
  - "UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinPackage.cs"
  - "UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinLayout.cs"
  - "UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinComponent.cs"
  - "UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinSignatureKind.cs"
  - "UotanStudio.Tests/Firmware/HarmonyUpdateBinTests.cs"
  - "UotanStudio.Firmware/Application/Formats/FirmwareFormatRegistry.cs"
  - "VERIFICATION.md"
session_ids:
  - "cb232d97-1587-4fe2-9fe7-462b70dc3faf"
memory_body_ids:
  []
---

# HarmonyOS update.bin 容器格式与 UotanStudio 只读处理器实现（2026-10-26）

HarmonyOS 侧的 `update.bin` 是一个带签名元数据 TLV 的组件容器。本轮把它作为**只读格式**接进 UotanStudio 固件层：只解析/解包，不打包。本文记录容器结构（便于后续维护与自写解析器）、处理器的接入点与契约、真实样例实测结果，以及明确未覆盖的边界。

相关：并行会话的 ADB 直连问题文档 [ADB 直连三重根因](documents/active/adb-crc-cnxn-tls-dns-sd-2026-10-26-b3f83b64.md)（其"未验证边界"一节曾记录本组 HarmonyOS 新文件的 `FINALNEWLINE` 格式门失败，已在本轮修复）。

## 1. 容器结构

新增实现集中在 `UotanStudio.Firmware/VendorFirmware/HarmonyOS/`（`VendorFirmware` 是厂商私有的只读格式目录）：

| 文件 | 行数 | 职责 |
| --- | --- | --- |
| [HarmonyUpdateBinReader.cs](UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinReader.cs) | 521 | 头部、组件表、元数据 TLV 解析与有界校验 |
| [HarmonyUpdateBinFormatHandler.cs](UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinFormatHandler.cs) | 315 | `CanHandle` 探测、元数据读取、流式解包 |
| [HarmonyUpdateBinPackage.cs](UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinPackage.cs) | 38 | 解析结果聚合（布局、组件、数据偏移、TLV 类型） |
| [HarmonyUpdateBinLayout.cs](UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinLayout.cs) | 16 | L1 / L2 枚举 |
| [HarmonyUpdateBinComponent.cs](UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinComponent.cs) | 44 | 单个组件的名称/类型/大小/SHA-256/输出名 |
| [HarmonyUpdateBinSignatureKind.cs](UotanStudio.Firmware/VendorFirmware/HarmonyOS/HarmonyUpdateBinSignatureKind.cs) | 15 | `Signature` / `HashAndSignature` |

### 头部与组件表

- 固定 180 字节头部，**组件表长度记在偏移 `178`**；其后 16 字节保留区，再后是元数据 TLV 与数据载荷。
- **首个字是布局提示**：`0x11` = L1、`0x01` = L2。仅凭 `.bin` 扩展名无法判定，必须由该提示 + 组件表解析成功共同确认，这也是 `CanHandle` 的探测依据（`requireLayoutHint: true`）。
- 组件表记录：

| 布局 | 记录长度 | 名称字段 |
| --- | --- | --- |
| L1 | 71 字节 | 16 字节 |
| L2 | 87 字节 | 32 字节 |

- 记录内字段按**名称长度动态偏移**（`nameLen`，即实际名称字段宽度）：
  - 组件类型 = `nameLen + 4`
  - 数据大小 = `nameLen + 15`
  - SHA-256 = `nameLen + 23`
- 名称按**严格 UTF-8** 解码，并做跨平台安全文件名检查（拒绝路径分隔符/保留名），重复名同样拒绝。

### 元数据 TLV

TLV 头 6 字节：`ushort kind`（小端）+ `uint length`（小端）。支持两种形态：

- 直接 `0x08`（signature）；
- 链式 `0x06`（hash-info）→ `0x07`（hash-data）→ `0x08`（signature），顺序错误即抛 `InvalidDataException`。

未知 kind 一律拒绝。所有偏移/长度/组件数/名称都做有界校验；TLV 头不足 6 字节判截断。`0x08` 载荷**只跳过、不验签**。

### 输出命名算法（沿用参考实现）

| 组件名 | 输出名 |
| --- | --- |
| `board_list` | `BOARD.list` |
| `version_list` | `VERSION.mbn` |
| `software_ver_list` | `SOFTWARE_VER_LIST.mbn` |
| 其它，类型 `0` | `<名称>.img` |
| 其它，类型 `1` | `<名称>.zip` |

## 2. 处理器契约与接入点

`HarmonyUpdateBinFormatHandler` 实现 `IReadonlyFormatHandler` + `IRepackMetadataProvider`：

- `FileExtensions => ["bin"]`，但身份由头部/组件表探测确认，不是扩展名级绑定。
- `SupportsMetadataOnlyUnpack => true`：可只发布 `metadata.json` 而不写组件文件。
- `ConfigureRepackMetadata` 固定 `CanPackDirectory = false` 并在 `PackingNotes` 说明"容器带签名元数据 TLV，只支持解包"——**打包不实现**，因为重打包会破坏签名元数据。
- 解包：128 KiB 流式复制缓冲；支持按组件名或输出名筛选、`VerifyChecksums` 在流式写出时校验记录的 SHA-256、输出预算、覆盖策略、`CancellationToken` 与原子写入（临时文件校验后替换）。
- 注册位置：[FirmwareFormatRegistry.cs:64](UotanStudio.Firmware/Application/Formats/FirmwareFormatRegistry.cs#L64) **在 `UpdateApp` 之前**。这是关键顺序约束：两者都吃 `.bin`，update.bin 必须先行结构探测，否则会误进 `UPDATE.APP`。
- 探测链路上 `DocumentFormatPolicy.IsFirmwareExtension` 用注册表聚合的固件扩展名（`FirmwareFormatRegistry` 的懒加载 `_firmwareExtensions`），因此新格式自动进入"已知固件扩展名"集合，无需改宿主侧策略。

## 3. 真实样例实测

样例 `D:\Download\update_full_cust_HAD-W24_all_cn.zip`（zip SHA-256 `e1f28c16…197aed`）内的 `update.bin`（35,778 B）：

- 布局 **L2**、组件数 4、`DataOffset=2851`、`PayloadBytes=32927`、元数据 TLV `signature`。
- `VerifyChecksums=true` 解出结果与组件表记录逐一相同：

| 输出 | 大小 | SHA-256 |
| --- | --- | --- |
| `package_info.img` | 121 B | `da56c604…258f74bc` |
| `BOARD.list` | 20 B | `68496808…7ac510af` |
| `ptable.img` | 18 B | `d32f7851…388881` |
| `version.img` | 32,768 B | `f2cf68c8…522406e` |

同时发布 `metadata.json`（705 B，`canPackDirectory=false`）。样例只解压到 `%TEMP%` 用临时工装验证后删除，**未**加入 `TestData/`、测试语料或仓库。

## 4. 验证

- 定向：`HarmonyUpdateBinTests` 23/23（L2/L1、输出名、SHA-256 校验与失败回滚、筛选、预算、覆盖、取消、截断、非法名称、重复名、未知 TLV、metadata-only）。
- 全 `UotanStudio.Tests.Firmware` 命名空间：4357 总数 / 4355 通过 / 2 跳过（`UOTAN_EROFS_CORPUS` 未设置）/ 0 失败，约 32 s。
- 架构门全绿：格式注册、命名空间归属、依赖基线、Hash 边界、ZIP 容器边界。
- `dotnet build UotanStudio.slnx --no-restore -c Debug`：0 错误；3 条既有 `xUnit1051`/`xUnit1069` 警告来自无关测试文件。
- `dotnet format` 的 `whitespace` 与 `style` 门：全解零报告（首轮 `WHITESPACE`/`FINALNEWLINE` 失败源于 `insert_final_newline = false` 约定与表达式体换行，已按 `.editorconfig` 修正）。
- NativeAOT publish 成功，`UotanStudio.Desktop.exe` 155,762,176 B，仅有第三方既有 IL2026/IL2104/IL3053 警告。

## 5. 未覆盖边界

- `0x08` 签名载荷**只跳过不验签**；签名链的有效性不在本处理器内。
- ZIP 外壳、组件内部的 `HARMONY!`/HVB 封装、以及组件内文件系统的二次解包均不处理。
- 打包（repack）未实现；`metadata-only` 解包只产元数据。
- 除上述 35 KB 小样外，尚未用多 GB 整包或真实 L1 包实测；L1 路径目前只由合成用例覆盖。
- 本轮全量测试期间的 1 个失败来自并行会话的 ADB sync v2 拉取用例（`gone!`），与本改动无关。

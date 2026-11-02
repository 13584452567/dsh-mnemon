---
id: "1d17fe25-755e-4d34-a207-81ee15fb1bde"
title: "酷派 CPB 固件包布局与 UotanStudio 只读处理器（2026-10-27）"
description: "酷派 CPB 552 字节头 + 76 字节镜像头的解码、密钥 0x66666667 高半乘（实为除 5）与 CRC 覆盖解码后头的两处参考实现怪癖、固件层处理器与宿主接线、魔数规则重生成、测试与 NativeAOT 实测数字、CS2012 artifacts-path 环境坑与未验证边界。"
status: "active"
created_at: "2026-10-26T21:43:35.560Z"
updated_at: "2026-10-26T21:43:35.560Z"
content_hash: "4d8624fdb08c6218aaa18d304f7cd9ff10c57c620839e3346a1b927886bf8fe1"
source_paths:
  - "UotanStudio.Firmware/VendorFirmware/Coolpad/Cpb/CpbFormat.cs"
  - "UotanStudio.Firmware/VendorFirmware/Coolpad/Cpb/CpbFormatHandler.cs"
  - "UotanStudio.Tests/Firmware/CpbTests.cs"
  - "UotanStudio.Tests/TestInfrastructure/CpbContainerBuilder.cs"
  - "UotanStudio/Services/Firmware/ImageInputService.cs"
  - "UotanStudio.Firmware/Application/Formats/FirmwareFormatRegistry.cs"
  - "UotanStudio.Firmware.MagicGen/corpus/Magdir/uotan-firmware"
  - "VERIFICATION.md"
session_ids:
  - "5bb67545-8225-420f-b693-9b586d950279"
memory_body_ids:
  []
---

# 酷派 CPB 固件包：头部/镜像目录布局与 UotanStudio 只读处理器（2026-10-27）

酷派 CPB（`.cpb`）作为**只读格式**接入 UotanStudio 固件层：只解析头与目录、只解包，不做重打包。参考实现为仓库内 vendored 的 `reference/MIO-KITCHEN-SOURCE/src/c_extension/cpb_file/cpbtool.c`，两处非直觉行为按参考实现逐位复刻并在测试里钉住。同类的厂商只读格式接入模式见 [HarmonyOS update.bin 容器格式解析与 UotanStudio 只读处理器实现](documents/active/harmonyos-update-bin-uotanstudio-2026-10-26-1ded92c8.md)（id `1ded92c8-51ea-4402-b7f3-22f6b828ab17`）。

## 1. 容器结构

- 头固定 552 字节：魔数 `CP` + 版本 `1.7`、6 个"校正字"、机型/硬件版本/版本号/来源四段文本、镜像头结束位置、CRC16。
- 其后每条镜像头 76 字节：64 字节名称、偏移、长度、CRC16。
- **偏移与长度是异或混淆的**：用校正字推导出的密钥对 `idx + keyByte` 做自逆 XOR（同一变换用于加密与解密）。
- 容器校验和按**解码后**的镜像头累加，而不是磁盘上的混淆字节。

### 密钥推导（复刻的怪癖之一）

校正字之和的"除以 6"在参考 C 里实际是 `0x66666667` 高半乘再右移一位——等价于**除以 5**，不是除以 6。厂商默认校正字因此得到密钥 `0x00310000`；`DeriveKey_VendorCorrectionWords_ProducesTheDocumentedSeed` 钉住该常量。

### CRC-16 比较（复刻的怪癖之二）

CRC-16 累加器**不截断到 16 位**，所以头部与镜像头都用完整 32 位比较；同时接受低 16 位一致（对应以 16 位寄存器实现的产出方），避免误拒真实固件。`BitwiseCrc16` 用逐位移位独立复算这条低 16 位等价关系。负载损坏仍写出但报校验失败；头部校验和不一致或范围截断则拒解且不留暂存文件。

## 2. 实现与接入点

| 文件 | 职责 |
| --- | --- |
| `UotanStudio.Firmware/VendorFirmware/Coolpad/Cpb/CpbFormat.cs` | 552 字节头 + 76 字节镜像头解码、CRC-16 表、混淆变换、名称净化、越界/上限校验 |
| `UotanStudio.Firmware/VendorFirmware/Coolpad/Cpb/CpbFormatHandler.cs` | 发布格式名 `Coolpad CPB Package`（后缀 `cpb`）；`ReadMetadataAsync` 只读头与目录，`UnpackAsync` 按 1 MiB 有界缓冲流式解出、逐条目校验 CRC、临时文件校验后原子落盘，支持条目筛选/上限/覆盖/进度/取消 |

- 注册：`UotanStudio.Firmware/Application/Formats/FirmwareFormatRegistry.cs`。
- 魔数：`UotanStudio.Firmware.MagicGen/corpus/Magdir/uotan-firmware` 新增 `0 string CP\\x01\\x07` 规则（strength +60）并重新生成 `CompiledMagicData.{g.cs,zst}`（顶层规则 4205 → 4206，blob 3,045,111 B → zstd 317,709 B）；`DetectDetailed` 报 `StrongMagic`。
- 宿主文件类型：`UotanStudio/Services/Firmware/ImageInputService.cs` 新增 `ImageFormatKind.Cpb`（标签 `cpb`，`ParseFormatLabel` 可回读），`ProbeFormat` 经已注册处理器 + `.cpb` 后缀双重识别；`ImageViewModel` 的解包模式扫描登记该类型。
- 界面：属性面板复用既有 `FormatDetector` + `ReadMetadataAsync` 路径展示 CPB 头字段；`.cpb` 归入固件包图标，导入与新建工程选取器加入 `*.cpb`；`FirmwareInfoViewModel.Metadata.cs` 增 `FirmwareInfo_CpbSummary`，并补 `FirmwareField_{CpbVersion,Model,HardwareVersion,RomFrom,Mismatch}` 中英双份 resx 与 `LocaleKeySets`。

## 3. 实测门禁

- `CpbTests` 18/18（头字段与镜像目录、掩码校验和、位级 CRC 等价、密钥常量、解包字节、负载损坏、头部校验不一致、截断、条目筛选、名称净化、取消、版本字节判定、魔数拒绝、过短文件、`DetectDetailed` 强魔数、混淆自逆）。
- 全 Firmware 命名空间 4378 总数 / 4376 通过 / 2 跳过（未设 `UOTAN_EROFS_CORPUS`）/ 0 失败；`UotanStudio.Tests.Services` 1799（1 跳过）、`UotanStudio.Tests.Architecture` 16，均 0 失败。
- 定向：`ImageInputServiceTests` 77、`LocaleDictionaryTests` 6、`FileMagicTests` 34、`CompiledMagicCoverageTests` 5、`FirmwareArchitectureGuardTests` 12、`FirmwareNamespaceFolderTests` 2。
- `restore -p:Configuration=Debug` + `build UotanStudio.slnx -c Debug`：0 错误 0 警告；`dotnet format` 的 `whitespace` 与 `style --verify-no-changes` 零报告。
- NativeAOT：`publish -c Release -r win-x64 --self-contained -p:PublishAot=true --artifacts-path %TEMP%\uotan-aot-cpb` 成功，产物 155,977,728 B；只有第三方既有 8×IL2104 + 4×IL3053（Avalonia.Controls.DataGrid、Common.Logging、Dock.Avalonia、Iciclecreek.Avalonia.Terminal、LTRData.Extensions、Semi.Avalonia.Dock、Ursa、Ursa.Themes.Semi、YamlDotNet），无 `UotanStudio.*` 裁剪/AOT 警告。

### 构建环境坑（可复现）

共享工作树里其他会话同时构建 Release 时，默认输出目录会以 `CS2012`（"无法在使用用户映射区域打开的文件上执行"）失败在 `UotanStudio.Hash`/`Net`/`Media`/`Desktop` 等无关项目的 obj 产物上，不是代码错误。用 `--artifacts-path` 隔离发布输出即可；发布后需按 `-p:Configuration=Debug` 重新还原再构建 Debug，否则 `Dock.Avalonia.Diagnostics` 的 `IncludeAssets` 条件按 `$(Configuration)` 求值为空，`#if DEBUG` 内的 `AttachDeveloperTools`/`AttachDockDebugOverlay` 会以 `CS1061` 失败。

## 4. 未验证边界

- 仓库内没有真实酷派 CPB 固件样本；解包正确性只由测试自建容器覆盖（`TestInfrastructure/CpbContainerBuilder.cs` 按参考 C 的布局独立构造，不是生产代码的往返），未与厂商工具产物做逐字节对照。
- 固件信息视图与属性面板未启动 GUI 目视验证。
- 未实现重打包，CPB 保持只读。

上述边界同时记录在仓库 `VERIFICATION.md` 的"酷派 CPB 解包实测"一节。

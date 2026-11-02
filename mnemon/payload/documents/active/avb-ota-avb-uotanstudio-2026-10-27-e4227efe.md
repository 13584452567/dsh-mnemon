---
id: "e4227efe-8c13-4001-a3a6-a01ec226c99d"
title: "AVB 全链签名/页脚/OTA 能力与宿主 AVB 工具实施（UotanStudio，2026-10-27）"
description: "AVB 密钥/页脚/校验/OTA 载荷修补的固件层新增 API（AvbDescriptorTable、AvbSigner 部分类、AvbImageVerifier、PayloadPatcher、PayloadSignatureVerifier）、宿主 AvbOtaService 与 Tools→AVB 工具接线、实测测试与 NativeAOT 数字、真实厂商 OTA 与 GUI 等未验证边界。"
status: "active"
created_at: "2026-10-26T23:37:04.667Z"
updated_at: "2026-10-26T23:37:04.667Z"
content_hash: "e546fbe7a0eee7f26f1a4afcb4069e99d2f611e3360d3213cdcfafc47f9fa7ac"
source_paths:
  - "UotanStudio.Firmware/BootSecurity/Avb/Signing/AvbDescriptorTable.cs"
  - "UotanStudio.Firmware/BootSecurity/Avb/Signing/AvbSigner.DescriptorTable.cs"
  - "UotanStudio.Firmware/BootSecurity/Avb/Signing/AvbSigner.Footer.cs"
  - "UotanStudio.Firmware/BootSecurity/Avb/Verification/AvbImageVerifier.cs"
  - "UotanStudio.Firmware/OtaDelta/Payload/PayloadPatcher.cs"
  - "UotanStudio.Firmware/OtaDelta/Payload/Verification/PayloadSignatureVerifier.cs"
  - "UotanStudio/Services/Firmware/AvbOtaService.cs"
  - "UotanStudio/ViewModels/Shell/AvbToolViewModel.cs"
  - "UotanStudio/Views/Shell/AvbToolWindow.axaml"
  - "UotanStudio.Delivery/Build/OtaZipPatchService.cs"
  - "UotanStudio/UiAbstractions/IWindowService.cs"
  - "UotanStudio/ViewModels/Shell/MenuViewModel.cs"
  - "UotanStudio.Tests/Services/AvbOtaServiceTests.cs"
  - "UotanStudio.Tests/Firmware/AvbImageVerifierTests.cs"
  - "UotanStudio.Tests/Firmware/AvbFooterRemovalTests.cs"
  - "UotanStudio.Tests/Firmware/AvbDescriptorTableTests.cs"
  - "UotanStudio.Tests/Firmware/PayloadPatcherPublicTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "a9104abb-0c6f-4e61-8a67-ce58559cfe85"
memory_body_ids:
  []
---

# AVB 全链能力与宿主 AVB 工具（实施记录）

## 目标与结果
在 UotanStudio 中打通 AVB 全链操作：密钥生成/导入、VBMeta 描述符表级签名、镜像尾部（footer）增删与校验、A/B OTA（`payload.bin` 或 OTA ZIP）分区替换后重签并重打包，并作为 Tools → AVB 工具接入宿主 UI。未提交任何改动；工作区中其他并行修改（EROFS 编解码、GPT 等）保持原样。

## 固件层新增 API（UotanStudio.Firmware）

| 文件 | 职责 | 关键约束 |
| --- | --- | --- |
| `BootSecurity/Avb/Signing/AvbDescriptorTable.cs` | 原样读取并按需替换 VBMeta 描述符表 | 未知描述符类型不丢弃；链式描述符（chain partition）不会被静默重定向 |
| `BootSecurity/Avb/Signing/AvbSigner.DescriptorTable.cs` | `SignVbmetaWithDescriptorTableAsync` | 支持保留属性、内核命令行、链式等各类描述符 |
| `BootSecurity/Avb/Signing/AvbSigner.Footer.cs` | 公开 `RemoveFooterAsync` | 返回去掉尾部记录后的原始镜像字节并报告大小，用于恢复未签镜像 |
| `BootSecurity/Avb/Verification/AvbImageVerifier.cs` | 一次完成尾部解析 + VBMeta 签名校验 + 内容/FEC 校验 | 同时支持 hash 与 hashtree 镜像 |
| `OtaDelta/Payload/PayloadPatcher.cs` | 公开分区替换 + 重建 Payload + 可选 RSA 签名 | 面向完整 `payload.bin`，构造后自洽 |
| `OtaDelta/Payload/Verification/PayloadSignatureVerifier.cs` | 独立校验 payload 的 metadata 与 payload 两个 RSA 签名槽 | 两个签名槽分别验证，不依赖外部 avbtool |

## 宿主层（UotanStudio）

- `Services/Firmware/AvbOtaService.cs`：接受原始 `payload.bin` 或 A/B OTA ZIP。
  - 重签所选分区 → 更新并重签顶层 `vbmeta` → 必要时 `disable-avb` → 重建 Payload；
  - ZIP 输入经 `OtaZipPatchService` 重新发布，支持载荷签名与整包签名；
  - `otaverify` 校验 ZIP 结构、载荷签名（证书可从 `META-INF/com/android/otacert` 回退取得）以及以 `vbmeta` 为根的 AVB 链。
- `UotanStudio.Delivery/Build/OtaZipPatchService.cs`：新增带 `IDeliverySigningKeyProvider` 的公开重载，用于整包签名（Delivery 层仍不引用宿主 UI 或 `KeyLibraryService`）。

## UI 接线

- `Views/Shell/AvbToolWindow.axaml` + `.axaml.cs` + `ViewModels/Shell/AvbToolViewModel.cs`，入口 Tools → AVB 工具，三页：
  - **密钥**：生成、导入、删除 AVB 密钥，导出 `avb_pkmd.bin`；
  - **页脚**：添加 hash/hashtree 尾部、校验镜像、移除尾部、查看详情；
  - **A/B OTA**：加载分区、选择分区、`disable-avb`、Payload/ZIP 密钥、修补与校验。
- 接线点：`HostWindowKind.AvbTool`、`UiAbstractions/IWindowService.cs` 与 `Views/UiWindowService.cs`、`ViewModels/Shell/MenuViewModel.cs`、`Views/Shell/MainWindow.axaml` 菜单项。
- 本地化：`Locale/Common.resx`、`Common.zh-CN.resx`、`LocaleKeySets.cs`、`Menu.resx`、`Menu.zh-CN.resx` 新增 98 个 `Avb_*` 键（中英成对）。

## 验证（实测）

- 全解决方案 Debug 构建：0 错误。
- 新增 AVB/OTA 定向测试 23/23 通过：`AvbDescriptorTableTests` 6、`AvbImageVerifierTests` 6、`AvbFooterRemovalTests` 4、`PayloadPatcherPublicTests` 4、`AvbOtaServiceTests` 3（原始载荷 / 带载荷签名的 OTA ZIP / 带整包签名与 otacert 回退）。
- `UotanStudio.Tests.Firmware`：4398 项 / 4396 通过 / 0 失败 / 2 跳过（缺 `UOTAN_EROFS_CORPUS`）。
- 全解决方案：7662 项，两次完整运行均 7655 通过 / 7 跳过 / 0 失败。并行工作区负载下复跑时各有 1 次无关时序/隔离失败（`ErofsRebuildFeatureReplayTests` 与 `UsbSerialCacheTests`），两类单独复跑分别 20/20、6/6；另有既有 `adb-sync` 拉取用例输出的 `gone!` 日志，未计入失败。
- `dotnet format` 的 whitespace 与 style 双门无差异。
- NativeAOT `win-x64` 自包含发布成功；仅出现既有第三方（Ursa/Dock/Avalonia/DataGrid/YamlDotNet/Common.Logging 与 `Makaretu.Dns.LinuxHelper`）IL2104/IL3053/IL2026 警告，没有指向新增 AVB 代码的裁剪警告。

## 未验证边界

- OTA 修补/校验只用本地构建的完整 Payload 与最小 OTA ZIP 验证；未在真实厂商 OTA、差分/source 操作、`super` 动态分区或真机启动链上接受。
- 顶层链式描述符保持原样且不能被重定向；选择链保护分区会明确失败而非静默改写。
- `otaverify` 校验载荷签名与 AVB 链，但不独立校验外层 ZIP 签名。
- AVB 工具窗口与原生文件选择器未做人工 GUI 验证。
- 未与 Android `avbtool` 做逐字节外部对照。

验证记录同时写入仓库根 `VERIFICATION.md`。相关并行工作区记录见文档 `6932403a-9982-4006-9bff-c84d127ccbdc`（其中提到本 AVB 服务曾处于未跟踪的并行修改状态）。

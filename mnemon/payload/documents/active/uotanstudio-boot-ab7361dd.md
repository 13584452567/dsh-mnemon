---
id: "ab7361dd-103d-4b30-a478-4f97ccae5e4c"
title: "UotanStudio Boot 修补模式解析与选项可见性规则（实测矩阵）"
description: "BootPatchWindow 的 Auto 模式解析（LKM/KPM/GKI）、Magisk 各版本复选框可见性实测矩阵、KMI/超级密钥/ramdisk 门控规则与验证证据。"
status: "active"
created_at: "2026-10-30T05:00:35.472Z"
updated_at: "2026-10-30T05:00:35.472Z"
content_hash: "baf3babd67ba92ebbad10fec64fdfa6900d3a945460027d0eb81782c2bda0b25"
source_paths:
  - "UotanStudio.Firmware/BootSecurity/Patcher/BootPatchCompatibility.cs"
  - "UotanStudio.Firmware/BootSecurity/Patcher/BootPatchService.cs"
  - "UotanStudio/Views/Shell/BootPatchWindow.axaml.cs"
  - "UotanStudio.Tests/Firmware/BootPatchModelsTests.cs"
  - "UotanStudio.Tests/Firmware/BootPatchModeTests.cs"
session_ids:
  - "dbb7b052-ae90-4fef-b2a8-f5ae902a9b75"
memory_body_ids:
  []
---

# Boot 修补模式解析与选项可见性规则（实测矩阵）

## 1. 范围

本文件记录 `BootPatchWindow` 的补丁**模式解析**（Auto → LKM/KPM/GKI）与**选项行可见性/开始按钮门控**规则，以及用 `reference/patch-pkg` 语料实测得到的复选框显示矩阵。在线版本下载与补丁包链路见文档 `6e1bc9bf-cdf8-477b-83dc-aab69e6bbd6d`，此处不重复。

## 2. 模式解析（`BootPatchCompatibility` / `BootPatchService.ResolvePatchMode`）

Auto 由包内容决定，而不是只看包类型：

- KernelSU 包内存在**已验证的模块负载** → `Lkm`。
- KernelSU 包只携带 `kpimg` → `Kpm`。
- `GkiKernel` 内核镜像或 AnyKernel3 包 → `Gki`。
- `KernelPatch` / `APatch` → KPM 路径（需要超级密钥）。

显式指定的模式在包不具备该能力时必须被拒绝（`InvalidDataException`），不能静默降级。

## 3. 选项可见性（按包内脚本驱动）

`BootPatchOption` → `Supports(...)` 是唯一来源，选项来自包内 `assets/boot_patch.sh`、`util_functions.sh`；`BootPatchAnalyzer` 逐包解析后喂给窗口。

| 包 | 检测到的选项 | 显示 |
| --- | --- | --- |
| Magisk v19.0–v21.4（ZIP 旧布局） | 无（`UnsupportedReason`） | 不显示，禁止开始 |
| Magisk v22.0–v23.0 | KeepVerity、KeepForceEncryption、RecoveryMode | 3 个 |
| Magisk v24.0–v25.2 | +`PATCHVBMETAFLAG` | 4 个 |
| Magisk v26.0–v26.1 | +`PREINITDEVICE` | 4 个 + 预启动设备框 |
| Magisk v26.2–v31.1 | +`LEGACYSAR` | 5 个全显示 |
| Magisk Alpha（28104…31100） | 六项全有 | 5 个全显示 |
| KernelSU v3.2.2–v3.3.1（APK） | 无 `boot_patch.sh`，`libksud` 内嵌模块 | 不进 Magisk 选项行，走 LKM + KMI 列表 |
| SukiSU v3.1.1–v3.1.9（仅管理器） | Unknown | 不支持（manager-only） |
| SukiSU v3.2.0 / v4.2.1-r1…v4.3.0 | 仅 kpimg | 走 KPM，需超级密钥 |
| APatch 10413–11282 | kpimg + 脚本 | 走 KPM/APatch，需超级密钥 |

结论：语料中没有"选项被检测到但不该显示"的反例；`PREINITDEVICE` 原先只是被检测出来、从不参与可见性，导致 Magisk < v26 错误显示预启动设备框 —— 已改为按脚本是否含该选项决定显示。

## 4. 必填项与门控

- **KMI/GKI 下拉**：KernelSU `Lkm` 显示模块 KMI 列表；GKI 包显示其发布的 GKI 版本列表并**必须选择**（未选则开始按钮禁用并提示 `BootPatch_GkiVersionRequired`）。GKI 显式选择与 boot 实际 KMI 不一致时由 `BootPatchCompatibility.Check` 拒绝。
- **超级密钥**：APatch 要求 8–63 位且同时含字母与数字；KernelPatch/KPM 要求非空。开始按钮用与修补器同一个 `IsSuperKeyAcceptable` 判定，避免点击后才在修补器里报错。
- **ramdisk/内核要求矩阵**：GKI、APatch、KernelPatch 与 KernelSU 内核类要求内核；Magisk 与 KernelSU-LKM 要求 ramdisk。对应空 ramdisk 的拒绝分支必须与模式联判，否则 GKI 路径会被误拦。
- 读取失败（boot 可解析但内核架构读不出、GKI 包版本读不出）按输入路径去重各弹一次警告，不刷屏。

## 5. 测试与遗留

- 语料（`UOTAN_PATCH_PKG_CORPUS=reference/patch-pkg`）：`BootPatchModelsTests` 68、`BootPatchModeTests` 11、`RootPatchPackageCorpusTests` 3、`RootPatchPackageTests` 39、`LocaleDictionaryTests` 6、`FirmwareArchitectureGuardTests` 12、`HostLayeringGuardTests` 4 —— 143 通过 / 0 失败；全量 Debug 构建 0 错误 0 警告；`dotnet format`（whitespace + style）无差异；NativeAOT Release 发布成功，裁剪/AOT 警告均来自第三方程序集。
- 既有用例 `PatchAsync_KernelSuPackageWithKpimg_...` 原按 Auto 期望 KPM，在"有模块负载走 LKM"规则下必然失败；已改为显式 `Kpm` 以保留原覆盖意图，而不是放宽解析规则。
- 未验证：真实窗口交互（选项随版本切换、GKI 必选、警告时机）无自动化，仓库不对 Avalonia 窗口做断言；真机/真镜像修补未执行。

## 6. 关键文件

- `UotanStudio.Firmware/BootSecurity/Patcher/BootPatchCompatibility.cs`（GKI 一致性、`IsSuperKeyAcceptable`）
- `UotanStudio.Firmware/BootSecurity/Patcher/BootPatchService.cs`（`ResolvePatchMode`、ramdisk/内核要求、包内条目读取）
- `UotanStudio/Views/Shell/BootPatchWindow.axaml.cs`（`UpdatePatchModeState`、`RequiresKmiChoice`、`UpdateStartEnabled`）
- `UotanStudio/Locale/Common.resx`、`Common.zh-CN.resx`、`Locale/LocaleKeySets.cs`（三处同步）
- `UotanStudio.Tests/Firmware/BootPatchModelsTests.cs`、`BootPatchModeTests.cs`

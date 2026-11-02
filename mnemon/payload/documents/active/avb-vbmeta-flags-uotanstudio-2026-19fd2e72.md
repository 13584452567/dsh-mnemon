---
id: "19fd2e72-2815-4c5f-a4ba-1ec06dc3ebea"
title: "AVB VBMeta flags 切换与固件信息摘要移除（UotanStudio，2026）"
description: "fastboot --disable-verity/--disable-verification 的位语义与字节级证据、AvbSigner.SetVbmetaFlagAsync 与既有 DisableVbmetaVerificationAsync 的差异、FirmwareInfo 视图按钮接线与 AVB 摘要移除理由、实测测试数字与未验证边界。"
status: "active"
created_at: "2026-10-29T03:58:20.647Z"
updated_at: "2026-10-29T03:58:20.647Z"
content_hash: "0fed1702ab2a77063a08c81910c475a966c26e0d3f033003475244bf7c7bb5ec"
source_paths:
  - "UotanStudio.Firmware/BootSecurity/Avb/Signing/AvbVbmetaFlags.cs"
  - "UotanStudio.Firmware/BootSecurity/Avb/AvbMetadataReader.cs"
  - "UotanStudio/ViewModels/Editors/FirmwareInfoViewModel.Metadata.cs"
  - "UotanStudio/ViewModels/Editors/FirmwareInfoViewModel.cs"
  - "UotanStudio/ViewModels/Editors/FileViewModel.cs"
  - "UotanStudio/Views/Editors/FirmwareInfoView.axaml"
  - "UotanStudio.Tests/Firmware/AvbVbmetaFlagTests.cs"
session_ids:
  - "fd5180fd-301a-4771-bc43-b2bd2cd8602d"
memory_body_ids:
  []
---

# AVB 标志位切换与固件信息摘要移除（UotanStudio 实施记录）

## 目标与结果

在 UotanStudio 固件信息视图里为 AVB 镜像提供与 fastboot 等价的 VBMeta flags 切换能力（`--disable-verity` / `--disable-verification`），并移除 AVB 文件的摘要卡片（标题为「显示文件声明的签名、哈希、回滚保护和描述符配置」的那段说明文字及其显示区域）。未提交改动。

相关前序工作见 [AVB 全链签名/页脚/OTA 能力文档](documents/active/avb-ota-avb-uotanstudio-2026-10-27-e4227efe.md)（id `e4227efe-8c13-4001-a3a6-a01ec226c99d`）。

## fastboot 语义证据（本地参考树 reference/fastboot）

`reference/` 在 `.gitignore` 内，源码只是本地参考树：从 GitHub AOSP 镜像 `aosp-mirror/platform_system_core` 稀疏拉取 `system/core/fastboot`（59 个文件、1.1 MB、含 LICENSE）。直接连接 `android.googlesource.com` 与 `git clone github.com` 均失败，最终通过镜像仓库取得。

- 选项注册/解析：`--disable-verification` / `--disable-verity` → `g_disable_verification` / `g_disable_verity`（`fastboot.cpp:2251-2252`、`2290-2293`）。
- 核心函数 `rewrite_vbmeta_buffer`（`fastboot.cpp:1110-1160`）：
  - 缓冲区不足 256 字节直接返回（`:1113`）；vbmeta 分区要求偏移 0 为 `AVB0`；boot 镜像用尾部 `AVBf` 中的 `vbmeta_offset` 定位（`:1124-1136`）。
  - **flags 位于 VBMeta 头偏移 120，32 位大端；bit0 = disable-verity，bit1 = disable-verification**（`:1140-1146`）。
  - 实际写入只有 `data[123 + vbmeta_offset] |= 0x01 / 0x02`（`:1147-1152`），即大端 flags 的最低字节。
  - **不重新签名、不清零认证块、不改其它任何字节**；补丁只在内存/临时 fd 中随刷写下发，不落盘改写源镜像。
- 生效条件：`fastboot flash vbmeta ...` 走 `is_vbmeta_partition`（`:2522`、`:2108`）；仅当设备无 vbmeta 分区时才回退修改 boot 尾部内嵌 vbmeta（`:1276-1285`、`:1162-1173`）。

## 与视图两行元数据的一致性

| 视图行 | 元数据字段 | 读取处 | fastboot 参数 | 位 |
| --- | --- | --- | --- | --- |
| 已禁用哈希树校验 | `HashtreeDisabled` = `(Flags & 1) != 0` | `UotanStudio.Firmware/BootSecurity/Avb/AvbMetadataReader.cs:72` | `--disable-verity` | bit0 (0x1) |
| 已禁用 AVB 验证 | `VerificationDisabled` = `(Flags & 2) != 0` | `UotanStudio.Firmware/BootSecurity/Avb/AvbMetadataReader.cs:73` | `--disable-verification` | bit1 (0x2) |

## 关键设计判断

- fastboot 只做 `|=`（置位），不产生清位路径，且补丁不落盘；本实现做成可置位也可清位的原地切换，落点仍是同一个大端 flags 字段，按字节比对与 fastboot 等价。定位规则：头部 `AVB0`，否则尾部 `AVBf` 声明的 vbmeta 偏移。
- **没有复用**已有的 `AvbSigner.DisableVbmetaVerificationAsync`：它同时置两位并清零认证块，与 fastboot 行为不等价。
- 修改 flags 后 vbmeta 自身 hash/签名必然失配，libavb 只在 `VERIFICATION_DISABLED` 置位时跳过签名校验，所以单独 `--disable-verity` 对已签名镜像通常不可用；官方用法是两个参数一起给。
- 摘要移除原因：AVB 元数据只是文件声明的配置，读取配置不代表签名可信、分区内容已通过验证，AVB 也不代表镜像加密，口径上不适合作为「摘要」呈现。实现上用 `HasSummary` 控制摘要卡片可见性，AVB 格式返回空摘要。

## 改动文件

- 新增 `UotanStudio.Firmware/BootSecurity/Avb/Signing/AvbVbmetaFlags.cs`：`AvbVbmetaFlag` 枚举 + `AvbSigner.SetVbmetaFlagAsync`，仅原地改 VBMeta 头 4 字节 flags（写入 + `Flush(true)` + 回读校验），不改哈希/签名/描述符；支持独立 `AVB0` 镜像与带 `AVBf` 尾部的分区镜像，非 AVB 输入返回失败且不写盘。
- `UotanStudio/ViewModels/Editors/FirmwareInfoViewModel.Metadata.cs`：给两行挂 `ToggleHashtreeDisabledCommand` / `ToggleVerificationDisabledCommand`，成功后 `LoadAsync` 重新读取元数据，失败走错误横幅与日志；`FlagToggleInProgress` 防重入。
- `UotanStudio/ViewModels/Editors/FileViewModel.cs`：`PropertyRow` 增加可选 `ActionCommand` / `ActionLabel` / `HasAction`。
- `UotanStudio/Views/Editors/FirmwareInfoView.axaml`：行内值右侧渲染「切换标志位」按钮，摘要卡片 `IsVisible` 改为绑定 `HasSummary`。
- `UotanStudio/ViewModels/Editors/FirmwareInfoViewModel.cs`：新增 `HasSummary`，重置路径清空。
- 本地化：删除 `FirmwareInfo_AvbSummary`（两个 resx + `LocaleKeySets`），新增 `FirmwareField_ToggleFlag`；工作区同时存在并行未提交改动（`Flash_*` 键），本任务只做自己那一处替换。
- 新增 `UotanStudio.Tests/Firmware/AvbVbmetaFlagTests.cs`。

## 实测验证

- `dotnet build UotanStudio.Tests/UotanStudio.Tests.csproj --no-restore --nologo -c Debug`：0 警告 0 错误（含 XAML 编译绑定）。
- 新增测试 4/4：独立 vbmeta 置位/清位、尾部内嵌 vbmeta、两位同置后第 123 字节为 `0x03`、非 AVB 镜像拒绝且不写盘。
- `--filter-namespace UotanStudio.Tests.Firmware`：4412 总计，4410 通过，2 跳过（`UOTAN_EROFS_CORPUS` 未设置），0 失败。
- 定向：`LocaleDictionaryTests` 6/6、`AvbSignatureVerificationTests` 15/15、`FirmwareNamespaceFolderTests` 2/2、`DocumentOpenRegistryTests` 24/24。
- `dotnet format` 的 whitespace 与 style 校验均无差异。

## 未验证边界

- 没有自动化覆盖，也未人工确认按钮点击后的 XAML 渲染与布局（需开一次 AVB 镜像目视确认）。
- 未刷入真实设备验证引导结果。
- 未跑全量 slnx 构建/测试/NativeAOT 发布（工作区存在其他并行未提交改动）。

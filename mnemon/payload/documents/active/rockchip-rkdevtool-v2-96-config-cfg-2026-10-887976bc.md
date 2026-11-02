---
id: "887976bc-7e87-4a39-9f1e-53ce4d9fa407"
title: "Rockchip 烧录页接线与 RKDevTool v2.96 config.cfg 结构（2026-10）"
description: "RKDevTool v2.96 控件/执行链还原、config.cfg 二进制结构（29B 头 + 610B 行）、UotanStudio 烧录页瑞芯微接线（cfg 服务、ViewModel 分部、XAML、本地化、测试）与已验证/未接线边界。"
status: "active"
created_at: "2026-10-28T12:55:13.941Z"
updated_at: "2026-10-28T12:55:13.941Z"
content_hash: "1c364f92f1f00366822ae916e8d1bb4e51619d23b1a033be2df000912aa481f8"
source_paths:
  - "UotanStudio/Services/Flash/RockchipConfigFile.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs"
  - "UotanStudio/Views/Shell/FlashView.axaml"
  - "UotanStudio/Views/Shell/FlashView.axaml.cs"
  - "UotanStudio/Locale/LocaleKeySets.cs"
  - "UotanStudio.Tests/Services/RockchipConfigFileTests.cs"
session_ids:
  - "bbb6d4c5-e078-4987-bc03-0ca79497f44d"
memory_body_ids:
  []
---

# Rockchip 烧录页接线与 RKDevTool v2.96 config.cfg 结构（2026-10）

## 背景与目标
分析 `D:\EXE Installer\RKDevTool_Release_v2.96` 的功能与实现逻辑、目录内 `cfg` 文件结构，并把对应能力接到 UotanStudio 烧录页（顶部通用按钮 + 瑞芯微二级菜单）。本机无外网（`web_fetch` 报 `fetch failed`、`web_search` 插件缺 CLI），在线开源实现按已知上游对照而非实抓。

## RKDevTool v2.96 界面事实（自 `Language\Chinese.ini` 控件 ID + `revision.txt` 还原）
- 三个标签页：`下载镜像` / `升级固件` / `高级功能`（`TABCTRL_1000_Tab_0..2`）。
- 顶部通用按钮：`固件`(1011)、`升级`(1013)、`设备分区表`(1052)、`加载分区表`(1008)、`清空`(1009)、`执行`(1006)、`切换`(1007)；另有固件/Loader 版本、芯片信息、`Boot:` 只读标签。
- 高级功能（二级菜单）：`擦除Flash`(1018)、`擦除所有`(1034)、`擦除扇区`(1054)、`读取FlashID`(1028)、`读取Flash信息`(1029)、`读取Chip信息`(1045)、`读取Capability`(1051)、`切换存储`(1057)、`获取当前存储`(1058)、`进入Maskrom`(1050)、`重启设备`(1031)、`测试设备`(1032)、`下载`(1027)、`LBA`(1044)、`强制按地址写`(1062)、`导出镜像`(1033)、`导出串口日志`(1055)、`清空序列号`(1053)、`检测安全模式`(1056)。
- 下载列表右键菜单：添加项/删除项/清空所有项/上移/下移/导入配置/导出配置；列：`地址/名字/路径/存储`。
- 执行链：`下载Boot → 下载IDB → 下载固件 → 校验`（`IDS_DOWNLOADBOOT_*`、`IDS_DOWNLOADIDBLOCK_*`、`INFO_WRITE_PARAM_PROGRESS`、`ERROR_BOOT_BEFORE_LBA`）；按 parameter 表把勾选项写到对应 LBA。`ERROR_PARTSIZE_UNKNOWN` ⇒ 分区大小必须来自文件或设备读回的 parameter；`ERROR_STORAGE_OP_SUPPORT`/`ERROR_SWITCH_NOT_SUPPORT` ⇒ 存储切换要能力位支持。
- 版本沿革（`revision.txt`）：v2.86 起支持按地址写，v2.87 起支持写入不同存储，v2.90 起支持自定义 VID/PID。

## config.cfg 逆向结构
`config.cfg`、`rock-5b-emmc.cfg`、`rock-5b-spinor.cfg` 样本均为 1249 字节 / 2 行，结构一致：

| 区段 | 偏移/长度 | 内容 |
| --- | --- | --- |
| 头 | `0x00`，29 B | `"CFG\0"` + `u16` 头长 `0x1D`(29) + 8 个 `u16` 日期时间样式块 |
| 行 | 每行 610 B | `u16` 标记 `0x0262`、名称 UTF-16 定长 80 B、路径 UTF-16 定长 512 B、尾块 16 B |

- 行数由文件长度决定：`(len - 29) / 610`；样本两行为 `Loader` 与 `Image`。
- 尾块后 8 字节按行地址解释、末 4 字节观测值恒为 1；日期块字段语义无法从样本唯一确定，实现中**原样保留**而非猜测。
- 解码约定：未知/未解释字节回写时逐字节保留，写入走同目录临时文件 + 原子替换。

## 落地改动
- 新增 `UotanStudio/Services/Flash/RockchipConfigFile.cs`：按上表解析/写回 cfg，损坏魔数与截断文件拒绝。
- 新增 `UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs`：读取芯片信息（Capability/FlashID/存储）、测试设备、获取/切换存储、整块擦除（带确认）、LBA 读/写/擦除扇区（支持 `0x` 十六进制与「强制按地址写」）、导入/导出二进制 cfg（替换原先导出文本的占位实现）。设备动作统一由 `UpdateRockchipActionState` 随协议/忙闲刷新。
- 顶部通用按钮接线：`读取信息` 增加 RKDL 分支（原先只处理 MTK/Fastboot）；`烧录选中` 把自定义 cfg 的 `Loader` 传给 `FlashRkdlAsync` 的 loader 过渡参数（原为 `null`），Maskrom 设备可先进 loader 再写入。
- 视图与本地化：`UotanStudio/Views/Shell/FlashView.axaml` 瑞芯微分类新增上述控件（`RkExpander` 内 `RkStorageTypeBox/RkGetStorageButton/RkSwitchStorageButton/RkEnterMaskromButton/RkReadInfoButton/RkTestDeviceButton/RkEraseFlashButton/RkLbaAddressBox/RkLbaCountBox/RkForceAddressBox/RkLbaReadButton/RkLbaWriteButton/RkEraseSectorsButton/RkConfigPickBox/RkConfigPathBox/RkChooseConfigButton/RkImportConfigButton/RkExportConfigButton`），`FlashView.axaml.cs` 增加事件转发，`Common.resx`/`Common.zh-CN.resx`/`Locale/LocaleKeySets.cs` 同步 7 个中英 key。
- 测试：新增 `UotanStudio.Tests/Services/RockchipConfigFileTests.cs`（往返逐字节一致、尾块保留、坏魔数/截断拒绝）。

## 上游对照
`rockchip-linux/rkdeveloptool`（同一 RKUSB 协议的 CLI：`db/ld/rd/wl/el/td/ef/rfi` 与本清单命令一一对应）、`linuxerwang/xrock`（Python，含 maskrom/loader 切换与 LBA 读写）、`linux-rockchip/rkflashtool`。仓库内 `UotanStudio.Comm/Protocols/Bootrom/` 已具备 `ReadLBA/WriteLBA/EraseLbaRange/EraseAllFlash/ChangeStorage/ReadStorage/ReadCapability/TestDeviceAsync` —— 协议事实与实测见文档 `5cf3d40c-3504-4b41-a3a5-9157978b6c62`。

## 验证与未验证边界
- `dotnet build UotanStudio.slnx --no-restore -c Debug`：0 警告 0 错误。
- `RockchipConfigFileTests` 5/5 通过，`LocaleDictionaryTests` 6/6 通过；`dotnet format` whitespace 与 style `--verify-no-changes` 对改动文件干净（新文件初版触发 `FINALNEWLINE` 与 `IMPORTS` 需按 `.editorconfig` 修正：末尾不留换行、导入排序）。
- 未验证：无真机，Maskrom 与 loader 两阶段的存储切换、整块擦除、LBA 读写的灰机 USB 行为未实测；界面未人工点检。
- 明确未接线（缺少依赖，未臆造实现）：`update.img` 一键整包升级、读取 IDB、清空序列号、检测安全模式、导出串口日志（分别依赖固件整包流程、Secure Boot 证据、串口日志源）。

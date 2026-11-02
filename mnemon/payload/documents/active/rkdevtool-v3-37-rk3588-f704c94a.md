---
id: "f704c94a-fdd2-4686-b99d-8c228311d606"
title: "RKDevTool v3.37 芯片信息展示逻辑与 RK3588 芯片名映射修复"
description: "反编译 RKDevTool v3.37「读取 Chip 信息」执行链（sub_4197E0 只回显 Chip Tag、老家族 switch 表无 35xx）、UotanStudio GetChipName 的 8853→RK3588 修复与契约、门禁实测与未验证边界。"
status: "active"
created_at: "2026-11-01T06:31:00.311Z"
updated_at: "2026-11-01T06:31:00.311Z"
content_hash: "efb3e7639f83792eb9c5af7114cf28a4b1f50a1fac2868d67133b3f24743e806"
source_paths:
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlConstants.cs"
  - "UotanStudio.Tests/Protocols/RkdlModeClassificationTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "7e7f4fb1-561e-4f4b-b09a-77e72a3b5373"
memory_body_ids:
  []
---

# RKDevTool v3.37 芯片信息展示逻辑与 RK3588 芯片名映射修复

## 背景

UotanStudio 烧录页对 RK3588 设备显示的不是型号，而是按线上字节序原样回显的倒序标签（`8853`）。为确认厂商工具到底怎么显示，反编译 `RKDevTool.exe` v3.37（IDA i64，完整 PE 已载入）定位「读取 Chip 信息」执行链，再按结论修正本仓库的显示源。

## RKDevTool v3.37 执行链（IDA 证据）

- 命令分发：消息映射处 `0x67fdbc` 的条目把命令 `0x48b` 指向 `sub_41E100`；该处理函数复位界面后 `AfxBeginThread(sub_405CD0, this, ...)`，线程体 `sub_405CD0` 直接调用 `sub_4197E0`（返回 `unsigned int`/`CWnd*`，符合 `AfxBeginThread` 的线程函数要求）。
- `sub_4197E0` 流程：
  1. 建立 COMM 对象（失败弹 `ERROR_CREATE_COMM`），打印资源串 `IDS_GETCHIPINFO_START`。
  2. 经虚表 `vtbl+4` 调用 `RKU_ReadChipInfo`，输出首 DWORD 即 tag。
  3. 只按原始字节打印 `Chip Tag:\t%02X %02X %02X %02X`（字符串在 `0x67abdc`，UTF-16 原始字节已核对：`C h i p   T a g : \t % 0 2 X ...`），不做任何型号翻译。
  4. `switch(tag)` 只识别老一代四字符标签（RK27、A372/cayman、RK28、RK281X、RKPANDA、RK29、RK292X、RK30、RK30B、RK31、RK32、RKSMART、NANO、RKCROWN），每支各往输出追加一个家族后缀 `-RK27`…`-RKCROWN`。**这张表没有 35xx/3588。**
  5. `default` 分支把四个线上字节按原序拼成 `"Ch" + b0 + "K" + b1 + b2 + b3` 的宽字符串，仅用于 `Image Chip Flag:\t%s` —— 这是映像比对串，不是给用户看的型号。
  6. 成功打印 `IDS_GETCHIPINFO_PASS`；失败写日志 `Error:ReadChipInfoProc-->RKU_ReadChipInfo failed,err=%d` 并打印 `IDS_GETCHIPINFO_FAIL`。
- 同批按钮同样不显示型号：`sub_419ED0`（Read Capability）打印 `Capability:\t%s` 加 13 行 `DirectLBA/VendorStorage/…: Enable|Disable`（含 `SwitchStorage`、`LBAparity`、`SwitchUsb3`）；`sub_41A6E0`（Read Flash Info）打印 `Flash Size:\t%dMB`、`Total Sector`、`LBA Size`、`Block Size`、`Page Size`、`ECC Bits`、`Access Time`、`Flash CS`。

**结论**：RKDevTool 从不把芯片标签翻译成型号显示，型号只能由调用方按 ID 自行映射。因此 UotanStudio 的唯一显示源仍是 `RkChipInfo.ChipName`，不能靠"照抄厂商工具"解决问题。

## 根因与修复

- 根因：`RkdlConstants.GetChipName` 表中 `0x350A` 映射为 RK3566；RK3588 的芯片编号按字节倒序上报（RKBOOT/RKFW 头部同一顺序），标签字节读作 `8853`，落进 `TryReadPrintableTag` 兜底，于是把倒序标签当型号显示。
- 修复：`GetChipName` 增加 `0x38383533`（标签 `8853`）→ `RK3588`，与既有 `0x33353838` 并列，两种线上字节序都解出型号。
- 契约写清：`ChipName` 是显示名，`Tag` 是线上字节序；显示型号只走 `ChipName`。
- 宿主侧信息行（`ReadRkDeviceInformation`）依次输出 capability 原值 + `Describe()` 各行 + `ChipName` + FlashID，存储选择框经官方位表映射，均不自行解释标签字节序。

## 落地改动与门禁实测

- 源文件：`UotanStudio.Comm/Protocols/Bootrom/RkdlConstants.cs`（`GetChipName` 映射与 `ChipName`/`Tag` 契约注释）。
- 测试：`UotanStudio.Tests/Protocols/RkdlModeClassificationTests.cs` 新增 `0x33353838`→RK3588、`0x38383533`→RK3588 两条。
- 记录：`VERIFICATION.md` 顶部新增本轮实测条目（含 IDA 地址与结论）。
- 门禁（Windows x64，Debug）：`dotnet build UotanStudio.Desktop/... -c Debug` 0 警告 0 错误；`RkdlModeClassificationTests` 41 通过 / 0 失败；触碰文件的 `dotnet format whitespace|style --verify-no-changes` 零报告。

## 未验证边界与待决

- 真实 RK3588 设备上的列表行与「读取芯片信息」输出本轮未复测（需接板）。
- RK3588 与 RK3588S 无法用标签区分（同一芯片 ID 只报 `8853`/`3588`），当前统一显示 "RK3588"，是否分列待定。
- `UotanStudio.Tests` 存在既存两条 `CS8604`（`DeviceControllerRockchipReconnectionTests.cs:150/159`），本轮未改动该文件。

## 相关文档

- `0141cf6a-e8ce-44ca-a858-54a6c8c72dbb`：RKDL 会话对齐 rkdeveloptool，能力位/芯片/闪存信息读取收敛在会话层。
- `887976bc-7e87-4a39-9f1e-53ce4d9fa407`：RKDevTool v2.96 控件/执行链与 config.cfg 结构、烧录页接线。

---
id: "5cf3d40c-3504-4b41-a3a5-9157978b6c62"
title: "RKDL（rockusb）后端补齐：flash 几何解码、528B 扇区写与擦除语义对齐 rkdeveloptool（2026-10）"
description: "以上游 rkdeveloptool/rkdevtool 只读源码为准补齐 RKDL 协议后端：FLASHINFO/CAPABILITY 完整解码、WriteSector 528B/32 扇区上限、EraseBlocks 坏块容忍、EMMC ERASE_LBA 与 ERASE_FORCE 整盘擦除；含实测门禁数字、NativeAOT sourcelink 文件锁阻塞与未验证边界。"
status: "active"
created_at: "2026-10-26T20:59:28.314Z"
updated_at: "2026-10-26T20:59:28.314Z"
content_hash: "0887a3d07d2a0ccf8361306f005f98051589d0fb4992712f92d640c1af5a47ad"
source_paths:
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlSession.Erase.cs"
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlSession.Commands.cs"
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlConstants.cs"
  - "UotanStudio.Tests/Protocols/RkdlFlashGeometryDecodeTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "dbdffcbe-3a02-44f2-8271-f7ce44a3a91d"
memory_body_ids:
  []
---

# RKDL（rockusb）后端补齐：以上游源码为准的 flash 几何/扇区写/擦除语义（2026-10）

## 起点
`RkdlSession.ReadFlashInfo` / `ReadCapability` 的旧注释明确写着「本仓库没有可核对的 `RKComm.h` 副本，因此只解首个 DWORD / 不解码能力位」。本轮把上游只读源码拉到仓库，按源码把协议层空缺补齐，并把「上游声明但从未下发」的 opcode 显式记录为不属于功能集。

## 只读参考源码
| 目录 | 来源 | 提交 | 许可 |
| --- | --- | --- | --- |
| `reference/rkdeveloptool` | rockchip-linux/rkdeveloptool | `304f073`（2025-03-07） | GPL-2.0-or-later |
| `reference/rkdevtool` | hiifong/rkdevtool（Rust/Tauri 第二实现） | `34257529` | 该提交无 LICENSE 文件，仅作行为参考 |

两处均写 `UPSTREAM.md`，记录来源、许可与「本仓库取用了哪些协议事实」；`/reference` 已被 `.gitignore` 覆盖（第 458 行），不入库。

## 落地的协议事实（对照 rkdeveloptool）
- `RkFlashInfo.Decode` 完整解 `STRUCT_FLASHINFO_CMD`：`uiFlashSize`（512B 扇区数）、`usBlockSize`、`bPageSize`、`bECCBits`、`bAccessTime`、`bManufCode`（含 SAMSUNG…INTEL 名称表）、`bFlashCS` 位掩码；派生出块字节数、块数与在位芯片下标。
- 新增公开类型 `RkCapability`：bit0 = 直接 LBA、bit2 = 前 4M 访问，保留原始 8 字节。
- `WriteSector`（0x05）：528 字节物理扇区（512 + 16 spare），单命令 ≤ 32 扇区（上游 `ERR_CROSS_BORDER` 上限），用于原始 NAND 扇区写。
- `EraseBlocks` 容忍 CSW status 1（上游 `ERR_FOUND_BAD_BLOCK`）并返回坏块数，不再一律抛错。
- `TestDeviceReady(byte subCode, …)` 支持 `0xFE/0xFD/0xFB/0xF9` 子码，新增 `GetUserSector`。
- 新增 `UotanStudio.Comm/Protocols/Bootrom/RkdlSession.Erase.cs`：
  - `IsEmmcFlash()`：READ_FLASH_ID == "EMMC"（小端 `0x434D4D45`）；
  - `EraseLbaRange`：按 32768 扇区分块，与上游 `EraseEmmc` 一致；
  - `EraseAllFlash`：eMMC／直接 LBA 走 `ERASE_LBA`，否则按 `bFlashCS` 每芯片 `ERASE_FORCE`、16 块/命令，复刻 `CRKDevice::EraseAllBlocks`。
- 枚举补 `EraseForce = 0x0B`；常量补 528 / 32 / 32768 / 11 / 512 / EMMC ID（改动 `RkdlConstants.cs`、`RkdlSession.Commands.cs`）。

## 明确不实现的部分
上游声明但从未下发的 opcode：READ_SECTOR、TEST_BAD_BLOCK、SDRAM 读/写/执行、EFUSE、SPI flash、SET_RESET_FLAG。它们不属于 rockusb 功能集，只在 `UPSTREAM.md` 记录，不写进生产代码。

## 验证（实测数字）
- `dotnet build UotanStudio.slnx --no-restore -c Debug`：0 警告 / 0 错误。
- `dotnet test --filter-namespace UotanStudio.Tests.Protocols`：119/119 通过，含新增 `UotanStudio.Tests/Protocols/RkdlFlashGeometryDecodeTests.cs` 7 例（结构映射、短响应拒绝、零块几何、能力位、EMMC 魔数）。
- `dotnet test --filter-namespace UotanStudio.Tests.Devices`：321 条，320 通过 / 0 失败 / 1 环境跳过。
- `dotnet format … whitespace` 与 `style --verify-no-changes`：清洁。新增文件初版因仓库 `insert_final_newline = false` 触发 `FINALNEWLINE` 错误，需去掉文件末尾换行。

## 未验证边界
- **NativeAOT 发布门本轮未通过，但与本改动无关**：`ilc` 三次一致报 `The process cannot access the file '…\\win-x64\\native\\UotanStudio.Desktop.sourcelink' because it is being used by another process`；ILC 日志中 `Rkdl/Bootrom` 出现 0 次，清理 `obj/Release` 并终止残留 `ilc.exe` 后仍复现，判断为并行会话/残留进程持锁。上一轮基线（见 `VERIFICATION.md`）发布成功，产物约 147–155 MB。需在无并发发布时重跑。
- 真实硬件未执行：528 字节 `WRITE_SECTOR`、`ERASE_FORCE` 坏块容忍、eMMC 整盘 `ERASE_LBA` 的线上行为只有协议层与单测覆盖。
- tool 级工作流未移植：parameter/GPT 生成与 `ppt`、sparse 镜像写、`update.img`(RKFW) 解析仍属工具侧能力（Comm 侧已有 `RkdlGpt` 备份表/磁盘尺寸与 `WriteGPT`），不属于 rockusb 协议层。

## 相关工作
以只读 `reference/` 源码为准对齐协议细节的做法，与 ADB 侧 `8d80767c-a89a-4634-a807-3e98c962ed3e` 一致。

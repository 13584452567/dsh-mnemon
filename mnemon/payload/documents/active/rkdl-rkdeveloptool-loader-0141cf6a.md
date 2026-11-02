---
id: "0141cf6a-e8ce-44ca-a858-54a6c8c72dbb"
title: "RKDL 会话对齐 rkdeveloptool：loader 下载、稀疏回放与官方存储表"
description: "Rockchip RKDL 协议层补齐（RK28 ID block、厂商 RC4、LBA 64 loader 升级、稀疏回放）、端点重捕获修复、宿主发送引导与官方存储表显示、CRC-16/XMODEM，以及测试与未验证边界。"
status: "active"
created_at: "2026-11-01T03:29:40.034Z"
updated_at: "2026-11-01T03:29:40.034Z"
content_hash: "10336445a8c82744630eefedea662cd41db2f6d5b73fdfde6477e93d74ada48a"
source_paths:
  - "UotanStudio.Comm/Protocols/Bootrom/Rk28IdBlock.cs"
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlRc4.cs"
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlSession.LoaderUpgrade.cs"
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlSession.Sparse.cs"
  - "UotanStudio.Hash/Crc16Xmodem/Crc16Xmodem.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.cs"
  - "VERIFICATION.md"
session_ids:
  - "69a01d75-0a3c-4800-920e-551ac0a8b21a"
memory_body_ids:
  []
---

# RKDL 会话对齐 rkdeveloptool：loader 下载、稀疏回放与官方存储表（实现记录）

记录 UotanStudio 中 Rockchip RKDL（Bootrom/loader）链路的一次成组实现：协议层补齐 rkdeveloptool 的核心步骤，宿主补上下发引导与存储表显示，以及随之调整的 Flash 行语义。文件行勾选/校验细节见 `ac021ea4-1790-442d-9260-1e7ddf79ce40`；loader 两阶段读取与门控维度另见 `4cf0b689-be12-4ab6-b574-db9d331f500a`。

## 1. 协议层（UotanStudio.Comm/Protocols/Bootrom）

- 新增 `Rk28IdBlock.cs`：构造 RK28 ID block（loader 下载前置的标识块）。
- 新增 `RkdlRc4.cs`：厂商私有 RC4 变体，用于 loader 升级通道的加扰，与标准 RC4 分开实现以免误用。
- 新增 `RkdlSession.LoaderUpgrade.cs`：LBA 64 处的 loader 升级通道。
- 新增 `RkdlSession.Sparse.cs`：稀疏镜像回放（按 sparse 头逐块写）。
- `RkdlSession.Commands/Transfer/Erase/Loader.cs`、`RkdlConstants.cs`、`RkdlLoaderTransitionOffer.cs` 同步调整；命令超时与 reset 容忍、MSC 设备排除表、官方存储位表与能力位/芯片/闪存信息读取都收敛在会话层。
- 设计取向：所有设备读写留在 `UotanStudio.Comm`，宿主只经 `DeviceController`/会话门面消费；超时与 reset 容忍做成会话内策略，避免调用方各自重试。

## 2. 校验和（UotanStudio.Hash/Crc16Xmodem）

- 新增 `Crc16Xmodem`：多项式 0x1021、初值 0、无末异或、检查值 0x31C3；与既有 `Crc16CcittFalse` 并存，二者初值不同不可互换。

## 3. 连接与端点（修复「节点消失后丢端点」）

- `DeviceController.Reconnection.cs`、`UsbReconnectIdentityPolicy.cs`、`UsbProtocolClassifier.cs`、`RockusbDevice.cs`：在租约存活期内捕获旧集线器链并经 `StickyLocationChain` 传递，重枚举后仍能定位同一物理口。
- 把实例 ID 尾部 `ROCKCHIP` 作为第三证据参与判定；描述符缺 `bcdUSB` 时仍按 0x0471 处理；未确认的启动阶段拒绝存储命令并给出诊断，而不是盲发。
- 下载前用 TEST_UNIT_READY / READ_CHIP_INFO / READ_FLASH_INFO 预热，重枚举后等 loader 就绪；配置里声明但实际缺失的 loader 路径直接失败。

## 4. 宿主（UotanStudio 闪存页）

- 恢复 RKDL 完整发送并等待重枚举；新增「发送引导」动作，按 loader 阶段门控。
- 存储下拉改走官方位表，写入后回读校验，并显示芯片型号与几何信息（扇区/块/页）。
- 取消 `FlashPartitionEntry.CanSelect`：行是否可选按 `Command` 决定行为，避免「有文件才能选」与「特殊行」两套状态打架；loader 结论（缺失/有效/已检查）每次导入只报一次，避免重复日志。
- 设置页新增 GC 单行摘要与 .NET SDK 版本行；riscv64/loongarch64 回落默认 GC 配置。
- 中英 resx 与 `LocaleKeySets` 同步；补齐 Node 家族名并加回归测试。

## 5. 测试与门禁

- 新增测试：`Rk28IdBlockTests`、`RkdlRc4Tests`、`RkdlSparseWriteSourceTests`、`RkdlStorageTableTests`、`Crc16XmodemTests`、`DeviceControllerRockchipReconnectionTests`、`RkdlLoaderProbeE2ETests`；`FlashRockchipFileRowTests` 增加「只报一次」用例，`FlashGlobalLogRoutingTests`/`FlashRootPartitionTests` 随 `CanSelect` 移除调整。
- 该批次共 56 个文件、+3990/−624（含 12 个新增文件），提交按 Conventional Commit 分 6 组（协议对齐、CRC、端点重捕获、宿主发送+存储表、本地化、验证记录）；结论落在 `VERIFICATION.md`。
- 未验证边界如实保留：真实设备上的 loader 下载、稀疏写入与存储表回读只按既有门禁记录，未在本轮重跑 `dotnet build`/测试；硬件相关结论不能视为已通过。

## 6. 协作注意

- 同一工作树内存在并行会话时，`git commit`/合并会互相穿插（出现过本地提交被并行会话合并进 `origin/master`）。整理提交前先确认是否有人正在同一 worktree 操作，避免把他人改动并入本次提交；不要用 reset/rebase 掩盖。

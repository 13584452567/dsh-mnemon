---
id: "7eea27ea-914e-4b61-a137-7f2c1eaf6822"
title: "单容器多张 GPT 表：表来源选择、副本回写规则与 1 MiB 探测窗口（UotanStudio）"
description: "ptable.img 三份 GPT 副本（LBA 1 空表 / 257 46 分区 / 291 118 分区）导致分区编辑器读到 0 分区的根因、GptImageEditor 按分区数选表与只回写镜像/空副本的规则、DocumentFormatPolicy 探测窗口 64 KiB→1 MiB，以及实测分区数与测试数字。"
status: "active"
created_at: "2026-10-26T04:29:46.350Z"
updated_at: "2026-10-26T04:29:46.350Z"
content_hash: "b36a5a6ab966da48b7eff5bc2bc1d0f51a3c297845881e0de331400ba991312a"
source_paths:
  - "UotanStudio.Firmware/BlockImage/Gpt/GptImageEditor.cs"
  - "UotanStudio.Firmware/BlockImage/Gpt/GptTableCopy.cs"
  - "UotanStudio/ViewModels/Editors/PartitionEditorViewModel.cs"
  - "UotanStudio/Services/Workspace/Documents/DocumentFormatPolicy.cs"
session_ids:
  - "fea3836a-5879-4362-8d26-17479b3579cf"
memory_body_ids:
  []
---

# 单容器多张 GPT 表：分区表来源选择与探测窗口（UotanStudio）

## 症状
打开 `ptable.img`（165,888 B，真实文件，非合成夹具）后分区编辑器显示 0 个分区，状态栏容量也异常；文件被当成分区表镜像加载成功但内容为空。

## 诊断路径
按 haucet-tools 的容器检测顺序逐项排除：无 `HARMONY!`、无 HVB footer、无 `rot\0`（RVT）、无华为 secimg、无 NVE/OEMINFO/sparse/ZIP 签名 —— 它不是别的容器格式，而是**同一个容器里打包了 3 份 GPT 表**。

实测副本分布（扇区对齐扫描 `EFI PART`，签名出现在 0x200 / 0x20200 / 0x24600）：

| 头部 LBA | 声明槽位 | 已用分区 | 内容 |
| --- | --- | --- | --- |
| 1 | 128 | 0 | 保护性 MBR + 头 + 全零条目数组（CRC 仍合法） |
| 257 | 128 | 46 | boot_ctrl / bl2_a / fw_* / teeos_a … |
| 291 | 128 | 118 | persist @1024-13311 … userdata @79286271（容量 40,594,571,264 B） |

## 根因
`GptImageEditor` 旧实现「取首个 CRC 合法副本作为表来源」→ 命中 LBA 1 的合法空表 → 读出的分区数为 0。真实多表镜像在 EDL 合并镜像、storage head 转储、仅含分区表的镜像中都会出现，不是损坏。

参考实现 `reference/haucet-tools/crates/tools-common/src/formats/gpt.rs`：扫描所有扇区对齐的 `EFI PART`，每个头同时尝试「声明 LBA 偏移」与「紧随头部块」两种条目数组位置，按分区数取最优并保留所有非空表。

## 修改（实现要点）
- `UotanStudio.Firmware/BlockImage/Gpt/GptImageEditor.cs`
  - 条目数组解析按候选（声明 LBA / 头部块之后 / 有界 CRC 搜索）取分区最多者；
  - 跨副本取**分区最多**的一份作为表来源（全空镜像仍可用，保持原有空镜像行为）；
  - 容量取 `max(备用头声明的容量, 末分区结束)`，使 128 KiB 偏移的「纯表镜像」也能得到正确容量；
  - `Patch` 只回写**与上报表逐字节一致或仍为空**的副本，携带另一张有效表的副本原样保留（避免覆盖并存的独立表）；
  - 新增 `DescribeCopies(byte[])` 返回每份副本的头部 LBA / 槽位数 / 分区数。
- `UotanStudio.Firmware/BlockImage/Gpt/GptTableCopy.cs`（新增）：副本描述记录类型。
- `UotanStudio/ViewModels/Editors/PartitionEditorViewModel.cs`：状态栏追加副本清单，例如 `GPT: 118 partition(s), disk 40594571264 bytes, 3 table copies: LBA 1 (0), LBA 257 (46), LBA 291 (118)`。
- `UotanStudio/Services/Workspace/Documents/DocumentFormatPolicy.cs`：分区表内容探测窗口 `PartitionHeaderLength` 64 KiB → 1 MiB，让头部不在 LBA 1（如 0x20000）的 ptable / storage-head 镜像被路由到分区编辑器，而不是十六进制或固件面板。

## 验证
- `dotnet build UotanStudio.slnx --no-restore -c Debug`：0 错误。
- 真实文件断言：`copies: LBA 1(0), LBA 257(46), LBA 291(118)`；回读 118 个分区（`persist@1024-13311` … `userdata@79286271`），容量 40,594,571,264 B；保存后 46 分区表的条目数组字节不变，空副本被填成同一张表，回读仍为 118 条，空副本头部 CRC 校验通过。
- `--filter-namespace UotanStudio.Tests.Firmware`：4334 总计 / 0 失败 / 2 跳过。
- 定向：`DocumentFormatPolicyTests` 23、`PartitionEditorBandTests` 11（新增端到端 `LoadAsync_GptImageWithSeveralTables_…`）、`PartitionTableLayoutTests` 5、`MbrDetectionTests` 15、新增 `GptOffsetTableTests` 5，全部通过。
- `dotnet format whitespace` / `style --verify-no-changes`：clean。

## 未验证
- NativeAOT 发布门禁未跑。
- 真实界面手工双击未做（VM 层已覆盖 `LoadAsync` 路径）。

---
id: "b97660e0-b43a-4826-aa33-52b6fbc2039f"
title: "RKDevTool v3.37 四个读取按钮执行链与 Chip Tag→型号推导定案（更正旧结论）"
description: "反编译 RKDevTool v3.37 四个高级功能读取按钮（FlashID/FlashInfo/ChipInfo/Capability）的共同骨架、各自线上事务与回显换算，证明官方 -RK+倒序标签 的型号规则，并列出 RkdlSession 对齐点、不可还原边界与通用化待决项；更正 f704c94a 的 default 分支结论。"
status: "active"
created_at: "2026-11-01T06:39:08.621Z"
updated_at: "2026-11-01T06:39:08.621Z"
content_hash: "593c64d01a70b141ea24a3de8ec1928da62974c68e9b098acc7e0b8125db25fc"
source_paths:
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlConstants.cs"
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlSession.Commands.cs"
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlSession.Transfer.cs"
  - "reference/rkdeveloptool/RKComm.cpp"
  - "reference/rkdevtool/src-tauri/src/device_ops.rs"
session_ids:
  - "4056be17-3c7e-400f-905b-d0ee55145d96"
memory_body_ids:
  []
---

# RKDevTool v3.37 四个「读取」按钮执行链与 Chip Tag→型号推导定案

## 0. 本文定位

对 `RKDevTool.exe`（v3.37，IDA i64）高级功能页四个按钮——读取 FlashID / 读取 Flash 信息 / 读取 Chip 信息 / 读取 Capability——做完整执行链还原，并交叉核对 `reference/rkdeveloptool`（C++）与 `reference/rkdevtool`（Rust/Tauri）两份参考实现，最后产出 UotanStudio `RkdlSession` 侧的对齐结论。

**本文更正** `f704c94a-fdd2-4686-b99d-8c228311d606`（《RKDevTool v3.37 芯片信息展示逻辑与 RK3588 芯片名映射修复》）的核心结论：该文认为官方 default 分支拼的是 `"Ch"+b0+"K"+b1+b2+b3`、`Image Chip Flag` 只是映像比对串。重新核对 `[ebp+Src]` 处的 dword 常量后，实际是 `0x52002D`，按 wchar 拆为 `'-'`、`'R'`，紧跟常量 `'K'`，因此官方构造的是 **`-RK` + 四个标签字节的倒序**，其结果就是给用户看的型号串。该旧文仍为 active，本文作为其更正版并存；旧文的其它内容（按钮映射、消息表、门禁记录）仍然有效。

## 1. 四个按钮的共同骨架

窗口消息表位于 `0x67fc10–0x67fe80`，每条 24 字节：`msgid @ +0`、`control = 0x0E @ +0x10`、`handler @ +0x14`。

| 按钮 | WM_COMMAND | handler | 线程体 | worker |
| --- | --- | --- | --- | --- |
| 读取 FlashID | `0x484` | `sub_41E080` | `sub_405CB0` | `sub_41AE50` |
| 读取 Flash 信息 | `0x485` | `sub_41E0C0` | `sub_405CC0` | `sub_41A6E0` |
| 读取 Chip 信息 | `0x48B` | `sub_41E100` | `sub_405CD0` | `sub_4197E0` |
| 读取 Capability | `0x48E` | `sub_41E220` | `sub_405D30` | `sub_419ED0` |

handler 行为完全同构：清输入行 → `sub_41DBD0` 复位日志控件 → 写阶段码 `[esi+0xE4]`（FlashID=9、ChipInfo=0xA、Capability=0xB、FlashInfo=0x11）→ `AfxBeginThread(worker, this, 0,0,0,0)`。**点击只挂线程，UI 不阻塞**。worker 内：建 COMM 对象（失败弹 `ERROR_CREATE_COMM`）→ 打 `IDS_*_START` → 经 vtable 调 `RKU_*` → 结果拼行交 `sub_44D4E0(日志控件, 级别, 保留, 文本)` → 打 `IDS_*_PASS` 或错误行。

## 2. 各命令的线上事务与回显

### READ_CHIP_INFO（vtable+4，worker `sub_4197E0`）
- CBW `READ_CHIP_INFO`，`dwCBWTransferLength=16`，读 16 字节。
- 打印 `Chip Tag:\t%02X %02X %02X %02X`：前 4 字节逐字节十六进制，**线上顺序**（RK3588 为 `38 38 35 33`）。
- 命中 14 项老表（`72KR/A372/82KR/X182/B282/X092/X292/A003/A013/B013/A023/C262/nano/CRON`）追加 `-RK27…-RKCROWN`；否则走 default 分支现拼型号串。
- 型号行是 `Image Chip Flag:\t%s`（格式串 `0x67acfc`），`0x419c37` 处格式化。
- 第 4..7 字节（Revision）官方不打印。

### READ_CAPABILITY（vtable+8，worker `sub_419ED0`）
- 8 字节，官方用 `RKU_Read_EX((BYTE*)&csw, sizeof(CSW))` 直接读进 CSW 缓冲的怪写法，再 `memcpy` 到输出（`reference/rkdeveloptool/RKComm.cpp:433`）。
- 打印 `Capability:\t%s` 原始位串 + 12 行 `Enable/Disable`：首字节位 0..6 = `DirectLBA/VendorStorage/First4M/ReadLBA/ReadComLog/ReadIDBConfig/ReadSecureMode`；次字节位 0..4 = `NewIDB/SwitchStorage/LBAparity/ReadOTPchip/SwitchUsb3`。
- 与本仓库 `RkCapability.Describe()` 的位表完全一致。

### READ_FLASH_INFO（vtable+0x14，worker `sub_41A6E0`）
- 请求 11 字节，实际最多读 512（`RKU_Read_EX`，`reference/rkdeveloptool/RKComm.cpp:388`）。
- 回显换算：`Flash Size:\t%dMB` = sectors ≫ 11；`Total Sector:\t0x%x`；`LBA Size(N):\t%d` = `byte[6] > 0xC8 ? byte[6]-0xC8 : 1`（否则打 `LBA Size:\t1`）；`Block Size:\t%dKB` = `word[4] ≫ 1`；`Page Size:\t%dKB` = `byte[6] ≫ 1`；`ECC Bits:` = `byte[7]`；`Access Time:` = `byte[8]`；`Flash CS:` 按 `byte[10]` 位图逐个 `%d `。
- 与 `RkFlashInfo.Decode`（`SectorCount/BlockSizeSectors/PageSizeSectors/EccBits/AccessTime/ManufacturerCode/FlashCs` + `RawResponse`）逐字段对齐。

### READ_FLASH_ID（vtable+0x10，worker `sub_41AE50`）
- **先发一次 READ_FLASH_INFO** 取 `byte[10]` 片选位图，再对每个置位 CS 各发一次 `READ_FLASH_ID`（5 字节）。
- 打印 `CS%d:\t%02X %02X %02X %02X %02X`；厂商/器件 ID 原样十六进制，**不解析成颗粒型号**。

## 3. Chip Tag → 型号：官方规则（已由二进制证明）

default 分支等价于：

```text
"-R" + "K" + tag[3] + tag[2] + tag[1] + tag[0]
```

即 **`-RK` + 四个标签字节的倒序**。RK3588 标签 `38 38 35 33` → `-RK3588`。完整回显：

```text
Chip Tag:        38 38 35 33
Image Chip Flag: -RK3588
```

因此 `GetChipName` 中已有的 `0x38383533 → RK3588` 正是该规则的逆运算，与官方行为一致，不是启发式猜测。

**通用化候选（未实施）**：在 `GetChipName` 默认分支前加一条「任意四位数字标签 → `RK` + 倒序」判定，比逐颗加表更贴合官方；代价是改变 `"350B"` 这类当前返回原标签的用例预期，需要用户确认后再改并补测试。

## 4. 边界与不可还原项

- **老芯片标签不是数字**（`320A`/`330A`/`350A`…），倒序拼不出型号，官方才保留 14 项家族表，且只到系列不到型号；`reference/rkdeveloptool/DefineHeader.h:43` 的 `ENUM_RKDEVICE_TYPE` 只到 `RK32_DEVICE=0x80`，`RKDevice.cpp:444 CheckChip` 仅用它比对映像头「芯片是否匹配」，从不显示型号。
- **两份参考实现都不还原型号**：`rkdeveloptool` 的 `read_chip_info`（`main.cpp:2390`）只 `PrintBuffer` 16 字节；`rkdevtool` 的 Rust 侧 `format_chip_info`（`reference/rkdevtool/src-tauri/src/device_ops.rs:244`）只输出 `Chip Info: [hex]`。二者映射的是**固件头部**芯片码（`firmware/info.rs:324` 的 `0x35 => "RK3588/RK3588S"`），与设备标签不可混用。
- **RK3588 与 RK3588S 无法区分**：`RockchipPacker.EncodeChip` 把二者同归 family `0x35` 并取四位数字，打包后同为 `3588`，设备标签层面无区分位。
- **maskrom 阶段只有 Chip 信息含 SoC 名称**：Capability、Flash Info、FlashID 都不含；更细型号需进 Android 读 `ro.board.platform`/device-tree compatible，或加载 loader 后看 boot 镜像头。
- 本轮未接真实设备复测回显；结论来自静态反汇编与源码对照。

## 5. 相关文档

- `f704c94a-fdd2-4686-b99d-8c228311d606`：旧结论（本文第 3 节将其更正；按钮映射与门禁记录仍有效）。
- `887976bc-7e87-4a39-9f1e-53ce4d9fa407`：烧录页接线与 RKDevTool v2.96 config.cfg 结构。
- `5c6c0a82-a5ff-4954-91c3-d91e13b9a7b3`：cfg 尾块 `'S'` 存储标记定案。

---
id: "5c6c0a82-a5ff-4954-91c3-d91e13b9a7b3"
title: "RKDevTool cfg 尾块定案：+3 存储标记 'S'（RKDevTool.exe 反汇编证据，更正 +0 未知字结论）"
description: "以 RKDevTool v2.96 二进制的读取/写出函数定案 cfg 尾块结构（+3 'S' 存储标记、+4 位掩码、+8 行地址、+12 勾选、+0..+2 填充），说明旧 cfg 空存储的成因、头部 +22/+23/+27 语义，记录 RockchipConfigFile 与测试改动及 51/51 验证与未验证边界。"
status: "active"
created_at: "2026-10-29T02:48:47.837Z"
updated_at: "2026-10-29T02:48:47.837Z"
content_hash: "9722a0fcbc763c48fd91a114a8356d50881f9180113e19621bb98083125066a9"
source_paths:
  - "UotanStudio/Services/Flash/RockchipConfigFile.cs"
  - "UotanStudio.Tests/Services/RockchipConfigFileTests.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs"
session_ids:
  - "48475515-de94-4b49-ba62-b1f0f5f0baac"
memory_body_ids:
  []
---

# RKDevTool cfg 尾块定案：+3 存储标记 'S'（反汇编证据）

对 `D:\EXE Installer\RKDevTool_Release_v2.96\RKDevTool.exe.i64`（32 位，基址 `0x400000`）做反汇编后，cfg 尾块的真实结构定案。本文更正 `669d2f16-a028-4b5a-93d9-b65c804ac507`（《RKDevTool cfg 尾块布局复核》）中「+0 存在未知 32 位字、含义未确立」的结论：那个 `0x53000000` 并不是一个字，而是**标记字节 `'S'` 落在尾块 +3**，按 u32 小端读才成了 `0x53 << 24`。

## 1. 厂商实现（关键函数）

| 函数 | 角色 | 证据 |
| --- | --- | --- |
| `sub_42A370` @ `0x42A370` | 读取（`LoadGridItem`） | 日志串 `LoadGridItem-->open %s failed`；`CFile::Open(..., 0x8000)` 后整文件读入 |
| `sub_429DF0` @ `0x429DF0` | 写出（`ExportGridItem`，带保存对话框） | 过滤器 `ConfigFile(*.cfg)|*.cfg|ParameterFile(*.txt)|*.txt|All File(*.*)|*.*||` |
| `sub_41BE60` @ `0x41BE60` | 另一窗口的写出 | 同一套 29/610/尾块布局 |
| `sub_407FC0` @ `0x407FC0` | 菜单/列名字符串表 | `IDM_LOAD_CONFIG`、`IDM_EXPORT_CONFIG`、列名 `Address`/`Name`/`Path`/`Storage` |

另外确认 `Load Config` 对话框同时接受 `*.cfg` 与 `*.txt`（parameter.txt），与 `RockchipConfigOperation.Parameter` 的设计一致。

### 头部

写出端：魔数 `0x474643`（`"CFG\0"`）、`+4` u16 `29`、`+6` 起 8 个 u16 本地时间（`GetLocalTime`）、`+22` 行数（内存中每项 28 字节）、`+23` u32 `29`、`+27` u16 `610`。

读取端不校验，直接把这三者当参数：

```c
while ((int)v18 < (unsigned __int8)v14[22])
  v19 = *(DWORD *)(v14 + 23) + v18 * *(WORD *)(v14 + 27);
```

即 `+23` = 首行偏移、`+27` = 行跨度、`+22` = 行数。本仓库按 `29/610` 硬编码并校验收到的几何（比厂商严格：几何不符直接 `InvalidDataException`，不会误读）。

### 行记录（`memset(rec, 0, 0x262)` 之后）

```c
rec[0] = 610;                        // +0  记录长度
名字 → rec + 2 / 82 … rec + 82 路径
*(DWORD *)(rec + 602) = item[12];    // 尾块 +8  行地址
*(DWORD *)(rec + 606) = item[24];    // 尾块 +12 勾选/标志
if (item[20]) {                      // 只有指定了存储才写
    rec[597] = 83;                   // 尾块 +3  = 'S' (0x53) 标记
    *(DWORD *)(rec + 598) = item[20];// 尾块 +4  = 存储位掩码
}
// 读取：v22[597] == 83 ? *(DWORD *)(v22 + 598) : 0；+602 地址；+606 标志
```

## 2. 尾块布局（定案）

| 尾块偏移 | 记录偏移 | 含义 | 证据 |
| --- | --- | --- | --- |
| +0..+2 | 594–596 | 填充，写出时被 memset 清零 | 写出端只写 +3/+4/+8/+12 |
| +3 | 597 | 存储标记 `'S'`，仅当该行指定存储时写入 | `rec[597] = 83`，读取端 `== 83` 才认存储 |
| +4 | 598 | 存储位掩码 `1 << id`（1=eMMC，2=SD，9=SPI NOR） | 与 rkdeveloptool `cs` 编号一致 |
| +8 | 602 | u32 行地址 | 读取端直接取，对应网格 Address 列 |
| +12 | 606 | 勾选/标志（1 = 勾选） | 读取端直接取；样本中带镜像行恒为 1 |

由此解释了三件此前悬置的事：

1. `rock-5b-emmc/spinor` 的 Image 行是 `'S'` + 存储位，Loader 行是 0 + 0 无存储 —— 不存在「未知首字」。
2. 四份「存储列显示为空」的旧 cfg（`D:\rk3568\images\config.cfg` 等）标记字节分别是 `0xEF`、`0x02`、`0xE9`、`0xDF`，全都不是 `'S'`，因此厂商读取端直接忽略存储字。
3. 这些文件名字字段 NUL 之后的垃圾字节与异常标记字节同源：v2.96 写出端有 `memset(rec, 0, 0x262)`，而这四份文件由未清零缓冲区的旧版本/旧路径写出（残留栈数据，故整份文件每行同一常数）。

## 3. 代码改动（实施）

- `UotanStudio/Services/Flash/RockchipConfigFile.cs`：新增 `StorageMarkerOffset = 3` 与 `StorageMarker = 0x53`；`Parse` 先看标记（`trailing[3] == 'S'` 才接受存储字并做单比特归一化，否则掩码为 0，原始字仍留在 `StorageWord`）；`WriteStorage` 按厂商语义写（指定存储→写 `'S'`+掩码；读入带标记而今不指定→显式清除；读入即无标记→完全不动，未解释字节原样透传）。
- 修掉的真实 bug：以前给新行设 eMMC 只写 +4 的 `2`、标记字节留 0，RKDevTool 打开会显示「无存储」。
- 把尾部 16 字节视为结构化尾块而非不透明透传，文档去掉「含义未确立」措辞。
- `UotanStudio.Tests/Services/RockchipConfigFileTests.cs` 新增 4 条：无标记时存储字被忽略；写存储时标记与字同写；清除存储时标记与字同清；有标记但多比特不算存储。另修正 1 条因新规则变动的尾块断言。

## 4. 验证

- `dotnet build UotanStudio.slnx -c Debug`：0 警告 0 错误；`dotnet format` whitespace + style 对改动文件无差异。
- `RockchipConfigFileTests`：51/51 通过（总计 51，失败 0）。
- 真实文件复核（7 份）：`rock-5b-emmc`、`rock-5b-spinor` 与 v2.96 自带 `config.cfg` 逐字节往返一致，存储分别识别为 EMMC / SPINOR；四份旧文件标记 `0xEF/0x02/0xE9/0xDF` 存储为空、字节不被改写。

## 5. 仍未确立 / 未验证

- 尾块 +12 的语义只到「读取端原样取出并在网格显示为勾选态」，反汇编里没有更强命名；样本中恒等于「该行是否带镜像」。
- 四份旧文件的具体来源版本与写出路径无法从 v2.96 二进制确定。
- 未做：把本仓库输出再用 RKDevTool GUI 打开确认存储列显示（需人工点界面）。

## 6. 相关文档

- `669d2f16-a028-4b5a-93d9-b65c804ac507` —— 此前基于样本逐字节 dump 的尾块复核（+4 掩码 / +8 地址）。其偏移结论仍有效，但「+0 未知字」与「+12 未确立」两处由本文更正。

---
id: "dbc55a39-9383-4727-8bef-35f5705f8b77"
title: "UotanStudio 注释规范化：审计口径、分批范围与实测基线"
description: "全仓注释清理的可复用方法：四类判定口径与保留的假阳性、按目录互斥分批的并发纪律、实测残留基线与优先顺序、未启用 GenerateDocumentationFile 的项目隐藏 XML 结构错误（含 11 处具体位置）以及构建/格式化验证门。"
status: "active"
created_at: "2026-10-27T00:27:45.076Z"
updated_at: "2026-10-27T00:27:45.076Z"
content_hash: "51d2be8d10af9486b2265200f7e00eeedea9eaab547aec001ee4df6ffb4c59bd"
source_paths:
  - "AGENTS.md"
  - "UotanStudio/Services/Devices/DevicePropertyReader.cs"
  - "UotanStudio.Benchmarks/Firmware/SparseDatBenchmarks.cs"
  - "UotanStudio.Comm/Protocols/Adb/AdbAuthentication.cs"
  - "UotanStudio.Comm/Protocols/Edl/Streaming/StreamingBadBlockTable.cs"
  - "UotanStudio.Tests/Firmware/DtsCoverageTests.cs"
  - "UotanStudio.Delivery/Models/Plan.cs"
session_ids:
  - "ed1bad61-827b-4065-9348-831f14f225c8"
memory_body_ids:
  []
---

# UotanStudio 注释规范化：审计方法、分批范围与实测基线

一次跨全仓的注释/Doc 清理会话留下的可复用方法、分批边界与实测数字。规则本体在仓库根 [AGENTS.md](AGENTS.md)（"代码注释和仓库文档只描述当前状态，不描述历史、草稿或演进"；XML 注释英文在前、中文放入同一标签的 `<para>`）；本文只记录落地方法与审计口径，不作为规则的替代。

## 清理口径（判定为"该改"的四类）

1. **行内注释** — 实现旁的 `//` 旁注一律删除（公共 API 不得用行间注释代替 XML 注释）。
2. **历史/演进表述** — "保持历史行为""旧版本写出的快照退化""此前使用""历史上导致""早期版本遗留"等，改写为直接的当前契约描述（如"否则使用惰性实例""会使…"）。
3. **其他项目溯源** — 引用第三方源码路径/函数名/工具语料的注释（`client/auth.cpp`、`RKBoot.cpp`、`mtkclient/`、`bkerler/edl`、`mkfs.erofs`、`AOSP dt_table.h`、`file(1)` 语料等），改为直接写协议/格式要求。
4. **双语格式不统一** — 缺中文 `<para>` 的英文单语注释补齐；中英顺序颠倒的改回"英文在前、中文在 `<para>` 中"。

假阳性需保留：`UotanStudio.Tests/Firmware/DtsCoverageTests.cs` 里的 `// line comment`、`/* inline */` 是 DTS 测试数据字符串而非注释；网络/工作流的"上游端口/上游输出"是字面本义；芯片"演进字"、撤销"历史"字段等是业务语义。逐条判断，不做全局正则替换。

## 分批与并发纪律

- 划分必须按**目录/项目完全互斥**，否则并发子代理会在同一文件上冲突。
- 子代理约束只改 `///` 注释行，保留 CRLF（仓库文件为 CRLF，全量转换需要单独恢复），禁止在子代理内执行构建与 git 操作。
- 提交前统一收口：XML 结构校验 → 残留扫描 → 全量 `dotnet build` → 两条 `dotnet format --verify-no-changes`。

## 实测基线（审计时的扫描值，随清理推进而下降）

| 指标 | 数量 |
| --- | --- |
| 行内注释残留 | 5（全部为 `DtsCoverageTests.cs`/`UntrustedXmlDocumentTests.cs` 的测试数据字符串） |
| 第三方源码路径/函数名引用（文档块） | 292 |
| `upstream` 一致性比较句（文档块） | 166（Comm 99、Firmware 38、宿主 18、其余零散） |
| 历史语义标记行 | 118（多数为假阳性，需逐条判断） |
| 引用第三方源码路径的行 | 532（Comm 504、Tests 19、Hash 5、Firmware 4），集中在 `Comm/Protocols/Edl/Streaming/*`、`Comm/Protocols/Edl/Programmer/*` |
| `AOSP` 提及行 | 117 |
| `erofs-utils`/`mkfs.erofs`/`include/erofs_fs.h` | 79 |
| 宿主 `UotanStudio` 纯英文公共 XML 注释 | 112（该项目未启用 `GenerateDocumentationFile`，不违规但与库项目风格不一致） |

优先顺序：`Comm/Protocols/Edl` 与 `Comm/Protocols/Adb`（占比最大），其次 `Firmware/FileSystem/Erofs`。

## 关键发现：未启用 `GenerateDocumentationFile` 的项目会永久隐藏 XML 结构错误

宿主 `UotanStudio` 与 `UotanStudio.Benchmarks` 未设置 `GenerateDocumentationFile`，因此编译器从不校验 XML 注释结构，浏览器/格式化工具也不会报错。人工扫描发现 11 处结构损坏——`<returns>` 内嵌 `<para>` 未闭合或缺 `</returns>` 结束标签：

- [DevicePropertyReader.cs](UotanStudio/Services/Devices/DevicePropertyReader.cs) 3 处（约 520/552/588 行的 `<returns>… <para>…。</returns>`）。
- [SparseDatBenchmarks.cs](UotanStudio.Benchmarks/Firmware/SparseDatBenchmarks.cs) 8 处（约 293–395 行的 `<returns>` 与 `<exception>` 块）。

这类缺陷只能靠独立的 XML 文档解析扫描发现；修改这两个项目的注释时应顺手跑一次结构校验，不能只依赖构建。

## 验证门（实测通过的形式）

```text
dotnet build UotanStudio.slnx --no-restore --nologo -c Debug        -> 0 error（仅既有 xUnit1051 提示）
dotnet format UotanStudio.slnx whitespace --no-restore --verify-no-changes -> 0
dotnet format UotanStudio.slnx style      --no-restore --verify-no-changes -> 0
```

纯注释改动时以 `git diff` 逐行核对"只有 `///` 行与少数诊断字符串变化"，并确认 CRLF 已恢复、工作区既有无关修改未被触碰；编译门覆盖即足够，未额外跑测试，也未做 Release AOT 发布（纯注释改动不触发裁剪风险，但该边界应在汇报中写明）。

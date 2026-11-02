---
id: "ccc1063c-92f3-4a7b-98cf-bc921c0c4b23"
title: "UotanStudio 内嵌资源 zstd 压缩可行性评估（2026-10-24）"
description: "UotanStudio 的 magic 规则表 / MiSans 字体 / ohos 设备 .so / update-binary 用 zstd 压缩的实测体积账、解压分配与耗时、分项可行性与风险、落地顺序，以及 Avalonia 12 字体原型结论。"
status: "active"
created_at: "2026-10-23T18:18:57.412Z"
updated_at: "2026-10-23T18:18:57.412Z"
content_hash: "00f80cdd6bfed822de0bba60a28e077a70dc7c2e32731a49b4334df814f15415"
source_paths:
  - "UotanStudio.Firmware/Core/Magic/Generated/CompiledMagicData.g.cs"
  - "UotanStudio.Firmware/Core/Magic/CompiledMagicEngine.cs"
  - "UotanStudio.Mirroring/UotanStudio.Mirroring.csproj"
  - "UotanStudio.Delivery/Build/Updater.cs"
  - "UotanStudio/App.axaml"
  - "UotanStudio/Assets/Fonts/NOTICE.md"
  - "UotanStudio.Tests/Firmware/Architecture/CompressionImplementationBoundaryTests.cs"
session_ids:
  - "5423575e-14c3-4dbf-9ead-20fe66e525c0"
memory_body_ids:
  []
---

# UotanStudio 内嵌资源 zstd 压缩可行性评估（2026-10-24）

评估目标：判断把安装产物中的大块内嵌资源改存为 zstd 压缩体是否值得做，以及各资源的价值/风险排序。所有数字均来自对 `UotanStudio.Desktop/bin/Release/net10.0`（framework-dependent）实际产物的字节级取证 + 独立 .NET 10 探针程序实测，非估算。

## 体积账（唯一结论表）

| 资源 | 原始占用 | zstd 19 后 | 比率 | 节省 |
| --- | --- | --- | --- | --- |
| magic(5) 规则表（base64 → UTF-16 IL 字符串） | 8,119,848 B | 317,678 B | 3.9% | 7.44 MiB |
| MiSans Regular | 8,122,324 B | 4,900,305 B | 60.3% | 3.07 MiB |
| MiSans Bold | 8,009,316 B | 4,979,674 B | 62.2% | 2.89 MiB |
| ohos 设备端 `.so`（15 个文件） | 11,204,026 B | 3,478,225 B | 31.0% | 7.37 MiB |
| update-binary-arm64 | 2,002,464 B | 838,349 B | 41.9% | 1.11 MiB |
| 合计 | 37,457,978 B | 14,514,231 B | 38.7% | 21.88 MiB |
| 仅程序集内（magic + 字体） | — | — | — | 13.40 MiB |

注意：magic 的 8.12 MB 是 **UTF-16** 表示（源码为 4,059,924 个 base64 字符），所以它既是安装体积浪费也是下载体积浪费；字体和 `.so` 属于"安装体积"收益，ZIP/MSI 对裸 TTF/ELF 本来也能压到近似比率，下载体积几乎不变。做决策前必须先区分这两个指标。

## 现状取证要点

- **字体原样内嵌**：在 `UotanStudio.dll`（23,382,016 B）中能逐字节命中 `MiSans-Regular.ttf` 头/中/尾 4 处 TTF 切片 → 16.13 MB 是裸 TTF，Avalonia `AvaloniaResource` 不做压缩（`UotanStudio/UotanStudio.csproj` 的 AvaloniaResource 项）。
- **magic 表是 base64 字符串**：`UotanStudio.Firmware.dll`（12,213,248 B）中 254/254 个片段可按 UTF-16 命中，占该 DLL 的 66%；运行时 `Rules => _rules ??= Decode()` 逐段 `Convert.FromBase64String` 再拼接（`UotanStudio.Firmware/Core/Magic/Generated/CompiledMagicData.g.cs`，消费点 `UotanStudio.Firmware/Core/Magic/CompiledMagicEngine.cs:68` 用 `new MemoryStream(CompiledMagicData.Rules)`）。**这里本来就有一层解码，换成 zstd 只是替换解码器，不新增架构层。**
- **ohos `.so` 是松散 Content**（`UotanStudio.Mirroring/UotanStudio.Mirroring.csproj:17`），运行时按 `AppContext.BaseDirectory/ohos/device` + 文件名解析版本与套接字布局（`SelectVideoLibraries`），推送走 `IHdcSyncClient.Send(localPath, remotePath)`。
- **update-binary 是松散 Content**，`Updater.MaterializeAsset` 复制到 `%TEMP%/UotanStudio/delivery-updater` 后用**长度**比较判断缓存（`UotanStudio.Delivery/Build/Updater.cs:41`）。

## 压缩与解压实测（.NET 10.0.402 + ZstdSharp.Port 0.8.8）

参照 CLI：zstd 1.5.7（WSL Alpine）。托管 L19 与 `zstd CLI -19 --no-check` 对 magic blob 输出**字节数完全相同（317,678）**，构建期可用任一工具。

- **级别边际收益极小**：magic L19=L22=10.4%；字体 L3 66.4% → L9 62.8% → L19 60.3% → L22 60.3%。构建期用 `-19`，总压缩耗时约 10 s（magic 1.1 s、每字体 2.3 s、3 MB `.so` 0.8 s）。
- **解压目标必须用预分配 span**：`Unwrap(compressed).ToArray()` 分配 2× 输出（8.12 MB 字体 → 16.24 MB；3.04 MB blob → 6.09 MB）；`Unwrap(z, destSpan)` 实测 **0 分配**。`ZstdBlockCodec.Decompress(source, expectedSize)` 正是后者。解压吞吐：magic 2.8 ms（≈1 GB/s）、uitest `.so` 0.62 ms（925 MB/s）、字体 8.8 ms 热 / 54 ms 冷。
- **Brotli 不值得**（虽然 `CompressionService` 里有）：magic q11 295,281 B（比 zstd 小 7%，但压缩 3.6 s）、SmallestSize 字体 4,773,853 B（58.8%，压缩 16.4 s）、解压反而更慢（magic 4.55 vs 2.8 ms；字体 48 vs 9 ms）。zstd 是正确选择。
- **solid 单帧 vs 逐文件**：ohos 15 个 `.so` 逐文件 3,478,225 B（31.0%），solid tar+zstd19 2,194,497 B（19.6%，多省 1.22 MiB）但失去随机访问/懒加载，不建议；字体 per-file 9,881,661 vs solid 9,739,396（差 142 KB），同样不值。

## 分项可行性

### A. magic 规则表 —— 唯一"纯赚"项，首选

改法：`MagicGen` 输出 `CompiledMagicData.bin.zst`（≈318 KB）+ 只含常量的小 `.g.cs`（`BlobByteLength`、`RuleCount`、`SHA256`、`GeneratedAt`）；csproj 用显式 `LogicalName` 的 `EmbeddedResource` 嵌入；`CompiledMagicData.Rules` 改为 `ZstdBlockCodec.Decompress(resourceBytes, BlobByteLength)` 并校验长度 + 生成的 SHA-256。

收益：`UotanStudio.Firmware.dll` 12.21 MB → ≈4.41 MB；加载 5.5 ms → 2.8 ms；分配从"3 MB 结果 + 254 个小数组（≈6 MB 临时）"变为一次 3 MB 精确分配；仓库 diff 从 4 MB 文本变 318 KB 二进制。风险最低。

### B. 字体 —— 技术可行，已做端到端原型

在 Avalonia 12.1.4（Win32 + Skia，`SetupWithoutStarting()`）验证：

- `FontCollectionBase` 是 public abstract、protected 构造，**唯一抽象成员是 `Key`（`abstract Uri get_Key()`）**；
- `TryAddGlyphTypeface(Stream, out GlyphTypeface)`（public）直接吃解压后的 `MemoryStream` → `True`，`FamilyName="MiSans"`，29,758 字形；
- `FontManager.Current.AddFontCollection(...)` + `FontFamily("fonts:UotanStudio/MiSans#MiSans")` 可解析出 typeface；
- **托管缓冲可释放**：丢掉解压数组并 GC 后托管堆增量 0.35 MiB，face 仍可用（Avalonia 复制到非托管内存）；代价是单张 face 的私有字节 +18.5 MB——与现有 avares 路径稳态内存相同，存储格式不改变稳态占用；
- 必须先完成 Avalonia 平台初始化（裸控制台抛 `Unable to locate 'Avalonia.Platform.IFontManagerImpl'`），应用内天然满足。

成本：首次排版 Regular ≈84 ms（解压 54 + TryAdd 23 + 解析 7），Bold ≈42 ms，热态解压 8.8 ms。改动面：一个 `FontCollectionBase` 子类、`UotanStudio/App.axaml:38` 的 FontFamily、`UotanStudio/Assets/Fonts/NOTICE.md` 中"原样内嵌"措辞（MiSans 许可证要求注明官方来源）。真要抠字体，**子集化收益远大于 zstd**，但会带来用户动态文本缺字风险，属于独立决策，不要混做。

### C. ohos 设备端 `.so` —— 收益大但要改运行时契约

压缩后必须从"文件名枚举 + 名称解析"改为**资源清单驱动**（名称/版本/套接字布局/原始长度/压缩长度/哈希），`SelectVideoLibraries` 读清单而非 `Directory.EnumerateFiles("*.so")`。推送方面 ADB 侧已有 `AdbSyncClient.PushStreamAsync(Stream, remotePath, ...)`，但 Mirroring 走 `IHdcSyncClient.Send(localPath, remotePath)`，**HDC 契约只有路径版**，需补 stream 重载或落临时文件（后者违背纯内存）。风险：设备端推送/权限/启动参数需真机或 HDC 验证，且 Comm/HDC 正在并发改动，属冲突面。

### D. update-binary —— 可行但收益小

DAT/exec 路径仍须把内容写到 `%TEMP%` 真实文件（exec 需要路径），只赚安装体积 1.11 MiB。注意 `MaterializeAsset` 用**长度**判断缓存，压缩后必须改成"解压后长度/哈希"比较，否则会误判缓存有效。

### E. 原生库 —— 不可行

`libusb-1.0.dll`、`libuotan_media_va.so` 等由 OS 加载器 / P‑Invoke 直接打开，必须是真实文件，无法 `LoadLibrary` 内存映像。合计 < 1 MB，保持现状。

## 约束与坑（按项目规则排序）

1. **架构边界**：`UotanStudio.Tests/Firmware/Architecture/CompressionImplementationBoundaryTests.cs` 禁止 `UotanStudio.Compress` 之外 `new` 内置压缩流；生产层必须走压缩门面，不要在 Firmware/宿主里直接 `using ZstdSharp`。
2. **NativeAOT/裁剪**：`EmbeddedResource` + `GetManifestResourceStream` 是 AOT 安全的；**不要**用 `Assembly.Location` 找旁挂文件（单文件发布下为空）。`LogicalName` 必须显式固定。落地后必须补一次 AOT publish 冒烟（读资源 + 解压）。
3. **失败要响亮**：资源按"可信但校验"处理——校验解压后长度 + 生成 SHA-256，解压上限用 `expectedSize`，别绕过 `ZstdBlockCodec` 的帧窗口上限（`maximumWindowLog`）直接用裸 `Decompressor`。
4. **惰性 + 预热**：magic 首次识别、字体首次排版才解压；字体 ~84 ms 建议启动时后台预热，否则体现为首帧卡顿。
5. **峰值内存**：字体每张 face 瞬时 +8 MB（可回收），ohos `.so` +3 MB；按字重懒加载，`TryAddGlyphTypeface` 后立刻清引用。
6. **测试**：magic 加"解压 = 长度/哈希"往返测试；`.so` 加"清单 vs 实际文件集合"一致性门防漂移；字体加 headless 冒烟或 `TryGetGlyphTypeface` 断言；在 `VERIFICATION.md` 记录实测数字。
7. **不要压已压缩/过小资源**：png/jpg/ico 与 < 32 KB 资源，帧开销与解压上下文不划算。

## 建议路线

| 顺序 | 内容 | 收益 | 风险 | 工作量 |
| --- | --- | --- | --- | --- |
| 1 | A. magic 表 | −7.44 MiB，加载更快 | 低 | ~0.5 天含测试 |
| 2 | C. ohos `.so` | −7.37 MiB | 中（清单 + HDC stream 契约 + 真机） | 1–2 天 |
| 3 | B. 字体 | −5.96 MiB，首帧 +80 ms | 中（自定义 collection + 回退链） | 1 天 |
| 4 | D. update-binary | −1.11 MiB | 低 | 顺带做 |
| — | E. 原生库 | — | — | 不做 |

只做一件事就做 A：唯一"体积更小 + 加载更快 + 风险最低 + 仓库 diff 从 4 MB 文本变 318 KB 二进制"的改动。

## 未验证边界

- 未跑完整 `dotnet publish -p:PublishAot=true`；上述体积由产物组件字节推算，AOT 后 exe 实际缩小幅度待测。
- 字体原型是独立 Win32+Skia 控制台，**未集成进应用**；Semi.Avalonia/Ursa 主题是否指定其它字体族、回退链是否受影响未验证。
- `.so` 清单方案与 HDC `Send(Stream)` 未实现，真机推送未验证。
- 未测整体冷启动时间，只有各步骤耗时。
- 审计期间 `ohos/device` 出现并行新增的 `libscrcpy_server0.z.so` 等工作区改动，账目按当时 15 个文件 / 11.2 MB 计。

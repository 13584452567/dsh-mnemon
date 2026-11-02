---
id: "1eda821d-1fe8-402c-9627-149b82049c6e"
title: "UotanStudio 测试偶发失败定点排查：Progress<T> 异步投递与分配测量两类根因（2026-10-29 后续）"
description: "全量门禁 7804 用例多轮复现后拆出的四条偶发线索：McpCancellation 进度竞态与 AudioPcmRingBuffer 分配测量的根因与修法、ADB 配对真实 30s 卡死（待加阶段名）、EROFS ambient 覆盖跨类干扰（待证明机制），含门禁数字与未验证边界。"
status: "active"
created_at: "2026-10-28T16:10:00.266Z"
updated_at: "2026-10-28T16:10:00.266Z"
content_hash: "e701d360ac97e1df841c9a562dc59896030fa284c6b2947f0293a63c99d3770a"
source_paths:
  - "UotanStudio.Tests/Protocols/AdbRemoteProtocolTests.cs"
  - "UotanStudio.Tests/Services/McpCancellationPropagationTests.cs"
  - "UotanStudio.Tests/Media/AudioPcmRingBufferTests.cs"
  - "UotanStudio.Media/Audio/AudioPcmRingBuffer.cs"
  - "UotanStudio.Tests/AGENTS.md"
session_ids:
  - "b341bbbd-2c2c-463d-8f04-f687eff97a06"
memory_body_ids:
  []
---

# UotanStudio 测试偶发失败定点排查：两类根因已修、两类仍有证据链（2026-10-29 后续）

对 `dotnet test -c Release` 全量门禁（7804 用例 / 跳过 8）多轮重跑后，把"偶发失败"拆成 4 条独立线索。全部实测在本机 Windows x64、共享工作树、Release 配置下完成；CI runner 上的表现未验证。

相关既有文档：`49ac1526-5aa6-4e33-8124-d8469eb37d45`（早期 15s 宿主超时与连带取消的实测，本文是其后续收敛）、`65b080df-c163-4577-8543-ecbb049d88ac`（host:pair 结果帧解析与进程内 SPAKE2 回退）。

## 1. 已修：McpCancellationPropagationTests 的进度断言（真实竞态）

- 根因在生产侧：`FirmwareToolFactory.ToTupleProgress` 用 `new Progress<T>` 包装接收端，而 `Progress<T>` 是异步投递（投到捕获的 SynchronizationContext / 线程池），`ExtractAsync` 返回时回调可能尚未执行。
- 测试原实现用普通 `int` 被 lambda 捕获、完成即读，因此偶发读到 0，形成断言语义错误而非产品错误。
- 修法（仅测试）：`CountingProgress` 用 `Interlocked.Increment` 计数 + `TaskCompletionSource` 信号，测试先 `await progress.WaitForFirstReportAsync(30s, token)` 再断言。
- 验证：该类连跑 20 轮，0/20 失败。

## 2. 已修：AudioPcmRingBufferTests.SteadyWritesAndReads_DoNotAllocate 的测量方式

- 失败值固定为 `Expected: 0, Actual: 4080`。逐行核对 `UotanStudio.Media/Audio/AudioPcmRingBuffer.cs`：构造时一次性分配 `_ring`/`_conversion`，`Write`/`Read` 全程 Span + `Lock`，无逐次分配点。
- 4080 字节来自测量循环首轮在测量线程上触发的 tier-1 JIT 提升，被打进 `GC.GetAllocatedBytesForCurrentThread()`。
- 修法：测量循环改为最多 3 轮、任一轮为 0 即通过；真泄漏每轮都会非零，仍会失败。
- 验证：连跑 20 轮，0/20 失败。

## 3. 仍有：AdbRemoteProtocolTests.PairAsync_StatusFramedResult_IsAccepted 真实卡死 30 秒

- 上一轮的墙钟/超时修复行为正确：不再出现宿主 15s 中止，也不再连带取消其它用例；但暴露出是真实卡死而非调度噪声。
- 证据：8 轮全量中 3 次失败，耗时都精确 30s 0xx ms，异常是产品自身的 `TimeoutException: ADB network operation exceeded 30 seconds.`，栈指向 `RunPairCommandAsync`。
- 已逐行核对双方协议，排除分段发送死锁：产品 `RequestAsync` 写 4 字节长度 + 正文，测试假 server `SingleReplyAdbServer.ReadRequestAsync` 用 `ReadExactlyAsync` 读 4 + N。
- `OpenHostStreamAsync` 设的 `ReadTimeout = 10000` 对异步读无效，因此唯一边界是 `RunPairCommandAsync` 里 30s 的 linked token。异常文本不区分卡在 connect / 写请求 / 读状态 / 读负载，无法继续收敛。
- 待办（未执行，属产品改动）：给该 `TimeoutException` 带上阶段名，再复现一次即可定位到具体 await。

## 4. 仍有：PackingOptionPropagationTests.Pack_PreservedErofsWithDotOmittedEnabledKey_Succeeds 覆盖值丢失

- 现象：`Expected: Enabled, Actual: Disabled`，即 `RepackMetadata.UsePackingOverrides` 的 `ErofsDotOmitted=Enabled` 未生效。
- 分布：全量 8 轮中 1 次；同一测试类单独连跑 15 轮 0 次 -> 指向跨类并发干扰，而非产品必然错误。
- 候选污染源（共用同一批 ambient 覆盖）：`DatFormatHandlerTests`、`Ext4CapacityHeadroomTests`、`FileSystemPreservationTests`、`MetadataJsonTests`、`RepackMetadataTests`、`PackingOptionPropagationTests`。
- 机械性未证实，因此未采用"收进同一串行 collection"的掩盖式修法；`UotanStudio.Tests/AGENTS.md` 允许对共享进程全局状态串行化，但需先证明机制。

## 当前工作树状态（本轮未提交）

- `UotanStudio.Tests/Protocols/AdbRemoteProtocolTests.cs`、`UotanStudio.Tests/Services/McpCancellationPropagationTests.cs`、`UotanStudio.Tests/Media/AudioPcmRingBufferTests.cs`，合计 +56/−29。
- 门禁：`dotnet format` whitespace/style 均退出 0；Debug 与 Release 构建 0 警告 0 错误。
- 最近一轮全量：7804 总数 / 1 失败（第 3 条）/ 8 跳过。改法生效后 8 轮统计：4 轮全绿、2 轮第 3 条、1 轮第 4 条、1 轮已修的分配用例。

## 复用要点

- 判定偶发失败前先分辨"测量手段引入的噪声"与"产品真实缺陷"：分配测量、`Progress<T>` 异步投递、墙钟超时三者都会制造假失败。
- 超时消息必须携带阶段名，否则 30s 卡死无法收敛；这是本轮得到的通用教训。
- 环境相关跳过项仍在：`UOTAN_DEVICE_SHELL_TARGET` 未设置导致 DeviceShellConnectionTests 真机用例跳过，ErofsInode 语料用例同样按门控跳过。

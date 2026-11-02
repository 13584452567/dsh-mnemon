---
id: "00a7134c-62cd-457b-b10c-88e2aa352b0e"
title: "投屏 ReadFrame 超时返回 null 被 ?? 误判为缺失实现：facade NRE 根因与回归测试"
description: "MirrorSessionFacade 用 ?? 串联两条投屏实现，导致一次视频读超时（合法 null）触发 NullReferenceException 并杀死宿主解码循环；含触发条件、修复、变异验证的回归测试与未验证边界。"
status: "active"
created_at: "2026-10-24T19:26:47.063Z"
updated_at: "2026-10-24T19:26:47.063Z"
content_hash: "858a3c85ff162a5cafcfd085dcd97b6561a5beed6320ff4f0ec7ac19756cbda7"
source_paths:
  - "UotanStudio.Mirroring/MirrorSessionFacade.cs"
  - "UotanStudio.Mirroring/Scrcpy/ScrcpyClient.cs"
  - "UotanStudio.Mirroring/ManagedMirrorSession.cs"
  - "UotanStudio/ViewModels/Editors/ScreenMirrorViewModel.cs"
  - "UotanStudio.Tests/Mirroring/MirrorSessionFacadeTests.cs"
session_ids:
  - "a146302e-48ea-4524-8e6e-f37db01b420e"
memory_body_ids:
  []
---

# 投屏 ReadFrame 超时与 `??` 回落导致的 facade 空引用

范围：`UotanStudio.Mirroring` 会话门面在一次视频读超时后抛出 `NullReferenceException`，进而杀死宿主解码线程，表现为"画面停住、触摸仍可用、切回也无法恢复"。前置的投屏故障隔离背景见 `64a77aaf-9665-44d1-8b37-0bfd4194e9e2`（该文记录的是**帧呈现异常**越过解码循环边界；本文记录的是**读取契约**被 `??` 误用，两者症状相同、根因不同）。

## 症状与诊断特征

日志栈为 `MirrorSessionFacade.ReadFrame(int)` → `NullReferenceException`，紧邻 60 秒级的 `ADB service read timed out` 噪声。诊断特征是"控制可用 + 视频静止"：控制通道与解码循环相互独立，控制仍透传说明会话与服务端链路没断，死的是宿主侧解码线程。

## 根因

`UotanStudio.Mirroring/MirrorSessionFacade.cs` 把两条实现用 `??` 串联：

```csharp
=> _scrcpy?.ReadFrame(timeoutMs) ?? _ohscrcpy!.ReadFrame(timeoutMs);
```

而 `ScrcpyClient.ReadFrame` 的契约是**超时返回 `null`**（`UotanStudio.Mirroring/Scrcpy/ScrcpyClient.cs`，读不满 12 字节包头即 `return null`）。走 ADB/scrcpy 路径时 `_scrcpy` 非空、`_ohscrcpy` 为 `null`，于是**第一次视频读超时**就会求值右侧的 `_ohscrcpy!` 并抛 NRE：`??` 在这里被当成"实现缺失"的回落，而返回值 `null` 实际是合法的"本次无帧"。

触发条件：屏幕被复合标签等界面完全遮挡后不再有输入，即使 `ScrcpyServerOptions.StayAwake` 保持屏幕不熄，编码器在画面静止时也不出新帧，读超时成为常态而非偶发。NRE 冒泡到 `UotanStudio/ViewModels/Editors/ScreenMirrorViewModel.cs` 解码循环的最外层 `catch`，打印 `Decode failed` 后退出循环，且该路径不重连，所以"切回来也恢复不了"。

## 修复

按当前实现分支，不再让合法的 `null` 落到另一条不存在的实现上：

```csharp
public ScrcpyVideoFrame? ReadFrame(int timeoutMs)
{
    if (_scrcpy is not null) return _scrcpy.ReadFrame(timeoutMs);
    return _ohscrcpy!.ReadFrame(timeoutMs);
}
```

同一文件里 `DeviceName`、`IsControlConnected` 也是 `??` 串联，但左值分别是 `string.Empty` 与 `bool`，不存在"合法的空结果"，保持不变。

原则：**可空返回值的语义是数据时，不得作为 `??` 的左操作数**；"实现缺失"要用显式的实现判空表达。这类缺陷在无流量时才会浮现，静态审查容易漏。

## 回归测试

`UotanStudio.Tests/Mirroring/MirrorSessionFacadeTests.cs` 新增 `ReadFrame_AdbSessionTimesOut_ReturnsNullWithoutThrowing`：

- 用真实的 `ScrcpyClient`（内部 `Func<string, Stream>` 构造）完成 scrcpy 握手（设备名 + `h264` + `PacketFlagSession | 宽` + 高），避免桩件掩盖真实读路径。
- 自定义 `StalledVideoStream : MemoryStream` 在握手字节耗尽后从 `Read` 抛 `TimeoutException`，从而走到 `TryReadExactInto` 的 `offset == 0` 超时分支并返回 `null`。
- 断言 facade 返回 `null` 且不抛异常。
- 变异验证：把生产代码改回 `??` 时该用例以 `System.NullReferenceException` 失败，改回修复版即通过，证明用例确实钉住了该 bug。

## 验证边界

- `dotnet build UotanStudio.slnx -c Debug` 0 警告 0 错误；`dotnet test --filter-namespace UotanStudio.Tests.Mirroring --minimum-expected-tests 1` 总计 112、通过 111、跳过 1（`UOTAN_HDC_MIRROR_DIRECT` 门控的硬件集成用例）。这与 `VERIFICATION.md` 的记录口径一致。
- 真正的终结性断流（服务端退出导致 `EndOfStreamException`）仍会结束解码循环并留下失败状态，没有自动重连；本次只修掉"一次读超时就把会话打死"。
- 未在真机上复现遮挡切换场景（本机设备库 `dlopen` 失败），结论来自代码路径、日志栈与单元级复现，硬件回归仍需实测。
- 工作区中 `MirrorSessionFacade.cs` 的 `DeviceLease?` → `IAsyncDisposable?` 两处签名改动属于并行会话的租约重构，不随本次修改。

## 源文件

- `UotanStudio.Mirroring/MirrorSessionFacade.cs`
- `UotanStudio.Mirroring/Scrcpy/ScrcpyClient.cs`
- `UotanStudio.Mirroring/ManagedMirrorSession.cs`
- `UotanStudio/ViewModels/Editors/ScreenMirrorViewModel.cs`
- `UotanStudio.Tests/Mirroring/MirrorSessionFacadeTests.cs`

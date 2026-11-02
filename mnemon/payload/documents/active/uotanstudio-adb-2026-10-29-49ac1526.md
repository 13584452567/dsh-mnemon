---
id: "49ac1526-5aa6-4e33-8124-d8469eb37d45"
title: "UotanStudio 全量门禁偶发：ADB 配对用例超时并连带取消无关用例（2026-10-29 实测）"
description: "Release 全量门禁实测：断言 0 失败但 AdbRemoteProtocolTests.PairAsync_StatusFramedResult_IsAccepted 在满载并行下撞自身 15s 超时并连带取消一条无关用例；含复现比例、单跑对照、FINALNEWLINE 空白违规与配置切换复核。"
status: "active"
created_at: "2026-10-28T15:37:27.316Z"
updated_at: "2026-10-28T15:37:27.316Z"
content_hash: "5854630c0b79aba1c98c0e98dc92875cc65e4de8a6364599b512f8b3f7072947"
source_paths:
  - "UotanStudio.Tests/Protocols/AdbRemoteProtocolTests.cs"
  - "UotanStudio.Comm/Protocols/Adb"
  - "VERIFICATION.md"
  - "AGENTS.md"
session_ids:
  - "6bde3060-dfe2-4d3a-919f-9ecb8235e3d8"
memory_body_ids:
  []
---

# UotanStudio 全量门禁的偶发失败：ADB 配对用例超时并连带取消无关用例

本文件记录 2026-10-29（Windows x64，Release 配置，共享工作树）重跑完整 CI 门禁时的实测现象与定点结论，供后续排查门禁红灯时对照。相关实现文档见 `1625ca1e-b4e4-4731-9cdf-933a93898b1b`（烧录页 Root 修补接线）与 `6e1bc9bf-cdf8-477b-83dc-aab69e6bbd6d`（Boot 补丁窗口与补丁包链路）。

## 1. 门禁命令与稳定项

与 `.github/workflows/ci.yml` 相同的 Release 链路：`dotnet restore UotanStudio.slnx` → `dotnet build -c Release` → `dotnet format ... whitespace --verify-no-changes` → `dotnet format ... style --verify-no-changes` → `dotnet test --solution UotanStudio.slnx -c Release --minimum-expected-tests 6000`。

- 构建每轮都是 0 警告 0 错误；两道 `dotnet format` 门禁在修复下面第 2 节的空白违规后转绿并保持稳定。
- 全量跑一次约 1m25s–1m27s（Release，共享工作树）；用例总数 7804、跳过 8。
- 结论：**断言失败始终为 0**，红灯全部来自第 3 节的超时类偶发。

## 2. 空白门禁的真实坑：`FINALNEWLINE`

仓库根 `.editorconfig` 要求 `insert_final_newline = false`。已提交的 5 个节点编辑器文件带了文件末尾换行，`dotnet format whitespace --verify-no-changes` 因此报 `FINALNEWLINE`：`ViewModels/Nodes/NodeEditorViewModel.cs`、`NodeEditorViewModel.Navigation.cs`、`Views/Nodes/NodeEditorView.axaml.cs`、`NodeEditorView.ExecutionLog.cs`、`NodeEditorView.Navigation.cs`。删掉末尾空行后两道格式门禁均为零差异。日常 `dotnet build` 不暴露这类问题，只有 `--verify-no-changes` 会拦。

## 3. 偶发：`PairAsync_StatusFramedResult_IsAccepted` 撞自己的 `Timeout = 15000`

现象（`UotanStudio.Tests/Protocols/AdbRemoteProtocolTests.cs`）：

```text
失败 (已取消) ...AdbRemoteProtocolTests.PairAsync_StatusFramedResult_IsAccepted (15s 005ms)
  Xunit.MicrosoftTestingPlatform.XunitException: Test execution timed out after 15000 milliseconds
```

- 同一轮里还会有一条**无关**用例被报「失败 (已取消)」，每次不同：一次是 `Firmware.ErofsInodeAdviseAcceptanceRegressionTests.Capture_Restore_NlinkOneAdviseSource_...`，一次是 `Services.McpCancellationPropagationTests.ExtractTool_NoProgressSinkAndLiveToken_...`。所以「2 个失败」计数里通常只有 1 个是真凶。
- 复现比例：7 次全量运行 3 次全绿、4 次出现该超时；该类单独跑 `--filter` 为 14/14 通过（约 2–4.6 s）。
- 该用例用 `SingleReplyAdbServer`（`TcpListener` 绑 loopback 随机端口）做假 ADB server，等待由墙钟驱动；满载并行（全量跑时同时有大量 MSBuild/dotnet 进程占用 CPU）下 15 s 上限不再裕量充足。
- 与该轮功能改动无关：`UotanStudio.Comm/Protocols/Adb` 不在改动范围内（`Comm` 只改了 `Diag/DiagSession.cs`、`Edl/Crash/CrashDumpCollector.cs` 各 1 行删除）。
- 未验证边界：本机（共享工作树、Windows x64）只能证明本地不稳定；`windows-latest` 的 runners 是否会同样偶发无法在本机证明。

建议修法（未执行）：把 socket/accept 的等待预算与 xUnit 的 `Timeout` 解耦——例如给 `SingleReplyAdbServer` 显式短超时并在超时后立即 `ThrowOperationCanceledException` 于自身，或改用 `TaskCompletionSource` 的信号等待而非墙钟轮询，使超时只影响本用例、不再连带取消同类并行的其它用例。

## 4. 配置切换的既有约束（复核结论）

CI 门禁走 Release 后若要回到日常 Debug 构建，必须用 `dotnet restore UotanStudio.slnx -p:Configuration=Debug` 重新还原；否则 `Dock.Avalonia.Diagnostics` 的 `IncludeAssets` 条件求值为空、`#if DEBUG` 里的 `AttachDockDebugOverlay` 会以 `CS1061` 失败。该约束已写在根 `AGENTS.md`，此处只记录本轮实测确认其仍然成立（复原后 Debug 构建 0 警告 0 错误）。

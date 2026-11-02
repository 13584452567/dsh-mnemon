---
id: "b4c0eb13-4c71-4066-b8eb-d5e87d096ba1"
title: "UotanStudio NativeAOT 发布瘦身与本地构建加速（2026-10-27）"
description: "ILCompiler 原生 PDB 的关闭条件（DebugType=none）、去掉命令行 PublishAot 的依据、项目级构建实测耗时，以及发布不稳定等已知陷阱与未验证边界。"
status: "active"
created_at: "2026-10-26T21:39:09.961Z"
updated_at: "2026-10-26T21:39:09.961Z"
content_hash: "28cf882febb9fe05c6d07b9c17cae177b8176b5ff0c5266425cddb5cd5137272"
source_paths:
  - "UotanStudio.Desktop/UotanStudio.Desktop.csproj"
  - ".github/workflows/release.yml"
  - "AGENTS.md"
  - "UotanStudio.Desktop/AGENTS.md"
session_ids:
  - "89eb8286-0db6-42ea-8633-4b1cde99c3cb"
memory_body_ids:
  []
---

# UotanStudio NativeAOT 发布瘦身与本地构建加速（2026-10-27）

本轮把本地发布从"产出 482 MB 原生 PDB"改为"只出可执行文件"，并去掉命令行上冗余的 `-p:PublishAot=true`、把日常宿主/固件构建收敛到项目级。本文记录**为什么这样改**（ILCompiler 目标文件里的实际条件）、**实测数字**，以及本轮观察到但未修复的既有问题，供后续维护与 CI 调整参考。

## 1. 原生 PDB 的来源与关闭方式

NativeAOT（Microsoft.DotNet.ILCompiler）自己推导符号开关，不读 `DebugSymbols`：

| 文件:行 | 条件 |
| --- | --- |
| `Microsoft.NETCore.Native.targets:25` | `NativeDebugSymbols = true` 当且仅当 `DebugType != 'none'` 且非空 |
| `Microsoft.NETCore.Native.targets:246` | 给 ilc 加 `-g` 的前提是 `NativeDebugSymbols == 'true'` |
| `Microsoft.NETCore.Native.targets:252` | `--sourcelink:` 同样只在 `NativeDebugSymbols` 为 true 时附加（同时需要 `EnableSourceLink`、win） |
| `Microsoft.NETCore.Native.Windows.targets:91, 97` | `/SOURCELINK:` 与 `/DEBUG` 链接参数都受 `NativeDebugSymbols` 门控 |
| `Microsoft.NETCore.Native.targets:406, 410` | 非 Windows、非 Apple 的 strip 步骤也看这两个属性 |

因此唯一有效开关是 **`DebugType=none`**。落地方式是在 `UotanStudio.Desktop/UotanStudio.Desktop.csproj` 顶部加一个 Release 专属属性组：

```xml
<PropertyGroup Condition="'$(Configuration)' == 'Release'">
  <DebugType>none</DebugType>
  <DebugSymbols>false</DebugSymbols>
</PropertyGroup>
```

效果经实测确认：`obj/<cfg>/<rid>/native/` 下的 `UotanStudio.Desktop.ilc.rsp` 不含 `-g`，`link.rsp` 不含 `/DEBUG`，发布目录不再出现 482,512,896 B 的 `UotanStudio.Desktop.pdb`（linked 阶段仍会生成 `.obj`、`.def`、`link.rsp`）。

## 2. 实测数字（本机 win-x64）

| 项 | 改动前 | 改动后 |
| --- | --- | --- |
| MSBuild `Exec` 阶段（ilc + 链接） | 349.8 s | 237.7 s（−112 s，−32%） |
| 发布墙钟 | 353 s | 305.9 s（含托管重编）／378.9 s（清空 `native/` 全冷） |
| `UotanStudio.Desktop.exe` | 155,932,672 B | 155,977,728 B |
| 原生 `UotanStudio.Desktop.pdb` | 482,512,896 B | 不产出 |

本轮参考构建的其它实测值：托管阶段 `Csc` 72.5 s / 13 次调用；fresh `dotnet restore UotanStudio.slnx -r win-x64 -p:Configuration=Release` 3.7 s。

## 3. `-p:PublishAot=true` 为什么可以去

`PublishAot` 已由 `UotanStudio.Desktop.csproj` 声明；命令行传入会作为**全局属性**渗透到所有被引用的项目。去掉后 fresh restore（`-r win-x64`）仍会解析 `Microsoft.DotNet.ILCompiler`，ilc 真实执行、exe 约 156 MB，**没有**回退成托管 apphost。`release.yml` 的 Publish 步骤保留 `-p:DebugType=none`、`-p:DebugSymbols=false` 作为显式契约（与 csproj 默认一致），RID/self-contained/Version 参数原样保留。同步更新了根 `AGENTS.md` 与 `UotanStudio.Desktop/AGENTS.md` 里的发布命令示例。

## 4. 日常构建走项目级而不是全量 slnx

| 命令 | 覆盖面 | 实测 |
| --- | --- | --- |
| `dotnet build UotanStudio.Desktop/UotanStudio.Desktop.csproj -c Debug` | 宿主 + 全部库（约 12 个项目） | 37.1 s |
| `dotnet build UotanStudio.Firmware/UotanStudio.Firmware.csproj -c Debug` | 只到固件层 | 18.8 s |
| 全量 `UotanStudio.slnx` 冷构建 | 再加 Tests（650 源文件）+ Benchmarks | 49–59 s |

全量 slnx 多出的尾段就是 `UotanStudio.Tests` 与 `UotanStudio.Benchmarks`；提交前与最终验证仍按全量 slnx + `dotnet format` + 测试。没有引入 `.slnf`（`.slnf` 叠 `.slnx` 的 SDK 支持面未验证，不在本轮赌）。

Debug 构建必须显式 `-c Debug`、还原带 `-p:Configuration=Debug`：`UotanStudio.csproj` 中 `Dock.Avalonia.Diagnostics` 的 `IncludeAssets` 按 `$(Configuration)` 求值，还原阶段属性为空会剔除编译资产，导致 `#if DEBUG` 的 `AttachDockDebugOverlay` 以 `CS1061` 失败。

## 5. 已知陷阱（本轮观察到，未修复）

- **发布不稳定**：8 次尝试多次失败。症状包括：残留 `ilc.exe`（3.4–3.8 GB 工作集）占着 `native/*` 文件；一次 `CS2012`（`VBCSCompiler` 占着 `obj/.../win-x64/UotanStudio.dll`）；一次 ilc 无诊断 exit 1。发布前 `Stop-Process ilc` + `dotnet build-server shutdown` 可立即绕开。
- **`publish/` 仍有约 4 MB 托管 PDB**（各库 Release 编译产物）。MSBuild `Copy` 保留时间戳，旧文件不会自动清掉；`release.yml` 打包时本来排除 `*.pdb`，本轮只去掉了原生 PDB。
- **删除被占用的旧 PDB 要显式报错**：`Remove-Item -Recurse -Force` 对锁定的 482 MB PDB 会静默失败（表现为"删了还在"），需 `-ErrorAction Stop` 或先杀进程。
- 先前结论复测仍成立：关闭 Trim/AOT 分析器对 `UotanStudio.Firmware -t:Rebuild` 无收益（13.97 s vs 13.99 s），因此未动分析器配置。

## 6. 未验证边界

- 其余 5 个 RID（win-arm64、linux-x64、linux-arm64、osx-x64、osx-arm64）的 `DebugType=none` 行为与非 Windows strip 分支未实测。
- CI 上的实际耗时未测量（仅本机 win-x64）。
- 仓库 `VERIFICATION.md` 是按轮次记录的实测日志，本轮未更新。

## 涉及文件

- `UotanStudio.Desktop/UotanStudio.Desktop.csproj`（Release `DebugType/DebugSymbols`）
- `.github/workflows/release.yml`（Publish 步骤参数）
- `AGENTS.md`、`UotanStudio.Desktop/AGENTS.md`（构建/发布命令与项目级构建说明）

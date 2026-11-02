---
id: "94b8e112-b244-4a3d-8376-549a9d84dcd0"
title: "设备 Shell 终端改为进程内 IPtyConnection：约 10 秒静默期根因与共享会话复用（2026-10-16）"
description: "GUI 设备 shell 前 ~10 s 无可见输出的时间线拆解（设备层初始化/全量扫描/TLS 握手）、被排除的抢占与 PTY 假设、进程内 DeviceShellConnection 适配 IPtyConnection 的架构与接线链、验证数字与未验证边界。"
status: "active"
created_at: "2026-10-26T20:15:28.062Z"
updated_at: "2026-10-26T20:15:28.062Z"
content_hash: "a3ed172144e6fc81bdabfe0bb3c9a88b8b980cbeb5883b6dc8f245bc778f3738"
source_paths:
  - "UotanStudio/Services/Terminal/DeviceShellConnection.cs"
  - "UotanStudio.Desktop/Program.cs"
  - "UotanStudio/Services/Terminal/DeviceShellService.cs"
  - "UotanStudio.Tests/Devices/DeviceShellConnectionTests.cs"
session_ids:
  - "184d2d60-517b-4f6c-b0ba-cad632b044b6"
memory_body_ids:
  []
---

# 设备 Shell 终端：从“子进程 PTY”改为“进程内 IPtyConnection”（2026-10-16）

## 症状

GUI 里点“打开设备 Shell”后，终端面板**前约 10 秒完全没有可见字符**（ConPTY 初始化序列不可见），用户判断为“连接失败/无输出”。宿主的 `--managed-device-shell <Adb|Hdc> <serial>` 入口本身能成功拿到 `dada:/ $` 提示符。

## 根因度量（PTY 探针实测时间线，旧实现）

```
+148ms  ← ConPTY 初始化 ESC[1t ESC[?1004h ESC[?9001h（不可见）
+9850ms ← dada:/ $    ← 第一行可见输出
```

代价拆分为三段：`DeviceMonitorService.Create()`（设备层/USB 后端初始化）4–5 s + `Scan()`（子进程重新全量扫描设备）~2 s + TLS/ADB 握手并拉起 `shell:` ~3 s。

已排除的假设：不是无线调试单连接抢占（并行两个进程同时连同一 host:port 都能拿到提示符）；不是 Porta.Pty 替换环境变量导致（最小环境集合下仍可工作）；ConPTY 下 GUI 子系统的 stdout/stderr 正常。

## 第一版修复（保留）

`UotanStudio.Desktop/Program.cs` 的 `--managed-device-shell` 分支在每个阶段立即 `Report()` 一行进度（失败原因也写入终端，同时保留 stderr + 退出码 1 的既有契约）：

```
+408ms  [UotanStudio] Opening the Adb shell for <serial>.
+5576ms [UotanStudio] Resolving the target; a full device scan runs in this process.
+8019ms [UotanStudio] Target resolved; opening the interactive shell.
+11369ms dada:/ $
```

## 第二版：进程内方案（当前架构）

新增 `UotanStudio/Services/Terminal/DeviceShellConnection.cs`：把 `DeviceMonitorService.RunInteractiveShellAsync` 的转发包装成 `Porta.Pty.IPtyConnection`，因此 `TerminalView` 侧仍是 `AttachConnection(connection)`，视图与泵代码不变。

- 两个方向各一条**带背压**的 `ShellByteChannel`（`Channel<byte[]>`，64 块上限，写满即等待），替代 PTY 管道。
- 转发结束/失败：先向终端写入 `[UotanStudio] <原因>`，再收尾；退出码通过新增 `Exited` 事件上报。**不能**触发 `IPtyConnection.ProcessExited`：`Porta.Pty.PtyExitedEventArgs` 的唯一构造函数是 `internal`（`ctor internal (Int32)`，反射确认），无法从外部构造——两次尝试后改用自有事件。
- `Kill()`/`Dispose()` 取消转发并完成两条通道，读端返回 0；`Resize` 空实现（裸 `shell:` 没有尺寸协议，旧方案同样没有）。

接线链：`DeviceViewModel` → `DeviceShellService.OpenInteractiveShell` → `WorkspacePanelService.OpenDeviceShell` → `TerminalDockService.CreateDeviceShell(device)` → `TerminalViewModel`/`TerminalSessionViewModel`（不再走 `PtyProvider`，改 `DeviceShellService.CreateConnection(App.DeviceMonitor, device)`）。设备 shell 的工具栏配置退化为纯标签（不再拼 `--managed-device-shell` 命令行），并加 `CloneProfile` 守卫，避免在新终端里把宿主自身再启动一次。命令行入口保留。

收益：同一进程、复用宿主**共享 ADB 会话**，没有进程启动、没有二次全量扫描、没有第二条竞争传输；**直连 USB 目标也能开 shell**（旧子进程方案在宿主独占句柄下根本拿不到传输）。

## 验证

- `dotnet build UotanStudio.slnx -c Debug`：0 警告 / 0 错误。
- `DeviceShellConnectionTests`（回显+退出码、失败写原因+退出码 1、Kill 收尾）：3 通过 / 1 跳过（未设 `UOTAN_DEVICE_SHELL_TARGET`）。
- 真实设备（无线调试 host:port 经环境变量注入）：进程内连接开壳、写 `echo uotan-shell-ok`、读回输出 → 通过（13 s）。
- `DeviceShellServiceTests` 13/13、`UotanStudio.Tests.Architecture` 16/16 通过。
- `dotnet format … whitespace/style --verify-no-changes`：改动文件干净（注意 `.editorconfig` 的 `insert_final_newline = false`，曾报 FINALNEWLINE）。

## 未验证 / 待办

- Avalonia 界面真实点击后的渲染与输入回显未人工冒烟（仓库规则要求 Dock/XAML 改动人工验证）；不能把编译通过写成界面已验证。
- 未跑 NativeAOT 发布；新代码无反射、无动态注册。
- `UotanStudio.Desktop/AGENTS.md` 第 17 行“`CreateDeviceShellProfile` 传三个参数”的说法已过时；该文件写入被拒（`cannot modify`），需人工更新。
- 终端不能改窗口尺寸（ADB 裸 shell 无 resize 协议）。

## 环境备注

- 编辑 `E:\UotanStudio` 下部分文件时 `ReplaceFileW EIO (Win32 1175)` 偶发失败，需重试写入（重试后成功）。

## 相关文档

- `dddf19d9-142d-4ea5-a267-24369c23a9f0`：ADB 交互式 shell 的 5 秒空闲读期限击穿托管 TLS 记录层；本方案的 `RunInteractiveShellAsync` 复用链路依赖该修复后的共享会话。

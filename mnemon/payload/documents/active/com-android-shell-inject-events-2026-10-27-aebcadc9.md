---
id: "aebcadc9-f15a-4be7-97f2-2d7676ce4f5a"
title: "投屏输入注入预检（com.android.shell INJECT_EVENTS）+ 指针捕获触摸注入（2026-10-27）"
description: "scrcpy 投屏“触摸/按键被设备拦截”的只读预检契约（三态判定、复用共享 ADB 会话、3 秒节流重检、标题栏橙色提示）、鼠标指针捕获式触摸注入与本地化键，含验证数字与真机未验证边界。"
status: "active"
created_at: "2026-10-26T18:15:45.668Z"
updated_at: "2026-10-26T18:15:45.668Z"
content_hash: "da60426434cc261d0da06c1636f1c27f1e80d9c18188d5ce64ddc70cc155de17"
source_paths:
  - "UotanStudio/Services/Devices/InputInjectionProbe.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Operations.cs"
  - "UotanStudio/ViewModels/Editors/ScreenMirrorViewModel.cs"
  - "UotanStudio/Views/Editors/ScreenMirrorView.axaml"
  - "UotanStudio/Views/Editors/ScreenMirrorView.axaml.cs"
  - "UotanStudio/Locale/Common.resx"
  - "UotanStudio/Locale/Common.zh-CN.resx"
  - "UotanStudio/Locale/LocaleKeySets.cs"
  - "UotanStudio.Tests/Devices/InputInjectionProbeTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "f7f35dee-5a0c-4b27-95fb-7003def74c16"
memory_body_ids:
  []
---

# 投屏输入注入预检与指针捕获触摸注入（2026-10-27，Windows x64 / Debug）

## 要解决的问题

部分国产 ROM（例如 ColorOS）默认不授予 `com.android.shell` 的 `INJECT_EVENTS`，scrcpy 会话能建立、视频正常，但触摸与按键全无反应，用户只看到“投屏卡住”而没有任何解释。宿主需要在不重启会话、不抢设备端点的前提下，判定“注入被设备拦截”，并给出可操作的开关位置。

## 预检契约（`Services/Devices/InputInjectionProbe.cs`）

只读命令：

```text
dumpsys package com.android.shell | grep -E 'Package \[|INJECT_EVENTS'
```

三态判定（必须可区分“确认被拦”与“探测失败”，否则会误报）：

- 输出中必须出现 `Package [com.android.shell]` 才认为探测有效；缺包标题（命令失败、输出为空、grep 无匹配）→ 返回**未知**，不显示提示。
- `android.permission.INJECT_EVENTS: granted=true` → 可用。
- `granted=false`、只有 requested 行、或权限行完全缺失 → 判为**被拦**：shell 没申请或没拿到权限，两种情况都无法注入。

## 会话内复用与界面提示

- `DeviceMonitorService.Operations.ProbeInputInjection`（`DeviceMonitorService.Operations.cs:242`）走 `RunAdb`：投屏占用直连会话时**借用同一个共享 ADB 会话**（server 路由时用 server 会话），不另开传输、不抢端点。这与既有“端点独占 + `AcquireForMirror` 有界等待”的取舍一致（见 `13d2945f-9a23-4bc7-8aa9-ac82e702b5c8`）。
- `ScreenMirrorViewModel`（`ScreenMirrorViewModel.cs:175` 起）：投屏开始后且 `control=true`、会话为 scrcpy 时探测一次；被拦时在状态栏下方显示橙色（`LogWarnBrush`）提示条（`ScreenMirrorView.axaml:24`）。
  - 中文：设备未授予注入权限，请在开发者选项开启“USB 调试（安全设置）”，然后重新点击画面。
  - English：Touch and key control are blocked by the device: enable “USB debugging (Security settings)” in Developer options, then tap the picture again.
- 提示可见期间每次触摸以 **3 秒节流**在后台重跑预检（`RecheckInputInjection`，`ScreenMirrorViewModel.cs:301`）：用户打开开关后下一次点击即清除提示，**不需要重启投屏会话**。停止会话、切换设备、关闭文档都会清掉提示状态。

## 指针捕获式触摸注入

`ScreenMirrorView.axaml.cs` 的鼠标/触摸路径改为显式指针捕获，避免拖动被别的控件截走：

- `OnPointerPressed` 记录 `_pressedPointer` / `_pressedButton` / `_lastTouch` 并 `Capture`，按下即发 `ActionDown` 与按钮掩码；
- `OnPointerMoved` 只在指针仍是捕获者时发 `ActionMove`，帧尺寸（`width`/`height`）变化时补发一次；
- `OnPointerReleased` 校验按钮一致后发 `ActionUp`；
- `OnPointerCaptureLost` 走 `EndGesture(cancel: true)`，用 action `3` 取消手势，防止设备端遗留按下状态。
- `ScreenMirrorViewModel.InjectTouch` 的压力值按动作区分：`ActionUp`/取消为 `0`，其余为 `ushort.MaxValue`；`actionButton` 只在按下/抬起时非零。

## 本地化

`Common.resx` 与 `Common.zh-CN.resx` 各新增 `ScreenMirror_InputInjectionBlocked`（索引约 456 行），并同步生成式键集合 `Locale/LocaleKeySets.cs:143`；`LocaleDictionaryTests` 会校验中英键集合与生成索引一致。

## 验证（Debug，主树工作区）

- 新增 `UotanStudio.Tests/Devices/InputInjectionProbeTests.cs`：7/7（granted、denied、仅 requested、其它权限 granted、缺包标题、空输出）。
- `LocaleDictionaryTests` 6/6、`MirrorSessionFacadeTests` 13/13、`UotanStudio.Tests.Devices` 305/305、`Protocols` 112/112。
- 全量：**7579 总数 / 7573 通过 / 6 跳过 / 0 失败**（2m43s，跳过项为既有环境门控，含 `UOTAN_HDC_MIRROR_DIRECT` 硬件用例）。
- Debug 全解构建 0 警告 0 错误；`dotnet format` whitespace/style 对 `UotanStudio.Comm`、`UotanStudio.Tests` 与本轮宿主文件零报告。注意根 `.editorconfig` 是 `insert_final_newline = false`，新增文件末尾多一个换行会报 `FINALNEWLINE`。
- NativeAOT publish（Release / win-x64 / self-contained / `PublishAot=true`）成功，仅剩第三方既有 IL2026/IL2104/IL3053 警告（LTRData.Extensions、YamlDotNet、Common.Logging）。

## 未验证边界

- 提示条本身未在受限真机（ColorOS）上人工确认，只由单元测试覆盖解析规则。下次连接受限制设备时应观察：标题栏出现橙色提示；打开“USB 调试（安全设置）”后点一下画面，提示消失且触摸生效。
- 若设备上 `dumpsys package com.android.shell | grep INJECT_EVENTS` 本身输出为空，预检按“未知”处理并保持静默，这是刻意的误报抑制，不是漏检。

## 相关文档

- 端点独占与投屏启动有界等待：`13d2945f-9a23-4bc7-8aa9-ac82e702b5c8`
- ohscrcpy 投屏重写与控制通路：`fe4bd5e7-924b-45ec-aa91-975d63f3b737`
- 投屏断流与 HDC 连接故障隔离：`64a77aaf-9665-44d1-8b37-0bfd4194e9e2`

---
id: "44550dcf-0e22-40ac-b0f2-ea4aac9f0e48"
title: "UotanStudio 启动窗改造：字体解压可见化与阶段状态（2026-10-24）"
description: "启动窗把内嵌 MiSans 的 zstd 解压显示为第一个阶段、启动窗改用平台默认字体的取舍、进度条恒为不确定态、11 段启动阶段文本的修正，以及运行时冒烟 / 全量测试 / NativeAOT 实测。"
status: "active"
created_at: "2026-10-23T20:07:15.095Z"
updated_at: "2026-10-23T20:07:15.095Z"
content_hash: "d4b4194d7cd3e5bde3e030ebf8531f60a24bd109d17637e02d8576069f5fb0f2"
source_paths:
  - "UotanStudio/ViewModels/Shell/SplashViewModel.cs"
  - "UotanStudio/Views/Shell/MainSplashWindow.axaml"
  - "UotanStudio/App.axaml.cs"
  - "UotanStudio/Fonts/ZstdEmbeddedFontCollection.cs"
  - "UotanStudio/Fonts/MiSansFontData.cs"
  - "UotanStudio/Locale/Common.resx"
  - "UotanStudio/Locale/Common.zh-CN.resx"
  - "UotanStudio/Locale/LocaleKeySets.cs"
  - "UotanStudio/AGENTS.md"
  - "VERIFICATION.md"
session_ids:
  - "adf916ae-2c03-4671-928e-abb72dbd611e"
memory_body_ids:
  []
---

# UotanStudio 启动窗改造：字体解压可见化与阶段状态（2026-10-24）

把内嵌资源（MiSans 字体的 zstd 帧）的解压从 `App.Initialize`（启动窗出现之前，用户完全看不到）移到启动序列的第一个可见阶段，并修正启动期间"状态文本与真实进度不一致"的多处问题。前置的体积账、字体原型与约束见文档 `ccc1063c-92f3-4a7b-98cf-bc921c0c4b23`；本文只记录落地后的形态与实测。

## 启动阶段顺序（当前契约）

```
启动窗出现
→ 正在解压内置资源…      （MiSans Regular + Bold，两张字重约 187 ms）
→ 正在读取设置…
→ 正在应用主题、语言和服务设置…
→ 正在扫描设备…
→ 发现 N 台设备 / 未发现设备   （复用原闲置的 Splash_FoundDevices，两块分支文本都验证过）
→ 正在启动应用服务…
→ 正在准备编辑器…
→ 正在构建工作区…
→ 正在准备主界面…
→ 正在打开启动文件…
→ 初始化完成
```

顺序规则的权威描述在 `UotanStudio/AGENTS.md`（启动顺序小节），改启动流程时先看那份再动 `SplashViewModel`。

## 字体"鸡生蛋"问题与取舍

全局样式把每个控件的 `TextElement.FontFamily` 设为 `fonts:UotanStudio/MiSans#MiSans`，而字体恰恰要到启动窗显示之后才解压。直接沿用该样式会先解析一个空的 `fonts:` 集合，落到回退字体并留下不一致的首帧。

处理方式：

- `SplashViewModel.SplashFontFamily` 用 `FontManager.TryMatchCharacter(...)` 取平台默认字体（本机 `Segoe UI`），作为启动窗三个自绘文本块的**本地值**覆盖全局样式；`fonts:` 家族不出现在启动窗上（Ursa 模板自带标题仍用主题字体 `Inter`）。
- 字体注册失败不再把用户困在启动窗：`App` 侧 try/catch 只写 `LogService` 错误日志并回退平台字体，启动序列继续。参考实现见 `SplashViewModel.cs` 的 `SplashFontFamily` 与 `App.axaml.cs` 中 `SplashPhase.Resources` 分支。

**不可兼得的取舍**：启动窗整个生命周期都用系统字体，这是让"解压步骤"可见的前提；若要启动窗也用 MiSans，只能把解压移回启动窗之前，此阶段就无法显示。主界面及之后所有文本仍是 MiSans。

## 进度条与状态文本修正

- `MainSplashWindow.axaml` 只保留一条 `BusyProgress`，`IsIndeterminate="True"`、`Value=0`、`Maximum=100`、可见；删掉了 `Value="100" Maximum="100"` 的"完成态"第二条进度条与 `IsInitializing` 绑定。整个启动期间恒为加载动画，不设目标进度。
- 修掉的不一致：`ApplyingSettings` 原本被推进两次；`McpExposureService.Apply` 落在无文本窗口期；主窗预构建失败时永远到不了"初始化完成"；`Services/Nodes/Workspace` 原写实现细节（"正在创建工作区布局"）改为用户可读的阶段描述。

## 实测（Debug / win-x64）

- 运行时冒烟（真实 `App` + Win32/Skia 宿主，未跑消息循环）：`SplashFontFamily='Segoe UI'`；5 个文本块中三个自绘块字体均为 `Segoe UI`；`BusyProgress` 属性符合预期；上面 11 段状态文本按序逐条匹配，`发现 3 台设备` 与 `未发现设备` 两分支都通过；"custom family never resolved into splash text: True" 作为回归断言保留在冒烟程序里。
- 全量测试：7229 总数 / 7222 通过 / 7 跳过 / 0 失败，约 1m26s。
- NativeAOT：`dotnet publish UotanStudio.Desktop/UotanStudio.Desktop.csproj -c Release -r win-x64 --self-contained -p:PublishAot=true` 成功，`UotanStudio.Desktop.exe` 154,245,120 B（147.1 MiB），仅第三方既有 IL2026/IL2104/IL3053 警告；`正在解压内置资源`、`未发现设备`、`Splash_Resources`、`BusyProgress` 均可在 exe 内检索到。
- 格式门：改动文件 `whitespace` / `style` 两道零报告；`LocaleDictionaryTests` 6/6。

## 环境坑与未验证边界

- 本机若有 `UotanStudio.Desktop` 实例在跑，全解构建的 Desktop 复制步骤会被输出目录锁挡住（MSB3021/MSB3027，"文件被 UotanStudio.Desktop (17964) 锁定"）。Host 与 Tests 可分别构建；这是环境冲突，不是代码错误。
- 真实窗口首帧观感（进度动画、字体观感）与整体启动时长没有自动化覆盖，按仓库规则不得写成"界面已验证"；启动窗无自动化 UI 测试，运行时冒烟是替代证据。
- 冒烟用的临时工程放在主机 `%TEMP%\uotan-splash-smoke`，验证完应删除，不要留在仓库。
- 启动文件失败只记日志、不阻断主窗口的既有策略未改。

---
id: "10a83a9a-0d60-45c3-a914-e4889ba4749c"
title: "Flash 烧录页忙碌态控件启用策略：范围收窄到侧边操作区（UotanStudio）"
description: "Flash 页 IsBusy 启用策略的根因、唯一名单 BusyDisabledControls、SetControlEnabled/SetBusy 的不变量、回归测试要求与验证记录；含构建期 pdb/apphost 文件锁的处理顺序。"
status: "active"
created_at: "2026-11-02T01:10:57.416Z"
updated_at: "2026-11-02T01:10:57.416Z"
content_hash: "baf0c950dc9f442fde3896970c59046f6f764117dde8d0d2e09858efe5f2b95f"
source_paths:
  - "UotanStudio/ViewModels/Flash/FlashDeviceUiPolicy.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Gating.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Root.cs"
  - "UotanStudio/Views/Shell/FlashView.axaml"
  - "UotanStudio.Tests/Flash/FlashDeviceUiPolicyTests.cs"
session_ids:
  - "8b9c88d9-eb8b-48e9-8a2f-9e6cd6e70046"
memory_body_ids:
  []
---

# Flash 烧录页忙碌态控件启用策略（UotanStudio）

## 1. 背景与回归根因

上一版把忙碌态（`IsBusy`）实现成"整页具名控件全量禁用"：`ApplyBusyUiGate` 遍历 `GetLogicalDescendants()` 关闭所有具名控件，其中包含整页滚动容器 `PageScroll`。
退出忙碌时只能恢复"曾经记录过启用意图"的控件（`_controlIntent`），而 `PageScroll` 等大量控件从未记录意图，于是永久停在禁用状态——表现为"获取设备信息结束后整个界面仍然禁用、无法滚动"。

结论性教训：**忙碌态不能对未登记意图的控件做批量禁用**，"禁用即记录、恢复即写回"必须严格配对。

## 2. 现行策略（唯一事实来源）

- `UotanStudio/ViewModels/Flash/FlashDeviceUiPolicy.cs`：`BusyDisabledControls` 是唯一的忙碌禁用名单，只覆盖**侧边操作区**：
  - 通用动作：加载引导、获取设备信息、回读、烧录、擦除、切换槽位；
  - 刷写选项：6 个复选框 + AVB 选项面板 + Root 选项面板；
  - 二级菜单：EDL / MTK / 瑞芯微 / 镜像 四个面板。
  - `AlwaysEnabledControls` = `CancelFlashButton` / `LogBox` / `ClearLogButton`，`SetControlEnabled` 直接把意图钳为 `true`，任何流程都无法关闭。
- `UotanStudio/ViewModels/Flash/FlashViewModel.Gating.cs`：`SetControlEnabled` 是唯一的启用写入口，判定为
  `control.IsEnabled = !(_busy && FlashDeviceUiPolicy.IsBusyDisabled(name)) && enabled && IsOfferedByDevice(name)`；
  `RefreshDeviceUiGate`（换设备重算）也只对名单内控件施加忙碌限制。整页 `GetLogicalDescendants()` 遍历已删除。
- `UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs` 的 `SetBusy`：忙碌时按名单逐个写回，退出忙碌时对名单逐个恢复并调用一次 `UpdateModeVisibility()` + `FlushPendingLog()`，使运行期间改变的包/选中状态立即反映到界面。

由此得到的不变量：**忙碌期间只有侧边操作区被关闭；`PageScroll`、刷机包栏、分区表与日志区完全不被触碰，滚动与日志在运行中始终可用。**

## 3. 回归测试

`UotanStudio.Tests/Flash/FlashDeviceUiPolicyTests.cs`：
- `BusyDisabledControls_KeepThePageScrollableAndTheLogUsable`：断言 `PageScroll`/`LogBox`/`ClearLogButton`/包栏/分区表**不在**忙碌名单，六个通用动作与四个面板**在**名单内。
- `BusyDisabledControls_ListEveryControlOnceAndEachExistsInTheView`：逐个核对名单中的名字确实存在于 `UotanStudio/Views/Shell/FlashView.axaml`，防止改名后静默失效。

新增控件时的要求：先加进 `BusyDisabledControls`（若属于侧边操作区）并同步上述两条测试，否则名单外控件在忙碌期不受限、名单内拼错名则静默失效。

## 4. 验证记录

| 门禁 | 结果 |
| --- | --- |
| `dotnet build UotanStudio.Desktop/UotanStudio.Desktop.csproj -c Debug` | 0 错误 / 0 警告 |
| `FlashDeviceUiPolicyTests` | 21/21 通过（新增 2 条回归） |
| `--filter-namespace UotanStudio.Tests.Flash` | 93/93 通过 |
| `LocaleDictionaryTests` | 6/6 通过 |
| `dotnet format` whitespace + style | 无差异 |

未验证边界：真实 usbplug 设备上的界面点验（禁用观感、运行中侧边面板滚轮滚动）无自动化覆盖。

## 5. 构建期文件锁（环境经验）

改动后连续构建时可能遇到 `MSB3021/MSB3027`（`apphost.exe` → `UotanStudio.Tests.exe` 被残留测试宿主锁定）或 `CS2012`（`obj/Debug/net10.0/*.pdb` 被 `.NET Host` 锁定）。处理顺序：先确认占用进程（`Get-Process` / `Get-CimInstance Win32_Process` 按 `StartTime` 与 `CPU` 判断是否为陈旧句柄），必要时结束残留宿主，再 `dotnet build-server shutdown` 关闭 VB/C# 编译器与 MSBuild 服务器后重建；不要通过删 `obj`/`bin` 或加 `NoWarn` 绕过。

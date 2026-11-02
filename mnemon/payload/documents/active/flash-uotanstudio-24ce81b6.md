---
id: "24ce81b6-1a5a-4a89-be15-0fb27385a294"
title: "Flash 页忙碌态门控补正：取消按钮归属、面板继承与处理器守卫审计（UotanStudio）"
description: "更正 CancelFlashButton 不属于 AlwaysEnabledControls、忙碌名单的面板继承效应、无 _busy 守卫处理器清单、SetBusy 配对语义与验证/未验证边界；基线策略见文档 10a83a9a。"
status: "active"
created_at: "2026-11-02T01:41:59.855Z"
updated_at: "2026-11-02T01:41:59.855Z"
content_hash: "0f6c8f21ef4bc3f9f1a1666aba6f670741c2686a875bce287323b6fbe412c064"
source_paths:
  - "UotanStudio/ViewModels/Flash/FlashDeviceUiPolicy.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Gating.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs"
  - "UotanStudio/Views/Shell/FlashView.axaml"
  - "UotanStudio/Views/Shell/FlashView.axaml.cs"
  - "UotanStudio.Tests/Flash/FlashDeviceUiPolicyTests.cs"
session_ids:
  - "a19f198d-95d8-49e5-bf32-ad5ec0614900"
memory_body_ids:
  []
---

# Flash 页忙碌态门控：取消按钮归属补正与处理器守卫审计（UotanStudio）

本文只记录相对既有策略文档 **10a83a9a-0d60-45c3-a914-e4889ba4749c**（«Flash 烧录页忙碌态控件启用策略»）的**更正与补充**，不重复其全文。基线结论仍然有效：`FlashDeviceUiPolicy.BusyDisabledControls` 是唯一忙碌禁用名单，`SetControlEnabled` 是唯一启用写入口，判定为
`control.IsEnabled = !(_busy && FlashDeviceUiPolicy.IsBusyDisabled(name)) && enabled && IsOfferedByDevice(name)`。

## 1. 更正：`CancelFlashButton` 不属于常启用集合

- 旧行为：`AlwaysEnabledControls` 曾包含 `CancelFlashButton`，而 `SetControlEnabled` 在入口把常启用条目的意图钳为 `true`，于是**空闲态取消按钮也是可点的**——XAML 里写的是 `IsEnabled="False"`，但任何一次 `SetBusy(false)` 走完后它就被点亮，与"启用状态跟随 busy"的原始意图直接冲突。
- 现行：`AlwaysEnabledControls` 只保留日志区 `LogBox` / `ClearLogButton`（`UotanStudio/ViewModels/Flash/FlashViewModel.Gating.cs:79`）；`CancelFlashButton` 的启用完全由 `SetBusy` 独占写入（依据 `_flashCancellation` 是否为空），布局声明在 `UotanStudio/Views/Shell/FlashView.axaml:351`，按钮点击走 `OnCancelFlashClick`。
- 回归锁：`AlwaysEnabledControls_CoverTheLogAreaAndNeverTheCancelEntry`。任何"把取消按钮加进常启用集合"的改动都会被这条测试拒绝。

判定口径：**常启用集合只能装日志区这类与运行无关的控件；凡是状态机驱动的按钮（取消、烧录、擦除、切换槽位）都必须留在`SetBusy`/`RefreshDeviceUiGate` 的可写范围内。**

## 2. 补充：忙碌名单的作用域与"父面板继承"效应

`BusyDisabledControls`（`UotanStudio/ViewModels/Flash/FlashDeviceUiPolicy.cs:130`）共 18 项：

| 分组 | 控件 |
| --- | --- |
| 通用动作（6） | `LoadBootButton`、`ReadInfoButton`、`ReadbackSelectedButton`、`FlashSelectedButton`、`EraseSelectedButton`、`SwitchSlotButton` |
| 刷写选项（8） | `DisableVbmetaBox`、`AddRootBox`、`EraseDataBox`、`RebootAfterBox`、`ShowAllPartitionsBox`、`DeleteCowBox`、`AvbOptionsPanel`、`RootOptionsPanel` |
| 二级分类面板（4） | `EdlPane`、`MtkPane`、`RkPane`、`ImagePane` |

关键事实：`ActionPane`（`FlashView.axaml:299`）内 134 个具名控件里有 **82 个位于上述四个面板内部**，父面板被禁用即整体失效，因此名单**不需要**逐个列出 `AvbDisableCommandBox`、`EdlSetBootableButton`、`EdlPartitionDigestButton` 之类的子项；同理 `AvbOptionsPanel`（span 398–406）与 `RootOptionsPanel`（span 407–425）覆盖了各自的输入框。

反向边界：`ActionPane` 中既不在名单、也不被任何禁用面板覆盖的控件，在运行期间仍然可交互（实测枚举到了 `CancelFlashButton`、`RootPartitionBox`/`RootFileSourceBox`/`RootVersionBox`/`RootPackagePathBox`/`ChooseRootPackageButton`、`DeviceStatusPanel`、`EdlExpander`）。这是有意的：设备状态面板与取消入口必须在运行中保持可用；若要进一步收紧，可把四个 `*Expander`（已在设备门控表中）加进忙碌名单，成本很低但会牺牲运行中查看分类信息的能力。

## 3. 审计：处理器侧的 `_busy` 守卫

- 侧边操作区的主要处理器都带 `_busy` 早退（`OnClearLogClick` 除外，它属于常启用日志区）。
- 仍**没有** `_busy` 守卫的 14 个处理器（`UotanStudio/ViewModels/Flash/FlashViewModel.Operations.cs`）：`OnReadbackSelectedClick:787`、`OnLoadBootClick:862`、`OnReadInfoClick:1671`、`OnFlashSelectedClick:1850`、`OnEraseSelectedClick:2180`、`OnSlotResetCurrentClick:2288`、`OnSlotSetAClick:2291`、`OnSlotSetBClick:2294`、`OnEdlResetClick:3063`、`OnEdlSetBootableClick:3230`、`OnEdlPartitionDigestClick:3258`、`OnMtkSetSpeedClick:4339`、`OnMtkRebootClick:4378`、`OnMtkXmlPropertyQueryClick:4436`。
  这些触发控件全部落在忙碌名单或其父面板内，忙碌期界面已灰，双击/键盘聚焦不会造成重入；**新增这类处理器时不要依赖父面板禁用，仍应显式写 `_busy` 早退**。
- 观测注意：按 `Click="OnXxx"` 在 `FlashView.axaml` 中反查时，`OnEdlResetClick`/`OnMtkSetSpeedClick`/`OnMtkRebootClick` 查不到（它们在 `UotanStudio/Views/Shell/FlashView.axaml.cs:308/309/315` 有转发方法）。这是名称检索的盲区而非死连线的证据——核对 XAML 连线时以 code-behind 转发方法为交叉证据。

## 4. `SetBusy` 配对与恢复语义

- `SetBusy`（`FlashViewModel.Operations.cs:1507`）先遍历名单写 `!value`，再单独写 `CancelFlashButton`，退出忙碌时补一次 `UpdateModeVisibility()` 重算派生项。
- 36 处 `SetBusy(true)` / 36 处 `SetBusy(false)` 全部配平，且都发生在 `_flashCancellation` 赋值之后，因此取消按钮的可见可点状态不会出现"已恢复但令牌已清"的缝隙。

## 5. 验证记录与未验证边界

| 门禁 | 结果 |
| --- | --- |
| `dotnet build UotanStudio/UotanStudio.csproj -c Debug` | 0 错误 / 0 警告（`dotnet build-server shutdown` 后干净重建） |
| `UotanStudio.Tests` 构建 | 0 错误 |
| `FlashDeviceUiPolicyTests` | 22/22 通过（本轮新增 3 条） |
| `--filter-namespace UotanStudio.Tests.Flash` | 95/95 通过 |
| `LocaleDictionaryTests` | 6/6 通过 |
| `dotnet format` whitespace + style | 无差异 |

未验证边界：真实 usbplug 设备上的观感与交互点验仍需人工确认（禁用状态视觉、运行中在侧边面板滚动滚轮、取消按钮在空闲态确为不可点）。

构建期补充经验：若 `MSB3021/MSB3027` 报告 `UotanStudio.Tests.exe` 锁定输出 dll，用 `Get-CimInstance Win32_Process` 按 `CommandLine`（含 `--server dotnettestcli`）确认残留测试宿主后再结束，随后 `dotnet build-server shutdown`；不要删 `obj`/`bin` 或加 `NoWarn` 绕过。

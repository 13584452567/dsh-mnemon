---
id: "4cf0b689-be12-4ab6-b574-db9d331f500a"
title: "UotanStudio Flash 瑞芯微 loader 阶段切换、门控与全局日志路由（实测）"
description: "Flash 面板瑞芯微 maskrom→loader 两阶段读取、loader 门控维度、cfg 导入行 Replace 语义、全局日志与 Comm 插桩的门禁实测与未验证边界。"
status: "active"
created_at: "2026-10-30T17:16:59.382Z"
updated_at: "2026-10-30T17:16:59.382Z"
content_hash: "3eeaa89fea26432ff08b54883bd07429493485a2770d991a6211bd12b6f667d4"
source_paths:
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs"
  - "UotanStudio/ViewModels/Flash/FlashDeviceUiPolicy.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Gating.cs"
  - "UotanStudio/Services/LogService.cs"
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlSession.cs"
  - "UotanStudio.Tests/Flash/FlashGlobalLogRoutingTests.cs"
session_ids:
  - "c359e0d6-5b60-4e35-b82b-a35a35564b14"
memory_body_ids:
  []
---

# UotanStudio Flash 瑞芯微 loader 阶段与全局日志路由（实测）

范围：Flash 面板的瑞芯微「获取信息」路径、loader 门控、cfg 导入行处理、全局日志与 Comm 插桩。日期 2026-10-31，Windows x64，Debug，共享工作树。

## 1. maskrom 下的存储命令是本轮根因

- 未声明 loader 时，`RunRkReadInfoAsync` 只发送 maskrom 能应答的命令（CAPABILITY、READ_FLASH_ID），`session.IsMaskromMode` 为真时输出 `Flash_RkStorageNeedsLoader`，不再调用 `ReadStorage`。
- 症状对应关系：日志里 `读取芯片信息: 10000030 (bootrom)` 紧接 `数据错误(循环冗余检查)`，来源是把 CBW 存储命令（0x2B 等）送进 maskrom，只能在等满超时后报 CSW 校验失败。阶段判定本身由 `bcdUSB` bit 0（`RkdlConstants.ClassifyMode`）给出，见 VERIFICATION.md 的 RKDL 模式识别实测一节。
- 声明了 loader 时走两阶段：先在 maskrom 租约上 `session.DownloadBootLoaderAsync`，释放租约、等重新枚举，再用新会话读存储与芯片信息（`RunRkdlLoaderTransitionAsync`）；会话已是 loader 阶段则跳过下载、不重建会话。

## 2. loader 门控是独立可用性维度

- `FlashDeviceUiPolicy` 增加第二个维度 `RockchipLoaderControls`（`ReadInfoButton`、`ReadbackSelectedButton`、`FlashSelectedButton`、`EraseSelectedButton`、`RkPane`）与重载 `IsAvailable(name, mode, hasRockchipLoader)`；原有设备模式维度保持不变。
- `HasRockchipLoader => EffectiveRkLoaderPath is not null`；loader 行增删经 `RefreshRockchipLoaderGate` 立即重算，通用功能区的烧录、获取信息与瑞芯微二级菜单随之一并开关。
- 定向测试 `UotanStudio.Tests.Flash.FlashDeviceUiPolicyTests` 10/10。

## 3. cfg 导入行的两个坑

- `ApplyRkConfigRows` 原用 `Partitions.Apply(Partitions.Count, rows)`，与 `RefreshVisiblePartitions` 的 `Replace` 语义不一致：行在首次 `Reset` 前不进入 `VisiblePartitions`，导致新加行上的「选择文件路径」单元格从不出现。已改为 `Partitions.Replace(rows)`。
- 导入时跳过路径指向另一份 `.cfg` 的行（`RockchipConfigFile.LooksLike(entry.Path)`），不再把配置文件当镜像行发布。回归用例 `FlashGlobalLogRoutingTests.ImportRkConfigAsync_ConfigurationRowPointingAtAnotherConfiguration_StaysOutOfTheGrid` 断言日志作用域为 `flash` 而非 `loader`。

## 4. 全局日志与 Comm 插桩

- `AppendLog` 逐行写 `LogService`（页面日志仍按 32 行 / 4096 字符 / 100 ms 合批）；46 处 `catch` 改为 `AppendError(ex)`，输出 `{异常类型}: {消息}` 并展开内层异常。
- `LogService.ApplyDebugMode` 同时驱动 `CommTrace.Level`/`CaptureRawPayloads` 与 `UsbTrace.IsEnabled`/`CaptureFrames`；RKDL 侧在会话打开与关闭、CBW/CSW、DATA 收发、厂商请求分块、loader 下载前后、maskrom 拒绝存储命令处插桩，原始数据经 `CommTrace.Payload` 输出。
- 顺带修正 `Surface` 的 5 处 `is null` 守卫（原先调用的是会抛异常的属性），并让未附加页面的 `Field<T>` 返回分离虚拟控件。
- 新增 `UotanStudio.Tests.Flash.FlashGlobalLogRoutingTests` 4/4。

## 5. 门禁实测

- `dotnet build UotanStudio.slnx -c Debug`：0 错误 0 警告。
- `dotnet format` 的 whitespace 与 style `--verify-no-changes`：均零报告。
- `dotnet publish UotanStudio.Desktop/UotanStudio.Desktop.csproj -c Release -r win-x64 --self-contained`：成功，无指向 `UotanStudio.*` 的裁剪/AOT 警告。
- 定向：`UotanStudio.Tests.Flash` 47/47、`UotanStudio.Tests.Protocols` 175/175。
- 全量：8115 总数 / 8092 通过 / 21 跳过 / 2 失败，两条都不在本次改动文件上：
  - `HostLayeringGuardTests.Views_DoNotImportApplicationServiceNamespacesOutsideTheFrozenAllowList` —— `UotanStudio/Views/Shell/AvbToolWindow.axaml.cs` 的冻结豁免表与现状不匹配。
  - `RockusbPipeSelectionTests.SelectPipes_OutEndpointDeclaredFirst_StillMapsInToZeroAndOutToOne` —— 并行会话新增的未跟踪用例，期望 in=0/out=1，实测 in=1。
- `ErofsInodeAdviseAcceptanceRegressionTests` 首轮全量报过 1 条，单跑 8/8、复跑未复现。

## 6. 未验证边界

- 本机无瑞芯微真机：maskrom→loader 重新枚举后的实际读取只在合成设备与单元测试上验证；`UOTAN_RKDL_E2E=1` 闭环未执行。
- 配置行「选择文件路径」按钮、loader 门控的启用/禁用、全局日志窗口实时输出属真实界面行为，未启动 GUI 目视。
- 改动未提交；工作树含并行会话修改，上述两条失败用例属他人改动面。

## 7. 关键文件

- `UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs`（`RunRkReadInfoAsync`、`ApplyRkLoaderPath`、`RefreshRockchipLoaderGate`、`ApplyRkConfigRows`）
- `UotanStudio/ViewModels/Flash/FlashDeviceUiPolicy.cs`、`FlashViewModel.Gating.cs`
- `UotanStudio/Services/LogService.cs`、`UotanStudio.Comm/Protocols/Bootrom/RkdlSession.cs`
- `UotanStudio.Tests/Flash/FlashGlobalLogRoutingTests.cs`、`UotanStudio.Tests/Devices/RockusbPipeSelectionTests.cs`

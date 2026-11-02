---
id: "79f5570c-d0a2-4cdb-a00f-88e062a47556"
title: "瑞芯微 maskrom→usbplug 切换导致设备选中项跳到无关设备：选择保持修复与验证"
description: "DeviceViewModel 扫描后按 Key 恢复选中、maskrom→usbplug 序列号/容器 ID 改写导致 GroupId 不匹配而回退 Devices[0]；选择保持（30s 窗口、唯一替代端点接管、TimeProvider 可测）设计、FlashViewModel loader 通道接入、测试数字与并行会话锁/编译错误等环境边界。"
status: "active"
created_at: "2026-11-02T00:56:28.019Z"
updated_at: "2026-11-02T00:56:28.019Z"
content_hash: "5294bb8deab9ab259e4a6bac8b118588d4ab30aef027e96b56e63dda22df7413"
source_paths:
  - "UotanStudio/ViewModels/Devices/DeviceViewModel.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs"
  - "UotanStudio.Comm/Protocols/Bootrom/RkdlConstants.cs"
  - "UotanStudio.Tests/Devices/DeviceSelectionHoldTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "3df97d33-5939-4a43-897d-cc881b93b49a"
memory_body_ids:
  []
---

# 瑞芯微 maskrom→usbplug 切换时设备列表选中项跳到无关设备：根因与选择保持修复

## 现象
发送/加载引导程序（loader）时设备从 maskrom 切换到 usbplug，设备列表选中项不再是原设备，而是跳到列表首位——常常正是那台无关设备。

## 根因
`DeviceViewModel.RefreshDevicesCoreAsync` 每次扫描结束后用扫描前的 `SelectedDevice.Key` 找回选中项，找不到就无条件回退 `Devices[0]`。

瑞芯微 maskrom → usbplug 切换会**改写上报序列号**（数字 → `ROCKCHIP`）并可能更换操作系统容器 ID，因此旧端点的分组键（`NetworkAlias ?? PhysicalId ?? Key`，即 GroupId）在新扫描中不再匹配，匹配失败即落到列表首位。这与 `VERIFICATION.md` 已记录的「maskrom 序列号为数字、loader 阶段序列号为 `ROCKCHIP`」一致。

## 修复设计
**`UotanStudio/ViewModels/Devices/DeviceViewModel.cs`** 引入显式「静态等待」（selection hold）：

- `BeginSelectionHold()` 记录当时端点键集合作为基线，并开启 30 秒窗口（与 `RkdlConstants.LoaderRelinkTimeoutMs` 同量级）。
- `ApplySelectionAfterScan` 改为三段判定：键匹配优先 → 等待期内走保持逻辑 → 才回退原有逻辑。
- 等待期内若选中端点消失：基线中**唯一新增**的端点被认作替代节点并自动接管选择；有多个新增或尚无新增时保持原选择；窗口期满才按原逻辑回退。
- 新增可选 `TimeProvider` 构造参数（默认系统时钟），使时间窗口可确定性测试。
- `SelectDevice`（用户主动点选）结束等待，避免保持卡住真实的手动切换。
- 节点真正掉线（无等待、无新增端点）的行为完全不变。

**`UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs`**：在 `DownloadRkLoaderAsync` 中、调用 `RunRkdlLoaderTransitionAsync` 之前开启等待——该方法是 loader 传输的唯一通道，因此「加载引导」与瑞芯微二级分类两条入口同时被覆盖；失败/取消/结束路径需清除等待。

已修正的实现细节：扫描恢复选择时若走 `SetProperty` 会触发 `ClearSelectionHold()`，而节点仍在线时两次扫描之间会因此错误解除保持；改为不经过该副作用。

## 验证
| 项目 | 结果 |
| --- | --- |
| `UotanStudio.Tests/Devices/DeviceSelectionHoldTests.cs`（新增 8 项） | 8 通过 / 0 失败 |
| `DeviceLinkDisplayTests`（回归） | 33 通过 / 0 失败 |
| `dotnet build UotanStudio.Desktop/UotanStudio.Desktop.csproj -c Debug` | 0 错误 0 警告 |
| 三个触碰文件的 `dotnet format` whitespace / style | 零报告 |

新增测试覆盖：端点仍在无等待时回退、等待中保持、唯一替代端点接管、多个新增继续等待、超时回退、手动点选结束等待、无选中项时选首个。实测结果已同步写入 `VERIFICATION.md`。

## 环境陷阱（复现时会遇到）
- **构建被运行实例锁住**：用户已启动的 `UotanStudio` 进程占用 `UotanStudio/obj` 与 `bin` 下的 `UotanStudio.pdb`、`UotanStudio.dll`；用 `-p:OutputPath` 重定向输出仍会命中 `obj` 锁，需同时重定向中间目录。
- **不要重定向 `BaseIntermediateOutputPath` 到仓库内新目录**：与既有 `obj` 并存会让 `AssemblyInfo.cs` 被重复包含，产生成片的 `CS0579` 程序集特性重复错误；这类临时目录必须清理后回到常规构建。
- **测试宿主 bin 被并行会话锁定**：`UotanStudio.Tests/bin` 常被另一会话的测试进程占用（`MSB3027/MSB3021/MSB1006`），定向测试只能走一次性 `-p:OutputPath` 目录运行，跑完删除。
- **并行会话的未跟踪文件**：`UotanStudio.Tests/Protocols/RkdlMaintenanceAndDevicesTests.cs` 存在既有 `CS1503`（`Span<byte>` → `byte[]`）编译错误，与本改动无关；定向测试经一次性 props 排除该文件，不修改它。
- **既有测试超时**：`HdcServerTransferTests.Install_HapPackage_...` 在 30 s 预算内超时（`OperationCanceledException` + Test Cleanup Failure），属既有 HDC 传输测试环境问题。
- **测试过滤器**：组合过滤器不被支持，设备选择与链接显示两组测试要分开运行；`--filter` 命中零测试时退出码为 8。

## 未验证边界
- **真机未复测**：maskrom → usbplug 切换期间列表选中项是否真的不跳动，需接瑞芯微板发送引导程序目视确认。
- 全量 `UotanStudio.slnx` 测试未能在本环境跑完（测试宿主被锁），仅做了定向测试与全解决方案构建。

## 相关文档
- 设备扫描/监视与 `DeviceViewModel` 既有问题：`a8f55ea5-bea4-4769-840d-683303866320`
- 顶部设备选择框变更通知链路：`ef58c9e1-8522-4442-a366-c48602055d04`

## 源码入口
- `UotanStudio/ViewModels/Devices/DeviceViewModel.cs`（`RefreshDevicesCoreAsync`/`ApplySelectionAfterScan`/`BeginSelectionHold`/`SelectDevice`）
- `UotanStudio/ViewModels/Flash/FlashViewModel.Rockchip.cs`（`DownloadRkLoaderAsync`）
- `UotanStudio.Comm/Protocols/Bootrom/RkdlConstants.cs`（`LoaderRelinkTimeoutMs`）
- `UotanStudio.Tests/Devices/DeviceSelectionHoldTests.cs`

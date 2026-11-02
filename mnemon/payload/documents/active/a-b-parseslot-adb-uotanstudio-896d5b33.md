---
id: "896d5b33-3b4c-4126-bb54-75943ef8bc61"
title: "设备显示名与 A/B 槽位：平台名拼接、ParseSlot 映射与 adb 链路类别推断（UotanStudio）"
description: "顶部设备选择框与设备管理器共用的 DisplayName/StateLabel 规则：Android/HarmonyOS 平台名、Slot A/B 拼接与 A-only 抑制、_a→Slot A 解析、adb devices-l 缺 usb: 描述符时的 TCP/USB 推断，含测试结果与未验证边界。"
status: "active"
created_at: "2026-10-27T12:41:55.108Z"
updated_at: "2026-10-27T12:41:55.108Z"
content_hash: "f4d91959f9a5493d0105b90d775ba706de431d8bf9696eebe1dbb6df59df67de"
source_paths:
  - "UotanStudio/ViewModels/Devices/DeviceInfo.cs"
  - "UotanStudio/ViewModels/Devices/DeviceViewModel.Details.cs"
  - "UotanStudio/ViewModels/Devices/DeviceViewModel.Hdc.cs"
  - "UotanStudio/Services/Devices/DevicePropertyReader.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Discovery.cs"
  - "UotanStudio.Comm/Protocols/Adb/AdbTargetInfo.cs"
  - "UotanStudio/Comm/Devices/Services/DebugBridgeSessionOpener.cs"
  - "UotanStudio.Tests/Devices/DeviceLinkDisplayTests.cs"
  - "UotanStudio.Tests/Devices/DevicePropertyReaderTests.cs"
  - "UotanStudio.Tests/Protocols/AdbTargetInfoTests.cs"
session_ids:
  - "acaf18b2-33b9-4043-9ba7-561818ec5da6"
memory_body_ids:
  []
---

# 设备显示名与槽位：Android/HarmonyOS 平台名、A/B 槽位与 adb 链路类别推断（UotanStudio）

本轮把设备在「顶部设备选择框」与「设备管理器」中的显示统一起来：平台名（Android / HarmonyOS）、A/B 槽位标签（`Slot A`/`Slot B`）以及 server 枚举得到的链路类别与端口。本文记录显示规则、推断依据与验证边界。

## 1. 显示名组装规则

- 单一来源：`DeviceInfo` 的 `DisplayName` / `StateLabel` 是顶部选择框（`SelectedDeviceName`、下拉项）与设备管理器的共用字符串，改一处两处同时生效。
- 就绪态映射：ADB 就绪状态（`device`）显示 `Android`；HDC 就绪态显示 `HarmonyOS`；离线、未授权等非就绪状态保持原状态文本，不拼平台名。
- 槽位拼接：检测到槽位时在括号内追加，例如 `R58M12345 - SM_G9730 (Android Slot A)`；fastboot 路径同样追加。
- A-only 设备：`ro.boot.slot_suffix` 为空或解析为 `--` 视为「无槽位」，只显示 `Android`，不拼槽位。判定依据是 `HasDetectedSlot`（非空且不等于 `--`）且状态属于就绪态。
- 槽位值变化会触发 `StateLabel`/`DisplayName` 变更通知，顶部选择框实时刷新。

## 2. 槽位解析（`DevicePropertyReader.ParseSlot`）

| 输入形态 | 结果 |
| --- | --- |
| `_a`、`a`、`current-slot: a` | `Slot A` |
| `_b`、`b`、`current-slot: b` | `Slot B` |
| 空串/无法识别 | `--` |

- 解析前做 `Trim + ToLowerInvariant`，因此 ` B ` 也可识别。
- ADB 属性写入路径（`DeviceViewModel.ApplyAdbProperties`）此前直接保存原始 `_a`/`_b`，本轮改为调用 `ParseSlot`，与 fastboot 路径（`ApplyFastbootProperties`）保持一致；否则界面会出现原始后缀。
- HarmonyOS（HDC）侧沿用 `ohos.boot.slot_suffix`/`ohos.boot.slot`/`ohos.boot.currentslot` 证据，`1`/`2` 映射为 `Slot A`/`Slot B`，其余回落到 `ParseSlot`。

## 3. adb server 枚举的链路类别与端口

问题：`adb host:devices-l` 的部分目标没有 `usb:` 描述符，旧实现把这些端点标成 `Unknown`，链路与端口列因此空白。

推断规则（`AdbTargetInfo`）：缺少 `usb:` 描述符时按串号判定——
- 网络端点（`host:port` 形态）、`emulator-*`、DNS-SD 实例名 → `TCP`；
- 其余 → `USB`。

显示层配套：
- server 链路标签比较改为大小写不敏感；HDC 过滤与 `IsSupportedHdcTarget` 共用同一映射。`AdbServerServices` 的相关调用（`DebugBridgeSessionOpener`、`AdbQrPairingService`）同批调整。
- 所选端点没有链路类别时，回退到同组其他端点的类别；TCP/UDP 的链接端口优先显示 server 解析出的 `host:port`。

## 4. 测试与验证

| 项目 | 结果 |
| --- | --- |
| 新增 `DeviceLinkDisplayTests` | 18/18 |
| 新增 `AdbTargetInfoTests` | 6/6 |
| `DevicePropertyReaderTests` | 51/51 |
| `DeviceProtocolMappingTests` | 29/29 |
| Debug 全量构建 | 0 警告 0 错误 |
| `dotnet format` whitespace/style `--verify-no-changes` | 通过 |
| Release `win-x64` NativeAOT publish | 成功；仅既有第三方裁剪/AOT 警告 |

## 5. 未验证边界与已知坑

- 本机无真实 adb/hdc 设备，未做真机枚举与界面人工确认；不同 adb 版本 `devices-list` 的链路字段形态差异未覆盖。
- HDC 的 UART 目标按既有语义仍不列出。
- 测试批次并发会触发仓库已记录的 30s 超时（`AdbTlsTransportTests`、`HdcServerTransferTests`）；单独复跑通过，不是回归。
- 构建偶发 `MSB3021`：`UotanStudio.Desktop` 输出目录的 `UotanStudio.Mcp.dll` 被运行中的进程占用，关闭实例或重试即可。

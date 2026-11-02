---
id: "a5d269a7-9135-4931-b07e-4fd83229ccdc"
title: "Comm 全库缺陷排查结论与 ADB 入站原始帧日志契约"
description: "UotanStudio.Comm 排查：UsbStream 部分写静默丢数据与 USB 枚举异常细节两项修复、ADB 入站 24 字节头日志契约（Error 必落盘 / UOTAN_COMM_RAW 预览）与判读表、串口/HDC/sync-over-async 观察项、确认非缺陷清单与验证数字。"
status: "active"
created_at: "2026-10-24T19:38:02.815Z"
updated_at: "2026-10-24T19:38:02.815Z"
content_hash: "7b39e2729cb8a729104feb2490ac06cbe57faada903bd38285d60a011da7e101"
source_paths:
  - "UotanStudio.Comm/Transports/Usb/UsbStream.cs"
  - "UotanStudio.Comm/Transports/Usb/Providers/UsbApiProviderBase.cs"
  - "UotanStudio.Comm/Protocols/Adb/AdbConnection.cs"
  - "UotanStudio.Comm/Protocols/Adb/AdbMessage.cs"
  - "UotanStudio.Tests/Usb/UsbStreamWriteTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "d1ee7e89-4eb6-4e78-a57d-f22a911c2e13"
memory_body_ids:
  []
---

# Comm 全库缺陷排查结论与 ADB 入站原始帧诊断契约

本文只记录本轮新增的排查结论与运行时插桩契约；ADB "Invalid ADB message magic" 的症状链、启动时序竞态根因与 magic 字段增强已由文档 `78613e8c-9182-4f03-8607-b11ce652e00d` 覆盖，此处不重复。

## 已修缺陷

1. **USB 批量写部分成功被静默丢弃**（`UotanStudio.Comm/Transports/Usb/UsbStream.cs`）
   - 批量写按块循环，超时或出错时只写进一部分块；`Stream.Write` 无法表达部分成功，原实现把剩余字节直接丢弃，调用方误以为写完整帧。
   - 改为"写入字节数不足即抛 `IOException`"，并补 2 个回归测试。
   - 测试夹具 `UotanStudio.Tests/Usb/UsbStreamWriteTests.cs` 的 `ShortWriteSession` 需同步实现 `IUsbDeviceSession.ReadPacket(byte[], int, int, int)` 与 `ReadInto(Span<byte>, int)`，否则 CS0535 编译失败（接口新增了这两个成员）。
   - 关联的既有失败语义：`AdbConnection` 侧部分写路径已先 `ResetPeerReady()`、丢弃 transport 并抛 `IOException("ADB transport accepted N of M bytes; the connection was closed.")`，二者口径一致。

2. **枚举失败丢失异常细节**（`UotanStudio.Comm/Transports/Usb/Providers/UsbApiProviderBase.cs`）
   - 原来只记 "unknown exception"，丢掉异常类型与消息；排查 USB 枚举失败时无法定位来源，现保留类型与文本。

## ADB 入站原始帧日志（新增诊断契约）

实现位置 `UotanStudio.Comm/Protocols/Adb/AdbConnection.cs`（`ReadMessageAsync` / `ThrowIfLoopFaulted`）。

- 每帧 24 字节头以十六进制 + ASCII 预览记录：**首帧为 Debug 级**（便于 WS 客户端不设环境变量也能看到），其后为 Trace 级。
- 头部解析失败（`AdbMessaging.ParseHeader` 抛 `InvalidDataException`）时的那条日志 **固定 Error 级、不受 `UOTAN_COMM_LOG` 影响**，因此握手失败一定落盘；内容包含完整 24 字节原文、解出的 `command/magic`、期望 magic 与 `first8`。
- 负载正文默认不落盘；仅当 `UOTAN_COMM_RAW=1` 且日志级别为 Trace 时按 256 字节预览。
- 入站负载超限（`payloadLength > InboundPayloadLimit()`，上限为对端 `A_MAX_PAYLOAD` 与协议 `MaxPayload` 的较小值）以及 `ThrowIfLoopFaulted` 包装前，各有一条 Error 并带传输实现名（如 `WinUSBTransport`）。

### 判读表（拿到日志后先看 24 B 原文）

| 24 字节头内容 | 判读 |
| --- | --- |
| `43 4E 58 4E` 或 `OKAY/WRTE/CLSE/OPEN` 等命令字 | 帧本身已对齐，失败原因在后续校验（CRC/负载/时序） |
| `4F 48 4F 53`（"OHOS"）等可读文本 | 读到 HDC 或其它协议通道——接口/路由选错 |
| `12 01 00 02` 一类 USB 描述符字节 | 读到 USB 描述符，端点选错（误读接口 0） |
| 形似某条消息负载的一段 | 管道复位后入站流失步（"复位 + 重试"设计的已知副作用） |
| `00 00 00 …` 全零 | 端点读回零填充缓冲（僵死传输），与 `CancelIoEx`/1168 线索同源 |

**处置方向**：若确认为失步，正确修法不是在这里扫边界重同步，而是让整条 ADB 连接 fault、由命令池重建会话（管道复位后必须丢弃残留半帧）。

## 观察项（未改动，需真机/独立验证）

- `SerialTransport` 用 `Task.Run(阻塞串口调用, token)`，取消无法中断已开始的阻塞读，可能错切串口帧边界。
- `HdcServerTaskChannel` / `AdbServerServices` 存在 sync-over-async（阻塞等待异步结果），高并发下有线程池饥饿风险。
- 13 处 `Thread.Sleep`（含 `HdcForwardTunnel`），在异步路径上浪费线程且不可取消。

## 确认为非缺陷

- `EdlPhysicalRange`、`AdbQrPairingService` 中的 `_ =` 是"调用即校验"（异常在调用栈内立即抛出），非吞异常。
- `DeviceController.Dispose` 逐租约捕获异常，保证全部释放。
- `DeviceEndpointId` 是 `record struct`，字典/集合相等性正确。
- ADB 入站负载长度有协议上限，不会因对端宣告超大长度而无限分配。

## 验证

- `UotanStudio.Comm` 构建 0 警告 0 错误。
- 测试：`Diagnostics` 27/27、`Devices` 247/247、`Protocols` 37/37、`UsbStreamWriteTests` 2/2。
- 三个改动文件的 `dotnet format whitespace/style --verify-no-changes` 零报告；证据已记入 `VERIFICATION.md`。
- **未验证边界**：无真机，新增的 rx 头日志尚未在硬件上产生过一条记录；失步处置方向（fault + 命令池重建）尚未实现。

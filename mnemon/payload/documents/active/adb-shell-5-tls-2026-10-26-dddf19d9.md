---
id: "dddf19d9-142d-4ea5-a267-24369c23a9f0"
title: "ADB 交互式 shell 无法打开：5 秒空闲读期限击穿托管 TLS 记录层（2026-10-26）"
description: "无 server 直连 ADB 时 shell 报“target is not ready”/closed-failed TLS 连接的根因链（空闲读期限 → BouncyCastle 托管 TLS 致命关闭 → 死会话占租约）、读/写期限分离与 SO_KEEPALIVE 修复、共享池立即淘汰、验证数字与未验证的真机边界。"
status: "active"
created_at: "2026-10-25T20:56:30.955Z"
updated_at: "2026-10-25T20:56:30.955Z"
content_hash: "c92a9d13cf7effa5711de02a1faa2e40823c17fb8732834fed8c0ce7da6f8988"
source_paths:
  - "UotanStudio.Comm/Protocols/Adb/AdbConnection.cs"
  - "UotanStudio.Comm/Transports/Tcp/AdbTlsTransport.cs"
  - "UotanStudio.Comm/Transports/Tcp/TcpKeepAlive.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.SharedAdb.cs"
  - "UotanStudio.Tests/Protocols/AdbRemoteProtocolTests.cs"
session_ids:
  - "5e1e0a00-c8fc-4764-b6f1-918becc3cacc"
memory_body_ids:
  []
---

# ADB 交互式 shell 无法打开：5 秒空闲读期限击穿托管 TLS 记录层（Windows x64 / .NET 10）

## 表现

Windows 上直连无 server 的无线调试（`_adb-tls-connect`）后，shell 打不开并报 `The selected ADB target is not ready.`；日志伴随 `IOException: Cannot write application data on closed/failed TLS connection` 与 `ObjectDisposedException: System.Net.Sockets.Socket`。设备端看到的是连接被主动关闭。

## 根因链

1. `AdbConnection.ReadExactIntoAsync` 对每次传输读都传 `TransportDeadlineMs = 5000`，包括“等下一帧”的空闲读（`allowIdle`）；修复前空闲分支只做重试。
2. Windows 上 `_adb-tls-connect` 的客户端证书私钥是临时 CNG 密钥，Schannel 拒绝，`AdbTlsTransport` 回退 BouncyCastle 托管 TLS，并在每次读前设置 `Socket.ReceiveTimeout`。
3. 只要 5 秒没有入站字节，底层 `NetworkStream.Read` 抛 `IOException(SocketError.TimedOut)`。
4. 反编译确认 BC 2.7.0 行为：`TlsProtocol.SafeReadRecord` 把**任何**底层 `IOException` 当作致命记录错误 → `HandleException(80, …)` 先向设备发 fatal alert → `HandleFailure()` 置 `m_closed = m_failed` 并关闭套接字。托管 TLS 的记录层不可承受“被打断的读”，超时读等于自毁连接。
5. 之后该链路上的写全部失败（`closed/failed TLS connection`），属性查询命中已关闭 socket 抛 `ObjectDisposedException`。
6. shell 有两个失败入口：`DeviceShellService.OpenInteractiveShell` 先 `ProbeAdb` 借共享会话做健康检查，死会话直接返回 “not ready”；`--managed-device-shell` 子进程自己的会话也会在 5 秒无输出后死掉（交互式 shell 常态无输出，必然触发）。死会话还占着端点独占租约，替代会话被 `DeviceEndpointBusyException` 拒绝，形成反复失败的循环。

## 修复

- `AdbConnection.cs`：等下一帧（`allowIdle && totalRead == 0`）改用 `Timeout.Infinite`，只有已开始到达的帧才用 5 秒期限；读失败（I/O、超时、已释放）统一走 `InvalidateFailedTransport`，连接立即失效而不是继续复用半帧管道；异步写补 5 秒期限，避免托管 TLS 忽略取消令牌后无限挂起。
- `AdbTlsTransport.cs`：读/写期限分离（读只设 `ReceiveTimeout`，写只设 `SendTimeout`），已升级 TLS 的流被打断即置 `_faulted`（`FaultInterruptedTlsStream`），交给共享池淘汰重建；新增 `TcpKeepAlive`（空闲 30 s、间隔 10 s、3 次探测）作为空闲读无期限后的存活兜底。
- `TcpKeepAlive.cs`：两个 TCP 传输连接后启用 `SO_KEEPALIVE`；平台不接受细粒度选项时保留默认节奏。
- `DeviceMonitorService.SharedAdb.cs`：池内已失效会话立刻移出并立刻拆除（连同独占端点租约），不再等借用方归还；拆除加幂等门 `TryBeginTeardown`，避免重复释放。使 `ProbeAdb`、logcat、属性查询与 shell 子进程都能重新打开端点。
- `AdbConnection.cs`：TLS 链路不再做 foreign-stream 探测（300 ms 有界读会破坏记录层），探测改回带期限。

设计取舍：把“空闲无期限”限定在首字节之前的等待，帧内仍保留有界期限；同时用 TCP keepalive 而不是应用层心跳承担空闲存活检测，避免在 TLS 记录层之上再插入定时读。

## 验证

- `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 错误。
- 新增回归 `AdbRemoteProtocolTests.MessageLoop_WaitingForTheNextFrame_UsesNoReadDeadline`：断言空闲首帧读无期限、负载读仍带期限。反向验证：临时还原旧行为后失败（`Expected: -1, Actual: 5000`），恢复后通过。
- `UotanStudio.Tests.Protocols` 58/58、`UotanStudio.Tests.Devices` 276/276 通过；`dotnet format` whitespace/style 对改动文件零报告。
- 改动前全量单轮：7408 总数 / 7402 通过 / 6 跳过 / 0 失败。最后一次全量跑出 2 条失败，全部来自并行会话的新代码（`UotanStudio.Comm/Protocols/Adb/AdbSyncClient.V2.cs` 直接 `new BrotliStream`、`UotanStudio.Comm/Transports/Tcp/AdbTlsTransport.cs` 直接 `SHA256.HashData` 违反压缩/哈希边界守卫），与本次修复无关，未回滚。

## 未验证边界

- 真机（Android 11+ 无线调试）上的 shell/logcat/属性查询与断线重连复测未执行。
- “空闲读取无期限”只有单元测试覆盖，没有真机托管 TLS 复现。
- 建议复测：`adb-tcp` 连上后让 logcat 空闲 10–30 秒，再打开 shell 执行命令，确认日志不再出现 `closed/failed TLS connection` 与 `ObjectDisposedException: System.Net.Sockets.Socket`。

## 相关文档

- `cd5e7bab-64ff-4918-887a-bab9cd978987`：失效 TLS 连接被共享会话池反复复用（配对后 adbd 重启 TLS 服务），本轮的池淘汰幂等门与之衔接。
- `b1898c91-0104-446e-9ebf-b27f6dcb61c6`：ADB/HDC 远端连接修复与真机矩阵，含 TLS 版本与读故障处理的早期结论。

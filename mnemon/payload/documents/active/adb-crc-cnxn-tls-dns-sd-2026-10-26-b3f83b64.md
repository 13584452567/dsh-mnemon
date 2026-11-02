---
id: "b3f83b64-e6de-4c4f-b765-00d33be373e2"
title: "ADB 直连三重根因：入站 CRC 校验误拒 CNXN、托管 TLS 全双工死锁、组播被过滤后的单播 DNS-SD 兜底（2026-10-26）"
description: "无 server 直连 ADB 的三个真实根因与修复：入站 data_check 校验偏离上游 AOSP、BouncyCastle 托管 TLS 用信号量串行化 await 读写导致消息循环死锁、网络过滤组播 mDNS 时改用设备 5353 单播 DNS-SD；含上游源码依据、改动文件、真机验证与非破坏性边界。"
status: "active"
created_at: "2026-10-26T05:58:58.010Z"
updated_at: "2026-10-26T05:58:58.010Z"
content_hash: "5d918b95d29ff956dae7f932b5ebcee554206b3651c7a2349823387f241a960b"
source_paths:
  - "UotanStudio.Comm/Protocols/Adb/AdbConnection.cs"
  - "UotanStudio.Comm/Protocols/Adb/AdbConstants.cs"
  - "UotanStudio.Comm/Transports/Tcp/AdbTlsTransport.cs"
  - "UotanStudio.Comm/Protocols/Adb/Pairing/AdbQrPairingService.cs"
  - "UotanStudio.Comm/Protocols/Adb/AdbSyncClient.V2.cs"
  - "reference/adb-aosp/transport.cpp"
  - "VERIFICATION.md"
session_ids:
  - "7af82c6b-aecb-402d-8d1b-8b4bad0a14f0"
memory_body_ids:
  []
---

# ADB 直连三重根因（Windows x64 / .NET 10，2026-10-26）

本轮把"无 server 直连 ADB"的剩余失败一次性定位到三个彼此独立的缺陷，并用真机（Wi-Fi 192.168.1.144:40443）做非破坏性验证。相关背景文档：`cd5e7bab-64ff-4918-887a-bab9cd978987`（配对后 TLS 服务重启导致死会话复用）、`a1040553-a8fc-487d-948f-157183fd1cfa`（扫码后自动发现端口 + 共享 ADB 会话）、`b1898c91-0104-446e-9ebf-b27f6dcb61c6`（ADB/HDC 远端连接基础修复）、`5cfde066-d749-4e66-8f39-c819d853361e`。

前两份文档都把失败归到"端口/会话复用"，本轮证明端口解析在单播兜底后正确，另有两条协议层与一条传输层根因。

## 1. `ADB payload CRC mismatch`：入站校验是上游从不做的

依据仓库内 `reference/adb-aosp` 上游源码：

- `transport.cpp:send_packet`：只有 `protocol_version < A_VERSION_SKIP_CHECKSUM (0x01000001)` 才写负载累加和，否则 `data_check = 0`。
- `transport.cpp:check_header`：**只校验 magic 与 data_length，从不校验 `data_check`**。

原实现按"协商版本 ≤ `A_VERSION_MIN` 才校验"，而收到对端 CNXN 之前协商版本仍是 0（v1 语义），于是把现代设备 CNXN 的 `data_check = 0` 判成错误——USB 直连与 TLS 直连都会复现。

修复（`UotanStudio.Comm/Protocols/Adb/AdbConnection.cs`、`AdbConstants.cs`）：

- 移除入站校验拒绝，仅在不匹配时留 Debug 诊断；
- 出站 `data_check` 按协商版本决定（收到对端 CNXN 前按上游写累加和，之后写 0），新增 `AdbConstants.ProtocolVersionSkipChecksum`；
- CNXN 的版本与最大负载按上游 `update_version` 取 `min(≤A_VERSION, ≤MAX_PAYLOAD)`；
- TLS 模式下忽略 AUTH（对齐上游 `if (t->use_tls) break`）；
- 删除 `SendAsync` 遗留的 `Console.WriteLine("SEND-BEGIN …")`；
- `AdbTlsTransport` 的 SHA-256 改走 `UotanStudio.Hash`（架构边界门要求）。

## 2. "连上后 shell/日志/属性全不可用"：托管 TLS 全双工死锁

Windows 回退路径的 BouncyCastle `TlsStream` 用一个信号量串行化 **await 式**读写。ADB 消息循环常驻一次 read，导致所有写（OPEN/WRTE/OKAY）永久阻塞——实测 `OpenServiceStream` 挂起超过 120 秒，表现为"已连接但交互全失败、TLS 被关闭"。

修复（`UotanStudio.Comm/Transports/Tcp/AdbTlsTransport.cs`）：托管路径改为**阻塞 I/O**——read 在线程池线程执行、write 在调用线程执行（BC 的 `m_recordWriteLock` 与 ADB 发送锁仍串行化写），期限交给套接字 `ReceiveTimeout`/`SendTimeout`；空闲帧首字节仍按连接层"无限等待"规则处理。

结论性经验：托管 TLS 实现若把读写放进同一个 async 信号量，与"常驻读 + 并发写"的协议消息循环必然死锁，只能改用同步 I/O 或自建双工封装。

## 3. 无 server 端口发现：本网络过滤组播 mDNS 应答

实测本机组播浏览 0 条、`adb server` 的 mDNS 表也为空，但**单播 DNS-SD** 直接向设备 `:5353` 询问可拿到 `adb-6a082bda-lZgxIP._adb-tls-connect._tcp.local 192.168.1.144:40443`。已在三处加入单播兜底（`UotanStudio.Comm/Protocols/Adb/Pairing/AdbQrPairingService.cs`）：配对成功后解析、握手失败后重发现、手动端口预填；实现为把 PTR/SRV/A/AAAA 查询直接发到设备 UDP 5353。

## 验证（Wi-Fi，无 server，非破坏性）

- 单播发现得到 `192.168.1.144:40443`；
- `AdbSession` + `AdbTlsTransport` 直连握手成功 `peer=0x01000001`，`exec:echo uotan-crc-ok` → `uotan-crc-ok`；
- 空闲 7 秒后再次 `exec` → `uotan-after-idle`（空闲不破坏会话），释放正常；
- gate：`Protocols` 112/112（新增"零校验和 CNXN 可完成握手""出站校验和按版本策略"）、`Devices` 276/276、`HashBoundary` 5/5；改动文件 `dotnet format` whitespace/style 干净；NativeAOT publish 成功 155,761,152 B（仅既有第三方 IL2026/IL2104/IL3053 警告）；
- 全量：7555 总数 / 7547 通过 / 7 跳过 / 1 失败。

## 未验证与已知并行失败

- **USB 直连本轮无可用设备，未复测**；理论上同一 `AdbConnection` 代码路径已被覆盖，仍需插线重验。
- 唯一全量失败与本轮无关：`UotanStudio.Comm/Protocols/Adb/AdbSyncClient.V2.cs:554/567` 直接 `new BrotliStream`，违反"通用压缩只在 `UotanStudio.Compress` 内实现"的架构门（`CompressionImplementationBoundaryTests`），而 Comm 按依赖表不引用 `Compress`——需该改动作者补允许列表理由或调整分层。
- 并行新增的 HarmonyOS 文件（`HarmonyUpdateBin{Layout,Package,Reader,SignatureKind}.cs`）出现 `FINALNEWLINE` 格式门失败，不在本轮改动范围。

---
id: "2c8758fd-cdbb-4569-8533-645110211a65"
title: "DevEco 投屏设备库来源与重新提取（15 文件 + server0 回退）"
description: "UotanStudio.Mirroring/Assets/ohos/device 的 DevEco Studio 26.0.0.821 来源、JAR/逐文件摘要、tools/deveco-mirroring 重新提取流程、server0 回退排序理由，以及真机不可加载与构建被并行改动阻塞的未验证边界。"
status: "active"
created_at: "2026-10-23T18:40:09.752Z"
updated_at: "2026-10-23T18:40:09.752Z"
content_hash: "592b2281a54da251130ff0149ea6274420089b2c954663864a09e205a1214edf"
source_paths:
  - "UotanStudio.Mirroring/Assets/ohos/device"
  - "UotanStudio.Mirroring/Assets/ohos/README.md"
  - "UotanStudio.Mirroring/Ohscrcpy/OhscrcpyServerManager.cs"
  - "tools/deveco-mirroring/extract.ps1"
  - "tools/deveco-mirroring/README.md"
  - "UotanStudio.Mirroring/AGENTS.md"
  - "VERIFICATION.md"
  - "UotanStudio.Tests/Mirroring/OhscrcpyServerManagerTests.cs"
session_ids:
  - "ce6b1eda-323f-4bd4-b72e-949f57b7ee79"
memory_body_ids:
  []
---

# DevEco 投屏设备库来源与重新提取（15 文件）

相关设计文档：`fe4bd5e7-924b-45ec-aa91-975d63f3b737`（ohscrcpy 投屏重写，其中“14 个文件、8,174,853 字节”的描述已被本文取代）。

## 来源与完整性证据

- 来源：本机 DevEco Studio **26.0.0.821** 的 `plugins/screen-mirroring/lib/hosScrcpy-1.0.20-beta.jar`，JAR SHA-256 `62cba3701b635a9d9c906a70f18cbc43efd60b7bfd61e2bb18fb88b2c7dfb187`。
- 落地目录：`UotanStudio.Mirroring/Assets/ohos/device/`，共 **15 个**设备端文件、合计 **11,204,026 字节**：
  - `libscrcpy_server0/1/2/3.z.so`、`libscrcpy_server_5.10-20260114.z.so`、`libscrcpy_server_unix_6.3.1/6.4/6.5/6.8/6.9` 系列；
  - `uitest_agent_1.1.3/1.1.5/1.1.12/1.2.3` 与 `uitest_agent_x86_1.1.12`。
- 此前已集成的 14 个文件与 `reference/HdcMirror` 副本逐字节一致；本次仅新增 `libscrcpy_server0.z.so`（3,029,173 字节，SHA-256 `09468185a89a141f23484d73f4c2d3f140604dddae2e52f4663eeadce4834bb4`）。
- 逐文件 SHA-256、来源说明与再分发边界集中在 `UotanStudio.Mirroring/Assets/ohos/README.md`；`UotanStudio.Mirroring/AGENTS.md` 与 `VERIFICATION.md` 记录同一组摘要与提取方式。

## 重新提取

- 工具：`tools/deveco-mirroring/extract.ps1` + `tools/deveco-mirroring/README.md`。默认读取本机 DevEco 安装，重新导出全部 15 个文件并打印 SHA-256；已实际运行，输出与上文摘要一致。
- 约束：JAR 内的设备库是二进制资产，升级 DevEco 后必须用该脚本重新提取，不要手工改动 `.so`。

## server0 回退的理由

`OhscrcpyServerManager.SelectVideoLibraries` 原先排除 `server0`。现在把它作为**最后回退候选**：先尝试 `server1/2/3` 与 `5.10`，全部失败后才用 `server0`，因为旧版设备只认该版本。`OhscrcpyServerManagerTests` 12/12 通过，含新增的回退排序断言；`UotanStudio.Mirroring` 构建通过。

## 未验证边界

- 提取成功只证明文件来源与完整性，**不证明设备可加载**。DevEco 的 JAR 只含采集库与 UI Test 代理，不含设备系统的 `libgrpc.z.so` / `libprotobuf`；本机真机（HarmonyOS 7.0.0.109）protobuf ABI 偏旧、模拟器镜像缺 `libgrpc.z.so`，视频路径仍未在硬件上跑通。
- 本次改动完成时，整个解决方案的构建与 NativeAOT 复发布**未能执行**：并行的 Comm HDC 同步/任务通道重构使 `HdcSyncClient.Expect`、`CreatePackageTar`、`BuildInstallConfig` 等符号缺失，与本次提取无关；待该重构落地后需重跑完整构建与发布验证。

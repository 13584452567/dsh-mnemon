---
id: "fe4bd5e7-924b-45ec-aa91-975d63f3b737"
title: "ohscrcpy 投屏重写：UI Test 扩展 + 标准 gRPC/HTTP2 视频 + UI Test JSON 控制（2026-10-24）"
description: "UotanStudio.Mirroring 的 ohscrcpy 从 HAP 安装改为 UI Test 扩展 + 自实现 h2c gRPC + UI Test JSON/uinput 控制的设计理由、所有权模型、验证数字与真机未验证边界。"
status: "active"
created_at: "2026-10-23T17:48:50.391Z"
updated_at: "2026-10-23T17:48:50.391Z"
content_hash: "992bf9cf92356f155c18854e1e73e7bdacc101b8c93797b56811f6892b7fb251"
source_paths:
  - "UotanStudio.Mirroring/Ohscrcpy/OhscrcpyServerManager.cs"
  - "UotanStudio.Mirroring/Ohscrcpy/OhoscrcpyGrpcTransport.cs"
  - "UotanStudio.Mirroring/Ohscrcpy/OhscrcpyGrpcProtocol.cs"
  - "UotanStudio.Mirroring/Ohscrcpy/OhscrcpyProtocol.cs"
  - "UotanStudio.Mirroring/Ohscrcpy/OhscrcpyClient.cs"
  - "UotanStudio.Mirroring/Ohscrcpy/OhscrcpyControlChannel.cs"
  - "UotanStudio.Mirroring/Ohscrcpy/ScrcpyToOhscrcpyControlAdapter.cs"
  - "UotanStudio.Mirroring/MirrorSessionFacade.cs"
  - "UotanStudio.Mirroring/Ohscrcpy/ManagedOhscrcpySession.cs"
  - "UotanStudio.Comm/Transports/Tcp/HdcForwardTunnel.cs"
  - "UotanStudio.Comm/Contracts/HdcSessionContracts.cs"
  - "UotanStudio/Services/Devices/DeviceMonitorService.Ohscrcpy.cs"
  - "UotanStudio/ViewModels/Editors/ScreenMirrorViewModel.cs"
  - "UotanStudio.Mirroring/UotanStudio.Mirroring.csproj"
  - "VERIFICATION.md"
  - "UotanStudio.Mirroring/AGENTS.md"
session_ids:
  - "f97ee7a0-317e-47d8-bf04-6de248a89f6b"
memory_body_ids:
  []
---

# ohscrcpy 投屏重写：UI Test 扩展 + 标准 gRPC/HTTP2 视频 + UI Test JSON 控制

## 背景与替换范围

原实现通过 HAP 安装 + 扩展进程投屏。重写以 `reference/HdcMirror` 的 DevEco 路径为蓝本，改为 **UI Test 扩展 + 标准 gRPC/HTTP2 视频 + UI Test JSON 控制**，并把所有外部二进制（`hdc.exe`、`uitest.exe` 调用等）替换为 `UotanStudio.Comm` 的托管 HDC（`IHdcDeviceSession`）。

- 删除 HAP 下载/安装链路：`OhscrcpyServerService`、`OhscrcpyServerReleaseClient` 及宿主启动检查。
- 删除 `OhscrcpyHdcTunnel.cs`（旧的隧道包装），视频转发隧道所有权移交给 gRPC transport。
- 设备库随包分发：`UotanStudio.Mirroring/Assets/ohos/device/`（14 个文件，8,174,853 字节），由 `UotanStudio.Mirroring.csproj` 以 `<Content>` 链接到输出/发布目录 `ohos/device/`，匹配 `OhscrcpyServerManager.RuntimeDirectory`（`AppContext.BaseDirectory/ohos/device`）。

## 设备端链路（固定顺序）

1. 探测 UI Test 版本：低于 `MinimumUiTestVersion` 直接失败；高于 `UnixSocketMinimumVersion` 才使用 Unix 虚拟 socket。
2. 复用或启动不带扩展的 `uitest start-daemon singleness` 控制守护进程；需要新建时推送控制代理，替换前备份原 `agent.so` 并在清理时恢复。
3. 按设备架构/版本从 `ohos/device` 选采集 `.so`，以唯一文件名推送到 `/data/local/tmp`。
4. 启动扩展：`uitest start-daemon singleness --extension-name <唯一名> -scale 1 -frameRate ... -bitRate ... -p 5000 -iFrameInterval ... -repeatInterval ...`。
5. `HdcForwardTunnel` 把设备端 `localabstract:scrcpy_grpc_socket`、`localabstract:uitest_socket`（旧版本退回 `tcp:5000` / `tcp:8012`）映射到回环端口。

清理只处理本会话拥有的对象：只停 `ControlDaemonHandle` / `VideoDaemonHandle` 记录的 pid，删除本会话推送的扩展与控制代理、恢复备份，然后按 转发 → 会话 → 租约 释放；不按进程名 `pkill`，不删非本会话文件。

## 视频通路

- 自实现最小 gRPC HTTP/2 客户端（明文 h2c 先验知识），**不引入 Grpc/Protobuf 包**：保持 NativeAOT 兼容并避免 trim 元数据需求。
- `/ScrcpyService/onStart` 返回 `ReplyMessage`，其 `payload["data"]` 是 Annex-B H.264 访问单元；`onRequestIDRFrame` 请求关键帧。
- 首个访问单元解析 SPS/PPS 与尺寸；帧包装成 `ScrcpyVideoFrame`（config / key-frame 标志与 scrcpy 一致），继续复用既有 GPU 解码管线。
- 相关：`OhoscrcpyGrpcTransport.cs`（隧道 + 复用/请求编排）、`OhscrcpyGrpcProtocol.cs`（HTTP/2 帧与 protobuf 编解码）、`OhscrcpyProtocol.cs`（SPS 几何与帧包装）。

## 控制通路

- 触摸/鼠标/轴手势走 UI Test JSON socket；按键与文本走 shell `uinput`（`uinput -K -d/-u/-t`）。
- `ScrcpyControlMessageWriter` 产生的 scrcpy 控制消息由 `ScrcpyToOhscrcpyControlAdapter` 翻译；无对应物或控制通道不可用时 `OhscrcpyClient.SendScrcpyControl` 返回 `false`。
- 滚轮一次步进 = 轴手势 + `AxisStop` 两条命令；文本注入按 UTF-8 字节数受 `OhscrcpyConstants.MaximumTextLength` 限制。
- 剪贴板与 UHID 明确不支持。
- 控制通道实现：`OhscrcpyControlChannel.cs`（手势泵 + shell 泵、未完成手势释放、有界超时）。

## 宿主与会话所有权

- `MirrorSessionFacade.cs`：探测设备 → 确保控制守护进程 → 启动视频守护进程 → 建视频/控制两条 `HdcForwardTunnel` → 构造 transport / control / client，失败时按相反顺序释放已建对象。
- `ManagedOhscrcpySession.cs`：会话持有 client、HDC 会话、两个守护进程句柄与租约，Dispose 顺序固定，避免 fport/监听器泄漏。
- `DeviceMonitorService.Ohscrcpy.cs` 改为打开**直连** HDC 租约（HDC server 路由没有文件传输通道）；`ServerOnly` 模式仍拒绝直连；HDC server 占用 USB 时给出明确提示。
- `ScreenMirrorViewModel.cs` 的 HDC 分支不再显示"安装 server"，帧率/码率由镜像设置生成。

## 验证结果（Debug / win-x64）

- Debug 构建：0 错误；仅一条既有 `SpacemiTFlashTests` xUnit1051 警告。
- 定向测试：`OhscrcpyProtocolTests` 15、`OhscrcpyServerManagerTests` 12、`OhscrcpyClientTests` 8、`ScrcpyToOhscrcpyControlAdapterTests` 16、`MirrorSessionFacadeTests` 12；`UotanStudio.Tests.Mirroring` 命名空间 96 通过 / 1 跳过（跳过项为受环境门控的硬件用例）。
- 全量套件：7185 总数 / 7177 通过 / 1 失败 / 7 跳过。唯一失败是并行的 `CompressionImplementationBoundaryTests`（指向 `UotanStudio.Comm/Protocols/SpacemiT/SpacemiTFlasher.cs` 的 `GZipStream`），与本次 HDC/Mirroring 改动无关。
- 架构门：`HashBoundaryGuardTests`（Mirroring 内 MD5 已移除，改走推送语义）、`HostLayeringGuardTests`、`ProjectReferenceDirectionGuardTests` 通过。
- 格式门：改动文件 whitespace 与 style `--verify-no-changes` 通过。
- NativeAOT：`dotnet publish … -p:PublishAot=true` exit 0，发布目录含 `ohos/device` 14 个文件与可执行文件；仅有 Ursa/Dock/Avalonia/YamlDotNet 等既有第三方 trim/AOT 警告。
- 备注：整解 Debug 构建曾因运行中的 `UotanStudio.Desktop` / Visual Studio 锁住输出 DLL 而 MSB3027 失败；单独构建测试宿主与 Release AOT 发布均成功，不是编译错误。

## 真机 / 模拟器边界（未验证）

- 真机（HarmonyOS 7.0.0.109）加载 `ohos/device` 下 2026 年采集库时，`dlopen` 因缺少更新版 protobuf/gRPC 符号失败；模拟器缺 `libgrpc.z.so`，其 `uitest start-daemon singleness` 报 `Client nativeCode not exist`。因此视频解码/播放与界面呈现在本机硬件**未验证**。
- 受 `UOTAN_HDC_MIRROR_DIRECT` 门控的 `OhscrcpyHdcIntegrationTests`（支持 connect key 或 `host:port`）已加入，但真机尝试被 DevEco 的 HDC server 挡住：Comm 的 fail-closed USB 路由拒绝直连。需先停止 HDC server 或在无 server 主机上重跑才能验证直连 fport/控制链路。相关 HDC 直连前提见 [ADB/HDC 远端连接修复与真机验证](adb-hdc-2026-10-24-b1898c91.md)（id `b1898c91-0104-446e-9ebf-b27f6dcb61c6`）。
- 边界与实测数字已同步写入 `VERIFICATION.md` 与 `UotanStudio.Mirroring/AGENTS.md`；未执行提交、推送或历史改写。

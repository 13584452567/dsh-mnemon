---
id: "7bfa005b-61f7-40f8-a0d8-4c81a9e001e8"
title: "远程文件传输选型：SSH.NET + FluentFTP 的 NativeAOT 实测与火绒误报处置（2026-11）"
description: "SSH.NET 2026.0.0 与 FluentFTP 55.1.0 的 NativeAOT 实测（0 裁剪警告、体积代价、依赖与许可）、SCP/host key 等陷阱、火绒 TrojanDownloader 误报的成因分层与处置优先级，以及 UotanStudio.Net 远程文件抽象的落点建议与未验证边界。"
status: "active"
created_at: "2026-11-01T06:10:39.762Z"
updated_at: "2026-11-01T06:10:39.762Z"
content_hash: "dc28ec1c9b926761dc9757653522955f6a1456e2d178cecdcff08ee0f396ac97"
source_paths:
  - "UotanStudio.Net/AGENTS.md"
  - "UotanStudio.Desktop/UotanStudio.Desktop.csproj"
  - "UotanStudio.Assets/Assets/manifest.tsv"
  - "VERIFICATION.md"
session_ids:
  - "6f90caf7-598f-40b4-b587-b948ea3c00d7"
memory_body_ids:
  []
---

# 远程文件传输选型：SSH.NET + FluentFTP 的 NativeAOT 实测与火绒误报处置

> 关联：设备侧文件传输走 HDC 路由（见文档 id `15613614-f9e3-4daa-b1c0-a2aa9cd9528d`）；本文处理的是**主机侧**对外 SFTP/SCP/FTP 的库选型与发布期杀软误报。
> 实测环境：Windows x64，.NET SDK 10.0.403（ILCompiler 10.0.14），`PublishAot=true` + `TrimmerSingleWarn=false`，探针工程放在 `%TEMP%` 下，验证后删除。

## 1. 结论

| 用途 | 选型 | 版本 | 许可 | AOT 实测 |
| --- | --- | --- | --- | --- |
| SFTP / SCP / SSH 命令 | SSH.NET（`Renci.SshNet`） | 2026.0.0 | MIT | 0 条裁剪/AOT 警告，原生 exe 可运行 |
| FTP / FTPS | FluentFTP | 55.1.0 | MIT | 0 条裁剪/AOT 警告，原生 exe 可运行 |

两者合用的发布探针：`IL2026`/`IL3050` 等警告 **0 条**，publish 退出码 0，原生 exe 运行后 SFTP/SCP/SSH/FTP 四条链路都进入真实 socket 路径（分别抛 `SocketException` / `IOException`），BouncyCastle 的 Ed25519 签名验签与 Argon2 在 AOT 下结果正确。

## 2. 判定依据（为什么"0 警告"不是假绿）

- 两个程序集都自带 `IsTrimmable` 元数据，裁剪分析器会真正分析其 IL，而不是因为"未标注就跳过"。
- IL 中没有 `RequiresUnreferencedCode` / `RequiresDynamicCode` 标注（RUC=0、RDC=0），调用这些 API 不会产生抑制型静默。
- SSH.NET 直接提供 `lib/net10.0` 资产（另有 net9.0 / net8.0 / netstandard2.0 / net462）；FluentFTP 最高到 `lib/net9.0`，net10 上解析到 net9 资产属正常。
- NuGet 对 SSH.NET 2025.0.0 / 2025.1.0 标注 high severity 漏洞提示，因此直接取 2026.0.0。

## 3. 实测 API 面（覆盖真实调用链，而非 hello world）

- **SFTP**：connect、`ListDirectory`、`DownloadFileAsync`、`UploadFileAsync`、`GetAttributes`、`Exists`、`SetLastWriteTimeUtc`、`CreateDirectory`、`RenameFile`、`DeleteFile`、`SymbolicLink`、`Open`（`SftpFileStream`）、`ChangeDirectory`。
- **SCP**：`Download` / `Upload`，使用非过时构造 `new ScpClient(ci, RemotePathTransformation.ShellQuote)`。
- **SSH**：`RunCommand`（同步 + `BeginExecute/EndExecute`）、`CreateShell`、`ForwardedPortLocal` 端口转发。
- **FTP**：`FtpConfig{EncryptionMode=Explicit, ValidateAnyCertificate, AutoPassive}`、`GetListing(FtpListOption.AllFiles)`、`GetFileSize`、`GetChecksum`、`DownloadStream`/`UploadStream`、`OpenRead`/`OpenWrite`、`CreateDirectory`、`Rename`、`MoveFile`、`DeleteFile`、`HasFeature`、`AutoConnect`。

## 4. 体积代价（单文件原生 exe，含运行时）

| 变体 | 体积 |
| --- | --- |
| 空控制台 AOT 基线 | 933 KB |
| 仅 SSH.NET（+BouncyCastle） | 1.93 MB |
| 仅 FluentFTP | 3.65 MB |
| 两者 + 全 API 面 | 7.48 MB |

"仅 SSH.NET"的体积是**手工执行 `link.exe`** 得到的实测值，原因见第 6 节第 1 条。

## 5. 依赖与许可

- SSH.NET `net10.0` 依赖组：`BouncyCastle.Cryptography 2.7.0`、`Microsoft.Extensions.Logging.Abstractions 8.0.3`。BouncyCastle 无 net8+ 专用资产，以 net6.0 资产参与裁剪；4.9 MB IL 大部分被裁掉，只在后量子 KEX / 部分椭圆曲线算法上被触及。
- `FluentFTP.GnuTLS 1.0.40`：**LGPL-2.1-only**，且带原生 dll。仅在有 TLS 1.3 硬需求时按需启用，商业分发前需评估。
- 三个包的 nuspec 均为表达式型 license 字段（前两个 MIT）。

## 6. 陷阱与易错点

1. **杀软删产物伪装成构建错误**：SSH.NET 统一发布时报 `MSB3030: 无法复制文件 native\X.exe`，手工跑 `link.exe` 却成功产出 1.93 MB exe，文件随后数分钟内被静默删除；同期运行的 `HipsMain`/`HipsTray`（火绒）即为元凶。看到 `CopyNativeBinary` 阶段丢文件，先怀疑杀软，不要误判为 AOT 不兼容。
2. **SCP 旧构造全部 `[Obsolete]`**：`new ScpClient(host, port, user, pass)` 报 CS0618，原因是未转义路径可能导致远程命令注入；必须显式传 `IRemotePathTransformation`（`ShellQuote` / `None`）。官方建议能 SFTP 就别用 SCP。
3. **主机密钥必须自行校验**：SSH.NET 默认不拒绝未知 host key，生产实现要订阅 `HostKeyReceived` 做 known_hosts / 指纹比对。
4. **FluentFTP 无 net10 资产**：解析到 net9 属正常，但意味着该库在更高 TFM 上不保证新增 API 面。

## 7. 发布期杀软误报（火绒 `TrojanDownloader/W64.Agent.m!crit`）

**判定**：`.m` 是火绒机器学习/泛化家族分支而非精确哈希，`!crit` 是最危险等级、直接隔离。同机同构建下的对照——7.48 MB 的 `Probe.exe` 存活、1.93 MB 的 `Diag.exe` 被删——说明"含 SSH 就杀"不成立，判定与 PE 结构、熵、导入与字符串特征强相关且不稳定。.NET NativeAOT 被 Defender/国产引擎误报是记录在案的长期问题（dotnet/runtime#105959）。

**命中"Downloader"画像的原因**（UotanStudio.Desktop 现状）：

| 特征 | 现状 |
| --- | --- |
| 无 MSIL、高熵、内含完整运行时 | NativeAOT 单文件，像加壳/打包器 |
| exe 内嵌压缩资源 | `UotanStudio.Assets` 的 `<path>.zst` 内嵌帧，运行时解压释放载荷 |
| 启动即联网 | `UotanStudio.Net` 的 GitHub 镜像 / Release API |
| 启动即写盘 | 资产抽取、临时文件、原子替换 |
| 无签名、版本资源单薄 | Release `DebugType=none`，未签名；存活产物仅有 `Company=Probe Product=Probe` |

**处置优先级**：

1. 止血（零成本）：开发机只把 publish 目录加入火绒信任区；向火绒论坛"病毒查杀问题反馈 / 误报申诉"提交（附 SHA-256、火绒版本+病毒库版本、检测名全串、复现方式，注明 .NET 10 NativeAOT 产物、未签名、非下载器）；分发形态改为 ZIP + 说明或 MSI/MSIX 安装到 `Program Files`。
2. 结构性降特征：代码签名（OV 起，EV 更稳）是唯一系统性降权手段，签完仍需申诉一次；补全 `CompanyName`/`ProductName`/`FileDescription`/`LegalCopyright`/`FileVersion`；把内嵌 zst 资产外置（`AssetStore` 已支持外部路径，主要改 `manifest.tsv` 的消费方式）；镜像与 Release 端点走配置而非连续明文 URL；自更新改为"固定目录 + 哈希/签名校验 + 用户确认"，避免下载到 `%TEMP%` 再执行；锁定 `global.json` SDK 与 ILCompiler 版本，减少产物漂移导致的误报抽奖。
3. 发布门禁：发布后算 SHA-256 并做多引擎扫描；维护"误报台账"（版本/引擎/检测名/是否申诉/何时消失）；新版本灰度。

**明确不要做**：加壳、混淆、手改 PE 字节或导入表（把 ML 偶发命中变成确定性命中，并破坏签名）；让用户关闭杀软或加白全盘；反复重编译碰运气。

## 8. 落点建议（`UotanStudio.Net`）

远程文件访问放在叶子库 `UotanStudio.Net`，不新增 `ProjectReference`，不影响 `Hash`/`Compress`/`Firmware` 的依赖方向：

- `UotanStudio.Net/RemoteFile/` 定义 `IRemoteFileClient`（`List`/`OpenRead`/`DownloadAsync`/`UploadAsync`/`Rename`/`Delete`/`Stat`，全部带 `CancellationToken` 与有界缓冲），实现 `SftpRemoteFileClient`、`FtpRemoteFileClient`；SCP 仅作显式开关的兼容路径。
- 连接参数用 options 绑定 + 源生成 `JsonSerializerContext`，DI 沿用现有源生成注册方式，不走反射。
- 传输语义按仓库不变量：流式、大小上限、临时文件校验后原子替换、失败清理。

## 9. 未验证边界

本机无可连 SSH/FTPS 服务端，以下必须补一次真实互通才能写进 `VERIFICATION.md`：真实服务端握手与算法协商（尤其 BouncyCastle 覆盖的 PQ/Ed25519 KEX）、大文件与断点续传、FTPS implicit 模式、中文与特殊字符远端路径。误报处置中的代码签名与资产外置尚未实施，申诉结果也未回收到检测库更新。

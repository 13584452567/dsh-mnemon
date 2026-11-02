---
id: "6e1bc9bf-cdf8-477b-83dc-aab69e6bbd6d"
title: "UotanStudio Boot 补丁窗口在线版本与补丁包链路（2026-10-28）"
description: "BootPatchWindow 在线版本/补丁包链路的设计落点：Net 叶子库 Release 列表与资产投影、宿主下载校验与原子发布、镜像测速与类型族校验、窗口接线、实测验证与未验证边界。"
status: "active"
created_at: "2026-10-28T12:55:26.956Z"
updated_at: "2026-10-28T12:55:26.956Z"
content_hash: "096b146d187972db01687dcd73b6565b5f8e360efa086c4eed608368ebbf43c5"
source_paths:
  - "UotanStudio.Net/GitHub/GitHubReleaseClient.cs"
  - "UotanStudio.Net/GitHub/GitHubRelease.cs"
  - "UotanStudio.Net/Components/Root/RootPatchReleaseClient.cs"
  - "UotanStudio/Services/PatchPackages/RootPatchPackageService.cs"
  - "UotanStudio/Services/PatchPackages/RootPatchSources.cs"
  - "UotanStudio/Views/Shell/BootPatchWindow.axaml.cs"
  - "UotanStudio.Tests/Architecture/HostLayeringGuardTests.cs"
session_ids:
  - "52715689-9aab-46c5-befe-987111ca2a86"
memory_body_ids:
  []
---

# UotanStudio Boot 补丁窗口的在线版本与补丁包获取链路（2026-10-28）

本轮把「Boot 修补」窗口从"仅本地自选文件"接通为"在线版本列表 → 下载 → 校验 → 检测 → 修补"，并把上游 Release 元数据读取收敛到 `UotanStudio.Net` 叶子库。本文记录可复用的设计约束与实测边界；逐行实现以代码为准。

## 1. 分层落点

- Release 元数据与资产投影只能在 `UotanStudio.Net`（叶子库，不引用任何 `UotanStudio.*`）：新增 `UotanStudio.Net/Components/Root/RootPatchReleaseClient.cs`，与 `Components/Scrcpy/` 同级，属于同一类"上游组件 Release 记录 + 资产投影"。
- 下载、长度/摘要校验、原子发布属于宿主 **应用服务**：`UotanStudio/Services/PatchPackages/RootPatchPackageService.cs` 与 `RootPatchSources.cs`。Net 只做 HTTP、镜像枚举与 JSON，不持有下载目录策略。
- 窗口只做编排：`UotanStudio/Views/Shell/BootPatchWindow.axaml.cs`。宿主的分层守卫测试要求 Views 通过 ViewModel/冻结白名单访问应用服务，本窗口需登记冻结例外。

## 2. 元数据读取约定

- `GitHubReleaseClient.ListAsync`/`ParseReleases` 读 `/releases?per_page=N`：**丢弃草稿、保留预发布、顺序保持 API 顺序**；1 MiB 响应体上限不变；响应读取抽成共用私有方法，避免两处重复。
- `GitHubRelease` 增加 `draft`/`prerelease` 字段与 `List<GitHubReleaseDocument>` 源生成元数据；JSON 一律走 `GitHubJsonContext`，禁止反射式序列化/动态类型（NativeAOT 硬要求）。

## 3. 上游仓库与资产选择

| 类目 | 上游仓库 |
| --- | --- |
| `magisk` | `topjohnwu/Magisk` |
| `magisk-alpha` | `vvb2060/Magisk` |
| `kernelsu` | `tiann/KernelSU` |
| `sukisu` | `SukiSU-Ultra/SukiSU-Ultra` |
| `apatch` | `bmax121/APatch` |

资产优先级：**发行版 APK > 普通 APK > 发行版 ZIP > ZIP**，并排除 debug/sources 一类资产。理由：同一 Release 常同时挂多个产物，按"可安装性"而不是名称匹配能避免抓到调试包或源码包。

## 4. 下载与校验策略

- 下载走 `UotanStudio.Net` 的 `HttpDownload`：**先对镜像与源站测速再按速度取值**，镜像失败仍可回落源站。
- 校验同时核对**公布长度 + SHA-256**；未公布摘要的 Release 只走源站（镜像内容不可自证时不做盲信）。
- 原子发布使用同目录暂存 + 单次覆盖移动（`IO/AtomicPublish`），失败删除暂存。
- 存放位置与 scrcpy 服务端共用同一个下载资源根：`%APPDATA%/UotanStudio/<命令>/<版本>/<资产>`；scrcpy 存储目录被自定义时自动跟随其父目录。覆盖项：`patchPackage.downloadDir`、镜像开关 `patchPackage.useMirrorList`。
- `ValidateAsync` 校验自选文件：类型族、ABI、可选项，并核对**来源期望类型族**（Alpha → Magisk、SukiSU → KernelSU），防止把错族的包喂给修补器。

## 5. 窗口接线

- 切换类目即在线拉取版本并填入版本下拉框；选中版本后立即下载 + 校验，并把检测结果喂给原有架构/KMI/选项面板；点「开始」前再 `EnsureReleasePackageAsync` 确保本地文件存在。
- 自选文件路径走 `ValidateAsync`，未通过时显示原因并禁用「开始」，不进入修补。
- 新增 5 条本地化键（`BootPatch_VersionsLoading`/`VersionsEmpty`/`PackageDownloading`/`PackageReady`/`PackageInvalid`），`Locale/Common.resx`、`Common.zh-CN.resx` 与 `LocaleKeySets.cs` 三处必须同步。

## 6. 验证（实测）

- `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 错误 0 警告。前提是先用 `-p:Configuration=Debug` 执行 restore，否则 `Dock.Avalonia.Diagnostics` 的条件包含会失效。
- 定向测试：`UotanStudio.Tests.Net` 43/43、`RootPatchPackageTests` 17/17、`ScrcpyServerReleaseTests` 19 通过 1 跳过（随包 jar 门控）、`LocaleDictionaryTests` 6/6。
- whitespace 与 style `--verify-no-changes` 对全部触及文件干净。
- `dotnet publish UotanStudio.Desktop -c Release -r win-x64 --self-contained` 退出码 0；警告仅 Dock/Avalonia XAML trampoline、LTRData、YamlDotNet、Common.Logging 的 IL2026/IL2104/IL3053，没有一条指向新增代码（**未与既有基线逐条比对**，只能说未观察到回归）。

## 7. 已知环境陷阱

- 多个并行会话同时构建时，`UotanStudio.Net.pdb` 被内存映射占用会报 `CS2012 无法打开 ... .pdb 以进行写入`。处置：给 publish 加 `--artifacts-path`（例如 `%TEMP%\uotan-aot-check`）把中间产物挪走，**不要**终止不属于本会话的 `dotnet` 进程。
- 并行会话遗留的 `HostLayeringGuardTests.Views_DoNotImportApplicationServiceNamespacesOutsideTheFrozenAllowList` 失败来自已在 HEAD 的 `UotanStudio/Views/Controls/DocumentFileActionsMenuItem.cs` 导入 `UotanStudio.Services.Firmware`，非本轮改动引入；按仓库 Git 纪律不擅自改他人文件。

## 8. 未验证边界

- 本沙箱无外网：真实 GitHub 版本列表、镜像测速与下载链路**未实测**，只有单测覆盖解析与选择逻辑。
- 窗口交互（下拉框联动、下载进度、APatch 超级密码流程）需人工验证，不能写成已通过。
- 未在真机上验证修补产物可启动。

## 相关文件

- `UotanStudio.Net/GitHub/GitHubReleaseClient.cs`、`UotanStudio.Net/GitHub/GitHubRelease.cs`、`UotanStudio.Net/Components/Root/RootPatchReleaseClient.cs`
- `UotanStudio/Services/PatchPackages/RootPatchPackageService.cs`、`UotanStudio/Services/PatchPackages/RootPatchSources.cs`
- `UotanStudio/Views/Shell/BootPatchWindow.axaml.cs`、`UotanStudio/Locale/Common.resx`、`UotanStudio/Locale/LocaleKeySets.cs`
- 测试：`UotanStudio.Tests/Net/GitHubReleaseListTests.cs`、`UotanStudio.Tests/Net/RootPatchReleaseClientTests.cs`、`UotanStudio.Tests/Services/RootPatchPackageTests.cs`、`UotanStudio.Tests/Architecture/HostLayeringGuardTests.cs`

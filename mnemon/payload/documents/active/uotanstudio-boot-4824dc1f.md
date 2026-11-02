---
id: "4824dc1f-7c5c-4c92-b4df-7c87c0e0bcef"
title: "UotanStudio 下载源测速与最优源固定（Boot 补丁链路）"
description: "Boot 补丁窗口启动即对所有镜像与源站测速、固定最快源供后续下载复用：HttpDownload 选源契约、服务/窗口接线、清单为何保留源站的信任理由、实测结果与未验证边界。"
status: "active"
created_at: "2026-10-28T13:36:58.928Z"
updated_at: "2026-10-28T13:36:58.928Z"
content_hash: "52449f37cf520186ab1dd92414b0bb01cffda15e9ad44d187491654a932b9f3f"
source_paths:
  - "UotanStudio.Net/Http/HttpDownload.cs"
  - "UotanStudio/Services/PatchPackages/RootPatchPackageService.cs"
  - "UotanStudio/Views/Shell/BootPatchWindow.axaml.cs"
  - "UotanStudio/Locale/LocaleKeySets.cs"
  - "UotanStudio.Tests/Net/HttpDownloadSourceOrderTests.cs"
  - "UotanStudio.Tests/Services/RootPatchPackageTests.cs"
session_ids:
  - "c9f061ce-0951-4e5e-92ae-9bb5a0694c46"
memory_body_ids:
  []
---

# UotanStudio 下载源测速与最优源固定（Boot 补丁链路）

关联文档：`6e1bc9bf-cdf8-477b-83dc-aab69e6bbd6d`（Boot 补丁窗口在线版本与补丁包链路）——本文只补充「测速选源、最优源固定、清单为何不走镜像」这一组约束，其余分层与校验约定以该文为准。

## 1. 需求与落点

用户要求：窗口启动时对所有镜像源测速，选取最快的源用于清单拉取与下载。

最终落点（保留清单在源站，见第 4 节）：

1. 窗口打开 → 从源站拉取版本清单；
2. 清单到手后立即对**最新版本资产**的全部候选源（源站 + `GitHubMirrorSite` 列表中的镜像）并发测速；
3. 记住胜出源，后续下载把它排在候选首位，失败按排名回退；
4. 首次下载也使用最快源，而不是每个文件各测一遍。

## 2. `HttpDownload` 的选源契约（UotanStudio.Net，叶子库）

- `EnumerateSources`/`MirrorUrl`：镜像前缀已自带被替换的源站片段（`…/https://github.com` 或 `gh`、`github.com` 短别名），`github.com` 地址只贡献路径与查询串，否则会拼出重复源站；非 `github.com` 主机保留完整地址。
- `SelectSourcesBySpeedAsync(url, client, ct, requestHeaders)`：每个候选源发一次 `Range: 0..256 KiB-1` 的探测请求，按读到数据为止的耗时排序；单源不测速直接返回；个别源失败（含 5 s 超时）排最后；**全部失败时退化回源站优先的固定顺序**，不抛异常；外部取消原样向上抛。
- `DownloadAsync(..., sources)`：传入已排好序的候选时**跳过整轮测速**，按给定顺序尝试；未传时内部先测速。每个源失败只收集 `host: message` 继续下一个，全部失败抛 `IOException` 汇总；取消直接抛出、不再尝试后续源。返回实际使用的源地址。

设计理由：一次测速覆盖 27 个镜像，若放在每次下载内部，每个 APK 都要重新测 27 个源；把测速结果显式作为参数下传，才能做到「一次测速、多次下载」。

## 3. 服务与窗口接线

- `RootPatchPackageService`：新增 `ProbeSourcesAsync`、`PreferredSourceAuthority`、`OrderSources`；下载前把记住的胜出源排到候选首位再交给 `HttpDownload.DownloadAsync`，长度与 SHA-256 校验流程不变；每次下载成功都把实际使用的源更新为新的最优源（镜像退化时自动纠正）。
- `BootPatchWindow.axaml.cs`：`RefreshReleasesAsync` 拉完清单 → 用 `releases[0].DownloadUrl` 调 `ProbeSourcesAsync`（独立 `CancellationTokenSource`，切换类目即取消）→ 再选中首个版本，因此首次下载即走最快源；测速失败只记 Warn 日志，不阻断版本列表。
- 新增本地化键 `BootPatch_ProbingSources`（测速期间状态栏），须同步 `Locale/Common.resx`、`Common.zh-CN.resx` 与 `Locale/LocaleKeySets.cs` 三处。

## 4. 清单仍走 api.github.com（有意保留）

- 镜像代理的是 `github.com` 的文件地址，不代理 `api.github.com`；把清单也指向镜像需要给 `HttpDownload` 增加 `api.github.com` 的改写与开关。
- 更重要的是信任模型：**公布字节长度 + SHA-256 是让镜像交付的载荷可信的唯一依据**。清单若也经镜像，镜像就能同时伪造版本与摘要，受影响的包会被直接打进 boot 分区。因此清单只有几 KB，走源站；测速收益留给约 10–15 MB 的下载链路。
- 推论：未公布摘要的 Release 仍只走源站，与上文「镜像内容不可自证时不盲信」一致。

## 5. 验证（实测）

- 全量 `dotnet build UotanStudio.slnx --no-restore --nologo -c Debug`：0 错误 0 警告。
- `RootPatchPackageTests` 21/21：新增用例「只有源站应答时，测速结果与固定源均为 `https://github.com`，随后下载确实从该源完成」。
- `HttpDownloadSourceOrderTests` 1/1：给定 `sources` 时不再向任何镜像发探测请求，只请求 `github.com`。
- `LocaleDictionaryTests` 6/6；触及文件的 whitespace/style 校验干净（含 `FINALNEWLINE` 修复）。

## 6. 仍未验证

- 真实网络下镜像可用性与测速耗时分布（本机仅离线/单源场景验证）。
- GUI 中「正在测速镜像源…」状态与选版联动、以及切换类目时的测速取消，需人工在联网环境确认。
- 未新增 `api.github.com` 镜像改写路径，故「清单也走镜像」这一可选变体仍未实现。

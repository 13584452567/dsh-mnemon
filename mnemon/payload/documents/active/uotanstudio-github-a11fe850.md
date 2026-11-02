---
id: "a11fe850-25f2-4449-9cbd-103b94411c53"
title: "UotanStudio GitHub 镜像源清单维护规程"
description: "HttpDownload.MirrorSites 镜像前缀清单的落点、准入判定、实测证据（删 rapidgit/tbedu、加 g.blfrp.cn）、枚举与测试验证步骤及工作区陷阱。"
status: "active"
created_at: "2026-10-28T13:36:48.335Z"
updated_at: "2026-10-28T13:36:48.335Z"
content_hash: "4e02fe1ed79a23dbf2b2db457a06f06bf9be699debb2185bef53f6698f3c0450"
source_paths:
  - "UotanStudio.Net/Http/HttpDownload.cs"
  - "UotanStudio.Net/GitHub/GitHubMirrorSite.cs"
  - "UotanStudio.Tests/Net/HttpDownloadMirrorUrlTests.cs"
session_ids:
  - "af5c5934-5c88-4af7-bd63-0dc4db94ae3e"
memory_body_ids:
  []
---

# UotanStudio GitHub 镜像源清单维护规程

## 1. 落点与契约

- 清单的唯一事实来源是 `UotanStudio.Net/Http/HttpDownload.cs` 的 `MirrorSites`，**不是** UI 设置项；宿主通过 `GitHubReleaseClient` / `ScrcpyServerReleaseClient` / `RootPatchReleaseClient` / `AtomicPublish` 消费它。
- `GitHubMirrorSite`（`UotanStudio.Net/GitHub/GitHubMirrorSite.cs`）只有 `Name` / `BaseUrl` / `IsEnabled` 三个 `required init` 属性；`BaseUrl` 为空表示源站直连条目，枚举与测速中固定首位且不做地址改写。
- 条目是纯前缀形态：`BaseUrl + 原始 GitHub 地址`。`HttpDownload.DownloadAsync(..., IReadOnlyList<Uri>? sources = null)` 允许调用方传入已排好序的候选源以跳过重复测速；不传时仍走 `SelectSourcesBySpeedAsync`（256 KB 测速探针，源站始终参与）。
- 同一份清单同时服务 release 资产与 raw 路径（只替换 marker），所以条目必须两种形态都成立。
- 形态用例集中在 `UotanStudio.Tests/Net/HttpDownloadMirrorUrlTests.cs` 的 InlineData：覆盖尾斜杠形态、query 形态（`down.npee.cn/?`）、路径形态（`xget.xi-xu.me/gh`、`cors.isteed.cc/github.com`）与空前缀源站。

## 2. 准入判定（任一条不满足即不加入）

1. **前缀可拼接 release 资产** —— 排除 clone-only 端点（`gitclone.com`、`githubfast.com`、`ssh://git@ssh.github.com:443/`），混入会被测速当成坏源。
2. **同一前缀能服务 raw 路径** —— 排除 `fastly.jsdelivr.net/gh` 这类 `owner/repo@branch/path` 适配器，单列表结构表达不了。
3. **本机实测可连通** —— `gh.catmak.name` 在上游列表里未注释，但本机 release 与 raw 均超时，不纳入。
4. **上游注释条目视为已废弃** —— 需实测确认后删除，不凭注释直接删。

## 3. 本轮实测证据（删 2 条、加 1 条）

| 条目 | 上游说明 | 本机实测 |
| --- | --- | --- |
| `rapidgit.jjda.de5.net` | `挂了，522超时` | 503 / 1.3 s |
| `github.tbedu.top` | `DNS_PROBE_FINISHED_NXDOMAIN` | DNS 无记录 / 43 ms |
| `https://g.blfrp.cn`（新增，上游「日本 3」） | 非注释 | 探针下载 `scrcpy-server-v2.4` 成功 69007 字节，与源站一致 |

探针横向结果：该主机上多数镜像对 256 KB 测速请求返回 `ERR 12010/12038`（连接被中止 / 任务取消）或 404 / 429，仅 `GitHub 直连` 与少数镜像返回 206。结论是连通性高度因站点与网络而异，必须逐个实测，不能按上游顺序信任。

清单现状：源站直连固定首位，其后镜像顺序与上游 `download_url` 一致，新条目追加在末尾（本轮为 源站 + 31 个镜像）。

## 4. 验证步骤

1. `dotnet build UotanStudio.Net/UotanStudio.Net.csproj --nologo -c Debug` → 期望 0 警告 0 错误。
2. `dotnet format` whitespace `--verify-no-changes`（对触及文件）。
3. 枚举实测：临时 file-based app 引用 `UotanStudio.Net`，断言 `RELEASE_SOURCES == DISTINCT == 条目数`、首条为源站、地址中无重复 `github.com`、raw 地址同样条数且首个镜像为 `…/https://raw.githubusercontent.com/…`、非 GitHub 地址仍只 1 条；跑完删除临时工程。
4. 端到端：`HttpDownload.DownloadAsync` 拉一个真实 release 资产，比对字节数。
5. 定向测试 `--filter-class UotanStudio.Tests.Net.HttpDownloadMirrorUrlTests`。

临时探针脚本与临时工程一律放 `%TEMP%` 并在用完后删除，不留仓库或用户目录。

## 5. 陷阱

- 宿主工程可能因**他人未提交改动**无法构建：本轮 `UotanStudio/Views/Editors/RockchipConfigEditorView.axaml(77,37)` 报 `AVLN2000: Unable to resolve property or method of name 'HasSelection'`，导致 `UotanStudio.Tests` 跑不起来。按仓库 Git 纪律不回滚他人文件，把新增 InlineData 记为仅静态核对。
- PowerShell 探针用单引号内联数组容易触发 `',' 后缺少表达式` 解析错误；把脚本写成 `.ps1` 文件再执行，不要塞进 `-Command`。
- 只改清单与对应 InlineData，不要顺手动工作区里的并行改动。

---
id: "7fd11d8c-2139-4784-a57d-567e1ccf0b07"
title: "UotanStudio Root 修补清单预热与一次性测速（会话级缓存契约）"
description: "Root 修补包：每个来源清单会话内只读一次、整会话只测速一次并固定最优源、启动预热与日志去重的契约变化、实测数字与未验证边界。"
status: "active"
created_at: "2026-10-29T00:32:37.542Z"
updated_at: "2026-10-29T00:32:37.542Z"
content_hash: "bcc9295f605a0f6cb5834bbe15b667f9626e4346a5d4f0af3e86453e5a411d34"
source_paths:
  - "UotanStudio/Services/PatchPackages/RootPatchPackageService.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.Root.cs"
  - "UotanStudio/ViewModels/Flash/FlashViewModel.ViewBridge.cs"
  - "UotanStudio.Tests/Services/RootPatchPackageTests.cs"
  - "VERIFICATION.md"
session_ids:
  - "838e6630-b2fc-4f9f-aeb1-034ecf6c4e6c"
memory_body_ids:
  []
---

# UotanStudio Root 修补清单预热与一次性测速（会话级缓存契约）

关联文档：`4824dc1f-7c5c-4c92-b4df-7c87c0e0bcef`（下载源测速与最优源固定）、`1625ca1e-b4e4-4731-9cdf-933a93898b1b`（烧录页 Root 修补接线）。本文只记录「清单与测速改为会话内一次」这一组契约变化；分层、校验、UI 接线沿用上述两篇。

## 1. 变更动机

原行为是「每次切来源 / 每次进烧录页都重新拉清单并重新对全部镜像测速」，导致同一会话内重复的上游请求与重复日志（下载源列表 5 个来源 × 多镜像）。新契约把开销压到启动一次：清单每个来源只读一次，测速整会话只做一次，日志只在真正发生网络动作时输出。

## 2. 服务层契约（UotanStudio/Services/PatchPackages/RootPatchPackageService.cs）

- `ListPackagesAsync`：每个来源只读取一次 Release 源并缓存结果；并发调用共享同一个在途请求（同一 Task），失败不写入缓存，因此失败后仍可重试。
- 新增 `CatalogLoaded(source)` / `TryGetCatalog(source, out …)`：只读查询缓存状态，供界面判断是否真要显示「正在获取版本列表…」。
- 新增 `WarmUpAsync`：并行读取全部 5 个来源（Magisk / Magisk Alpha / KernelSU / SukiSU / APatch）的清单，随后用最新条目对源站与镜像做**唯一一次**测速并固定最快源。单个上游失败只记一条警告并跳过，不阻断其余来源。
- `ProbeSourcesAsync` 改为会话内单次：首次真正测速并固定站点部分，之后直接返回已固定的结果，不再联网。

## 3. 宿主接线

- `FlashViewModel.Root.cs` / `FlashViewModel.ViewBridge.cs`：烧录页构造时调用一次 `WarmRootCatalogs()`；修补 Boot 窗口随后只读缓存，切来源、切版本都不再触发请求。
- 日志压缩策略：`BootPatch_VersionsLoading`（正在获取版本列表）只在 `CatalogLoaded` 为 false 时输出；`BootPatch_ProbingSources`（正在测速镜像源）只在尚未固定源时输出；窗口状态栏同一判断。「正在下载 X…」/「补丁包 X 已就绪。」只在本地尚未存有该版本时输出，并记住已准备的版本，使同一版本的重复选中、列表刷新、起刷前二次解析都不产生第二组日志；缓存命中时包解析完全静默。

设计理由：把「用户是否真的要用这个来源/版本」从触发条件降级为缓存命中判断，使界面日志对应真实的网络动作，而不是每次交互的重放；并行预热让冷启动的网络代价与用户配置其它选项的时间重叠。

## 4. 验证（2026-10-29，Windows x64，Debug，共享工作树）

- 定向：`RootPatchPackageTests` 24/24（新增三条：重复调用只读一次清单、预热缓存 5 个来源且不重复读源、第二次测速不再发请求）；`UotanStudio.Tests.Flash` 39/39。
- 全量 `dotnet test --solution UotanStudio.slnx --no-build -c Debug`：7830 总数 / 7822 通过 / 8 跳过 / 0 失败（1m38s）。
- 构建 0 错误 0 警告；`dotnet format` 的 `whitespace` / `style` 对本轮文件零报告。
- NativeAOT 发布成功，`UotanStudio.Desktop.exe` 156,262,912 B；仅第三方既有 IL2104/IL3053，无指向本轮代码的裁剪/AOT 警告。数字已记入 `VERIFICATION.md`。

## 5. 未验证边界

- GUI 中「预热完成」与修补窗口打开的时序、真实镜像的测速耗时与命中率没有自动化覆盖，需人工在界面确认一次。
- 预热为最优努力（best effort）：若启动阶段网络不可用，后续首次打开修补窗口仍需自行补拉清单。

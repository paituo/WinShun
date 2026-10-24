# 方案二：WinShun 作为 amane 前端

- 文档日期：2026-10-24
- 目标分支：`feat/media-library`
- 对照文档：[design-media-library.md](design-media-library.md)（方案一：C++ 重写检索核心，把 amane 的能力下沉进 WinShun）
- 参考实现：amane 0.22.1（本地副本 `10c4b35`）

---

## 1. 结论摘要

**可行，且是两条路线里见效更快的一条。** amane 已经把「前端需要的一切」做成了 HTTP 契约：REST 资源、OpenAPI 描述文件（`scripts/export_openapi.py` 直接导出 `web/openapi.json`）、WebSocket 广播、AG-UI 的 SSE 流、HttpOnly cookie 鉴权，以及一份现成的 Windows 监督壳（`winapp/`：互斥体、Job Object、退出码 3/4、token 文件、3 秒轮询 `/api/system/desktop`）。WinShun 需要补的是「Qt 版监督壳 + QML 客户端 + 本地预览」，而不是重写检索。

可行性评级：

| 维度 | 评级 | 说明 |
|---|---|---|
| 接口完备度 | 高 | 检索、详情、分类、演员、任务、智能体、保存查询、资源图、播放均有端点 |
| 契约稳定性 | 中高 | 有 OpenAPI 可导出，可生成 C++ DTO；但有版本漂移风险 |
| 开发量 | 中 | 主要工作在 QML 页面与图片/SSE 管道，约 8 个页面 |
| 运行成本 | 中高 | 需随包分发 Python onedir（体积、首次启动、杀软误报） |
| 许可证风险 | 中 | GPLv3 独立进程 + HTTP 聚合，明显优于把代码搬进 MIT 仓库 |
| 长期架构 | 中 | 两套前端（React + QML）要同时维护；检索能力受限于 amane |

**八条核心判断：**

1. **唯一数据源是 amane。** WinShun 不打开 amane 的数据库、不解析 NFO、不判断番号；它只发 HTTP、渲染、预览本地文件。
2. **搜索主路径是 `GET /api/metadata?search=...`。** 返回里的 `search_mode`（`like/keyword/vector/hybrid`）与 `search_degraded` 必须直接驱动 UI 的「精确 / 语义 / 降级」提示；分页深度不能超过服务端召回窗口。
3. **契约优先，代码生成。** 用 `scripts/export_openapi.py` 导出 schema，生成 C++ 结构体与枚举，只对少量端点手写薄封装。这样 amane 升级时差异是编译期可见的。
4. **图片鉴权走 HttpOnly cookie。** amane 的中间件在首次 Bearer 认证成功后下发 `amane_token` cookie（`path=/api`），专门为 `<img>` 与 WebSocket 无法带 header 的场景准备。QML 侧要么共享同一个 `QNetworkAccessManager` 的 cookie jar，要么用自建 `QQuickImageProvider` 带 header 取图。
5. **进程监督照抄 `winapp/` 的语义，用 Qt 重写。** 单实例互斥、`CreateJobObject` + `KILL_ON_JOB_CLOSE`、退出码 0/130/143 视为正常、3 视为重启、4 视为启动失败并展示原因。
6. **默认不启用 WinShun 的本地索引。** 双索引会带来语义分叉（谁决定一条文件属于哪个作品）。本地索引只在「服务不可用时的离线浏览」里作为可选降级，见 §6.3。
7. **刮削入口从 UI 上消失，但后端自动化要显式关掉。** 前端不提供批量刮削/整理按钮；同时库的 `automation` 需设为 `none`，否则会出现「前端没让刮、后台在刮」的认知不一致。
8. **「打开 Web UI」保留为逃生舱。** amane 的 `feeds/schedules/plugins/network/settings` 这类低频页面不重做，直接调用系统浏览器打开本地 SPA。这样能用 8 个页面覆盖 80% 的日常使用。

**一句话取舍**：这条路线把 WinShun 从「搜索引擎」变成「amane 的原生桌面客户端」，用最少的改造换取完整的元数据、向量检索、智能体能力；代价是安装包、双前端维护，以及长期被 amane 的 API 牵着走。**建议先用它做 MVP 验证产品价值，再决定是否走方案一的「下沉」路线。**

---

## 2. 范围定义

### 2.1 「前端」包含什么

- 进程监督：启动/停止/重启 amane 服务，展示启动失败原因，崩溃自动拉起
- 原生 UI：搜索、结果、详情、分类、演员、任务、智能体、库管理
- 本地能力：视频首帧/文档预览、外部播放器拉起、文件定位、截图/复制
- 桌面集成：全局快捷键唤起、托盘、开机自启、深链（`winshun://meta/123`）

### 2.2 不包含什么

- 不重写检索、分词、向量、融合（全部交给 amane）
- 不解析 NFO、不识别番号、不刮削、不整理文件
- 不内嵌 amane 的 React 前端（除非选择形态 F2，见 §4）
- 不在 WinShun 进程内嵌 Python 解释器
- 不直接读写 amane 的 SQLite（并发写会破坏 WAL 语义与业务不变量）

### 2.3 与方案一的关系

| | 方案一（下沉） | 方案二（前端） |
|---|---|---|
| 检索实现 | C++ 重写 | amane 提供 |
| 数据权威 | WinShun 的 SQLite | amane 的 SQLite |
| 依赖 | 无 Python | Python onedir |
| 开发量 | 大（数据层+检索+UI+智能体） | 中（UI+管道+监督） |
| 许可证 | 需独立实现 | 独立进程聚合 |
| 适合 | 长期产品化 | 快速验证 / 已有 amane 用户 |

---

## 3. amane 后端接口盘点

前缀统一为 `/api`；除 `/api/health` 外均需 Bearer token 或同名 cookie。

### 3.1 运行与生命周期

| 端点 | 用途 | 前端要点 |
|---|---|---|
| `GET /api/health` | 就绪探针 | 唯一免鉴权端点，返回 `{status, version}` |
| `GET /api/system/desktop` | 桌面壳状态 | 返回 `{version, data_dir, supervised}`，可 3 秒轮询 |
| `POST /api/system/restart` | 请求重启（202） | 服务以退出码 3 退出，由监督者拉起 |
| `GET /api/system/release` | 检查更新 | 返回 `{newer, html_url}`，前端可直接开浏览器 |

### 3.2 库与文件

| 端点 | 用途 | 前端要点 |
|---|---|---|
| `GET/POST/PATCH/DELETE /api/libraries` | 库 CRUD | `automation` 取 `none/watch/scrape`；`ingest` 取 `native/clouddrive/everything` |
| `GET /api/libraries/{id}` | 库详情 | 含根目录、模板、扩展名等 |
| `GET /api/files?path=...` | 服务端目录浏览 | 用于「添加库」时选目录；受 `safe_dirs` 限制 |
| `GET /api/media` | 文件列表 | 支持 `search`（路径或番号）、`status`、`library_id`、`has_subtitle`、`mosaic`、`uncensored`、`definition`、`content_type`，**没有 `metadata_id` 过滤**；按作品取文件用详情接口 |
| `GET /api/media/{id}` | 文件详情 | 返回 `path`（本地绝对路径；CloudDrive 库可能是虚拟路径）、`size/duration/codec/has_subtitle/definition` |
| `PATCH /api/media/{id}` | 改文件属性 | 只改数据库标记，不动磁盘 |
| `POST /api/media/batch/delete` | 批量删记录 | **危险**：前端需二次确认且明确「只删记录」语义 |

### 3.3 检索与详情（核心）

| 端点 | 用途 | 前端要点 |
|---|---|---|
| `GET /api/metadata` | 列表 + 检索 | 参数：`search`、`offset`、`limit≤200`、`sort_by`、`order`、`actor_id/tag_id/studio_id/series_id/library_id...`、`has_files`、`has_subtitle`、`mosaic`、`uncensored`、`definition`、`content_type`、`saved_query_id` |
| `GET /api/metadata/{id}` | 详情 | 返回 `metadata` + `files` + `user_tags` + `comments` + 各类 facet id 映射 |
| `PATCH /api/metadata/{id}` | 局部更新字段 | 只改数据库；受 `locked_fields` 约束 |
| `PUT /api/metadata/{id}/locks` | 替换锁定字段集合 | UI 用锁图标 |
| `POST /api/metadata/{id}/merge` | 合并重复作品 | 低频，可放二级菜单 |
| `GET /api/metadata/schema` | 字段 JSON Schema | 可用于生成编辑表单 |
| `POST /api/metadata/batch/scrape` | 批量刮削 | **前端不暴露** |

`GET /api/metadata` 的响应带 `search_mode` 与 `search_degraded`，含义：

- `like`：未启用索引的子串扫描（`total` 精确）
- `keyword`：FTS5 关键词（`total` 精确）
- `vector` / `hybrid`：向量路或混合路，`total` 是**召回窗口内**的候选数，不是全库计数

因此分页必须满足 `offset + limit ≤ recall_window`，否则后续页没有可信顺序。UI 应当在到达窗口边界时禁用「下一页」。

### 3.4 分类、演员、用户注解

| 端点 | 用途 |
|---|---|
| `GET /api/facets/{kind}` | 某类 facet 列表（`kind` 见 `FacetKind`） |
| `GET/POST/PATCH/DELETE /api/facets/{kind}/rules[/{id}]` | 别名/屏蔽规则 |
| `POST /api/facets/{kind}/batch`、`POST /api/facets/{kind}/merge` | 批量与合并 |
| `GET /api/actors`、`GET/PATCH /api/actors/{id}` | 演员列表/详情/编辑 |
| `POST /api/metadata/{id}/comments`、`PATCH/DELETE /api/comments/{id}` | 评论 |
| `POST /api/metadata/batch/user-tags`、`/api/actors/batch/user-tags` | 用户标签批量关联 |

### 3.5 智能体与保存查询

| 端点 | 用途 | 前端要点 |
|---|---|---|
| `POST /api/agent/sessions`、`GET /api/agent/sessions` | 会话创建/列表 | |
| `GET /api/agent/sessions/{id}/trace` | 整段历史 | 首次进入会话时拉全量 |
| `POST /api/agent/sessions/{id}/agui` | 启动一个回合 | 返回 SSE（AG-UI 协议事件）；**同一会话并发会 409** |
| `GET /api/agent/sessions/{id}/agui/events?after_seq=` | 订阅「页面契约行」 | 页面渲染只认这条通道；回合结束且追平后自动关闭 |
| `POST /api/agent/sessions/{id}/agui/cancel` | 取消回合 | 客户端断开订阅**不会**取消后台回合，必须显式调用 |
| `GET /api/saved-queries`、`.../{id}`、`.../{id}/result` | 保存查询 | `result` 直接返回行数据，适合表格视图 |

SSE 分帧为 `data: {json}\n\n`；`events` 通道发的是页面契约行（气泡、工具卡片），`agui` 通道发的是 AG-UI 协议事件。**前端渲染走 `events` 通道，`agui` 通道只负责启动。**

### 3.6 实时事件、资源图、播放

| 端点 | 用途 | 前端要点 |
|---|---|---|
| `WS /api/ws` | 服务端单向广播 | 事件类型：`task.started/progress/completed/failed`、`file.discovered/removed`、`log` |
| `GET /api/resources/{url_hash}` | 本地缓存图片 | 支持 `ETag` + `If-None-Match`，SR 可能就地覆盖 |
| `GET /api/resources/proxy?url=` | 外链图片代理 | 前端把 `http(s)` 封面统一换成此地址 |
| `GET /api/playback/sources` | 播放源列表 | 依赖 amane 播放插件 |
| `GET /api/playback/{source}/{metadata_id}/...` | HLS 清单/分片/字幕 | 可选；本地文件直接交给系统播放器更简单 |

### 3.7 任务

| 端点 | 用途 |
|---|---|
| `GET /api/tasks`、`GET /api/tasks/{id}`、`GET /api/tasks/{id}/children`、`.../record` | 任务列表/详情/子任务/日志 |
| `GET/POST /api/tasks/worker/{pause,resume,enable,disable}` | 工作器状态 |
| `POST /api/tasks/batch/submit` | 批量提交（含刮削/整理类型，前端按需隐藏） |

---

## 4. 集成形态选择

| 形态 | 描述 | 开发量 | 包体积 | 体验 | 风险 | 结论 |
|---|---|---|---|---|---|---|
| **F1 原生 QML 客户端** | WinShun 用 QML 重画 amane 的 8 个页面，HTTP/WS/SSE 取数 | 中 | 中（+Python） | 最好：键盘流、秒开、原生预览 | 双前端维护 | **推荐** |
| F2 内嵌 Web UI | 用 WebView2 承载 amane 的 React SPA，WinShun 只做窗口与托盘 | 小 | 中（+WebView2 运行时，Win10/11 通常已有） | 一般：快捷键、拖拽、预览都受限 | 前端与桌面壳耦合；Qt 无官方 WebView2 封装，需要 HWND 原生宿主验证 | 备选 |
| F3 混合 | 搜索/详情用原生，设置/任务/日志用 WebView2 | 中 | 中 | 好 | 两套渲染栈 | 可选折中 |
| F4 进程内嵌 Python | 静态链接解释器或 `pybind11` 嵌入 | 大 | 大 | 与 F1 同 | GPL 边界最差、崩溃互相拖累 | 不推荐 |

**推荐 F1**，并把 F2 作为「打开 Web UI」的逃生舱（外部浏览器，不内嵌）。理由：WinShun 的核心价值是键盘优先的即时反馈，这恰恰是浏览器壳最弱的地方；而 amane 的页面信息密度高，用 QML 重画 8 个页面可控。

若后续发现 QML 重画成本超预期，可局部切到 F3，把 `settings/tasks/logs/plugins` 换成 WebView2 页面。

---

## 5. 目标架构

### 5.1 部署拓扑

```
WinShunMedia.exe（Qt6/QML，MIT）
 ├─ 监督器（QProcess + Job Object + 互斥体）
 ├─ QML UI（8 个页面 + 预览浮层）
 ├─ AmaneClient（HTTP / WS / SSE）
 └─ 本地能力（FilePreview / ThumbnailProvider / 外部播放器）
        │  http://127.0.0.1:18000/api（Bearer / cookie）
        ▼
amane/  onedir（PyInstaller，GPLv3，独立进程）
 ├─ FastAPI + uvicorn（--host 127.0.0.1）
 ├─ SQLite（media.db + search_vec.db）
 ├─ 检索栈（FTS5 + vec0 + hybrid）
 ├─ 智能体（pydantic-ai + AG-UI）
 └─ 来源插件 / 播放插件（按安装情况）
```

安装包布局：

```
%ProgramFiles%\WinShunMedia\
  WinShunMedia.exe
  amane\                 ← onedir，整目录
    Amane.Server.exe
    _internal\...
  web\dist\              ← amane 的 SPA（逃生舱用）
```

数据目录沿用 amane 约定：`%LOCALAPPDATA%\amane\`（`AMANE_DATA_DIR`）。

### 5.2 生命周期状态机

```
        ┌──────────┐  spawn   ┌───────────┐  /health ok  ┌────────┐
        │ Starting │────────▶ │ Connecting│────────────▶ │ Ready  │
        └──────────┘          └───────────┘              └────────┘
              │  exit 4 / 超时      │ 501/超时                  │ 断连
              ▼                     ▼                          ▼
        ┌──────────┐          ┌──────────┐              ┌──────────┐
        │ Failed   │◀─────────│ Retrying │◀─────────────│ Degraded │
        └──────────┘          └──────────┘   重连成功     └──────────┘
              │ 用户重试 / 改端口
              └──────────────────────────────────────────────▶ Starting
```

要点：

- **单实例**：命名互斥体（`Local\com.winshun.media`），第二个实例唤起已有窗口。
- **端口**：默认 `18000`；被占用则从 `18000–18099` 选空闲端口，写入子进程环境，前端以实际端口为准。
- **令牌**：优先读环境变量 `AMANE_TOKEN`；否则读 `%AMANE_DATA_DIR%\token`（服务首次启动生成）。`AMANE_TOKEN=off` 表示关闭鉴权（仅回环）。
- **绑定**：`AMANE_HOST=127.0.0.1`，绝不监听 `0.0.0.0`；`AMANE_SAFE_DIRS` 只放库根目录，不用 `ALLOW_ALL`。
- **退出码**：`0/130/143` 正常结束 → 退出应用；`3` → 立即重启（重启请求）；`4` → 启动失败，展示最后一条 `ERROR` 行并停止自动重试（或退避重试）。
- **Job Object**：`KILL_ON_JOB_CLOSE`，保证 WinShun 崩溃时 Python 一起退出，不留下孤儿进程占端口。
- **停止**：先 `POST /api/system/restart` 触发优雅退出（等服务自己走完 shutdown），超时 5 秒再 `TerminateProcess` 整个进程树。
- **日志**：子进程 stdout/stderr 重定向到 `%LOCALAPPDATA%\WinShun\media-shell\server.log`，保留末尾 40 行用于错误展示。

### 5.3 C++ 客户端分层

| 层 | 类 | 职责 |
|---|---|---|
| 传输 | `AmaneTransport` | `QNetworkAccessManager`、baseUrl、Bearer、cookie jar、超时、重试、错误归一化 |
| 契约 | `AmaneApi`（生成） | DTO 结构体、枚举、路径常量、查询参数序列化 |
| 薄封装 | `AmaneClient` | 组合查询参数、分页、取消、缓存策略 |
| 实时 | `AmaneEvents` | WebSocket 客户端、指数退避重连、事件分发 |
| 智能体 | `AmaneAgentStream` | SSE 增量解析（按 `\n\n` 切帧）、回合生命周期、取消 |
| 图片 | `AmaneImageProvider` | `QQuickImageProvider`，带 Bearer 取图、磁盘缓存、ETag |
| 模型 | `MetadataListModel`、`MediaFileListModel`、`TaskListModel`、`AgentTimelineModel` | `QAbstractListModel`，虚拟化友好 |
| 搜索 | `MediaQueryController` | 250 ms 防抖、请求取消、模式切换、窗口边界 |

**契约生成流水线**：

```
amane/scripts/export_openapi.py  →  web/openapi.json
        → tools/openapi2cpp（自建，Python 脚本，构建期运行）
        →  src/media-client/Generated/{Types,Enums,Endpoints}.h/.cpp
```

生成器只需覆盖本项目用到的 30 余个端点与 20 余个 schema；未覆盖的用 `QJsonObject` 兜底。构建时校验 `openapi.json` 的哈希，schema 变了则重新生成，避免「代码与契约悄悄分叉」。

### 5.4 线程与事件循环

- 所有 HTTP/WS/SSE 都是异步，不阻塞 GUI 线程。
- 单页响应可能较大（详情含 `raw`），JSON 解析放 `QtConcurrent::run`，回主线程组装模型。
- 图片解码走 `QQuickImageProvider` 的 `ImageResponse`，异步 `QImage`，避免主线程解码大图。
- 不做本地 SQLite，因此没有数据库线程问题。
- WS 重连与 SSE 重连都在主线程用 `QTimer` 调度。

### 5.5 鉴权与安全

- Bearer token 只放在 header；**不进 URL**（amane 明确设计如此，避免日志泄露）。
- 首次带 Bearer 的成功请求会下发 `amane_token` cookie（`HttpOnly`、`SameSite=Lax`、`path=/api`）。QML 图片加载需要它：
  - 首选：给 `QQmlEngine` 设置自定义 `QQmlNetworkAccessManagerFactory`，返回与 `AmaneTransport` 共享 cookie jar 的 NAM。
  - 备选：`AmaneImageProvider` 自己带 Bearer 取图并缓存（更可控，推荐）。
- 深链里只放实体 id（`winshun://meta/123`），不放 token。
- 若用户把 amane 跑在非回环地址，前端必须提示「远程模式：凭据经明文 HTTP」，并建议反向代理 + HTTPS。

### 5.6 页面映射

| 优先级 | QML 页面 | amane 对应 | 主要端点 |
|---|---|---|---|
| P0 | `MediaSearchPage.qml` | `/`（AgentHome 的搜索区） | `GET /api/metadata` |
| P0 | `MediaResultGrid.qml` / `MediaResultList.qml` | 结果区 | 同上 + facets |
| P0 | `MediaDetailPage.qml` | `/meta/$metadataId` | `GET /api/metadata/{id}`、`/api/media` |
| P0 | `MediaPreview.qml` | 详情页预览 | 本地文件 + `FilePreview` |
| P0 | `ShellStatus.qml` | 桌面壳状态/日志 | `/api/system/desktop`、WS |
| P1 | `CatalogPage.qml` | `/catalog`、`/catalog/$kind/$facetId` | `GET /api/facets/{kind}` |
| P1 | `ActorPage.qml` | `/actors`、`/actors/$id` | `GET /api/actors` |
| P1 | `AgentPage.qml` | `/`（AgentHome 对话） | agent sessions + SSE |
| P1 | `SavedQueryPage.qml` | `/saved-queries` | saved-queries + result |
| P2 | `TaskPage.qml` | `/tasks` | `/api/tasks` + WS |
| P2 | `LibraryPage.qml` | `/libraries` | `/api/libraries` |
| P2 | `SettingsPage.qml` | `/settings` | `GET/PATCH /api/config`（或直接跳 Web） |
| 逃生舱 | 「打开 Web UI」 | `/feeds`、`/schedules`、`/plugins`、`/network`、`/logs` | 系统浏览器 |

### 5.7 搜索交互

搜索框是产品的主入口，行为按 WinShun 的肌肉记忆设计：

1. 输入即搜：250 ms 防抖；新输入取消上一个 `QNetworkReply`（`abort()`），避免旧结果覆盖新结果。
2. 结果先出「本地即时项」：若启用可选本地索引，先用文件名命中占位；HTTP 结果到达后替换（默认关闭，见 §6.3）。
3. 模式徽标：`精确`（`like`）、`关键词`（`keyword`）、`语义`（`vector`）、`混合`（`hybrid`）；`search_degraded=true` 时显示「语义不可用，已退回关键词」。
4. 番号命中：整串是番号时，第一条直接是精确作品，并显示「番号精确匹配」标签。
5. 过滤芯片：演员/厂牌/系列/标签/年份/分辨率/有无字幕，与 `actor_id` 等参数一一对应。
6. 键盘：`↑/↓` 选择、`Enter` 打开详情、`Alt+P` 预览、`Ctrl+Enter` 播放、`Esc` 关闭。
7. 分页：到达 `recall_window` 边界时禁用「下一页」，提示「语义检索仅保证前 N 条顺序」。

### 5.8 详情与预览

详情页数据来自 `GET /api/metadata/{id}`：

- 顶部：封面（`poster_url` / `poster_urls`，外链统一走 `/api/resources/proxy`）、番号、标题、发行日、时长、厂牌、系列、评分。
- 文件区：每个 `MediaFileResponse` 展示 `path`、`size`、`duration`、`codec`、`definition`、`mosaic`、`has_subtitle`；双击用系统播放器打开本地 `path`。
- 演员区：头像 + 别名；点击进 `ActorPage`。
- 注解区：用户标签、评论（可写）。
- 相似推荐：调用 `GET /api/metadata?search=<标题>&limit=8` 或按演员过滤；assistant 生成的关系不入库。
- NFO 原文：若详情响应含 `nfo_path`（待 Phase 0 确认），用 `FilePreview` 的文本模式渲染；否则隐藏该块。

预览复用 WinShun 现有 `FilePreview`：图片/文本/文档/文件夹已覆盖；新增视频首帧（`Kind::Video`）与「媒体信息卡」（时长、分辨率、编码、字幕）。这样即使 amane 没有提供缩略图，也能本地出图。

### 5.9 智能体

- 入口：搜索页右侧抽屉或独立 `AgentPage`（`Ctrl+/` 唤起）。
- 会话：`POST /api/agent/sessions` 建会话；首次进入用 `/trace` 拉历史。
- 发消息：`POST /agui` 启动回合（后台执行），同时订阅 `GET /agui/events?after_seq=<last>` 渲染气泡与工具卡片。
- 取消：显式调 `/agui/cancel`；关闭页面只断开订阅，回合仍在跑，重进要接 `after_seq`。
- 审批：遇到 `ApprovalRequired` 时，服务端返回中断；UI 渲染「批准/拒绝」卡片，再以新的消息继续。
- 结果下钻：工具卡片里的实体 id 直接跳 `MediaDetailPage`。
- 未配置模型：`POST /agui` 返回 503（「助理 Agent 未配置」），UI 引导打开 Web 设置页。

### 5.10 任务与实时通知

- 订阅 `WS /api/ws`，把 `task.*` 事件映射到状态栏进度；`file.discovered` 触发当前列表的软刷新（只刷新可见页，避免全量重载）。
- 不做轮询；只在 WS 断开且重连失败时退化为 30 秒轮询 `/api/tasks?limit=5`。
- `log` 事件进「诊断」抽屉，默认折叠。

### 5.11 降级与错误

| 场景 | 表现 |
|---|---|
| 服务启动中 | 骨架屏 + 「正在启动媒体服务…」；3 秒后仍无 health 则展示日志末尾 |
| 启动失败（exit 4） | 对话框展示最后一条 `ERROR`，提供「复制日志」「打开数据目录」「重试」 |
| 运行中断连 | 顶部条「服务已断开，正在重连」，只读缓存仍可浏览已加载的详情 |
| 401 | 令牌失效：重新读取 token 文件并重试一次，仍失败则提示重新配对 |
| 503（智能体未配置） | 引导到 Web 设置 |
| 分页越界 | 禁用下一页并解释召回窗口 |
| 磁盘/库不可用 | 详情页仍显示元数据，文件区标注「文件不在线」 |

---

## 6. 与现有 WinShun 的关系

### 6.1 保留

- 窗口框架、主题、缩放、多显示器、托盘、全局快捷键
- 输入框、列表/网格虚拟化、键盘导航、右键菜单
- `FilePreview`（+ 新增 `Video`）、`ThumbnailProvider`、QuickLook/Seer 外接
- 外部播放器拉起逻辑
- MCP 服务（可选：把 amane 的智能体能力再暴露给其他客户端）

### 6.2 停用（媒体版构建）

应用/系统设置搜索、网页搜索、收藏夹、对话框搜索、资源管理器集成、剪贴板历史、任务栏搜索框、OCR/截图/标注、内容全文搜索。

### 6.3 本地索引：默认关闭，作为可选降级

WinShun 的 MFT 索引如果同时运行，会出现两个「真相」：本地索引说文件在，amane 说不在（或反之）。因此：

- 默认 `media.localIndex = off`，启动器只做 HTTP。
- 可选 `on` 时，本地索引只用于两件事：
  1. 输入时的即时文件名建议（不显示元数据，不参与排序权威）；
  2. 服务不可用时的「离线浏览」（只显示路径与文件名，点开用系统播放器）。
- 绝不把本地索引结果与 amane 结果混排；两者用不同视觉分区。

### 6.4 若要让 WinShun 承担发现通道（可选，需改 amane）

amane 现有发现通道是 native(watchdog)/clouddrive(webhook)/everything(es.exe)，没有「外部推送文件事件」的通用 API。若希望用 WinShun 的 MFT 索引替代 watchdog：

- 方案 A：给 amane 增加 `POST /api/ingest/events`（批量 add/remove/rename 事件，带库根校验）。改动小、语义清晰，但要维护补丁分支。
- 方案 B：复用 `POST /api/webhooks/clouddrive`，把库设为 `ingest=clouddrive`，由 WinShun 把本地路径映射成虚拟路径后推送。**不推荐**：语义错位，虚拟路径规则与本地盘不一致。
- 方案 C：不改后端，关掉 amane 的 native watcher，靠定时 `scan`（当前没有公开扫描端点，需确认）或任务触发。体验最差。

**建议先不做**。方案二的目标是验证前端体验，发现通道保持 amane 原生；等确认要长期走这条路，再做方案 A 的补丁（并接受维护成本）。

---

## 7. 实施阶段

### Phase 0：契约与管道验证（go / no-go）

| 编号 | 验证 | 通过标准 | 失败退路 |
|---|---|---|---|
| P0-1 | 导出 `openapi.json` 并生成 C++ DTO | 30 个目标端点全部生成，编译通过 | 手写 6 个核心端点 DTO，其余用 `QJsonObject` |
| P0-2 | 进程监督 PoC | 启动 onedir、读到 token、health 通过、退出码 3/4 行为正确、Job Object 生效 | 用 `AMANE_UI_ONLY` 连外部已启动服务 |
| P0-3 | 图片鉴权 | QML `Image` 能通过共享 cookie 或 ImageProvider 取到图 | 全部走 `AmaneImageProvider`（必成） |
| P0-4 | SSE 增量解析 | `/agui/events` 能实时解析出行并渲染 | 改轮询 `/trace` |
| P0-5 | 搜索语义 | `search_mode/search_degraded` 与分页窗口行为符合预期 | 固定 `limit≤50`，不做深分页 |
| P0-6 | 本地文件可得 | `MediaFileResponse.path` 指向本机可读路径 | 远程模式下降级为「仅元数据」 |

产物：`docs/design-amane-frontend-phase0.md` + 验证脚本 + 一份生成的 DTO 头文件。

### Phase 1：启动器与搜索（P0）

- 监督壳（互斥体、端口、token、状态机、日志）
- `AmaneTransport` + `AmaneClient` + `MetadataListModel`
- `MediaSearchPage` + 结果列表/网格（虚拟化）+ 防抖/取消/分页边界
- 模式徽标与降级提示

验收：冷启动到可搜索；输入 300 ms 内出结果；结果与 amane Web 端一致（同参数对拍）。

### Phase 2：详情、预览、分类（P0/P1）

- `MediaDetailPage`（字段、文件、演员、标签、评论）
- `FilePreview` 新增 `Video` 与媒体信息卡
- 「用播放器打开」「在资源管理器中显示」「复制番号/路径」
- `CatalogPage` + `ActorPage` + facet 过滤芯片

验收：从结果到播放不超过 2 次操作；封面加载 P95 < 500 ms（本地缓存后 < 50 ms）。

### Phase 3：智能体（P1）

- 会话列表/历史、SSE 渲染、工具卡片、审批卡片、取消
- 结果下钻、保存查询

验收：完整跑通「找某演员今年 4K 无字幕作品」并打开详情；断线重连后能接上进度。

### Phase 4：任务、库、设置与逃生舱（P1/P2）

- `TaskPage` + WS 进度 + 工作器开关
- `LibraryPage`（创建库时强制 `automation=none`，并展示说明）
- 设置页只做常用项（端口、token、数据目录、外部播放器），其余跳 Web
- 「打开 Web UI」入口

验收：创建库后能立刻搜索到已刮削数据；关闭后台刮削的路径清晰可见。

### Phase 5：打包、升级与性能

- 安装包集成 amane onedir + `web/dist`
- 首次启动引导（数据目录、库根、模型/LLM 配置跳 Web）
- 升级策略：WinShun 与 amane 版本号绑定，安装时校验 `openapi` 指纹
- 长稳测试（8 小时）、休眠恢复、端口冲突、杀软误报处理

验收：安装包可离线安装；升级失败可回滚；连续运行无句柄/内存增长。

---

## 8. 风险与缓解

| 风险 | 等级 | 说明 | 缓解 |
|---|---|---|---|
| GPL 聚合边界 | 中 | amane 为 GPLv3，作为独立进程通过 HTTP 通信通常视为聚合，但分发时仍须提供其源码与许可 | 不静态链接、不复制代码；安装包内附 `amane` 源码获取方式与 LICENSE；法务复核 |
| Python 运行时体积 | 中高 | onedir 预计 150–250 MB，首次启动慢 | 只装 `vector`/必要 extra；预编译 `.pyc`；启动画面与进度 |
| 杀软误报 | 中 | PyInstaller 产物常见 | 代码签名、提交白名单、提供 zip 版 |
| API 漂移 | 中 | amane 迭代快 | 契约生成 + 启动时校验 `/api/system/desktop` 的 version；版本不匹配时提示 |
| 双前端维护 | 中 | React 与 QML 语义漂移 | QML 只覆盖 8 个高频页面；低频页面直接跳 Web |
| 深分页不可用 | 中 | 向量模式 `total` 受召回窗口限制 | UI 明确边界；提供「按番号/演员精确过滤」替代深翻页 |
| SSE 长连接 | 中 | 休眠/代理会断 | 重连 + `after_seq` 续传 |
| 端口冲突/多实例 | 中 | 18000 被占 | 端口扫描 + 单实例互斥 |
| 本地路径不可读 | 中 | 远程 amane 模式下 `path` 不在本机 | 检测 `path` 存在性；缺失则隐藏播放并标注 |
| 安全 | 中 | `ALLOW_ALL`、远程暴露、token 落盘 | 回环绑定、safe_dirs 收紧、token 文件 ACL、远程模式警告 |
| 图标/封面加载风暴 | 低中 | 网格一次请求上百张 | 并发上限（4–6）、懒加载、磁盘缓存、失败占位 |
| 用户预期 | 低 | 以为 WinShun 自带检索 | 首次启动说明「元数据与检索由本地服务提供」，展示服务状态 |

---

## 9. 决策点

| 编号 | 决策 | 选项 | 建议 |
|---|---|---|---|
| E1 | 形态 | F1 原生 / F2 WebView / F3 混合 | F1，保留 F3 作为退路 |
| E2 | 是否随包分发 amane | 同装 / 检测已装 / 首次下载 | 同装（离线可用），版本锁定 |
| E3 | 本地索引 | 默认关 / 默认开 / 仅离线 | 默认关，离线可选 |
| E4 | 是否给 amane 打补丁（入库事件 API） | 打 / 不打 | 先不打 |
| E5 | 智能体入口 | 独立页 / 搜索页抽屉 | 抽屉优先，深度使用再开独立页 |
| E6 | 低频页面 | QML 重画 / 跳 Web | 跳 Web |
| E7 | 版本策略 | 严格绑定 / 兼容窗口 | 严格绑定 + 启动校验 |

---

## 10. 验收标准

**用户旅程**（全部要脚本化或用例化）

1. 双击启动 → 3 秒内出现搜索框（服务已在运行）／15 秒内（冷启动含 Python）。
2. 输入番号 → 首条为精确作品 → `Enter` 进详情 → `Ctrl+Enter` 播放。
3. 输入自然语言 → 出混合结果 → 断网/模型未就绪时显示降级提示。
4. 演员页 → 过滤「4K + 无字幕」→ 结果正确 → 保存为查询。
5. 智能体：「找出某演员今年所有作品，按评分排序」→ 结果卡片可下钻。
6. 杀掉 Python 进程 → 10 秒内自动恢复，UI 状态正确。
7. 拔掉库所在磁盘 → 详情仍可看，文件区标注离线。

**性能**

- 搜索：输入到首屏 P95 < 300 ms（含本地 HTTP）
- 详情：打开 P95 < 400 ms（不含图片）
- 封面：本地缓存命中 < 50 ms
- 网格：1 万条滚动 60 fps

**稳定性**

- 连续 8 小时：句柄、内存无持续增长
- 休眠唤醒后 WS/SSE 自动重连
- 端口被占用时能换端口并提示

---

## 附录 A：环境变量与文件

| 名称 | 作用 | 前端用法 |
|---|---|---|
| `AMANE_DATA_DIR` | 数据目录（DB、日志、token） | 设置页展示、打开目录 |
| `AMANE_LOG_DIR` | 日志目录 | 诊断抽屉 |
| `AMANE_HOST` | 监听地址，默认 `127.0.0.1` | 强制回环 |
| `AMANE_PORT` | 端口，默认 `18000` | 冲突时改 |
| `AMANE_SAFE_DIRS` | 允许浏览的目录白名单 | 创建库时选目录 |
| `AMANE_TOKEN` | API token，`off` 关闭 | 配对 |
| `AMANE_SUPERVISED` | 是否由监督者启动 | 影响重启行为 |
| `AMANE_UI_DISABLED` | 关闭 Web UI 静态服务 | 可省内存 |
| `AMANE_WEB_DIST` | SPA 目录 | 逃生舱 |
| `%AMANE_DATA_DIR%\token` | 自动生成的 token | 无环境变量时读取 |

退出码：`0/130/143` 正常；`3` 重启；`4` 启动失败。

## 附录 B：为什么不做「WinShun 直接读 amane 的 SQLite」

- WAL 允许多读者，但**多写者会破坏业务不变量**：索引同步、任务队列、事件广播都在 Python 侧。
- 绕过 Repository 会绕过字段锁定、字段来源、别名投影等规则。
- 读侧也有风险：`search_vec.db` 是派生缓存，可能正在重建；直接读会读到半成品。
- 结论：只走 HTTP。若需要离线只读，另做导出（未来可选）。

## 附录 C：与方案一的衔接

如果方案二验证成功但遇到以下任一情况，应启动方案一的下沉：

- Python 运行时的体积/启动/误报成为主要投诉
- 需要把 WinShun 的 MFT 索引作为权威发现通道
- 需要单文件绿色分发或离线环境部署
- amane 的 API 演进速度超过前端跟进能力

两条路线共用同一套 QML 页面与交互模型——方案二写的 UI 可以直接迁移到方案一，只替换数据层（HTTP → `media::QueryService`）。这也是建议先做方案二的另一个理由。

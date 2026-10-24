# 媒体库整合方案：WinShun 索引层 × amane 元数据与检索层

- 文档日期：2026-10-24
- 目标分支：`feat/media-library`（自 `main` 切出）
- 涉及仓库：WinShun（MIT，C++/Qt6/QML，Windows）、amane（GPLv3，Python 3.14/FastAPI/React，版本 0.22.1，分析用本地副本 `10c4b35`）
- 文档性质：可行性分析 + 实现落地方案，不含代码改动

---

## 1. 结论摘要

**有同类工具，但没有完全等价的组合。** 调研到的工具可以归为三类路线（见 §4）：单二进制内嵌 SQLite + 内嵌推理（Stash、PhotoPrism、Whisparr）、服务端 + Postgres/pgvector + 独立 ML 服务（Immich、Khoj）、本地 sidecar + SQLite/sqlite-vec + 多信号融合（PrivateLens、media-indexer、rclip、VidXP）。它们各自覆盖了「媒体元数据管理」「向量检索」「本地文件监控」中的一到两项，但**没有一个同时具备 WinShun 级的 NTFS MFT/USN 实时索引、amane 级的番号/演员/系列一等公民模型、以及可用的智能体查询层**。因此用户的集成思路成立，但要按「取两者之长、不搬两者之壳」的方式落地。

**可行性结论：可行，建议按单进程 C++ 落地，不把 Python 引入运行时。** 关键判断：

1. **发现层用 WinShun 的 MFT + USN 取代 amane 的三通道。** amane 的 native(watchdog)/clouddrive(webhook)/everything(es.exe 轮询) 是跨平台与无特权环境下的折中；WinShun 直接读 MFT、追 USN 日志、`ReadDirectoryChangesW` 兜底，信息更全（FRN、精确重命名、删除），延迟更低。CloudDrive 类虚拟盘用 `EntryFlag::Offline/Remote` + 定时对拍补足。
2. **权威数据必须是 SQLite，而不是内存文件树。** `FileIndex` 的 `EntryId` 是重建后不稳定的数组下标，不能做媒体文件身份；而 `RecordTable` 已能把 NTFS 记录号（FRN）映射到 `EntryId`，USN 记录本身也携带 FRN。用 `(volume, frn)` 做稳定键、`path` 做投影，重命名/移动可以零成本同步到媒体库。
3. **检索分三层而不是一套索引。** L0 文件名/路径层直接用现有 `FileIndex`（微秒级，边打字边出）；L1 元数据层用 SQLite FTS5 + BM25；L2 向量层复用 `winshun.semantic` 已有的 int8 量化 + mmap 段文件 + 精确余弦（在十万级作品规模下足够，免去 sqlite-vec 依赖）。最后做候选集内 z-score 融合与精确番号置顶。
4. **「只管理刮削好的数据」意味着入口是 NFO，不是爬虫。** 删除 amane 的 crawlers/aggregate/organize 全链路，`MediaFile`/`Metadata`/`Resource` 保留；`Library` 退化为「根目录 + 排除规则 + 扩展名白名单」，不再有 move_mode、模板、trash、CD 检测等整理语义。
5. **智能体不重写一遍，挂到现有 MCP 上。** WinShun 已有 `src/mcp/`（McpServer/JsonRpc/PipeServer/relay），把 amane 的 `sql_explore/sql_deliver/inspect_result` + 少量写 Capability 实现成 MCP 工具，既复用现有 AI 客户端配置，又天然隔离权限。
6. **许可证是硬约束。** WinShun 为 MIT，amane 为 GPLv3。**不得复制 amane 源码进 WinShun**，否则整个 WinShun 需按 GPLv3 发布。方案按「依据其公开文档描述的规则做独立实现」推进（算法与数据模型思想不受版权保护，代码表达受保护）。此点需在开工前由项目负责人确认，见 §9 决策 D8。

**不建议做的事**：不把 amane 整套 FastAPI 服务塞进安装包作为运行时依赖；不为了复用 React 前端引入 QtWebEngine；不做刮削、整理、转码、去重（除文件级重复提示外）；不在第一阶段实现任何破坏性文件操作。

---

## 2. 需求与边界

### 2.1 目标拆解

| 编号 | 目标 | 来源 | 验收形态 |
|---|---|---|---|
| G1 | 磁盘检索与监视机制 | WinShun | 媒体目录新增/删除/重命名/移动在秒级反映到库中 |
| G2 | 视频管理 | amane | 作品（Metadata）与文件（MediaFile）分离，一番多文件、多版本、字幕、mosaic/破解/中字相位 |
| G3 | 向量检索 | amane | 自然语言（中文/日文/英文）找到作品，返回排序结果 |
| G4 | 智能体 | amane | 会话式查询、只读 SQL、结果可下钻、可保存为 Saved Query |
| G5 | 媒体分类 | amane | 番号前缀、厂牌、演员、系列、年份、标签等 Facet 过滤与聚合 |
| G6 | 查询 | 二者 | 番号精确、标题/演员/标签关键词、拼音、混合模式 |
| G7 | 展现 | 二者 | 列表/网格/详情/预览，键位沿用 WinShun 肌肉记忆 |
| G8 | 只管理已刮削数据 | 用户 | NFO/本地元数据导入，不发起任何网络刮削 |

### 2.2 明确不做

- 刮削（crawler、聚合择优、翻译、R18 导入、演员补全）
- 整理（重命名、移动、硬链接、STRM、模板渲染、回收站、CD 合并）
- 转码/放大（UPSCALE）、分布式节点
- 非媒体功能：应用与系统设置搜索、网页搜索、收藏夹、对话框搜索、资源管理器集成、剪贴板历史、任务栏搜索框（从媒体版构建中裁剪）
- 在线播放以外的媒体服务端能力（Emby/Jellyfin 对接仅保留「跳转到已配置服务器」的链接，不做 API 集成）

### 2.3 成功标准

- 冷启动 100 万文件的卷，索引完成时间与现有 WinShun 持平（媒体过滤带来的额外开销 < 5%）。
- 媒体库内 10 万个作品、30 万个文件：关键词查询 P95 < 50 ms；混合检索 P95 < 150 ms（CPU、无 GPU）。
- 重命名/移动在 2 秒内反映到详情页与缩略图路径。
- NFO 导入 1 万个文件 < 60 秒；番号解析成功率在样本库上 ≥ 95%。
- 向量检索在自建 200 条查询集上 Top-10 命中率 ≥ 80%（相对人工标注）。
- 全部写操作可回滚；第一阶段零破坏性文件操作。

---

## 3. 现状盘点

### 3.1 WinShun：可直接复用的资产

| 能力 | 落点 | 复用方式 | 结论 |
|---|---|---|---|
| 紧凑内存索引 | `src/core/FileIndex.h`（`Entry` 17 字节、`EntryId` uint32、`RecordTable` FRN→EntryId） | 原样保留，新增只读观察接口 | 复用 |
| MFT 全量 + USN 增量 | `src/core/NtfsBulk.*`、`NtfsIndexer.*`、`Crawler.*` | 原样保留 | 复用 |
| 索引编排/快照/恢复 | `src/core/IndexService.h`（Startup/Full/Walked/Volumes、snapshot、follow、recover） | 原样保留，挂载媒体变更回调 | 复用 |
| 目录监视兜底 | `src/core/ChangeWatcher.h`（ReadDirectoryChangesW、overflow 重扫、`kMaxWatches=62`） | 原样保留 | 复用 |
| SMB 共享索引 | `src/core/ShareIndexer.*` | 原样保留 | 复用 |
| 文件名检索/拼音/排序 | `src/core/SearchEngine.*`、`NameSearch.*`、`Pinyin.*`、`Query.*` | 保留，媒体版只开放媒体扩展名与媒体 Facet | 复用 |
| 内容全文索引 | `src/core/ContentIndex.*`、`ContentIndexer.*`、`DocExtractor.*` | 媒体版默认关闭（视频正文提取不属于本期） | 停用 |
| 预览 | `src/app/FilePreview.*`（Picture/Text/Handler/Facts/Message + QuickLook/Seer 外接） | 扩展 `Video` 类型与 NFO 卡片 | 扩展 |
| 缩略图 | `src/app/ThumbnailProvider.*` | 复用系统缩略图，增加磁盘缓存 | 扩展 |
| MCP 服务 | `src/mcp/`（JsonRpc/LineChannel/PipeServer/relay） | 增加媒体工具 | 扩展 |
| 插件宿主 | `src/plugin/`、`src/plugins-sdk/` | 媒体版仅保留 `PlaybackPlugin` 类能力 | 裁剪 |
| 语义检索插件 | `src/semantic-plugin/`（Embedder/Chunker/Store，int8 + mmap + 精确余弦） | 把 `Store` 的向量段思路提升为媒体层内置组件 | 复用+改造 |
| SQLite | 已通过 `Qt6::Sql` 使用（剪贴板历史，WAL） | 媒体库复用同一驱动，需验证 FTS5 | 复用（待验证） |
| ONNX Runtime | 已随 OCR 打包（`third_party/ppocr`、`onnxruntime.dll`） | 可用于本地嵌入模型，免新增重型依赖 | 复用（待验证） |

### 3.2 amane：保留什么、丢掉什么

| amane 模块 | 内容 | 处置 | 理由 |
|---|---|---|---|
| `db/models.py` | MediaFile/Metadata/Resource/Library/Actor/ActorAlias/Director/Tag/Studio/Publisher/Series/UserTag/Comment/FacetRule/Task/Schedule/Feed | **保留核心实体**，Task 收缩为 Job，Schedule/Feed 移除 | 只管理已刮削数据 |
| `search/`（tokenizer/store/embedding/service/indexer） | 分词、FTS5、vec0、hybrid、预热、降级、`wait_idle` | **全部保留为设计蓝本**，C++ 重写 | 检索是本项目核心 |
| `watcher/`（native/clouddrive/everything） | 三通道发现 | **替换**为 WinShun 索引 | 已有更强实现 |
| `crawlers/`、`aggregate/`、`handlers/`（SCRAPE/ORGANIZE/UPSCALE/R18_IMPORT/RESCRAPE） | 刮削与整理 | **移除** | 用户明确不要 |
| `organize/`（模板、move_mode、trash、CD 检测、STRM） | 文件整理 | **移除**；仅保留路径模板的「读取/展示」用途（用于详情页显示期望路径） | 不做写文件 |
| `media/nfo.py` | NFO 写出 | **反向使用**：只读解析，不写 | 入口是已刮削数据 |
| `agent/` | sql_explore/sql_deliver/inspect_result + 7 个 Capability、Saved Query、AG-UI | **保留能力设计**，落到 MCP 工具 + QML 会话页 | 智能体是目标 G4 |
| `api/routes/` | 全套 REST + WebSocket | 仅保留 health/media/metadata/actors/facets/libraries/agent/saved-queries 的语义，作为 QML 的 C++ 侧服务层 | 不做 Web 后端 |
| `web/`（React/Mantine） | Web 界面 | 不作为运行时；仅作交互与信息架构参考 | 避免 WebEngine |
| `plugins/`（FilmSourcePlugin/PlaybackPlugin） | 插件 API v1 | 只保留播放插件概念 | 不引入来源插件 |
| `llm/` | 模型配置、provider | 复用 WinShun `AiSettings`/`McpClients` | 避免重复配置体系 |

### 3.3 两边的结构性冲突（必须在设计里解决）

| 维度 | WinShun | amane | 本方案 |
|---|---|---|---|
| 权威数据源 | 内存树 + 快照文件 | SQLite（WAL）+ Alembic | SQLite 权威；内存树只做发现与 L0 检索 |
| 文件身份 | `EntryId`（重建后变化） | `path` UNIQUE（NFC） | `(volume, frn)` 稳定键，`path` 为投影 |
| 路径编码 | WTF-8 名字存储，路径按需拼装 | Python `str` + NFC 归一 | 入库时 NFC 归一；显示保留原名 |
| 中文分词 | 单字 + bigram（内容索引） | CJK bigram + ASCII 整词 + 番号归一原子 | 以 amane 规则为准，拼音作为附加字段 |
| 向量 | int8 段文件、mmap、精确余弦 | float32 `vec0`（sqlite-vec）+ 独立 `search_vec.db` | 先 int8 段；规模超阈值再评估 sqlite-vec/HNSW |
| 进程/语言 | C++/Qt 单进程 | Python 常驻服务 | C++ 单进程，无 Python 运行时 |
| 平台 | Windows | 跨平台 | Windows（媒体版不承诺跨平台） |
| 许可证 | MIT | GPLv3 | 独立实现，不复制源码 |

---

## 4. 同类工具调研

调研时间 2026-10-24，来源为各项目官方仓库/文档与 PyPI 页面。下表只列与本项目直接相关的机制。

| 工具 | 形态 | 发现/索引 | 向量与模型 | 元数据 | 展现 | 可借鉴点 |
|---|---|---|---|---|---|---|
| Immich | 服务端（Node）+ Postgres | 上传/外部库扫描（周期） | VectorChord（旧 pgvecto.rs）/pgvector + CLIP；ML 服务独立，GPU 后端多 | 照片/人脸/相册 | Web + 移动端 | ML 服务与业务服务解耦；向量库与业务库分离部署 |
| PhotoPrism | Go 单二进制 | 目录扫描 + 索引 | 内嵌 TensorFlow 推理；SQLite/MariaDB | 照片/标签/人脸 | Web | 单二进制内嵌推理，部署零依赖 |
| Stash | Go 单二进制 | 目录扫描 + 监视 | 无向量（主要靠元数据/标签） | 场景/演员/工作室/标签，NFO 导入导出 | Web（GraphQL + React） | 单二进制 + 内嵌 SQLite + GraphQL 单一入口；NFO 作为互操作格式 |
| Whisparr | .NET（*arr 家族） | 目录扫描 | 无 | 场景索引、NFO/图片 | Web | 「库根 + 文件命名」驱动的轻量索引模型 |
| Khoj | FastAPI + Postgres | 文档连接器 | pgvector + bi-encoder | 文档/对话 | Web/桌面/聊天 | 分块 + 嵌入 + 对话式检索的完整闭环 |
| PrivateLens | 本地 CLI/sidecar | 只读扫描 | SQLite + sqlite-vec + FTS5；CLIP + OCR + 人脸 + 元数据多信号融合 | 证据卡（时间、位置、命中原因） | CLI/JSON/NDJSON | 多信号融合与「证据卡」式结果解释；只读 sidecar 定位 |
| media-indexer | 本地服务（FastAPI + Typer） | 增量扫描，跳过未变文件 | SQLite + FAISS + CLIP，稀疏抽帧 | 视频片段 | localhost 网页 | 帧级采样与「未变文件跳过」的增量策略 |
| rclip | CLI | 目录递归、增量索引 | OpenCLIP ViT-B/32 | 图片 | 命令行 | 自然语言 + 反向查询 + 混合算术查询 |
| VidXP | Python 库/CLI/REST/MCP | 视频解析 | Whisper/WhisperX + PySceneDetect + CLIP/SigLIP + BGE/Nomic + FAISS/Chroma | 场景/字幕/时间戳 | API/MCP | 场景切分 + 字幕嵌入 + 时间戳返回；MCP 暴露检索 |
| MoviePilot / Nastools | Docker 服务 | 订阅/整理 | 无 | 刮削/整理 | Web | 「刮削与媒体服务器解耦」的产品分层（本项目正是只要后者） |
| Tdarr / FileFlows | 分布式节点 | 文件流 | 无 | 转码/处理 | Web | 重任务外置节点的思路（本项目不需要） |
| jellyfin-mcp | MCP 服务 | 读媒体服务器库 | pgvector + all-MiniLM-L6-v2，字幕分块 | 场景检索 | MCP | 用 MCP 把媒体检索暴露给通用 AI 客户端 |

### 4.1 三条技术路线

1. **单二进制 + 内嵌 SQLite + 内嵌推理**（Stash、PhotoPrism、Whisparr）。优点：部署简单、无外部依赖、桌面友好；缺点：模型升级与多语言生态受限，索引能力通常只是目录扫描。
2. **服务端 + Postgres/pgvector + 独立 ML 服务**（Immich、Khoj）。优点：规模与并发强、ML 可 GPU 横向扩展；缺点：对单机桌面过重，运维成本高。
3. **本地 sidecar + SQLite/sqlite-vec + 多信号融合**（PrivateLens、media-indexer、rclip、VidXP）。优点：贴近个人媒体库场景、可解释、增量友好；缺点：大多只做「检索」，不做完整的作品/演员/系列数据模型。

### 4.2 与本项目的对照

| 能力 | Immich | Stash | PrivateLens | VidXP | **本项目目标** |
|---|---|---|---|---|---|
| NTFS MFT/USN 级实时索引 | ✗ | ✗ | ✗ | ✗ | ✓（WinShun） |
| 番号/演员/系列/厂牌一等公民 | ✗ | 部分 | ✗ | ✗ | ✓（amane 模型） |
| NFO 互操作 | ✗ | ✓ | ✗ | ✗ | ✓ |
| FTS + 向量混合 | 仅向量 | ✗ | ✓（多信号） | ✓ | ✓（分层融合） |
| 会话式智能体 + 只读 SQL | ✗ | ✗ | ✗ | 部分（MCP） | ✓ |
| 桌面原生、零服务依赖 | ✗ | ✓ | ✓ | ✓ | ✓ |
| 不做刮削/整理 | — | ✓ | ✓ | ✓ | ✓ |

**结论**：技术上没有可直接套用的现成项目；但 `Stash` 的「单二进制 + NFO 互操作」、`PrivateLens` 的「SQLite 内多信号融合 + 证据卡」、`Immich` 的「向量库与业务库分离」、`VidXP` 的「MCP 暴露检索」四点值得直接吸收。

---

## 5. 目标架构

### 5.1 形态选择

| 方案 | 描述 | 优点 | 缺点 | 结论 |
|---|---|---|---|---|
| A 纯 C++ 单进程 | 元数据、检索、智能体全在 WinShun 进程内 | 部署最简、启动最快、无跨进程一致性 | 智能体需要自己实现工具编排；开发量大 | **推荐** |
| B C++ 外壳 + 内嵌 Python（amane 裁剪版） | 子进程跑 FastAPI，WinShun 当客户端 | 复用 amane 现成实现 | PyInstaller 体积、双运行时、GPL 传染风险、IPC 故障面 | 不推荐（仅作过渡验证） |
| C C++ 外壳 + 独立 amane 服务 | 两个进程通过 HTTP/管道通信 | 边界清晰、可单独升级 | 用户需安装两个东西；数据写冲突需仲裁 | 不推荐 |
| D 重写成跨平台服务 | 放弃 Qt 桌面 | 架构干净 | 丢掉 WinShun 最大的资产（MFT/USN + 原生 UI） | 不推荐 |

**推荐 A**，并在 Phase 0 用 B 作为「行为基准」做对比测试（不进入发布物）。

### 5.2 分层架构

```
┌──────────────────────────── WinShunMedia.exe (Qt6/QML) ────────────────────────────┐
│  展现层  MediaShell.qml                                                            │
│    ├─ 搜索栏（番号/关键词/拼音/自然语言，模式切换）                                │
│    ├─ 结果区（列表 / 网格 / 分组，Facet 侧栏）                                     │
│    ├─ 详情页（作品字段、文件列表、演员、标签、相似推荐、NFO 原文）                  │
│    ├─ 会话页（智能体，Saved Query）                                                │
│    └─ 预览浮层（复用 FilePreview，新增 Video/NFO）                                 │
├────────────────────────────────────────────────────────────────────────────────────┤
│  查询层  media::QueryService                                                       │
│    ├─ QueryParser（番号识别 / 模式路由 / 拼音展开）                                │
│    ├─ L0 文件名层 → wscore SearchEngine（现有）                                    │
│    ├─ L1 元数据层 → media::FtsIndex（FTS5 + BM25 权重）                            │
│    ├─ L2 向量层   → media::VectorStore（int8 + mmap + 精确余弦）                    │
│    └─ HybridRanker（候选内 z-score + zgate + word_weight + 精确番号置顶）           │
├────────────────────────────────────────────────────────────────────────────────────┤
│  领域层  media::（新模块）                                                         │
│    MetadataRepo / MediaFileRepo / LibraryRepo / FacetRepo / ActorRepo              │
│    ImportNfo（只读解析）/ MediaFilter（扩展名+路径规则）/ JobQueue                   │
├────────────────────────────────────────────────────────────────────────────────────┤
│  存储层                                                                            │
│    media.db (WAL, SQLite)  ← 权威：作品/文件/资源/分类/用户注解/保存查询            │
│    vectors/seg-*.bin       ← 派生缓存：int8 向量段（可重建）                        │
│    thumbs/…                ← 派生缓存：视频首帧/海报缩略图                          │
├────────────────────────────────────────────────────────────────────────────────────┤
│  发现层（现有 wscore）                                                             │
│    NtfsBulk(MFT) → FileIndex ← NtfsIndexer(USN) ← ChangeWatcher(RDCW)               │
│    ShareIndexer(SMB) · IndexService(编排/快照/恢复)                                 │
│    ↓ MediaChangeFeed（新增观察接口：add/remove/rename，携带 volume+frn）            │
├────────────────────────────────────────────────────────────────────────────────────┤
│  智能体层  McpServer（现有）+ media 工具集                                          │
│    sql_explore / sql_deliver（只读连接 + authorizer）· media_get · media_update_tags │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 模块映射

| 新模块 | 职责 | 基于 |
|---|---|---|
| `src/media/Db.*` | 打开/迁移/事务/只读连接 | Qt6::Sql（SQLite） |
| `src/media/Schema.*` | 版本化 DDL（`meta_schema.version`） | amane `db/models.py` 的实体关系 |
| `src/media/ImportNfo.*` | NFO/`.amane` 导出数据 → 实体 | `src/amane/media/nfo.py` 的字段语义 |
| `src/media/MediaFilter.*` | 视频扩展名、库根、排除规则、CD/字幕识别 | WinShun `IndexService::Options` + amane `Library` |
| `src/media/MediaChangeFeed.*` | 索引变更 → 库变更（FRN 键） | `NtfsIndexer`/`UsnApplier` 事件点 |
| `src/media/Tokenizer.*` | CJK bigram、ASCII 词、番号归一原子 | amane `search/tokenizer.py` 规则 |
| `src/media/FtsIndex.*` | `metadata_fts` 建表/写入/查询/BM25 权重 | amane `search/store.py` |
| `src/media/VectorStore.*` | 段文件、int8、mmap、精确余弦、签名校验 | `src/semantic-plugin/Store.*` |
| `src/media/Embedder.*` | 本地 ONNX 或 OpenAI 兼容 `/embeddings` | `src/semantic-plugin/Embedder.*` + `AiSettings` |
| `src/media/HybridRanker.*` | 融合、门控、置顶、降级 | amane `search/service.py` |
| `src/media/QueryService.*` | 对 QML 的唯一查询入口 | `SearchEngine` 的请求/取消模型 |
| `src/media/MediaCards.*` | 证据卡（命中原因：番号/标题/演员/向量近邻） | PrivateLens 的 explain 思路 |
| `src/app/qml/Media*.qml` | 媒体版界面 | WinShun 现有 QML 设计语言 |
| `src/mcp/MediaTools.*` | MCP 工具 | `src/mcp/McpTools.*` + amane `agent/` |

### 5.4 数据流

**发现流（写路径，热）**

```
USN/MFT/RDCW 事件
  → FileIndex 更新（现有）
  → MediaChangeFeed：
       add    → 扩展名命中 → 立即建 MediaFile 行（status=unmatched），入 JobQueue 待匹配
       remove → MediaFile 软删除（保留历史，不删 Metadata）
       rename → 按 frn 改 path（零解析）
  → JobQueue（后台，单写线程）：
       match   ：番号解析 → 命中已有 Metadata 则挂接；否则建 Metadata(status=imported)
       meta    ：读 NFO（同名 .nfo / 目录内 nfo）→ 补全字段（不覆盖已锁定字段）
       thumb   ：首帧/海报 → thumbs/
       embed   ：标题+剧情+演员+标签 → 向量 → vectors/
```

**检索流（读路径）**

```
输入
 → QueryParser：番号正则？模式开关？拼音？
 → 并行：
     L0 SearchEngine（文件名/路径，限媒体扩展名）
     L1 FTS5（MATCH tokens, bm25(权重)）
     L2 VectorStore（query 向量 → 精确余弦 topK）
 → HybridRanker：
     候选并集 → 各层分数在候选集内 z-score → zgate 过滤 → word_weight 加权
     → 精确番号层整层置顶 → 按作品聚合（Metadata 为结果单位）→ 分页
 → MediaCards：每条结果带「为什么命中」
```

**导入流（冷路径）**

```
选择库根 → 扫描存量（复用现有索引的一次遍历，不额外 walk 磁盘）
 → 批量建 MediaFile → 批量读 NFO → 批量写 Metadata
 → 单事务分段提交（每 1000 条），失败可续跑
```

### 5.5 存储设计

数据库文件 `%LOCALAPPDATA%\WinShun\media\media.db`（WAL，`busy_timeout=30s`，`synchronous=NORMAL`）。

核心表（与 amane 对齐处标注）：

| 表 | 关键列 | 说明 |
|---|---|---|
| `library` | `id, name, root, include_ext, exclude_globs, created_at` | 去掉 move_mode/模板/automation |
| `media_file` | `id, library_id, volume_id, frn, path, path_key, size, mtime, oshash?, duration_ms?, width?, height?, codec?, is_mosaic, has_subtitle, definition, status, metadata_id` | `(volume_id, frn)` 唯一；`path_key` 为 NFC+大小写折叠的索引列 |
| `metadata` | `id, number_key UNIQUE, number, title, plot, release_date, year, runtime_min, content_type, studio_id, publisher_id, series_id, cover_path, nfo_path, field_sources JSON, locked_fields JSON, rating?` | 作品是一等公民；`number_key` 归一 |
| `metadata_actor` | `metadata_id, actor_id, ord` | 多对多 |
| `actor` / `actor_alias` | `id, name, ...` / `alias, actor_id` | 别名命中计入演员分 |
| `studio` / `publisher` / `series` / `tag` | 同 amane | 分类维度 |
| `metadata_tag` / `user_tag` / `metadata_user_tag` / `comment` | 同 amane | 用户注解 |
| `resource` | `id, metadata_id, kind, locator, hint` | 不做 LRU，按引用 |
| `facet_rule` | `kind, pattern, action(alias/block), target` | 分类投影规则 |
| `saved_query` | `id, name, sql, entity, created_at` | 智能体产物 |
| `job` | `id, kind(import/match/meta/thumb/embed), target, state, error, attempts` | 轻量任务表，替代 amane Task/Schedule |
| `metadata_fts` | FTS5(`number_tokens, title_tokens, plot_tokens, actors_tokens, tags_tokens, series_tokens, studio_tokens, publisher_tokens`) | 外部内容表，`tokenize='unicode61'`（中文先由 Tokenizer 切好） |
| `meta_schema` | `version` | 迁移版本 |

派生缓存：

- `vectors/seg-<n>.bin`：段文件，记录 `(metadata_id, int8 向量, scale)`；`signature = model_id|dim|normalize|版本`，签名不符整目录重建（对应 amane 换模型要重建的约束）。
- `thumbs/<xx>/<hash>.jpg`：视频首帧或海报，`hash = (path_key, mtime, size)`。

**身份与路径规则**

1. 写入时 `path` 做 NFC 归一；比较键 `path_key = NFC(casefold(path))`，建唯一索引。
2. 稳定身份用 `(volume_id, frn)`；`volume_id` 取卷序列号 + 卷 GUID，不用盘符。
3. 无法取得 FRN 的来源（SMB、部分虚拟盘）退化为 `path_key` 身份，并在 `media_file` 记 `identity_kind = 'frn' | 'path'`。
4. 重命名只改 `path/path_key`，不重建 Metadata 关联。
5. `Metadata.number_key = NFKC → lower → 仅保留字母数字`（与 amane 一致），保证 `SSIS-001` 与 `ssis001` 同条。

### 5.6 检索栈细节

**分词（照 amane 规则独立实现）**

- CJK 连续串 → bigram；单字查询回退到 LIKE。
- ASCII 连续串 → 整词保留（`MIDE`、`SSIS`）。
- 番号：`normalize_number` 生成原子 token（`ssis001`），同时保留原始形态；查询时若整串是番号，走精确层。
- 字段：NUMBER / TITLE / PLOT / ACTORS / TAGS / SERIES / STUDIO / PUBLISHER，各字段独立列，便于按列加权。
- WinShun 增量：拼音（全拼/首字母）作为 TITLE/ACTORS 的附加 token 列，命中权重低于原词。

**BM25 权重**（沿用 amane 的比例，后续用样本调）

| 字段 | 权重 |
|---|---|
| NUMBER | 8.0 |
| ACTORS | 6.0 |
| TAGS | 6.0 |
| SERIES | 5.0 |
| STUDIO | 5.0 |
| PUBLISHER | 5.0 |
| TITLE | 3.0 |
| PLOT | 1.0 |

**混合融合**

- `KEYWORD_WINDOW = 1000`（关键词召回上限），向量召回 `topK = limit × 4`。
- 两路候选并集内做 z-score，`zgate` 去除低于阈值的向量噪声，`word_weight` 控制文本层相对权重。
- 精确番号层（整串命中 `number_key`）整层置顶，不参与 z-score。
- 任一向量层不可用（模型未就绪/嵌入失败）→ `degraded=true`，只走关键词，UI 明示「语义检索不可用」。
- `total` 语义：融合模式返回去重后的作品数；关键词模式返回 FTS 命中数（与 amane 的对外契约一致，避免 UI 分页错乱）。

**向量存储**

- Phase 2 用 int8 量化 + 每向量 scale（现成 `semantic::Store` 的做法），精确余弦扫描。10 万条 × 1024 维 int8 ≈ 100 MB，单次全扫在桌面 CPU 上为十几毫秒量级。
- 阈值：作品数 > 30 万或 P95 > 80 ms 时，再评估 sqlite-vec（vec0）或内存 HNSW；接口先按 `VectorStore` 抽象，后端可替换。
- 嵌入模型：优先本地 ONNX（与 OCR 共用 `onnxruntime.dll`），模型放 `third_party/` 之外的运行时目录并走许可证清单；备选 OpenAI 兼容 `/embeddings`（复用 `winshun.semantic` 的 `Embedder` 配置）。

**为什么不用 amane 的 vec0 方案**：amane 把向量放在独立 `search_vec.db` 是因为 alembic 反射与扩展加载的工程约束；WinShun 没有 alembic，但同样不想让向量库拖累主库事务。段文件方案与之等价且更省内存（mmap 不占私有工作集），并且已有实现可参考。

### 5.7 媒体分类

- **Facet 维度**：番号前缀（厂牌代码）、厂牌、发行商、系列、演员、年份、标签、内容类型、分辨率档（SD/HD/FHD/4K）、时长档、有无字幕、mosaic/破解、文件相位（正片/中字/花絮）。
- **FacetRule**：`alias`（把 `SSIS` 的多种写法归一）与 `block`（屏蔽噪声标签），规则命中在写入时投影，查询时只读投影结果。
- **聚合**：FTS 命中的作品集合上做 `GROUP BY` 计数，随结果一起返回；Facet 点击转为过滤条件（`facet_id IN (...)`）。
- **与 amane 的差异**：不做「自动分类入库」，只做「已导入数据的投影」。

### 5.8 智能体

- 通道：WinShun 现有 MCP（命名管道 + relay），QML 侧一个会话页消费同一套工具。
- 只读工具：
  - `media_sql_explore`：`SELECT`-only，独立只读连接（`SQLITE_OPEN_READONLY`），`sqlite3_set_authorizer` 白名单（禁 ATTACH/DETACH/PRAGMA 写/`load_extension`），超时与行数上限。
  - `media_sql_deliver`：把上一步结果以表格返回给模型，避免大结果进上下文。
  - `media_inspect`：按 id 取作品/文件的完整卡片。
- 写工具（Capability，二次确认）：`media_set_tags`、`media_set_user_tag`、`media_add_comment`、`media_facet_rule`。全部只改数据库，不碰磁盘。
- Saved Query：`saved_query` 表存 SQL + 实体类型，UI 可「重跑/Live 固定」。
- 上下文：会话消息落盘 `messages.json`（沿用 amane 的「消息是权威上下文」原则），事件流另存 `events.jsonl` 便于回放。
- 安全：智能体永不获得文件系统写权限；所有 SQL 走只读连接；工具返回结果裁剪到前 N 条 + 总数。

### 5.9 查询与展现

**界面（媒体版构建，QML 重写，不引入 WebEngine）**

| 页面 | 内容 | 复用 |
|---|---|---|
| 搜索主页 | 输入框 + 模式开关（关键词/语义/混合）+ 最近查询 | `InputBox.qml`、`Footer.qml` |
| 结果区 | 列表（紧凑）⇄ 网格（封面）⇄ 分组（按演员/系列/厂牌） | `ResultModel` 改造为 `MediaResultModel` |
| Facet 侧栏 | 多维过滤、计数、清除 | 新写，参考 WinShun 现有侧栏样式 |
| 详情页 | 作品字段、文件列表（多版本/字幕）、演员、标签、评论、相似推荐、NFO 原文 | `FileFacts`、`FilePreview` 扩展 |
| 会话页 | 智能体对话、工具调用卡片、Saved Query | `McpClients` |
| 库设置 | 库根、扩展名、NFO 策略、嵌入模型、索引状态 | `SettingsEditor` 新增一页 |

**键位沿用**：`Alt+P` 预览、方向键浏览、`Esc` 关闭、`Ctrl+Enter` 用外部播放器打开、`Ctrl+C` 复制路径/番号。

**性能**：网格必须虚拟化（只创建可视区行对应的 delegate）；缩略图异步加载 + LRU 内存缓存 + 磁盘缓存；滚动时暂停向量相似推荐计算。

### 5.10 预览与播放

- `FilePreview` 增加 `Kind::Video`：
  - 首帧：优先 Windows Shell 缩略图（`ThumbnailProvider`，覆盖 mkv/hevc 视系统解码器而定）；缺失时用 Media Foundation 取首帧；再缺失显示海报（NFO 同目录 `poster/fanart`）。
  - 摘要行：时长、分辨率、编码、字幕轨、文件大小、mtime。
  - 播放：`Ctrl+Enter` 交外部播放器（沿用 WinShun 现有的「打开方式」逻辑），媒体版可配置首选播放器。
- `Kind::Facts` 增强：NFO 字段卡片（番号、发行日、时长、厂牌、系列、演员），字段来源标注（NFO / 文件名 / 用户手改）。
- 不做内置播放器（解码器与字幕渲染成本高，且用户已有播放器）。

---

## 6. 关键设计决策

| 编号 | 决策 | 理由 | 影响 |
|---|---|---|---|
| D1 | SQLite 为权威，内存文件树只做发现与 L0 | 需要事务、唯一键、跨重启稳定身份 | 新增 `src/media/` 模块；`IndexService` 增加观察接口 |
| D2 | `(volume, frn)` 为文件稳定键 | USN/ MFT 都携带 FRN，重命名零解析 | 需在索引构建路径暴露 FRN（`RecordTable` 已有映射） |
| D3 | 不引入 Python 运行时 | 部署体积、启动速度、GPL 边界 | 检索与导入用 C++ 重写 |
| D4 | 向量先 int8 段 + 精确余弦 | 复用现成实现，十万级足够 | 超阈值再换后端 |
| D5 | 智能体走 MCP，不做自有协议 | 复用现有管道与 AI 客户端配置 | 工具需按 MCP schema 注册 |
| D6 | 只读 NFO，不写 NFO | 用户明确「只管理刮削好的数据」 | 导入器只有解析分支 |
| D7 | 媒体版是独立构建目标（`WinShunMedia`），不是隐藏开关 | 裁剪范围大，避免破坏现有启动器 | CMake 增加 target 与编译选项 |
| D8 | 不复制 amane 源码，按文档规则独立实现 | MIT 与 GPLv3 不兼容 | 需要代码评审时的「来源声明」流程 |
| D9 | 不引入 QtWebEngine | 体积与安全面 | 界面用 QML 重写；Web 版仅作为可选只读镜像 |

---

## 7. 分阶段实施

> 阶段之间是依赖关系，不给出时间承诺。每阶段结束必须有可运行产物与验收结果。

### Phase 0：可行性验证（决定 go / no-go）

| 验证项 | 做法 | 通过标准 | 失败时的退路 |
|---|---|---|---|
| P0-1 SQLite 能力 | 用 `Qt6::Sql` 打开临时库，建 FTS5 虚拟表；尝试 `load_extension` | FTS5 可用；扩展加载可用或可不依赖 | 若 FTS5 不可用：改为内置 sqlite3 静态编译（FTS5 直接编译进二进制） |
| P0-2 嵌入延迟与质量 | 本地 ONNX 小模型（中文友好）对 1 万条文本编码；抽样 50 条查询 | 编码 ≥ 200 条/秒（CPU）；Top-10 主观可用 | 退化为标题/标签关键词 + 演员别名 |
| P0-3 FRN 可得性 | 在 MFT 构建与 USN 追更两条路径打印/记录 FRN；重命名实测 | 两条路径都能拿到 FRN；重命名能定位到同一行 | 退化为 `path_key` 身份 + 重命名时按「旧路径→新路径」匹配 |
| P0-4 NFO 保真度 | 取 200 个真实刮削目录，统计番号/演员/标签/封面的解析成功率 | 番号 ≥ 95%，演员 ≥ 90% | 增加 `.amane` 导出 JSON / 目录名回退解析 |
| P0-5 视频首帧 | 对 mkv/mp4/hevc 各 20 个样本取缩略图 | 覆盖率 ≥ 90% | 缺失时用海报；再缺失用类型图标 |
| P0-6 行为基准 | 用 amane 对同一批 NFO 建库，记录检索结果与延迟 | 得到可对比的 golden 结果集 | —— |

产物：`docs/design-media-library-phase0.md`（验证记录）+ 可复现的验证脚本；决定是否继续。

### Phase 1：数据层与发现

范围：
- `src/media/` 骨架：`Db`、`Schema`（v1 DDL + 迁移框架）、`MediaFilter`、`MediaChangeFeed`。
- `NtfsIndexer`/`UsnApplier` 增加变更观察点（add/remove/rename，带 volume+frn），不改变现有索引行为。
- `TrackedFolder`/库根配置：复用 `IndexService::Options.folders`。
- NFO 导入器：只读解析，字段映射，`field_sources`/`locked_fields`。
- 初始批量导入：复用索引一次遍历的结果，不额外 walk。

交付：命令行/隐藏参数可建库、可导入、可打印统计；无 UI。

验收：
- 1 万个 NFO 目录导入成功，字段抽样正确；
- 重命名/移动/删除在 2 秒内反映到 `media_file`；
- 导入中断后可续跑且不重复；
- 数据库写入全部在单写线程，读连接不阻塞。

### Phase 2：检索

范围：
- `Tokenizer`（含番号归一、拼音附加列）、`FtsIndex`（建表/写入/BM25）、`QueryService`（模式路由）。
- `VectorStore`（int8 段、签名、重建）、`Embedder`（本地 ONNX 优先）、`HybridRanker`。
- 结果聚合到作品级；`MediaCards` 生成命中原因。
- 性能护栏：查询超时、召回窗口、降级标记。

验收：
- 10 万作品：关键词 P95 < 50 ms；混合 P95 < 150 ms；
- 番号精确查询首条命中率 100%；
- 向量层不可用时可降级且 UI 可见；
- 与 Phase 0 golden 集对比，Top-10 重合度 ≥ 80%。

### Phase 3：界面裁剪与媒体视图

范围：
- 新构建目标 `WinShunMedia`（CMake option），复用 `wscore`；
- 裁剪：应用/系统设置搜索、网页搜索、收藏夹、对话框搜索、资源管理器集成、剪贴板历史、任务栏框、OCR 相关入口；
- 新增 `MediaShell.qml` 及结果/网格/详情/Facet/库设置页面；
- `FilePreview` 新增 `Video` 与 NFO 卡片；`ThumbnailProvider` 增加磁盘缓存；
- 键位与现有启动器保持一致。

验收：
- 冷启动到可搜索 < 2 s（不含首次建库）；
- 10 万结果网格滚动 60 fps（虚拟化生效）；
- 所有旧功能的入口在媒体版中不可达，且不影响 `WinShun` 原 target 构建。

### Phase 4：智能体与 MCP

范围：
- `src/mcp/MediaTools.*`：`media_sql_explore`、`media_sql_deliver`、`media_inspect`、写 Capability；
- 只读连接 + authorizer 白名单 + 行数/超时限制；
- QML 会话页；Saved Query 存取与重跑；
- 事件日志（`events.jsonl`）与消息权威文件。

验收：
- 模型可完成「找出某演员今年 4K 且无字幕的作品」并给出可点击结果；
- 任何写工具需二次确认，且只改数据库；
- 恶意 SQL（ATTACH、PRAGMA、写语句、`load_extension`）全部被拒。

### Phase 5：可选 Web 镜像与收尾

范围（可选）：
- 只读 HTTP 服务（C++ 或独立小服务）读 `media.db`（WAL 支持多读者），提供浏览与检索；不提供写。
- 性能剖析、安装包、许可证清单（新增模型/库必须登记）、崩溃恢复演练。

验收：
- 安装包体积增量在可接受范围；
- 数据库损坏场景可从备份 + 重建派生缓存恢复；
- 许可证清单完整。

---

## 8. 风险与缓解

| 风险 | 等级 | 说明 | 缓解 |
|---|---|---|---|
| GPLv3 传染 | 高 | 复制 amane 源码会使 WinShun 受 GPLv3 约束 | D8：只依据文档描述的规则独立实现；代码评审记录来源；不引入 amane 运行时 |
| FTS5/sqlite-vec 不可用 | 中 | Qt 自带 SQLite 的编译选项不确定 | P0-1 先验证；失败则静态编译 sqlite3（自带 FTS5） |
| FRN 缺失或复用 | 中 | SMB/虚拟盘无 FRN；NTFS FRN 会被复用 | `identity_kind` 双轨；FRN 复用时以「删除+新增」处理，靠 NFO 兜底匹配 |
| Entry 内存增长 | 中 | 若给每个 Entry 加 FRN 字段，内存 +23% | 不扩 `Entry`；只在媒体变更回调里携带 FRN（`RecordTable` 已够用） |
| 大库网格卡顿 | 中 | 10 万封面 | 虚拟化 + 异步缩略图 + LRU + 占位色块 |
| 视频首帧失败 | 中 | 系统缺解码器 | 海报 → 类型图标降级；文档中明示覆盖率 |
| 中文分词不一致 | 中 | 与 amane 结果有差异 | Phase 0 golden 集对拍；分词规则单测 |
| NFO 字段贫瘠 | 中 | 并非所有刮削器都写全字段 | 目录名/文件名回退解析；支持 `.amane` 导出 JSON；字段来源可见 |
| 「一番多文件」聚合错误 | 中 | 同番号不同版本/字幕 | 作品级聚合 + 文件级展开；相位字段来自文件名规则 |
| 删除/移动误伤 | 高 | 媒体库最怕丢数据 | Phase 1-3 只读磁盘；索引事件只改数据库；不实现「删除文件」 |
| 数据库损坏 | 中 | WAL + 断电 | 定时备份、完整性检查、派生缓存可重建 |
| 用户预期落差 | 低 | 期待自动刮削 | 首次启动明确「仅导入已刮削数据」，并提供 NFO 缺失报告 |

---

## 9. 待决策事项

| 编号 | 决策点 | 选项 | 建议 |
|---|---|---|---|
| Q1 | 许可证策略 | (a) 接受 GPLv3 并开源整库 (b) 独立实现 (c) amane 作为独立进程聚合 | (b) |
| Q2 | 是否保留 Python sidecar | 保留 / 不保留 | 不保留（仅 Phase 0 作基准） |
| Q3 | 媒体版形态 | 独立 target / 运行时开关 | 独立 target，保护现有产品 |
| Q4 | 向量后端 | int8 段 / sqlite-vec / HNSW | 先 int8 段，留接口 |
| Q5 | 嵌入模型 | 本地 ONNX / 远程 OpenAI 兼容 | 本地默认，远程可选 |
| Q6 | 是否做 Web 展现 | 做 / 不做 | Phase 5 可选只读 |
| Q7 | 是否支持导入 amane 现有数据库 | 支持 / 不支持 | 支持一次性导入（用户已有数据） |

---

## 10. 测试与验收

**单元测试（新增到 `tests/`）**

| 测试 | 覆盖 |
|---|---|
| `tst_media_schema` | 迁移幂等、唯一约束、软删除 |
| `tst_media_tokenizer` | CJK bigram、ASCII 整词、番号归一、拼音、单字回退 |
| `tst_media_nfo` | 字段映射、缺失字段、编码（UTF-8/UTF-16）、异常 NFO |
| `tst_media_identity` | FRN 重命名、跨卷移动、FRN 复用、SMB 路径身份 |
| `tst_media_hybrid` | z-score/zgate/word_weight、精确番号置顶、降级 |
| `tst_media_facet` | 投影规则、别名、屏蔽、计数 |
| `tst_media_agent_sql` | 只读连接拒绝写/ATTACH/PRAGMA/扩展 |

**集成与实机**

- 1 万 / 10 万 / 100 万文件三档数据集：索引耗时、内存、查询 P50/P95。
- 变更风暴：批量重命名 1 万文件、删除目录、云盘离线/上线。
- 断电模拟：WAL 恢复 + 派生缓存重建。
- 兼容性：`WinShun` 原 target 构建与测试全绿（媒体改动不得回归）。

**验收出口**：§2.3 全部指标达标 + Phase 4 智能体脚本化用例通过。

---

## 附录 A：amane 文档 → 本方案映射

| amane 文档 | 本方案对应 |
|---|---|
| `docs/dev/data-model.md` | §5.5 表结构与身份规则 |
| `docs/dev/search.md` | §5.6 分词/BM25/融合/降级/写入路径 |
| `docs/dev/watcher.md` | 由 §5.4 发现流取代（三通道不再需要） |
| `docs/dev/agent.md` | §5.8 工具集与 Saved Query |
| `docs/dev/api.md` | §5.9 界面背后的服务语义（不实现 REST） |
| `docs/dev/architecture.md` | 启动编排参考（本项目由 Qt 应用生命周期承担） |
| `docs/analysis-plugin-and-everything.md` | §5.1 形态选择的边界依据 |

## 附录 B：WinShun 裁剪清单

**保留**：`src/core`（FileIndex / NtfsBulk / NtfsIndexer / ChangeWatcher / IndexService / ShareIndexer / SearchEngine / NameSearch / Pinyin / Query / WorkerPool / SettingsMatch 的部分）、`src/app` 的窗口/主题/输入/预览/缩略图/QML 设计系统、`src/mcp`、`src/plugin`（播放相关）、`src/extract`（仅保留 NFO/文本读取所需）。

**媒体版停用**：`AppCatalog`、`SystemCatalog`、`WebShortcut`/`WebSuggest`、`ClipStore`/`Clipboard`、`Favorites`、`History` 的网页/应用历史、`DialogBar`、任务栏搜索框、`Crawler` 的网页抓取、OCR/截图/标注、`ContentIndexer` 默认关闭。

**新增**：`src/media/`（见 §5.3）、`src/app/qml/Media*.qml`、`src/mcp/MediaTools.*`、`tests/tst_media_*`、`docs/design-media-library-phase0.md`。

## 附录 C：参考项目

Immich（`immich-app/immich`）、PhotoPrism（`photoprism/photoprism`）、Stash（`stashapp/stash`）、Whisparr、Khoj（`khoj-ai/khoj`）、PrivateLens、media-indexer、rclip（`ricklan/rclip`）、VidXP、MoviePilot、Nastools、Tdarr、FileFlows、jellyfin-mcp、clawgallery、pixmap-engine。

以上项目的具体机制以调研时的官方文档为准；本方案只吸收其架构思路，不复制代码。

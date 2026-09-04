# Telegram Desktop（tdesktop）源码分析系列

面向 Mike 的 **文档导向** 分析包：基于 GitHub 公开 API / raw 内容整理，**未完整克隆** `telegramdesktop/tdesktop` 仓库树。

## 先读这个：为什么实现得牛

**推荐首读** → [`docs/00-why-it-nails-it.md`](docs/00-why-it-nails-it.md)

用产品约束（超长会话列表、实时 MTProto、跨平台、流体 UI）串起本系列已核实的机制：多 `MainList` 数据面、滚动行缓存与 freeze、Instance/会话线程、TL codegen、自研 `lib_ui`、以及「薄测试 / 厚所有权」的取舍。读完再按章节下钻；文末有 HistoryView / storage eviction 等下一步。

## 上游

| 项 | 值（均来自 `gh api repos/telegramdesktop/tdesktop`，拉取时） |
|---|---|
| 仓库 | [https://github.com/telegramdesktop/tdesktop](https://github.com/telegramdesktop/tdesktop) |
| 官网 | [https://desktop.telegram.org/](https://desktop.telegram.org/) |
| 默认分支 | `dev`（另有 `master`） |
| 创建时间 | `2014-05-02T12:36:31Z` |
| 主语言 | C++ |
| 许可证 | GNU GPL v3.0（`LICENSE`；`LEGAL` 另述 OpenSSL 链接例外） |
| Stars / Forks | `32825` / `7172`（瞬时值，会变） |
| 最新已发布 tag（API `releases`） | `v7.1.5`（`2026-09-02T19:49:05Z`） |

> 说明：`dev` 分支上的 `changelog.txt` 可能领先于 GitHub Releases（例如 changelog 已出现 `7.2.x`，而 Releases 列表当时最新为 `v7.1.5`）。以 Releases/tag 为准核对「已发布」；changelog 作功能叙述补充。

## 推荐阅读顺序

0. **[`docs/00-why-it-nails-it.md`](docs/00-why-it-nails-it.md)** — **导读：为什么实现得牛**（约束 / 啊哈时刻 / 代价 / 下一步）  
1. [`docs/00-overview.md`](docs/00-overview.md) — 项目定位、许可证、技术栈与第三方依赖概览  
2. [`docs/01-history-timeline.md`](docs/01-history-timeline.md) — 可核验的时间线与大版本里程碑  
3. [`docs/02-architecture.md`](docs/02-architecture.md) — 顶层目录、子模块与设计原则  
4. [`docs/03-mtproto-networking.md`](docs/03-mtproto-networking.md) — MTProto、会话/连接、TL scheme 与 codegen  
5. [`docs/04-dialogs-chat-list.md`](docs/04-dialogs-chat-list.md) — 会话列表（Dialogs / Chat List）UI 与数据路径  
6. [`docs/05-dialogs-impl-memory-perf.md`](docs/05-dialogs-impl-memory-perf.md) — Dialogs 深潜：实现、内存、滚动/重绘、相关测试  
7. [`docs/06-memory-testing-jank.md`](docs/06-memory-testing-jank.md) — 横切：内存所有权、测试体系与卡顿治理  
8. [`docs/07-history-structure-entry.md`](docs/07-history-structure-entry.md) — 消息列表结构与 widget 树（打开会话 → 第一帧）  
9. [`docs/08-history-data-pagination.md`](docs/08-history-data-pagination.md) — 消息数据模型、SparseIds 切片与翻页  
10. [`docs/09-history-layout-virtualization.md`](docs/09-history-layout-virtualization.md) — 布局虚拟化与滚动锚点  
11. [`docs/10-history-media-memory.md`](docs/10-history-media-memory.md) — 行内媒体内存、keepAlive、unload  
12. [`docs/11-history-updates-jank.md`](docs/11-history-updates-jank.md) — 增量更新、动画与对照 Dialogs  
13. [`docs/history-SERIES.md`](docs/history-SERIES.md) — 消息列表分册索引  
14. [`SERIES.md`](SERIES.md) — 总目录（已完成表 + 12+ stubs）

## 资料来源与方法

- `gh api repos/telegramdesktop/tdesktop`（元数据）
- `gh api .../contents/`、`.../releases`、`.../tags`、`.../languages`、`.../commits`、`git/trees?recursive=1`
- `https://raw.githubusercontent.com/telegramdesktop/tdesktop/dev/` 下头/实现与 `.github/workflows`
- 子模块 raw：`desktop-app/lib_base`、`lib_crl`、`lib_ui`（所有权 / `crl` / `RpWidget`）
- **未**做全量 `git clone`；路径/模块名来自 Contents API、git tree 与 raw 文件

## 目录结构（本分析包）

```
tdesktop-analysis/
├── README.md                 # 本文件
├── SERIES.md                 # 章节规划（已完成 + stubs）
└── docs/
    ├── 00-why-it-nails-it.md # 导读：为什么实现得牛（推荐首读）
    ├── 00-overview.md
    ├── 01-history-timeline.md
    ├── 02-architecture.md
    ├── 03-mtproto-networking.md
    ├── 04-dialogs-chat-list.md
    ├── 05-dialogs-impl-memory-perf.md
    ├── 06-memory-testing-jank.md
    ├── 07-history-structure-entry.md
    ├── 08-history-data-pagination.md
    ├── 09-history-layout-virtualization.md
    ├── 10-history-media-memory.md
    ├── 11-history-updates-jank.md
    └── history-SERIES.md     # 消息列表分册索引
```

## 语言约定

正文以 **简体中文** 为主；仓库路径、API、命令与标识符保留英文，并以 `` `code` `` 标注。

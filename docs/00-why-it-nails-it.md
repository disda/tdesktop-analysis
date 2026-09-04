# 为什么 tdesktop 的实现牛（导读）

> **推荐首读。** 本文不复述协议教科书，而是带着「产品约束 → 关键机制 → 代价」走一遍：为什么同样做即时通讯桌面端，这套代码读起来会让人觉得 **实现得狠**。每条「啊哈」都锚在本系列已写章节与上游路径上；**没有**编造帧时/内存基准。

读完再按需下钻：[`00-overview`](00-overview.md) → … → [`06-memory-testing-jank`](06-memory-testing-jank.md)。

---

## 产品约束（先把靶子立住）

桌面 Telegram 不是「再做一个聊天窗口」：

| 约束 | 意味着什么 |
|---|---|
| **超长会话列表** | 主列表 + 归档 Folder + 每个 Chat Filter 各一份集合；Forum / Saved Sublists / Community 还要切视图（见 [`04`](04-dialogs-chat-list.md)） |
| **实时 MTProto** | 多 DC、TCP/HTTP/TLS/WebProxy、主线程回调与会话线程收发并存（见 [`03`](03-mtproto-networking.md)） |
| **跨平台单一代码库** | Win / macOS / Linux（另 Snap/Flatpak）；CMake + Qt 5.15/6，Apple 上 ObjC++（见 [`00`](00-overview.md)、[`02`](02-architecture.md)） |
| **流体 UI** | 惯性滚动、置顶重排动画、视频头像/动图预览——列表不能每帧整表重排版（见 [`05`](05-dialogs-impl-memory-perf.md)、[`06`](06-memory-testing-jank.md)） |

在这些约束下，「用 `QListView` + 默认样式 + 同步网络」会很快撞墙。tdesktop 的答案是：**协议自建、列表自绘、库化横切、测试偏手艺**。

---

## 啊哈时刻（4–6）

### 1. 会话列表不是控件，是一套「多宇宙」数据面

**现象**：左侧栏看起来像普通列表；代码里却是 `Dialogs::MainList` ×（主列表 / 归档内列表 / 每个 `FilterId`），行对象是 `Dialogs::Entry` 多态（`History` / `Folder` / `ForumTopic` / …）。

**机制**：`Dialogs::InnerWidget` 用 `_shownList` 按优先级切换数据源（Saved Sublists → Forum → Community → Filter → Folder 主列表），而不是把所有行塞进一个 Qt model。排序键由日期、`PinnedDialogPos`、可选「未读置顶」、`fixedOnTop`（归档等）合成 `uint64`。

**为何牛**：产品上的 Folders / 归档 / Forum 不是 UI 滤镜贴片，而是 **同一套 Entry ↔ MainList 代数** 的不同实例；增量更新走 `Session::applyDialog*` → `entryUpdated(Repaint|Height)` → 行级重绘，而不是「整表 reload」。

- 本系列：[`04-dialogs-chat-list.md`](04-dialogs-chat-list.md)（UI 分层、MainList、ChatFilter、MTProto→模型流）
- 上游：`Telegram/SourceFiles/dialogs/`、`data/data_session.*`、`data/data_chat_filters.*`、`dialogs/dialogs_inner_widget.*`

---

### 2. 滚动流畅靠「视口虚拟化 + 有期限的行位图缓存」

**现象**：几千会话惯性滑动时，文本/角标排版成本极高。

**机制**：`InnerWidget` 自绘（非 `QListView`）；`List::findByY` O(log n) 找可见首行，`paintEvent` 画到 clip 底即停。滚动中启用 `Ui::RowsScrollCache`：**最多 256 行位图、累计约 32 MiB**，停滚 **120 ms** 清空；动画预览与视频头像走 overlay，避免为了一帧动画整行 invalidate。另有 **2 s 列表 freeze**：mousemove 期间按日期重排进 `_pendingAdjust`，超时再冲刷——用短暂顺序过期换手感。

**为何牛**：这是明确的工程交易：**滚动期用内存换 CPU，静止期立刻还回去**；再叠加行级/矩形 `update` 与 `updateChatListEntryPostponed` 合并同帧刷新。

- 本系列：[`05-dialogs-impl-memory-perf.md`](05-dialogs-impl-memory-perf.md) §2–§3；[`06`](06-memory-testing-jank.md) §C.2
- 上游：`dialogs/dialogs_inner_widget.*`、`ui/rows_scroll_cache.*`、`dialogs/dialogs_list.*`

---

### 3. MTProto 按「Instance → 每 DC Session → 会话线程」切开

**现象**：IM 桌面端最容易把 socket、加密、UI 回调缠在一条线程上。

**机制**：`MTP::Instance` 门面（send/cancel/restart、Updates handler）；`Session` 主线程侧 + `SessionPrivate` 跑在专用 `QThread`；`SessionData` 用读写锁交接待发/已发/已收队列。连接抽象 `AbstractConnection::Create` → Tcp/Http/Resolving，socket 再按 secret 选 Tcp / TLS（`0xEE`）/ WebProxy。业务侧 `Sender` / `ConcurrentSender`；DC 用 shift（download/upload/config/cdn…）隔离用途。

**为何牛**：线程边界写在类型与文件名上——**主线程解析回调，会话线程管帧与密钥**——再配 `DcOptions` 的 Address×Protocol 变体与临时密钥槽，而不是「一个全局 socket 单例」。

- 本系列：[`03-mtproto-networking.md`](03-mtproto-networking.md)
- 上游：`Telegram/SourceFiles/mtproto/`（`mtp_instance.*`、`session*`、`connection_*`、`details/*`）、`Telegram/cmake/td_mtproto.cmake`

---

### 4. TL 是代码，不是注释：`api.tl` + codegen → `MTP*`

**现象**：协议一变，手写序列化必炸。

**机制**：`mtproto/scheme/api.tl` + `mtproto.tl` 经 `codegen_scheme.py`（依赖 `lib_tl/tl/generate_tl.py`）生成 `scheme.cpp/h`，目标 `tdesktop::td_scheme`；拉取时 `api.tl` 末尾可见 `LAYER 229`。`SourceFiles/api/*` 消费生成类型（如 `MTPUpdates`），传输层与业务 RPC 分层清晰。

**为何牛**：桌面端与官方 schema **同源 codegen**，LAYER 可对账；库化 `lib_tl` 让基础类型与生成器可复用，而不是每个产品拷一份解析器。

- 本系列：[`03`](03-mtproto-networking.md)「TL 方案 ↔ API ↔ codegen」；[`02`](02-architecture.md) 设计原则 4
- 上游：`Telegram/SourceFiles/mtproto/scheme/`、`codegen/scheme/codegen_scheme.py`、`Telegram/lib_tl`

---

### 5. UI 用 Qt 的「引擎」，不用 Qt 的「皮肤」

**现象**：依赖 Qt 5.15/6，观感却是 Telegram，不是 Fusion。

**机制**：架构原则写明——`lib_ui` 承载自研控件与绘制；列表是 `Ui::RpWidget` + `ElasticScroll` + 自管几何。响应式用 `lib_rpl`（`rpl::producer` / `lifetime`），并发用 `lib_crl`（`crl::async` / `on_main` / `guard`）。媒体解码：`Media::Clip::Workers` 最多 **8** 条 `QThread`，音频 `Loaders` 旁路，像素与 `update` 回主线程。

**为何牛**：跨平台拿到的是 Qt 事件循环与平台后端；**产品形状**（行高、Stories overscroll、置顶位移动画）完全自控。卡顿治理词汇表（虚拟化、缓存、freeze、GifPause、`anim::Disabled`、overlapped early-out）全是「主线程减负」同一哲学的不同旋钮。

- 本系列：[`02-architecture.md`](02-architecture.md)；[`05`](05-dialogs-impl-memory-perf.md)；[`06`](06-memory-testing-jank.md) §A / §C
- 上游：`Telegram/lib_ui`、`lib_rpl`、`lib_crl`；`SourceFiles/dialogs/`、`media/`（Clip/Player）

---

### 6. 所有权与「测什么」同样是设计选择

**现象**：巨型 `Data::Session` 持有 peers/photos/documents/`MainList`/filters…；CI 却几乎不跑逻辑单测。

**机制**：所有权工具箱显式——`unique_ptr` 实体表、`base::unique_qptr` 拥有 QObject、`base::has_weak_ptr`（如 `Dialogs::Entry`）、`rpl::lifetime` 订阅读写、`not_null` 不表达所有权。媒体 `keepAlive(shared_ptr<…Media>)` 防滚动闪烁。测试侧：未见 gtest/Catch2；公开 workflow 是 **编译门禁**；`DESKTOP_APP_TEST_APPS` 启自研 harness / `test_text` / `test_update_verify`；dialogs 仅有 Debug 热键与可选 `waitForChatsLoaded`。

**为何牛**：在「正确性主要靠人与场景」的前提下，他们把 **生命周期写进类型**，把 **卡顿写进缓存与线程边界**——用结构换测试覆盖。这是可争议的 tradeoff，但不是疏忽：缺口在文档里被直接点名（见 05 §7、06 §B）。

- 本系列：[`06-memory-testing-jank.md`](06-memory-testing-jank.md)；[`05`](05-dialogs-impl-memory-perf.md) §7
- 上游：`desktop-app/lib_base`（`unique_qptr`/`weak_ptr`）、`Data::Session`、`SourceFiles/test/`、`.github/workflows`

---

## 代价与取舍（不神化）

| 选择 | 换到的 | 付出去的 |
|---|---|---|
| 自绘列表 + 滚动缓存 | 可控帧成本、可变行高、产品级动效 | 无 Qt item view 生态；缓存/overlay/freeze 状态机要人维护 |
| 自定义 `lib_ui` vs Qt 默认皮肤 | 跨平台一致 Telegram 观感 | 样式/无障碍/平台边角需自担（有 accessibility 文件，但是自研面） |
| TL codegen + 深 MTProto | 与官方 LAYER 对齐、多 DC/代理可演进 | 新人坡度陡；`SessionPrivate` 单文件体量极大 |
| 薄自动化测试 / 厚编译 CI | 迭代不被脆弱单测拖死；更新验签等有专项小程序 | 列表排序、cache 上限、Filter 矩阵等 **无** 库存回归网；回归靠人与 Debug harness |
| 巨型 `Data::Session` 中枢 | 单一真相来源，applyDialog/Updates 路径清晰 | 模块边界靠目录与约定，精确依赖图需后续 CMake/`#include` 分析 |

一句话：**牛在约束下的结构选择，不在「没有技术债」。**

---

## 接下来挖哪里

> 初轮 07–11 / 12–24 已落地；**gap-fill 25–29** 补了双宿主、发送路径、多账号、搜索、Difference 深潜。下列为仍可继续挖的点。

1. **`HistoryView` 双宿主与发送** — 总图见 [`25`](25-history-dual-hosts.md) / [`26`](26-compose-send-path.md)；与 Dialogs `RowsScrollCache` 对照仍见 [`09`](09-history-layout-virtualization.md)、[`11`](11-history-updates-jank.md)。
2. **存储驱逐** — 见 [`16`](16-storage-cache.md)；`_heavyViewParts` 与 UI `keepAlive` 边界可再量化。
3. **Updates / Difference** — 总述 [`14`](14-api-updates.md)；差量/TooLong/range 见 [`29`](29-updates-difference-deep.md)；`feedUpdate` 全分支仍可逐 TL 展开。
4. **媒体全管线** — 见 [`18`](18-media-pipeline.md)；相册大图卸载与 Clip 三缓冲是否同哲学可再对照。

---

## 怎么用本系列

| 顺序 | 文档 | 用途 |
|---:|---|---|
| ★ | **本文** | 为什么牛：约束、啊哈、代价、下一步 |
| 1 | [`00-overview`](00-overview.md) | 许可证、栈、平台 |
| 2 | [`01-history-timeline`](01-history-timeline.md) | 可核验里程碑 |
| 3 | [`02-architecture`](02-architecture.md) | 目录与设计原则 |
| 4 | [`03-mtproto-networking`](03-mtproto-networking.md) | 协议与线程 |
| 5–6 | [`04`](04-dialogs-chat-list.md) → [`05`](05-dialogs-impl-memory-perf.md) | 列表产品面 → 实现/缓存 |
| 7 | [`06-memory-testing-jank`](06-memory-testing-jank.md) | 所有权、测试、jank 词汇表 |

（完）

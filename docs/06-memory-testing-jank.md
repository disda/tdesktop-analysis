# 06 · 横切：内存所有权、测试体系与卡顿治理

> 仓库级模式扫描（`dev` Contents/git tree + raw 头/实现；子模块 `desktop-app/lib_base`、`lib_crl`、`lib_ui`）。与 dialogs 交叉处见 [`05-dialogs-impl-memory-perf.md`](05-dialogs-impl-memory-perf.md)。推测已标。

## A. 内存与对象生命周期

### A.1 所有权工具箱（出现频率高）

| 工具 | 来源 | 典型用途 |
|---|---|---|
| `std::unique_ptr<T>` | 标准库 | Session 子系统、`PeerData`/`PhotoData`/`DocumentData`、列表 `Row`、媒体 loader |
| `std::shared_ptr` / `std::weak_ptr` | 标准库 | `PhotoMedia`/`DocumentMedia` 的 `keepAlive`；`GroupCall` 会议共享（`sharedConferenceCall` + `_conferenceCalls: flat_map<CallId, weak_ptr>`） |
| `base::unique_qptr<T>` | `lib_base` / `unique_qptr.h` | **拥有** `QObject`：内部 `QPointer` + `destroy()` `delete`；父销毁时指针自动变空，避免悬空 |
| `QPointer<T>` | Qt | 非拥有观察；test harness 强制用于跨 stage 控件 |
| `base::weak_ptr` / `base::has_weak_ptr` | `lib_base` / `weak_ptr.h` | 非 Qt 对象的弱引用；`Dialogs::Entry`、部分 `HistoryView::Element` 登记在 `_formattedDateUpdates` |
| `not_null<T*>` | GSL / 项目约定 | API 表面「永不为空」；**不**表达所有权 |
| `object_ptr<T>` / `Ui::CreateChild` | `lib_ui` | Qt 父子树所有权（parent `QObject` 删子） |
| `rpl::lifetime` | `lib_rpl` | 订阅 RAII；成员析构即断开；场景测试强调泄漏 lifetime 会在 `~Main::Session` 时 SIGSEGV |

### A.2 `Data::Session`：数据中枢的持有图（头文件可见）

**值嵌入**

- `Dialogs::MainList _chatsList`
- `Dialogs::IndexedList _contactsList` / `_contactsNoChatsList`

**`unique_ptr` 子系统（示例）**

`ChatFilters`、`Histories`、`Stickers`、`Reactions`、`Stories`、`SavedMessages`、`Streaming`、`CustomEmojiManager`、`NotifySettings`、… — 构造期创建，随 Session 销毁。

**实体表**

- `_peers: unordered_map<PeerId, unique_ptr<PeerData>>`
- `_photos` / `_documents` / `_webpages` / `_polls` / `_games` / `_botApps` / `_locations(CloudImage)` 等：`unique_ptr` 值
- `_folders: flat_map<FolderId, unique_ptr<Folder>>`
- `_messages` 及大量 `flat_map/set<not_null<HistoryItem*|ViewElement*>>` 反向索引（webpage/poll/game/contact/story 视图）

**缓存与保活**

- `Storage::Cache::Database &cache()` / `cacheBigFile()` — 落盘二进制缓存（实现在 `lib_storage`）
- `keepAlive(shared_ptr<PhotoMedia|DocumentMedia>)` — 短暂延长媒体解码/位图生命周期，避免滚动时闪烁
- `_photosScheduledForCacheClear` — 延迟清图片缓存调度
- `_heavyViewParts: flat_set<not_null<ViewElement*>>` — 「重」消息部件登记（**推测**用于卸载不可见 heavy content）

**定时清理**

- `_ttlCheckTimer` / `_mediaDestroyCheckTimer` / `_pollsClosingTimer` / `_watchForOfflineTimer` / `_formattedDateTimer`

**销毁入口**

- `Session::clear()`（公开）；成员 `rpl::lifetime _lifetime` 统一收订阅。
- `notifyHistoryUnloaded` / `historyUnloaded` producer — UI 释放与 History 相关的视图。

### A.3 UI / dialogs 侧缓存（与 05 呼应）

- `Ui::RowsScrollCache`：滚动期行 `QImage`，256 条或 32MiB，停滚 120ms 清空。
- `InnerWidget::_cachedRows`：与上配对的 overlay 元数据。
- `_chatsFilterTags: unordered_map<ChatsFilterTagsKey, TagCache>` — filter 彩色标签位图。
- `_videoUserpics: flat_map<PeerData*, unique_ptr<VideoUserpic>>`
- `_rightButtons: unordered_map<PeerId, RightButton>`（含复用的 `RippleAnimation`）
- `RightButton` 注释：ripple 按 peer 创建、跨 filter 复用，callback 每次 press 刷新。

### A.4 `crl` / `rpl` 与生命周期

- `crl::async`（`lib_crl`，平台分发 winapi/dispatch/qt/tmc）：把工作丢到后台队列。
- `crl::on_main` / `crl::guard(object, fn)`：回主线程且对象仍存活才执行（clip `Manager::callback` 用 `crl::on_main` + `Reader::SafeCallback`）。
- `Ui::PostponeCall` / `InvokeQueued`：合并到事件循环；`Entry::updateChatListEntryPostponed`、拖拽后 `performDrag` 等。
- `rpl::producer` + 成员 `lifetime()` / 局部 `rpl::lifetime`（如 `_openedForumLifetime`）：离开 forum 即拆订阅。

### A.5 图像 / 像素池（可见边界）

| 区域 | 观察 |
|---|---|
| 行滚动位图 | `RowsScrollCache`（上） |
| 剪辑帧环 | `Media::Clip::Reader` 内 `_frames[3]` + `QAtomicInt _step` 三缓冲；worker 写、UI 读 |
| 音频 | `Media::Player::Loaders` 在专用 `QThread`；`unique_ptr<AudioPlayerLoader>` ×3（audio/song/video） |
| 磁盘 | `Storage::Cache::Database`（细节属 `lib_storage`，本文未展开 API） |
| Emoji | test 集成里 `emojiCacheFolder()`；产品路径在 `Ui::Emoji`（**未**深读） |

> **推测**：通用 `Image`/`ImageSource` 与 `data/data_photo_media` 另有引用计数卸载策略；需专章跟 `unload`/`automaticLoad`。

---

## B. 测试体系：实际测什么

### B.1 目录地图（git tree 已枚举）

```
Telegram/SourceFiles/
├── tests/                    # 独立可执行小程序（非 gtest）
│   ├── test_main.{h,cpp}     # 迷你 QApplication + Integration
│   ├── test_text.cpp         # → 目标 test_text（DESKTOP_APP_TEST_APPS）
│   └── test_update_verify.cpp
├── test/                     # 进程内 task-test harness（Debug / TEST_APPS）
│   ├── README.md             # 长篇契约（Stage/Runner/Capture）
│   ├── test_runner.* / test_scenario.cpp / test_agent.*
│   ├── test_widgets / capture / history_fixtures / messages / …
│   └── …
dialogs/dialogs_top_bar_suggestion_test.cpp  # DEBUG 热键，不是单测
```

**未发现**：GoogleTest、Catch2、`*_tests.cpp` 惯例、`ctest` 集成。自研 `Check()` 宏（`test_update_verify`）与 harness `Test::Check`。

### B.2 构建门闩

| CMake 开关 | 作用 |
|---|---|
| `DESKTOP_APP_TEST_APPS` | `include(cmake/tests.cmake)` → 建 `test_text`；主目标编入 `SourceFiles/test/*` |
| `DESKTOP_APP_TEST_APPS` **或** `DESKTOP_APP_SPECIAL_TARGET` | `include(cmake/test_update_verify.cmake)` → 可执行文件 `test_update_verify` |
| CI `TDESKTOP_API_TEST=ON` | **仅**测试环境 API id/hash，**不是**跑单元测试 |

`Telegram/cmake/tests.cmake`：`test_text` 链 `lib_base`/`lib_crl`/`lib_ui` + emoji qrc，依赖挂在 `Telegram` 上但 **workflow 未见执行该二进制**。

### B.3 CI（`.github/workflows`）

已列工作流：`linux.yml` / `mac.yml` / `win.yml` / `snap.yml` / 打包与 bot 类（`canary.yml`、`winget.yml`、`stale.yml`…）。

- 主路径：checkout + 依赖 + **编译**（Docker/ccache 等）。
- 对 `linux.yml`/`mac.yml`/`win.yml` 文本检索：**无** `ctest`、`test_update_verify`、`test_text` 运行步骤。
- 结论：**CI = 编译门禁**；逻辑回归依赖本地/`DESKTOP_APP_TEST_APPS` 的 agent harness 与人工。

### B.4 测得到 vs 测不到（对照表）

| 测得到（有专用代码） | 基本未覆盖（源码侧未见） |
|---|---|
| v2 更新包验签（`test_update_verify`，大量 Check） | `Dialogs::List` 排序/freeze、`MainList` 未读聚合 |
| 文本/UI 小沙箱（`test_text`） | `RowsScrollCache` 正确性/内存上限 |
| 进程内场景：点击、抓图、History 注入、MTP 500 重试探针、窗口激活 | MTProto 状态机 / PTS 全分支 |
| chats 加载门闩 `waitForChatsLoaded` | Filter 规则 `ChatFilter::contains` 矩阵 |
| harness 自测（hover/activation/layer root/…） | 性能/帧时自动化 |

### B.5 Dialogs 相关测试备注

- `dialogs_top_bar_suggestion_test.cpp`：仅 Debug 热键验证顶栏 suggestion 挂载。
- harness README 明确：`waitForChatsLoaded` 非每个场景默认；列表 UI 正确性需场景自行 `PublishLiveWidget` / 抓图，**无**库存 dialogs 场景文件（`test_scenario.cpp` 仓库版为 no-op）。

---

## C. 卡顿 / 卡顿治理（Jank）

### C.1 线程模型（可见）

```
主线程 (Qt GUI + 大部分 Data/UI)
   ↑ crl::on_main / QueuedConnection / InvokeQueued
   │
Worker 池示例：
   • Media::Clip::Workers  — 最多 kClipThreadsCount = 8 条 QThread
       Manager::moveToThread；解码后 callback → crl::on_main
   • Media::Player::Loaders — 构造注入 QThread*；FFmpeg loader 在旁路线程
   • crl::async 通用队列（lib_crl 平台后端）
   • MTP/网络（见 03 章；此处不展开）
```

原则：**解码/IO 离开 GUI 线程；像素提交与 widget `update` 回主线程**。

### C.2 UI 线程上的减负手法

| 手法 | 位置 | 作用 |
|---|---|---|
| 视口虚拟化绘制 | `InnerWidget::paintEvent` + `findByY` | 只画 clip 内行 |
| 滚动行缓存 | `RowsScrollCache` | 惯性滚动少排版 |
| 列表 freeze | `List::freeze` + 2s timer | 合并 date 重排 |
| 矩形/行级 `update` | `updateDialogRow`、pinned 动画、entry refresh | 避免全 widget 重绘 |
| `PostponeCall` 合并 Repaint | `Entry::updateChatListEntryPostponed` | 同一事件循环多改合一帧 |
| `contentOverlapped` early-out | `paintEvent` 开头 | 被挡住不画 |
| `GifPauseReason` | `isGifPausedAtLeastFor` | 非活动窗口等停 GIF/视频头像 |
| `anim::Disabled()` | 置顶动画等 | 减动效时跳过插值 |
| DnD 时动画改 InvokeQueued | `SetScheduleWithInvokeQueued` | 避免与拖拽同步冲突 |
| 预加载与分页 | `PreloadHeightsCount=3`、`_loadMoreCallback` | 减少滚到底卡顿（换网络/解码峰值） |

### C.3 媒体离主线程（已见证据）

- **GIF/短视频**：`Media::Clip::Reader` + `Manager` 在最多 8 条 `QThread`；`QAtomicInt` 帧步与 `displayed`；UI `frameInfo`/`current` 取帧。
- **音频**：`Loaders(QThread*)` + `ChildFFMpegLoader`；`QMutex` 保护外部 packet 队列。
- **回主线程通知**：`Manager::callback` → `crl::on_main` → `SafeCallback`（Reader 可能已销毁）。

> **推测**：大图/相册 `PhotoMedia` 解码路径类似（cache + 后台），需跟 `data_photo`/`ui/image`；未在本次逐行证实。

### C.4 定时器与「预算」

- `base::Timer`：freeze、scroll-cache stop、Session TTL/media destroy、offline watch 等 — **单次/周期任务**，非渲染时钟。
- `Ui::Animations::Basic` / `Manager`：置顶位移等；可与 `InvokeQueued` 调度策略联动。
- 动画时长常量来自 style（如 `st::stickersRowDuration`），非写死帧率预算；**未见**显式「每帧最多 N ms」调度器（若存在可能在 `lib_ui` animations，**未**深读）。

### C.5 与「感觉卡」相关的产品开关（代码名）

- `anim::Disabled()` — 系统/设置减动效。
- `Window::GifPauseReason` — 按原因暂停动图。
- `Painter::setInactive` — 绘制走 inactive 外观（常与 pause 同时）。
- dialogs「未读置顶」等选项影响排序频率（见 04），间接影响重排成本。

---

## D. 跨章关系

| 章 | 关系 |
|---|---|
| 04 | 列表产品/数据路径概览 |
| 05 | Dialogs 实现级绘制/缓存/freeze |
| 06（本文） | 仓库级内存 / 测试 / 线程与 jank 词汇表 |
| 03 | 网络线程与 Updates（互补） |
| stub「数据与存储」「媒体管线」 | `lib_storage` 细 API、FFmpeg 全管线 |

## E. 后续缺口

1. `lib_storage` Cache 驱逐策略与内存上限数字  
2. `HistoryView::ListWidget` 是否复用 `RowsScrollCache` 或独立虚拟化  
3. CI 是否在非公开 runner 跑 `test_update_verify`（公开 yml 未见）  
4. `crl::object_on_thread` 在 MTProto/其他模块的使用点全表  
5. Instruments 级帧时间与主线程阻塞火焰图（需运行时）


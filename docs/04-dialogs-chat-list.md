# 04 · 会话列表（Dialogs / Chat List）

> 基于 `dev` 分支 Contents API 与若干头/实现文件前部的 raw 拉取（`dialogs/`、`data/data_session.*`、`data/data_chat_filters.*`、`data/data_folder.h`、`data/data_thread.h`、`history/history.h`、`window/window_*`、`apiwrap.h`、`api/api_updates.h`、`mainwidget.h`）。**未**克隆完整树；下列为可核对的类/路径关系与高层面数据流，非穷尽运行时语义。

左侧栏「会话列表」在 UI 上对应 `Dialogs::Widget` + `Dialogs::InnerWidget`；数据上对应 `Data::Session` 持有的 `Dialogs::MainList`（主列表 / 归档 Folder / 各 Chat Filter），行对象为 `Dialogs::Entry` 派生类。

## 代码落点（Where）

| 路径 | 角色（文件名 / 头文件可见） |
|---|---|
| `Telegram/SourceFiles/dialogs/` | 会话列表 UI + 行/索引/置顶数据结构 |
| `…/dialogs/ui/` | 行绘制、消息预览、Stories 条、搜索空态、Suggestions 等 |
| `Telegram/SourceFiles/data/data_session.*` | 会话数据中枢：`applyDialog(s)`、`chatsList`、置顶、未读徽章 |
| `…/data/data_chat_filters.*` | Chat Folders / filters：`ChatFilter`、`ChatFilters` |
| `…/data/data_folder.*` | 归档 Folder（`Folder::kId = 1`）及其内部 `MainList` |
| `…/data/data_thread.*` | `Data::Thread : Dialogs::Entry`（History / Topic / Sublist 共用） |
| `…/history/history.*` | `History : Data::Thread`，实现 `applyDialog` / `chatList*` |
| `…/window/window_session_controller.*` | 活动会话、打开 Folder/Forum、`activeChatsFilter` |
| `…/window/window_filters_menu.*` | 侧栏 Filters 菜单 UI |
| `…/api/api_chat_filters.*`、`api_updates.*` | Filter 相关 API；Updates 入口 |
| `apiwrap.h` | `requestDialogs` / `requestPinnedDialogs` / 分页加载 |
| `mainwidget.h` | 主布局持有 `Dialogs::Widget _dialogs` + `HistoryWidget` |

命名空间：列表 UI/结构在 `namespace Dialogs`；业务数据在 `namespace Data`；窗口编排在 `namespace Window`。

## UI 分层：从主窗口到行

```
MainWidget
  └─ Dialogs::Widget          (Window::AbstractSectionWidget)
        ├─ 搜索框 / Stories / Suggestions / 连接状态等 chrome
        └─ Dialogs::InnerWidget   (Ui::RpWidget，可滚动行列表)
              └─ Dialogs::Row → Dialogs::Key → Dialogs::Entry
```

### `Dialogs::Widget`（`dialogs_widget.h`）

- 布局模式：`Layout::{Main, Child}`（主栏 vs 子列表，如 Forum 等）。
- 公开能力：`showForum`、`searchMessages(SearchState)`、`jumpToTop`、`toggleFiltersMenu`、`resolveChatNext/Previous`。
- 搜索：持有 `Ui::InputField`；内部 `SearchProcessState`（请求缓存、分页 `nextRate`、`mtpRequestId`）；对接 `Api::PeerSearch`、`Api::SingleMessageSearch`；可请求 public posts。
- 其它可见 setup：`setupStories`、`setupTopBarSuggestions`、`setupMoreChatsBar`、`setupDownloadBar`、`setupConnectingWidget`、`setupMainMenuToggle`。

### `Dialogs::InnerWidget`（`dialogs_inner_widget.h`）

- **状态机**：`WidgetState::{Default, Filtered}`（普通列表 vs 本地/全局过滤结果）。
- **当前列表指针**：`not_null<IndexedList*> _shownList`；`FilterId _filterId`。
- `refreshShownList()`（实现可见）按优先级切换 `_shownList` 来源：
  1. Saved Sublists 列表  
  2. Forum topics 列表  
  3. Community chats 列表  
  4. 非零 `_filterId` → `chatsFilters().chatsList(filterId)->indexed()`  
  5. 否则 → `data().chatsList(_openedFolder)->indexed()`  
- `switchToFilter(FilterId)`：校验 filter 仍在 `ChatFilters::list()`；保存/恢复各 filter 滚动位置；对带 `Flag::NoRead` 的 filter **跳过**滚动恢复（注释：这类列表过「灵活」）。
- 导航上下文：`changeOpenedFolder` / `changeOpenedForum` / `changeOpenedCommunity` / `showSavedSublists`。
- 选择/打开：`chosenRow()` → `ChosenRow{ Key, MessagePosition, … }`；可预览 `scheduleChatPreview`。
- 搜索相关 producer：`searchRequests`、`changeSearchTabRequests`、`changeSearchFilterRequests`（`ChatTypeFilter::{All,Private,Groups,Channels}`）、`cancelSearchRequests` 等。

### 行：`Key` / `Row` / `FakeRow`

| 类型 | 文件 | 要点 |
|---|---|---|
| `Dialogs::Key` | `dialogs_key.h` | 薄包装，指向 `Entry*`；可构造自 `History` / `Folder` / `Thread` / `ForumTopic` / `SavedSublist` |
| `Dialogs::Row` | `dialogs_row.h` | 列表中的可视行：几何 `top/height`、角标（Stories 等）、ripple；`sortKey(filterId)` |
| `Dialogs::FakeRow` | 同上 | 搜索命中消息用的「伪行」（绑 `HistoryItem` + `MessageView`） |
| `RowDescriptor` | `dialogs_key.h` | `{ Key, FullMsgId }`，活动会话 / 跳转用 |

绘制辅助在 `dialogs/ui/`：`dialogs_layout.*`、`dialogs_message_view.*`、`dialogs_topics_view.*`、`dialogs_stories_list.*`、`dialogs_suggestions.*`、`chat_search_*` 等。

## 数据模型：Entry → MainList → Session

### 继承链（头文件）

```
Dialogs::Entry
  ├─ Data::Folder          （归档入口行 + 内部 chatsList）
  └─ Data::Thread
        └─ History         （普通 peer 会话；另有 ForumTopic / SavedSublist 等 Thread 子类）
```

`Entry::Type`：`History` / `Folder` / `ForumTopic` / `SavedSublist`。

### `Dialogs::MainList`（`dialogs_main_list.h`）

每个「可见会话集合」一份：

- `IndexedList _all`：按字母索引的完整列表（`SortMode` + 按日期排序版本号）。
- `PinnedList _pinned`：置顶顺序（可 `applyList` 自 `MTPDialogPeer` / Forum topic id 等）。
- 未读：`UnreadState` 本地聚合 + `updateCloudUnread(MTPDdialogFolder)` 云端补全；`unreadStateChanges()`。
- `FilterId _filterId`（主列表为 0；各 Chat Filter 各有一份）。

`IndexedList` 包一层 `List`（`dialogs_list.h`）：`vector<Row*>` + `map<Key, unique_ptr<Row>>`；支持 `adjustByDate`、`moveToTop`、`freeze/unfreeze`（拖拽重排时）。

### `Data::Session` 中的列表槽位（`data_session.h`）

- `_chatsList`：主会话列表 `Dialogs::MainList`。
- `_folders` → `Data::Folder`；`Folder` 内再建 `_chatsList`（归档内会话）。
- `_chatsFilters` → `Data::ChatFilters`；内部 `_chatsLists: flat_map<FilterId, unique_ptr<MainList>>`。
- API：`chatsList(Folder*)`、`chatsListFor(...)`、`chatsFilters().chatsList(FilterId)`。
- 应用批量对话框：`applyDialogs(Folder*, messages, dialogs, count?)` → 逐条 `applyDialog`。
- 未读徽章：`unreadBadge()` / `unreadBadgeChanges()` 等（窗口/托盘消费）。

### `Data::Folder`（归档）

- 固定 `kId = 1`；自身也是 `Dialogs::Entry`，可出现在主列表顶部（`fixedOnTopIndex` / `kArchiveFixOnTopIndex`）。
- `applyDialog(MTPDdialogFolder)`、`applyPinnedUpdate(MTPDupdateDialogPinned)`。
- 维护 `lastHistories()` 供列表行预览文案缓存。

## 排序、置顶与过滤

### 排序键（`dialogs_entry.cpp` 可见逻辑）

- 基础：`DialogPosFromDate(adjustedChatListTimeId())` → `_sortKeyByDate`。
- 选项 `dialogsUnreadOnTop`（`kOptionDialogsUnreadOnTop`）：未读未静音会话可用 `UnreadOnTopDialogPos(...)` 抬升。
- **置顶**：`lookupPinnedIndex(filterId) != 0` → `PinnedDialogPos(index)` 覆盖日期键（`computeSortPosition`）。
- **固定置顶类**：`fixedOnTopIndex()`（如归档 `kArchiveFixOnTopIndex = 1`、推广位 `kTopPromotionFixOnTopIndex = 2`）→ `FixedOnTopDialogPos`。
- `SortMode`（`dialogs_common.h`）：`Date` / `Name` / `Add`——`IndexedList` 在 Name 模式下维护字母分桶。

### 置顶列表（`PinnedList`）

- 按 `FilterId` + limit 管理；`setPinned` / `addPinned` / `reorder`。
- 云端：`applyList(owner, QVector<MTPDialogPeer>)`；Forum / SavedMessages 有重载。
- `Data::Session`：`setChatPinned`、`pinnedChatsOrder(Folder*|FilterId|Forum*|…)`、`pinnedChatsLimit(...)`、`pinnedDialogsOrderUpdated()`。
- 加载：`ApiWrap::requestPinnedDialogs` / `reloadPinnedDialogs`。

### Chat Folders / Filters（自 v2.0 产品线；代码名 `ChatFilter`）

产品 changelog（见 `docs/01-history-timeline.md`）：**v2.0.0** 引入 Chat Folders。当前代码：

**`Data::ChatFilter`**（`data_chat_filters.h`）

- 规则 flags：`Contacts` / `NonContacts` / `Groups` / `Channels` / `Bots` / `NoMuted` / `NoRead` / `NoArchived`；以及 `Chatlist` / `HasMyLinks` / `StaticTitle`；Business 例外 `NewChats` / `ExistingChats`。
- 成员集合：`always` / `never`（`History*` 集合）+ `pinned` 向量。
- TL：`FromTL(MTPDialogFilter)` / `tl()`；`contains(History*)` 判定会话是否落入该 filter。

**`Data::ChatFilters`**

- `load` / `reload` / `setPreloaded`；`list()` + `changed()`。
- `apply(MTPUpdate)` 匹配（实现可见）：
  - `MTPDupdateDialogFilter` → `set` 或 `remove`
  - `MTPDupdateDialogFilters` → `load(true)`
  - `MTPDupdateDialogFilterOrder` → `applyOrder`，失败则 reload
- 每 filter 一份 `Dialogs::MainList`：`chatsList(FilterId)`；`refreshHistory(History*)` 在会话属性变化时重算归属。
- 可分享 chatlist：links / `moreChats` / tags（`tagsEnabled`、folder 彩色标签）。

**窗口侧**

- `Window::FiltersMenu`：侧栏按钮列表、拖拽重排、Favorite、打开 filters 设置。
- `SessionController`：`activeChatsFilter()` / `setActiveChatsFilter`；`toggleFiltersMenu`；`openFolder` / `showForum`。

> **推测（未跟完整订阅图）**：`InnerWidget` 在 controller 的 `activeChatsFilter` 变化时调用 `switchToFilter`；精确 rpl 订阅点需再读 `dialogs_inner_widget.cpp` 构造函数后半。

## 未读徽章

- 聚合结构：`Dialogs::UnreadState`（messages/chats/marks/reactions/mentions/polls 及对应 muted 分量 + `known`）。
- 行级：`Entry::chatListUnreadState()` / `chatListBadgesState()`（纯虚，由 `History` / `Folder` / Topic 等实现）。
- 展示：`BadgesForUnread(UnreadState, CountInBadge, IncludeInBadge)` → `BadgesState`（counter、mention、reaction、poll 与 muted 标记）；受 `Core::App().settings().countUnreadMessages()` / `includeMutedCounter()` 影响。
- 列表级：`MainList::unreadState()`；全局：`Data::Session::unreadBadge*`；另有 `data_unread_value.*`。
- Folder/Filter 计数行为在 changelog 中持续演进（如「归档内 topics 计为一个 chat」「文件夹计数是否含静音」等）——细节以设置项 + 上述结构为准，本文不展开每条规则。

## 搜索入口（列表层）

| 入口 | 位置 | 说明 |
|---|---|---|
| 顶栏搜索框 | `Dialogs::Widget` `_search` | 驱动 `search()` / peer search / messages / posts |
| `SearchState` | `dialogs_key.h` | `inChat`、`fromPeer`、`tags`、`tab`、`ChatTypeFilter`、`fromArchive`、`query` |
| 本地过滤 | `InnerWidget` `_filterResults` | `WidgetState::Filtered`；`appendToFiltered` |
| 消息命中行 | `FakeRow` + `_searchResults` | 非会话行，而是消息预览行 |
| Hashtag | `onHashtagFilterUpdate` | 补全/过滤 |
| Tags | `SearchTags`、`searchTagsChanges` | 按 reaction tag 搜 |
| Peer 搜索 | `Api::PeerSearchResult` → `peerSearchReceived` | |
| 会话内搜索跳转 | `SessionController::searchInChat` | 从别处打开「在此聊天中搜」 |

## MTProto → 模型 → 列表刷新（高层面）

```
MTP updates / dialogs RPC
        │
        ▼
Api::Updates::applyUpdates / feedUpdate*
   或 ApiWrap::requestDialogs → messages.getDialogs* 结果
        │
        ▼
Data::Session::applyDialogs / applyDialog
        │
        ├─ MTPDdialog        → History::applyDialog → Entry 链入 MainList
        ├─ MTPDdialogFolder  → Folder::applyDialog
        └─ MTPDdialogCommunity → community 行 / 置顶延迟加载
        │
        ▼
Entry::updateChatListSortPosition / addToChatList / removeFromChatList
        │
        ▼
session().changes().entryUpdated(..., Repaint|Height|…)
        │
        ▼
Dialogs::InnerWidget 订阅变更 → refresh / repaintDialogRow / 滚动调整
```

**已核对的关键节点：**

1. **拉取**：`ApiWrap::requestDialogs(Folder*)`、`requestMoreDialogs`、`requestPinnedDialogs`；状态结构 `DialogsLoadState`（含 pinned 请求）。
2. **灌入**：`Session::applyDialogs` 先 `processMessages(..., NewMessageType::Last)`，再对每个 `MTPDialog` `match` → `applyDialog`；若带 `requestFolder` 与 `count`，更新 Folder 的 `setCloudListSize`。
3. **单条 dialog**：`History::applyDialog(requestFolder, MTPDdialog)` + `setPinnedFromEntryList`；处理 migrateFrom/To 时移出列表。
4. **Filter 更新**：`ChatFilters::apply` 直接吃 `updateDialogFilter*`（见上）；业务辅助在 `api/api_chat_filters.*`。
5. **UI 刷新触发**：`Entry::updateChatListEntry()` → `session().changes().entryUpdated(this, EntryUpdate::Flag::Repaint)`（可 `Postponed`）；高度变更走 `Flag::Height`。
6. **嵌入**：`MainWidget` 持有 `base::unique_qptr<Dialogs::Widget> _dialogs`，与 `_history` 并列为左右栏。

> **推测**：`Api::Updates::feedUpdate` 内对 `updateDialogPinned` / `updateFolderPeers` / 新消息导致的 dialog 顶栏变更等会回调 `History`/`Session` 方法；完整 `MTPUpdate` 分支表留待「API 更新 / PTS」专章对照 `api_updates.cpp`。

## 相关 UI 周边（同目录可见）

- **Stories**：`Dialogs::Stories::List` + `dialogs/ui/dialogs_stories_*`；`Widget::setupStories`。
- **Suggestions / Top peers**：`dialogs_suggestions.*`、`top_peers_strip.*`。
- **Quick actions**：滑动快捷操作（`dialogs_quick_action*`、`Ui::Controls::QuickDialogAction`）。
- **Community**：`dialogs_community_*`（与 v7 Communities 产品线相关，细节另章）。
- **无障碍**：`dialogs_inner_widget_accessibility.*`。

## 小结

会话列表不是单一 `QListView`，而是 **数据侧多 `MainList`（主列表 / 归档 / 每 Filter 一份）+ Entry 多态行**，由 **`Dialogs::InnerWidget` 动态切换 `_shownList`** 绘制；置顶与日期（及可选「未读置顶」）合成 `uint64` 排序键；Chat Folders 自 TL `MTPDialogFilter` 落入 `Data::ChatFilters`，并与 `Window::FiltersMenu` / `activeChatsFilter` 联动。网络侧以 `ApiWrap` 拉 dialogs、`Api::Updates` 推增量、`Session::applyDialog*` 写模型，再经 `entryUpdated` 驱动重绘。

## 深潜续篇

实现级绘制 / 滚动缓存 / 列表 freeze / 行所有权 / 相关测试面 → [`05-dialogs-impl-memory-perf.md`](05-dialogs-impl-memory-perf.md)。  
仓库级内存、测试体系与卡顿治理 → [`06-memory-testing-jank.md`](06-memory-testing-jank.md)。

## 本文未覆盖（建议后续）

- `HistoryWidget` / `history_inner_widget` 消息流与打开会话后的同步  
- `Api::Updates` 全部分支与 PTS/getDifference 对列表的精确影响  
- Forum topics 列表与 Saved Messages sublists 专论  
- Filters 设置 UI / Premium 锁定与 chatlist 邀请链路深潜  


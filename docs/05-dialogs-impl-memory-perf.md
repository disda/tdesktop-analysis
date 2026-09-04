# 05 · Dialogs 深潜：实现细节、内存与滚动/重绘

> 承接 [`04-dialogs-chat-list.md`](04-dialogs-chat-list.md) 的概览。本文聚焦 **实现路径**（绘制/滚动/虚拟化）、**列表侧内存与缓存**、以及 **触及 dialogs 的测试面**。材料来自 `dev` 上 raw 拉取的 `dialogs_inner_widget.*`、`dialogs_list.*`、`dialogs_entry.*`、`ui/rows_scroll_cache.*`、`dialogs_widget.*`、`config.h` 等；**未**全量克隆。推测处已标注。

## 1. 实现骨架：谁画、谁滚、谁持有行

### 1.1 宿主链

```
Dialogs::Widget
  └─ Ui::ElasticScroll _scroll
        └─ InnerList (layout)
              └─ Dialogs::InnerWidget   // 真正 paintEvent 画行
```

- `dialogs_widget.cpp` 构造：`_innerList = _scroll->setOwnedWidget(...)`，再 `_inner = _innerList->add(object_ptr<InnerWidget>(...))`。
- 滚动同步：`_scroll->scrolls(...)` / `geometryChanged` 驱动 `InnerWidget::visibleTopBottomUpdated`；行移动时 `_dialogMoved` → `_scroll->scrollToY` 调整视口，避免置顶重排时列表「跳动」。
- Overscroll：Stories 全高条用 `setOverscrollPullDistances(st::dialogsStoriesFull.height, 0)`。

### 1.2 不是 `QListView`

`InnerWidget` 直接继承 `Ui::RpWidget`，自管：

| 机制 | 落点 | 作用 |
|---|---|---|
| Y 索引命中 | `List::findByY` / `rowAtY` | O(log n) 找可见首行 |
| 几何 | `Row::top()` / `height()`；`List::height()` = 末行 bottom | 可变行高（forum / tagged 行更高） |
| 裁剪绘制 | `paintEvent` 内 `list.findByY(dialogsClip.top() - skip)`，超 clip 即 `break` | **部分绘制**（viewport 虚拟化） |
| 选择态 | 裸指针 `_selected` / `_pressed` 指向 `List` 拥有的 `Row*` | 行销毁时靠 `dialogRowReplaced` / `existenceChanged` 清零 |

### 1.3 行所有权（dialogs 数据侧）

`Dialogs::List`（`dialogs_list.h`）：

- `_rowByKey: map<Key, unique_ptr<Row>>` — **唯一所有者**；
- `_rows: vector<not_null<Row*>>` — 排序视图；
- `clear()` 清空两者；`remove` 可 `replacedBy` 做替换通知。

`Dialogs::Entry`（`dialogs_entry.h`）：

- `class Entry : public base::has_weak_ptr` — 允许 UI 侧 `base::weak_ptr` 安全观察；
- `_chatListLinks: flat_map<FilterId, RowsByLetter>` — 每个 filter 上指向 `Row*`（`RowsByLetter::main`），**不拥有** `Row`（由对应 `MainList`/`IndexedList` 拥有）。

过滤/搜索 UI 侧另有独立所有权：

- `_filterResultsGlobal: flat_map<Key, unique_ptr<Row>>`
- `_searchResults` / `_peerSearchResults` / `_previewResults`：`vector<unique_ptr<FakeRow|…>>`
- `_collapsedRows: vector<unique_ptr<CollapsedRow>>`

## 2. 绘制路径与滚动缓存（抗 jank 核心）

### 2.1 `paintEvent` 流程（已核对）

1. `Painter` + `isGifPausedAtLeastFor(GifPauseReason::Any)` → `p.setInactive` / `context.paused`（后台或策略暂停时停 GIF/视频头像动画）。
2. 若主内容被其它层重叠且非 Saved Sublists，`contentOverlapped` 则 **直接 return**（避免无用重绘）。
3. 构建 `Ui::PaintContext`（`st`、`filter`、`now=crl::now()`、`narrow` 等）。
4. 对每行 `paintRow`：
   - 计算 `RowsCacheKey(entry)`、物理像素 `cacheSize`；
   - **允许进缓存**当且仅当：`_rowsScrollCache.scrolling()` 且无 video/active/selected/quickAction/rightButton/expanding/childList 等；
   - 命中 → `paintRow` 位图 + `paintCachedRowOverlays`（再画视频头像 / emoji status / 动画预览）；
   - 未命中但允许 → 画进 `QImage` 写入 cache，并填充 `CachedRow{preview,badge,userpic,video}`；
   - 否则 → `Ui::RowPainter::Paint` 直接画到 widget。
5. Default 态：collapsed 行 → `findByY` 起遍历至 clip 底；拖拽置顶行最后单独画在上层。

### 2.2 `Ui::RowsScrollCache`（`ui/rows_scroll_cache.*`）

| 常量/行为 | 值 / 语义 |
|---|---|
| `kLimit` | 最多缓存 **256** 行位图 |
| `kMemoryLimit` | 累计约 **32 MiB**；超限且插入新 key 时 `clear()` |
| `kStopTimeout` | 停止滚动后 **120 ms** 清缓存并回调 `stopped`（InnerWidget 里清空 `_cachedRows` + `update()`） |
| `markScrolling()` | `visibleTopBottomUpdated` 在 Default 态视口变化时调用 |
| `invalidate(rowId)` | 单行失效；与 `_cachedRows.erase` 配对 |

**设计意图（据代码）**：快速惯性滚动时用整行 RGB32 位图复用，避免每帧重新排版文本/角标；动画预览与视频头像通过 overlay 仍可动。静止后立刻丢弃，控制峰值内存。

### 2.3 `CachedRow` 与 overlay

`InnerWidget::CachedRow`：`preview` / `badge` / `band` / `userpic` key / `video`。

- `animatedPreviewCached`：滚动中且预览/角标/视频元数据就绪时，允许 **不整行 invalidate**，只标 `bandDirty`。
- `invalidateLoadedUserpics`：滚动中扫可见区间，`userpicUniqueKey` 变化则 invalidate（头像下载完成不整表重绘）。
- `paintCachedRowOverlays`：窄栏跳过视频头像；否则 `paintUserpic` + `paintEmojiStatusFrame` + `paintAnimatedPreview`。

### 2.4 部分 `update` / 行级 repaint

- `updateDialogRow` → `rtlupdate(0, rowTop, width, height)` 或带 `updateRect`；按 `UpdateRowSection::{Default,Filtered,PeerSearch,MessageSearch}` 分区。
- `handleChatListEntryRefreshes`：同 filter 下 `from!=to` 时 `_dialogMoved.fire({from,to})`，并对 `[min(from,to), abs(delta)+height]` **矩形 update**（非整表）。
- `existenceChanged` → 清 selection/menu/`_filterResults` 后 `refresh()`。
- 置顶重排动画 `pinnedShiftAnimationCallback`：只 `update(0, updateFrom, width, updateHeight)`，高度按 `minHeight`/`maxHeight` 估预算。

## 3. 列表冻结：滚动/拖拽时的排序缓冲

### 3.1 为何 freeze

鼠标移动时（非 `_dragging`）调用 `_shownList->freeze()`，并用 `base::Timer _freezeTimer`：

```text
kFreezeTimeout = 2 * crl::time(1000)  // 2s
```

超时或显式 `unfreezeShownList` → `IndexedList`/`List::unfreeze()` 冲刷 `_pendingAdjust`。

`List::adjustByDate` 在 `_frozen` 时：

- 默认把行丢进 `_pendingAdjust`；
- **例外**：队列为空且该行是 `fixedOnTop` 或当前 filter 置顶 — 仍允许立即调整（归档/置顶不滞后）。

**效果**：短时间内大量 `entryUpdated` 导致的按日期重排不会在每次 mousemove 同步改 `_rows` 顺序，降低滚动卡顿；代价是冻结窗口内顺序可能短暂过期（超时后批量补齐；多条 pending 时可能 `sortByDate()`）。

`skipChatsListFreeze()`：正在 `_dragging` 时跳过 freeze（拖到 filter 时要最新顺序）。

## 4. 预加载与视口

### 4.1 `visibleTopBottomUpdated`

1. Default 态视口变 → `markScrolling()`；
2. `_visibleTop/_visibleBottom` 更新；
3. `preloadRowsData()`；
4. `loadTill = visibleTop + PreloadHeightsCount * viewportHeight`：
   - 触及 filtered peer-search 区 → `_loadMoreFilteredCallback`；
   - 触及列表底 → `_loadMoreCallback`（外层 `ApiWrap` 分页 dialogs）。

`PreloadHeightsCount = 3`（`config.h` 注释：还剩约 3 屏时发起 preload）。

### 4.2 `preloadRowsData`

- Default：对 `findByY(yFrom)…` 可见+预读带内 `entry()->chatListPreloadData()`（纯虚，由 `History`/`Folder`/Topic 实现 — **推测**含 userpic / 最后消息文本所需资源）。
- Filtered：对 filter / peerSearch / preview / search 结果分段 `loadUserpic()`。

## 5. 变更合流：Postponed 与 rpl

| API | 行为 |
|---|---|
| `Entry::updateChatListEntry()` | 清 `UpdatePostponed`，`session().changes().entryUpdated(..., Repaint)` |
| `Entry::updateChatListEntryPostponed()` | 置 flag，`Ui::PostponeCall` 合并到下一轮事件循环再 Repaint |
| `Entry::updateChatListEntryHeight()` | `Flag::Height` |
| InnerWidget 订阅 | `entryUpdates(Repaint\|Height)`、`chatListEntryRefreshes`、`dialogsRowReplacements`、`itemRemoved`、peer/userpic 变更等 |

拖拽开始时：`Ui::Animations::Manager::SetScheduleWithInvokeQueued(true)`，结束恢复 — 把动画 tick 排到 queued invoke，避免与 DnD 同步打架。

## 6. 置顶重排动画预算

- `_pinnedRows: vector<PinnedRow{ anim::value yadd; crl::time animStartTime }>`
- `_pinnedShiftAnimation: Ui::Animations::Basic`
- 时长：`st::stickersRowDuration`；曲线 `anim::sineInOut`
- `anim::Disabled()` 时直接 `now += duration`（无障碍/减动效下一帧到位）
- 边缘自动滚：`Ui::DraggingScrollManager::_draggingScroll`

## 7. 与 dialogs 相关的「测试」

| 路径 | 性质 | 是否覆盖列表逻辑 |
|---|---|---|
| `dialogs/dialogs_top_bar_suggestion_test.cpp` | `#ifdef _DEBUG` 热键：`Ctrl+Shift+T` 注入生日 suggestion UI，`Ctrl+Shift+A` 注入假授权更新 | **否** — 顶栏 suggestion chrome，非 `InnerWidget`/`MainList` |
| `SourceFiles/test/*`（`DESKTOP_APP_TEST_APPS`） | 进程内 task-test harness；`waitForChatsLoaded` / Strict 可等会话列表加载 | **弱相关** — 可作场景门闩，无 gtest 断言列表排序/绘制 |
| `SourceFiles/tests/test_*` | 独立小程序 `test_text`、`test_update_verify` | **不**测 dialogs |

**缺口**：未见针对 `List::adjustByDate`、`RowsScrollCache`、filter 切换滚动恢复、`UnreadState` 聚合的单元测试；列表正确性主要靠人工/Debug harness 场景。

## 8. 小结（实现 + 内存 + 性能）

1. 自绘 + `findByY` 部分绘制，替代 Qt item view。  
2. 滚动期行位图缓存（256 / 32MiB / 停滚 120ms 清空）+ overlay 保动画。  
3. 2s 列表 freeze + pending date-adjust，换平滑交互。  
4. 行级/矩形级 `update`，entry 刷新可 Postpone 合并。  
5. `unique_ptr<Row>` 归 `List`；`Entry` 用 `has_weak_ptr`；搜索结果自管 `unique_ptr`。  
6. 预读 3 屏 + bottom load-more 回调接 API。  
7. GifPause / `anim::Disabled` / overlapped early-out 降后台与减动效开销。  
8. 触及 dialogs 的自动化测试极薄（Debug 热键 + 可选 chats-loaded 门闩）。

## 本文未覆盖 / 待补

- `Ui::RowPainter::Paint` / `dialogs_layout` 文本与 badge 排版细节  
- `History::chatListPreloadData` 具体拉什么资源  
- Forum / Community / SavedSublist 分支绘制差异表  
- Instruments/Perf 实测帧时（源码无法给出）


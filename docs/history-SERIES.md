# 消息列表 / HistoryView 分册索引

> 对应上游 `Telegram/SourceFiles/history/` 与 `history/view/`（分支 `dev`）。本分册从打开会话第一帧写到更新与卡顿对照；总目录见 [`../SERIES.md`](../SERIES.md)，导读见 [`00-why-it-nails-it.md`](00-why-it-nails-it.md)。

## 阅读顺序

| 编号 | 文档 | 一句话 |
|---:|---|---|
| 07 | [history-structure-entry](07-history-structure-entry.md) | `HistoryWidget`/`ChatWidget` → `HistoryInner`/`ListWidget` 树与第一帧 |
| 08 | [history-data-pagination](08-history-data-pagination.md) | `History`/`HistoryItem`、SparseIds 切片、gap、MTP 翻页闸门 |
| 09 | [history-layout-virtualization](09-history-layout-virtualization.md) | 行高、clip 绘制、`scrollTopItem` / `ListMemento` 锚点 |
| 10 | [history-media-memory](10-history-media-memory.md) | 行内 Media 生命周期：heavy 登记、±2 屏 unload、keepAlive |
| 11 | [history-updates-jank](11-history-updates-jank.md) | newItemAdded / reveal / 主线程边界；对照 Dialogs 行位图缓存 |

## 上游锚点（速查）

```
history/history_widget.*          # 主中栏宿主
history/history_inner_widget.*    # 经典列表
history/history.*                 # History + HistoryBlock
history/history_item.*            # HistoryItem
history/view/history_view_list_widget.*
history/view/history_view_chat_section.*
history/view/history_view_element.*
history/view/media/history_view_media.*
data/data_histories.*
data/data_history_messages.*
storage/storage_sparse_ids_list.*
```

## 与 Dialogs 分册的关系

- 04/05：左侧会话列表（`Dialogs::*`、`RowsScrollCache`）
- 07–11：右侧消息流（`History*` / `HistoryView::*`）
- 06：两边共用的所有权、`crl`、测试与 jank 横切

（完）

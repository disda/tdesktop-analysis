# 消息列表 / HistoryView 分集索引

> 会话**内**消息列表（非侧栏 Dialogs）。上游核对：`telegramdesktop/tdesktop` `dev`。完整系列 TOC 见 [`../SERIES.md`](../SERIES.md)。

| Ep | 文档 | 一句话 |
|----|------|--------|
| H1 | [`07-history-structure-entry.md`](07-history-structure-entry.md) | 打开 peer → `HistoryWidget` / `HistoryInner` 挂树与首帧；对照 `ListWidget` |
| H2 | [`08-history-data-pagination.md`](08-history-data-pagination.md) | `History`/`HistoryItem`、GetHistory 分页、front block、`MessagesList` gap |
| H3 | [`09-history-layout-virtualization.md`](09-history-layout-virtualization.md) | 行高链、clip 绘制、`scrollTopItem` 锚点、与 Dialogs 行缓存对比 |
| H4 | [`10-history-media-memory.md`](10-history-media-memory.md) | 行内 Media、`shared_ptr`*Media、`_heavyViewParts` 与 ±2 屏 unload |
| H5 | [`11-history-updates-jank.md`](11-history-updates-jank.md) | `newItemAdded`/reveal、`crl::on_main`、对比 Dialogs `RowsScrollCache` |

阅读顺序：H1 → H5；可先扫 [`00-why-it-nails-it.md`](00-why-it-nails-it.md) 导读。

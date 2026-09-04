# 系列后续章节

> 已完成章节见「已完成」。**拟定后续（stubs）：全部完成。**
>
> **推荐首读**：[为什么实现得牛（导读）](docs/00-why-it-nails-it.md) — 再按 00–24 下钻。导读为独立文档，各章正文不再重复置顶块。
>
> **HistoryView 分集速览**：[docs/history-SERIES.md](docs/history-SERIES.md)

## 已完成

| 编号 | 标题 | 文档 |
|---:|---|---|
| ★ | **为什么实现得牛**（导读：约束 / 啊哈 / 代价 / 下一步） | [`docs/00-why-it-nails-it.md`](docs/00-why-it-nails-it.md) |
| 00 | 项目定位、许可证、技术栈与第三方依赖概览 | [`docs/00-overview.md`](docs/00-overview.md) |
| 01 | 可核验的时间线与大版本里程碑 | [`docs/01-history-timeline.md`](docs/01-history-timeline.md) |
| 02 | 顶层目录、子模块与设计原则 | [`docs/02-architecture.md`](docs/02-architecture.md) |
| 03 | MTProto 与网络层：`SourceFiles/mtproto`、会话/连接、TL/codegen | [`docs/03-mtproto-networking.md`](docs/03-mtproto-networking.md) |
| 04 | 会话列表 / Dialogs / Chat List：UI、数据模型与更新路径 | [`docs/04-dialogs-chat-list.md`](docs/04-dialogs-chat-list.md) |
| 05 | Dialogs 深潜：实现、内存、滚动/重绘与相关测试面 | [`docs/05-dialogs-impl-memory-perf.md`](docs/05-dialogs-impl-memory-perf.md) |
| 06 | 横切：内存所有权、测试体系与卡顿（jank）治理 | [`docs/06-memory-testing-jank.md`](docs/06-memory-testing-jank.md) |
| 07 | 消息列表结构与入口：打开会话 → `HistoryWidget` / `HistoryInner` 首帧 | [`docs/07-history-structure-entry.md`](docs/07-history-structure-entry.md) |
| 08 | 消息列表数据与分页：`History` / Item、向上加载、Gap、MTP | [`docs/08-history-data-pagination.md`](docs/08-history-data-pagination.md) |
| 09 | 消息列表布局虚拟化与滚动锚点 | [`docs/09-history-layout-virtualization.md`](docs/09-history-layout-virtualization.md) |
| 10 | 消息列表行内媒体与内存：Media、keepAlive、unload | [`docs/10-history-media-memory.md`](docs/10-history-media-memory.md) |
| 11 | 消息列表更新 / 动画 / 卡顿与 Dialogs 缓存对比 | [`docs/11-history-updates-jank.md`](docs/11-history-updates-jank.md) |
| 12 | 构建系统深潜：CMake helpers、`configure.py` 与依赖获取 | [`docs/12-build-system.md`](docs/12-build-system.md) |
| 13 | `desktop-app` 子模块图谱：`lib_*` 职责与组装 | [`docs/13-desktop-app-libs.md`](docs/13-desktop-app-libs.md) |
| 14 | API 层与更新机制：`SourceFiles/api`、`ApiWrap`、PTS/Updates、DC shift | [`docs/14-api-updates.md`](docs/14-api-updates.md) |
| 15 | UI 体系：`lib_ui`、`RpWidget`、主题与 style codegen | [`docs/15-ui-system.md`](docs/15-ui-system.md) |
| 16 | 数据与存储：`data` / `lib_storage` / 本地缓存边界与 eviction | [`docs/16-storage-cache.md`](docs/16-storage-cache.md) |
| 17 | Chat Folders / Filters 深潜：规则、`chatlist` 分享、设置 UI | [`docs/17-chat-folders-deep.md`](docs/17-chat-folders-deep.md) |
| 18 | 媒体管线：FFmpeg、Lottie、语音/视频消息与编辑器 | [`docs/18-media-pipeline.md`](docs/18-media-pipeline.md) |
| 19 | 通话与直播：`lib_webrtc`、`tgcalls`、`SourceFiles/calls` | [`docs/19-calls-streaming.md`](docs/19-calls-streaming.md) |
| 20 | 平台层：`SourceFiles/platform`（Win / macOS / Linux）差异表 | [`docs/20-platform-layer.md`](docs/20-platform-layer.md) |
| 21 | 安全相关表面：本地加密、passcode、webauthn/passkeys、代理（含 WEB proxy） | [`docs/21-security-surface.md`](docs/21-security-surface.md) |
| 22 | 打包与分发：Snap / Flatpak / 官方安装包 / 签名与更新通道 | [`docs/22-packaging-distribution.md`](docs/22-packaging-distribution.md) |
| 23 | 贡献与工程实践：Issues、Actions、分支策略（`dev` vs `master`） | [`docs/23-engineering-practice.md`](docs/23-engineering-practice.md) |
| 24 | 附录：术语表、API 凭证自建注意、许可证合规检查清单 | [`docs/24-appendix.md`](docs/24-appendix.md) |

## 拟定后续（stubs）

全部完成。

（完）

# 系列后续章节

> 已完成章节见「已完成」；其余为标题 stubs，顺序可按研究需要调整。

## 已完成

| 编号 | 标题 | 文档 |
|---:|---|---|
| 00 | 项目定位、许可证、技术栈与第三方依赖概览 | [`docs/00-overview.md`](docs/00-overview.md) |
| 01 | 可核验的时间线与大版本里程碑 | [`docs/01-history-timeline.md`](docs/01-history-timeline.md) |
| 02 | 顶层目录、子模块与设计原则 | [`docs/02-architecture.md`](docs/02-architecture.md) |
| 03 | MTProto 与网络层：`SourceFiles/mtproto`、会话/连接、TL/codegen | [`docs/03-mtproto-networking.md`](docs/03-mtproto-networking.md) |
| 04 | 会话列表 / Dialogs / Chat List：UI、数据模型与更新路径 | [`docs/04-dialogs-chat-list.md`](docs/04-dialogs-chat-list.md) |

## 拟定后续（stubs）

| 编号 | 拟定标题 |
|---:|---|
| 05 | 构建系统深潜：CMake helpers、`configure.py` 与依赖获取 |
| 06 | `desktop-app` 子模块图谱：`lib_*` 职责与版本锁定 |
| 07 | API 层与更新机制：`SourceFiles/api`、请求分发、分层 DC、PTS/Updates |
| 08 | UI 体系：`lib_ui`、自绘、主题与 codegen 样式 |
| 09 | 数据与存储：`data` / `lib_storage` / 本地缓存边界 |
| 10 | 历史消息视图：`history/`、`HistoryWidget`、打开会话后的同步 |
| 11 | Chat Folders / Filters 深潜：规则、`chatlist` 分享、设置 UI |
| 12 | 媒体管线：FFmpeg、Lottie、语音/视频消息与编辑器 |
| 13 | 通话与直播：`lib_webrtc`、`tgcalls`、`SourceFiles/calls` |
| 14 | 平台层：`SourceFiles/platform`（Win / macOS / Linux）差异表 |
| 15 | 安全相关表面：本地加密、passcode、webauthn/passkeys、代理（含 WEB proxy） |
| 16 | 打包与分发：Snap / Flatpak / 官方安装包 / 签名与更新通道 |
| 17 | 贡献与工程实践：Issues、Actions、分支策略（`dev` vs `master`） |
| 18 | 附录：术语表、API 凭证自建注意、许可证合规检查清单 |

（完）

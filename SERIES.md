# 系列后续章节

> 已完成章节见「已完成」；其余为标题 stubs，顺序可按研究需要调整。

## 已完成

| 编号 | 标题 | 文档 |
|---:|---|---|
| 00 | 项目定位、许可证、技术栈与第三方依赖概览 | [`docs/00-overview.md`](docs/00-overview.md) |
| 01 | 可核验的时间线与大版本里程碑 | [`docs/01-history-timeline.md`](docs/01-history-timeline.md) |
| 02 | 顶层目录、子模块与设计原则 | [`docs/02-architecture.md`](docs/02-architecture.md) |
| 03 | MTProto 与网络层：`SourceFiles/mtproto`、会话/连接、TL/codegen | [`docs/03-mtproto-networking.md`](docs/03-mtproto-networking.md) |

## 拟定后续（stubs）

| 编号 | 拟定标题 |
|---:|---|
| 04 | 构建系统深潜：CMake helpers、`configure.py` 与依赖获取 |
| 05 | `desktop-app` 子模块图谱：`lib_*` 职责与版本锁定 |
| 06 | API 层与更新机制：`SourceFiles/api`、请求分发、分层 DC、PTS/Updates |
| 07 | UI 体系：`lib_ui`、自绘、主题与 codegen 样式 |
| 08 | 数据与存储：`data` / `lib_storage` / 本地缓存边界 |
| 09 | 媒体管线：FFmpeg、Lottie、语音/视频消息与编辑器 |
| 10 | 通话与直播：`lib_webrtc`、`tgcalls`、`SourceFiles/calls` |
| 11 | 平台层：`SourceFiles/platform`（Win / macOS / Linux）差异表 |
| 12 | 安全相关表面：本地加密、passcode、webauthn/passkeys、代理（含 WEB proxy） |
| 13 | 打包与分发：Snap / Flatpak / 官方安装包 / 签名与更新通道 |
| 14 | 贡献与工程实践：Issues、Actions、分支策略（`dev` vs `master`） |
| 15 | 附录：术语表、API 凭证自建注意、许可证合规检查清单 |

（完）

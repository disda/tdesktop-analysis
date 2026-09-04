# 24 · 附录：术语表、API 凭证自建注意、许可证合规检查清单

> 材料：本系列 00–23、上游 `docs/api_credentials.md`、`LEGAL`、`LICENSE`、`README.md` Third-party 表、`.gitmodules`、`snap/snapcraft.yaml` 中的 API 注入点。附录以表为主；不提供绕过官方条款的方法。

## 1. 术语表（本系列常用）

| 术语 | 含义 |
|---|---|
| **tdesktop** | 官方桌面客户端仓 `telegramdesktop/tdesktop` |
| **MTProto** | Telegram 传输协议；实现于 `SourceFiles/mtproto` |
| **TL / scheme** | Type Language 与 `.tl` 描述；经 codegen 生成 C++ |
| **DC** | Data Center；会话可能 shift（第 14 章） |
| **PTS / Updates** | 更新序号与 `Api::Updates` 管道 |
| **MainList / Dialogs** | 会话列表数据面与 UI（第 04–05 章） |
| **HistoryView** | 消息列表：`HistoryWidget` / `HistoryInner` 等（第 07–11 章） |
| **lib_*** | `desktop-app` 工具库：`lib_ui`、`lib_storage`、`lib_webrtc`… |
| **RpWidget / rpl** | 自研 UI 基类与响应式流（第 15 章） |
| **EncryptionKey / CTR** | `lib_storage` 本地加密原语（第 16 / 21 章） |
| **Local passcode** | 锁本机盘面的口令；≠ 云端两步验证 |
| **Cloud password / 2FA** | 账号级密码；`PasscodeBox` + SRP |
| **Passkey / WebAuthn** | 平台认证器或 Cable/libfido2 登录 |
| **MTProxy** | Telegram 专用代理；secret 在 `password` 字段 |
| **WEB proxy** | `ProxyData::Type::Web`；经 WebView carrier 到 :443 relay |
| **App Translocation** | macOS 从隔离区运行只读副本的机制；需迁回原路径 |
| **v2 update / ES256** | 官方自动更新包格式与签名（`sign_update.py`） |
| **Canary** | 预发布签名构建轨（`public-canary` / `private-canary`） |
| **Snap / Flatpak** | Linux 沙盒分发；App id `org.telegram.desktop` |
| **Depot** | CI 赞助的高速 GitHub Actions runner |
| **tde2e** | Snap/构建 parts 中的端到端相关依赖名（见 snapcraft parts） |

## 2. API 凭证自建注意

上游 `docs/api_credentials.md` 原文要点：

1. **自建必须提供自己的 `api_id` 与 `api_hash`**
2. 申请说明见 [core.telegram.org/api/obtaining_api_id](https://core.telegram.org/api/obtaining_api_id)
3. 文档给出一组 **TEST ONLY** 凭证，**仅供本地测试**；用其**部署**会导致用户登录出现内部服务器错误
4. 本附录**不复制**该测试哈希到可执行脚本；请直接打开上游文档查看

构建注入点（分析时可见）：

| 位置 | 说明 |
|---|---|
| CMake `-DTDESKTOP_API_ID=` / `-DTDESKTOP_API_HASH=` | 官方 Linux/Snap 构建参数 |
| `snap/snapcraft.yaml` `cmake-parameters` | Snap 官方 id/hash（**勿**当个人应用凭证挪用） |
| 本地 `configure` / 构建文档 | 按 `docs/building-*.md` 传入自己的值 |

实践清单：

- [ ] 已用自己的 Telegram 账号在 my.telegram.org（或现行官方流程）申请应用
- [ ] CMake / 环境变量使用**自己的** id/hash，而非 TEST ONLY、而非 Snap 清单里的官方值
- [ ] 未计划把测试凭证打进公开分发安装包
- [ ] 若做签名更新 / 商店分发，另备自己的证书与隐私政策（官方 Key Vault / Developer ID 不可用）
- [ ] macOS 上理解未签名构建可能导致 WebAuthn `UnsignedBuild`（第 21 章）

## 3. 许可证合规检查清单

### 3.1 项目本体

| 项 | 状态 / 动作 |
|---|---|
| SPDX / GitHub license | `GPL-3.0` |
| `LICENSE` | GNU GPLv3 全文 |
| `LEGAL` | Copyright 2014–2026 The Telegram Desktop Authors；GPLv3(+)；**OpenSSL 链接特别例外** |
| 分发义务 | 提供对应源码、保留版权与许可声明、传达 GPLv3 条款 |
| 修改版本 | 需标明修改；若分发二进制则提供对应源 |

### 3.2 常见第三方（README Third-party 摘要）

抽查时逐项核对**实际链接版本**的许可证（下表为上游 README 声称，非法律意见）：

| 组件 | README 所称许可 |
|---|---|
| Qt 5.15 / 6 | LGPL |
| OpenSSL 3.x | Apache-2.0（叠加 LEGAL 的链接例外叙述） |
| WebRTC / tg_owt | BSD |
| zlib / liblzma / LZMA SDK | zlib / public domain |
| Breakpad | 自有 license 文件 |
| Crashpad | Apache-2.0 |
| OpenAL Soft / FFmpeg / Hunspell | LGPL |
| Opus / xxHash / range-v3 / GSL / QR / Ada | BSD / MIT / Boost / Apache 等 |
| 字体 Open Sans / Vazirmatn | Apache-2.0 / OFL |

`.gitmodules` 中还有 `rlottie`、`lz4`、`expected`、`kimageformats`、`fcitx5-qt`、`hime`、`nimf`、`tgcalls` 等——**发布清单应覆盖全部子模块与 ThirdParty 快照**。

### 3.3 分发形态检查

- [ ] **源码包**：含 `LICENSE`、`LEGAL`、对应 commit / tag、子模块 commit
- [ ] **二进制包**：提供获取完整对应源码的方式（同包或书面要约，按 GPLv3）
- [ ] **LGPL 组件**（Qt、FFmpeg、OpenAL…）：满足 LGPL 目标文件替换 / 源码提供要求
- [ ] **商店沙盒**（Snap/Flatpak）：元数据 `project_license` 与实际一致（metainfo 为 GPL-3.0）
- [ ] **未剥离**版权头；未把上游官方 api_id 标成自有应用而不自知
- [ ] 若专有产品**静态链接**本项目：通常与 GPLv3 **不兼容**——需独立法律评估（本系列不提供双许可方案）
- [ ] OpenSSL：依赖 `LEGAL` 中的 special exception；不要在去掉例外声明的分叉里假设仍可随意链接

### 3.4 本分析包

- 本仓库（`tdesktop-analysis`）为文档整理，**不包含** tdesktop 完整源码树
- 引用路径与片段来自公开 GitHub API / raw；大段许可证原文不在章节中整篇转载
- 不替代律师意见；商用分发前请对实际依赖树做 SPDX / `reuse` / 法务审查

## 4. 系列文档索引（00–24）

| 编号 | 文档 |
|---:|---|
| ★ | `00-why-it-nails-it.md` |
| 00–06 | 概览 → 架构 → MTProto → Dialogs → 内存/测试 |
| 07–11 | HistoryView 五集（见 `history-SERIES.md`） |
| 12–15 | 构建、`lib_*`、API/Updates、UI |
| 16–19 | 存储、Folders、媒体、通话 |
| 20–24 | 平台、安全、打包、工程、附录 |

## 5. 外部链接速查

| 用途 | URL |
|---|---|
| 上游仓 | https://github.com/telegramdesktop/tdesktop |
| 产品站 | https://desktop.telegram.org/ |
| API / 凭证 | https://core.telegram.org/api/obtaining_api_id |
| 翻译平台 | https://translations.telegram.org/ |
| Snap | https://snapcraft.io/telegram-desktop |
| Flatpak | https://flathub.org/apps/details/org.telegram.desktop |
| 贡献说明 | https://github.com/telegramdesktop/tdesktop/blob/dev/.github/CONTRIBUTING.md |

**要点**：自建先换 API 凭证；合规先认 GPLv3+OpenSSL 例外，再扫第三方法与商店元数据；术语表用于在 00–23 之间快速对齐名词。

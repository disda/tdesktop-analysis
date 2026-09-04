# 00 · 概览：Telegram Desktop（tdesktop）是什么

## 一句话

**Telegram Desktop** 是官方 Telegram 桌面客户端的完整源码与构建说明集合，基于 [Telegram API](https://core.telegram.org) 与 [MTProto](https://core.telegram.org/mtproto) 安全协议（见上游 `README.md`）。

- 上游仓库：`https://github.com/telegramdesktop/tdesktop`
- 产品页：`https://desktop.telegram.org/`
- GitHub 描述：`Telegram Desktop messaging app`
- Topics（API）：`messenger`、`multi-platform`、`telegram`、`telegram-desktop`、`telegram-solution`

## 许可证

| 来源 | 内容 |
|---|---|
| GitHub `license` 字段 | `GPL-3.0`（GNU General Public License v3.0） |
| `README.md` | 源码以 **GPLv3 + OpenSSL exception** 发布 |
| `LEGAL`（`dev` 分支 raw） | Copyright `(c) 2014-2026 The Telegram Desktop Authors.`；GPLv3 或其后版本；**特别例外**：允许与 OpenSSL 链接 |

完整文本见仓库根目录 `LICENSE` / `LEGAL`。本文不转载大段许可证原文。

## 支持的平台（上游 README）

最新版本宣称支持：

- Windows 7+（64 / 32 bit，含 portable）
- macOS 10.13+
- Linux 64-bit 静态构建
- Snap、Flatpak

旧系统截止版本（README 明文）：

| 截止版本 | 仍支持的旧环境（摘要） | GitHub Release 日期（`published_at`） |
|---|---|---|
| **4.9.9** | macOS 10.12；glibc &lt; 2.28 的 Linux 静态包 | `2023-09-19T13:10:18Z` |
| **2.4.4** | OS X 10.10–10.11；Linux 32-bit 静态包 | `2020-10-23T20:17:43Z` |
| **1.8.15** | Windows XP/Vista；更旧的 OS X | `2019-10-07T16:50:06Z` |

## 技术栈（可核验部分）

### 语言构成（`gh api .../languages`，字节数）

以 C++ 绝对主导：

| 语言 | 字节数（API 瞬时） |
|---|---:|
| C++ | 31642211 |
| Python | 429916 |
| Objective-C++ | 261170 |
| CMake | 188285 |
| 其它 | CSS / Shell / JS / GLSL / Dockerfile 等体量更小 |

### 构建与工程

- 根 `CMakeLists.txt`：`cmake_minimum_required(VERSION 3.25...3.31)`，`project(Telegram LANGUAGES C CXX ...)`；Apple 上启用 `OBJC`/`OBJCXX`
- 版本解析：`desktop_app_parse_version(Telegram/build/version)`（`cmake/version.cmake`）
- 大量逻辑在子模块 `cmake` → `https://github.com/desktop-app/cmake_helpers.git`
- 配置入口：`Telegram/configure.py` / `configure.sh` / `configure.bat`

### UI / 协议 / 媒体（README「Third-party」摘要）

上游 README 列出（节选，非完整审计）：

- **Qt 6** 与 **Qt 5.15**（略有补丁，LGPL）
- **OpenSSL 3.2.1**
- **WebRTC**（`tg_owt` 等）、**OpenAL Soft**、**Opus**、**FFmpeg**
- 崩溃上报：Breakpad / Crashpad
- 构建相关：GYP、Ninja、CMake
- 其它：zlib、LZMA、xxHash、Hunspell、range-v3、GSL 等

协议层：应用层走 Telegram API；传输层为 **MTProto**（源码中有 `Telegram/SourceFiles/mtproto` 目录，见架构篇）。

### 模块化子库（`.gitmodules`）

`desktop-app` 组织下拆出多枚子模块，例如：`lib_base`、`lib_ui`、`lib_rpl`、`lib_crl`、`lib_tl`、`lib_storage`、`lib_webrtc`、`lib_webview`、`lib_lottie`、`lib_qr`、`lib_spellcheck`、`lib_translate`、`codegen` 等。应用本体在 `Telegram/`，第三方在 `Telegram/ThirdParty/`。

## 默认分支与工作流提示

- API 显示 `default_branch`: **`dev`**
- 同时存在 `master`（与 `dev` tip SHA 不同；分析时需注明所依据的 ref）
- CI：README 徽章指向 GitHub Actions 的 Windows / macOS / Linux workflows；构建说明在 `docs/building-*.md`

## 本篇未覆盖

深入 MTProto 会话、UI 自绘管线、codegen 产物格式等见 [`SERIES.md`](../SERIES.md) 规划；当前包仅做概览级、可引用事实。

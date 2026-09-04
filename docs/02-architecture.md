# 02 · 架构速览：目录地图与设计原则

> 基于 `contents` API、根 `CMakeLists.txt`、`.gitmodules` 与 `README.md`。未克隆完整树，故 **不声称** 穷尽每个子目录文件数。

## 设计原则（从仓库结构归纳）

1. **跨平台单一代码库**：CMake 驱动；Windows / macOS / Linux（另有 Snap、Flatpak 发行形态）；Apple 目标启用 ObjC/ObjC++。  
2. **Qt 为底座，UI 大量自研**：依赖 Qt 5.15/6，同时以 `lib_ui` 等子模块承载 Telegram 风格自定义控件与绘制，而非「纯 Qt Widgets 默认皮肤」。  
3. **MTProto + Telegram API**：协议与会话相关代码集中在 `SourceFiles/mtproto`、`api` 等；网络语义与官方协议文档对齐。  
4. **Codegen 生成样板**：`Telegram/codegen` 子模块（`desktop-app/codegen`）与 `SourceFiles/codegen` 目录表明样式/方案/语言等有生成链路，减少手写重复。  
5. **库化拆分（desktop-app 生态）**：`lib_base`、`lib_rpl`（响应式）、`lib_crl`、`lib_tl`、`lib_storage`、`lib_webrtc`、`lib_webview` 等以 git submodule 引入，主仓 `Telegram/` 组装产品。  
6. **第三方隔离**：`Telegram/ThirdParty/` 挂 GSL、xxHash、rlottie、tgcalls、libfido2 等；根 README 列出许可证线索。

## 仓库顶层地图

`GET .../contents/`（`dev`）主要条目：

| 路径 | 类型 | 角色（简要） |
|---|---|---|
| `CMakeLists.txt` | 文件 | 根工程：解析版本、拉 Qt、`add_subdirectory(cmake)` / `Telegram` |
| `cmake/` | 子模块 | `desktop-app/cmake_helpers` — 选项、外部依赖、生成辅助 |
| `Telegram/` | 目录 | **应用主体**：源码、资源、子库、构建脚本 |
| `docs/` | 目录 | 官方构建说明（win/mac/linux/mas）、API credentials 说明等 |
| `lib/` | 目录 | （Contents 可见；细节需再下钻，本文未展开） |
| `snap/` | 目录 | Snap 打包相关 |
| `LICENSE` / `LEGAL` | 文件 | GPLv3 与 OpenSSL 例外声明 |
| `README.md` | 文件 | 产品说明、第三方、构建入口链接 |
| `changelog.txt` | 文件 | 面向用户的版本变更长日志 |
| `.gitmodules` | 文件 | 全部 submodule URL/path |
| `.github/` | 目录 | Actions 等 |

另有 `AGENTS.md` / `REVIEW.md` 等面向代理与评审的大型说明文件（本次未做内容分析）。

## `Telegram/` 下一层

| 路径 | 说明 |
|---|---|
| `SourceFiles/` | 主 C++ 业务与 UI 源码树（见下节） |
| `Resources/` | 资源 |
| `ThirdParty/` | 第三方 submodule 集合 |
| `codegen` | 代码生成工具链 submodule |
| `lib_*` | 多个功能库 submodule（见下） |
| `cmake/`、`CMakeLists.txt` | 目标定义与平台逻辑 |
| `build/` | 含 `version` 等构建元数据 |
| `shaders/` | 着色器 |
| `configure.py` / `.sh` / `.bat` | 配置入口 |
| `Telegram.plist` | macOS 相关 |

### 主要 `lib_*` / 工具子模块（`.gitmodules`）

| Submodule path | 远程（摘要） |
|---|---|
| `Telegram/lib_base` | `desktop-app/lib_base` |
| `Telegram/lib_ui` | `desktop-app/lib_ui` |
| `Telegram/lib_rpl` | `desktop-app/lib_rpl` |
| `Telegram/lib_crl` | `desktop-app/lib_crl` |
| `Telegram/lib_tl` | `desktop-app/lib_tl` |
| `Telegram/lib_storage` | `desktop-app/lib_storage` |
| `Telegram/lib_webrtc` | `desktop-app/lib_webrtc` |
| `Telegram/lib_webview` | `desktop-app/lib_webview` |
| `Telegram/lib_lottie` | `desktop-app/lib_lottie` |
| `Telegram/lib_qr` | `desktop-app/lib_qr` |
| `Telegram/lib_spellcheck` | `desktop-app/lib_spellcheck` |
| `Telegram/lib_translate` | `desktop-app/lib_translate` |
| `Telegram/codegen` | `desktop-app/codegen` |
| `cmake` | `desktop-app/cmake_helpers` |

另有 `ThirdParty/tgcalls`、`GSL`、`range-v3`、`libfido2` 等。

## `Telegram/SourceFiles/` 模块目录

Contents API 列出的 **目录名**（按字母序整理后的分组理解，非官方分层图）：

**会话 / 协议 / 数据**

- `mtproto` — MTProto 实现相关  
- `api` — API 调用封装  
- `data` — 客户端数据模型  
- `storage` — 本地存储协作  
- `core` / `main` — 核心与主流程  

**界面与窗口**

- `ui`、`window`、`dialogs`、`history`、`info`、`settings`、`menu`、`boxes`、`layout`、`overview`、`profile`  

**媒体与通话**

- `media`、`ffmpeg`、`calls`、`editor`  

**平台与集成**

- `platform` — 各 OS 差异  
- `webauthn`、`passport`、`payments`、`export`、`iv`、`inline_bots`、`poll`、`statistics`、`support`、`lang`、`countries`、`chat_helpers`、`intro`  

**工程辅助**

- `codegen`、`tests` / `test`、`_other`、`tde2e`  

> 「目录存在」≠「模块边界已文档化」；精确依赖图需后续用 CMake/`#include` 分析（见 `SERIES.md`）。

## 构建入口（文档层）

官方 `docs/`：

- `building-win.md` / `building-mac.md` / `building-linux.md` / `building-mas.md`
- `api_credentials.md` — 自建时 API ID/Hash 说明  

根 README 将 Windows / macOS / Linux(Docker) 构建分别链到上述文档。

## 架构示意（逻辑，非运行时精确）

```
┌─────────────────────────────────────────┐
│              Telegram Desktop           │
│  SourceFiles (ui/window/history/…)      │
├──────────────┬──────────────────────────┤
│ lib_ui / rpl │  lib_base / storage / tl │
├──────────────┴──────────────────────────┤
│     Qt 5.15/6  +  platform backends     │
├─────────────────────────────────────────┤
│  mtproto / api  ←→  Telegram DC         │
│  lib_webrtc / tgcalls / ffmpeg / …      │
└─────────────────────────────────────────┘
```

## 不确定 / 待后续核实

- `lib/` 顶层目录的具体用途（Contents 仅确认存在）。  
- codegen 输入 schema 与生成物落盘路径的完整列表。  
- `tde2e` 目录含义需读源码注释/CMake 目标名后再下定论。  

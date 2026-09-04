# 03 · MTProto 与网络层：`SourceFiles/mtproto`

> 基于 `dev` 分支 Contents API、`Telegram/cmake/td_mtproto.cmake` / `td_scheme.cmake` / `generate_scheme.cmake`，以及对若干 **头文件前部** 与 `scheme/*.tl` 开头的 raw 拉取。未克隆完整树；**不**声称穷尽每个方法的运行时语义。

官方协议说明（外部参考，非本仓源码）：[MTProto Mobile Protocol](https://core.telegram.org/mtproto)（`core.telegram.org/mtproto`，本次探测 HTTP 200，标题含 “MTProto Mobile Protocol”）。

## 代码落点（Where）

| 路径 | 角色（由文件名 / CMake / 头文件可见） |
|---|---|
| `Telegram/SourceFiles/mtproto/` | MTProto 客户端实现主目录 |
| `…/mtproto/details/` | 套接字、DC、密钥创建/绑定、序列化请求、RSA 等细节 |
| `…/mtproto/scheme/` | TL 方案：`api.tl`、`mtproto.tl` |
| `…/mtproto/web_proxy/` | Web 代理传输 / WebView / frame |
| `Telegram/SourceFiles/api/` | 业务 API 封装（贴纸、更新、发送等），消费 MTP 类型 |
| `Telegram/lib_tl`（submodule → `desktop-app/lib_tl`） | TL 基础类型与 `tl/generate_tl.py` |
| `Telegram/SourceFiles/codegen/scheme/codegen_scheme.py` | 调用 `generate_tl.generate`，生成 scheme C++ |
| `Telegram/cmake/td_mtproto.cmake` | 目标 `tdesktop::td_mtproto` |
| `Telegram/cmake/td_scheme.cmake` | 目标 `tdesktop::td_scheme` |
| `Telegram/CMakeLists.txt` | 另将 `session*` / `connection_*` / `mtp_instance*` / `web_proxy/*` 等列入主应用源 |

命名空间：头文件中统一为 `namespace MTP`，大量实现细节在 `MTP::details`。

## 构建切分（CMake 可见事实）

1. **`td_scheme`**：`generate_scheme()` 以  
   `mtproto/scheme/api.tl` + `mtproto/scheme/mtproto.tl` 为输入，脚本为  
   `SourceFiles/codegen/scheme/codegen_scheme.py`，并依赖  
   `lib_tl/tl/generate_tl.py`；产出构建目录下 `scheme.cpp` / `scheme.h` 与 `scheme-dump_to_text.*`。  
2. **`td_mtproto`**：链接 `tdesktop::td_scheme`；源列表含 `mtproto_auth_key`、`mtproto_config`、`mtproto_dc_options`、`mtproto_concurrent_sender`、`details/*` 套接字与密钥创建等（见 `td_mtproto.cmake` 全文列表）。  
3. **主目标**：`Telegram/CMakeLists.txt` 中另有 `mtproto/session.cpp`、`connection_tcp/http/resolving`、`mtp_instance`、`facade`、`config_loader`、`web_proxy/*` 等 —— 即会话/连接编排与 Instance 门面并不全部落在 `td_mtproto` OBJECT 库里。

> **推测（未跟完整链接图）**：业务可执行文件同时依赖 `td_mtproto` + 主目标内 MTProto 文件；精确边需后续 `#include`/链接分析。

## TL 方案 ↔ API ↔ codegen

### `scheme/mtproto.tl`

文件头注释写明 `// Core types (no need to gen)`，随后是授权密钥创建（`resPQ`、`req_DH_params`、`set_client_DH_params` 等）与系统消息类型（如 `msgs_ack`、`bad_msg_notification`）。对应客户端侧密钥/握手逻辑可见于 `details/mtproto_dc_key_creator.*`、`mtproto_dh_utils.*`、`mtproto_bound_key_creator.*`、`mtproto_dc_key_binder.*`。

### `scheme/api.tl`

Telegram API 的 TL 定义；文件末尾可见 `// LAYER 229`（拉取时）。体量约 281 KB（Contents API `size`）。生成类型前缀由 `codegen_scheme.py` 配置为 `MTP` / `MTPD` / `mtpc` / `MTP_`。

### `lib_tl`

- 公开别名：`desktop-app::lib_tl`  
- 源：`tl/tl_basic_types.*`、`tl_boxed.h`、`tl_type_owner.h`、`generate_tl.py`  
- `core_types.h` `#include "tl/tl_basic_types.h"` / `tl_boxed.h`，并定义 `mtpPrime`、`mtpRequestId`、`mtpMsgId`、`mtpBuffer` 等别名，以及 `DcId` / `ShiftedDcId` 与多种 DC shift 常量（download/upload/config/logout/updater 等）。

### `SourceFiles/api/`

大量 `api_*.cpp/.h`（如 `api_updates`、`api_sending`、`api_authorizations`）。`api_updates.h` 使用生成类型 `MTPUpdates` / `MTPUpdate`，并依赖 `Main::Session` / `ApiWrap` —— 属 **业务层**，不是传输层本身。

## 会话与连接组织（类名 / 文件名）

### 中心：`MTP::Instance`（`mtp_instance.h`）

- 继承 `QObject`；持有 `Config`、`DcOptions`、主 DC、`AuthKeysList` 等 `Fields`。  
- 可见 API：`send` / `sendSerialized`、`cancel`、`restart`、`killSession` / `stopSession`、`setUpdatesHandler`、`requestConfig` / `requestCDNConfig`、`logout` 等。  
- 注释区分 **Thread-safe** 与 **Main thread** 方法。  
- `Mode`：`Normal` / `KeysDestroyer`。  
- 默认主 DC 常量：`kDefaultMainDc = 2`。

### 会话：`MTP::details::Session` + `SessionPrivate` + `SessionData`

| 类型 | 文件 | 可见要点 |
|---|---|---|
| `Session` | `session.h` | 主线程构造；`start`/`stop`/`kill`/`restart`；`sendPrepared`；与 `Dcenter`、专用 `QThread` 关联 |
| `SessionData` | `session.h` | 待发/已发/已收队列 + `QReadWriteLock`；选项含 `useIPv4/IPv6`、`useHttp`、`useTcp`、代理 |
| `SessionPrivate` | `session_private.h` | 跑在会话线程；`connectToServer`、多路 `TestConnection`、收发包、`sendSecureRequest`、密钥更新 |

`SessionPrivate` 体量最大（Contents：`session_private.cpp` ≈ 86 KB）——细节语义留待专章，本文仅记结构。

### DC：`Dcenter`、`DcOptions`

- `details::Dcenter`：每 DC 的持久密钥 + 临时密钥槽（`TemporaryKeyType::{Regular, MediaCluster}`）。  
- `DcOptions`：端点表；`Variants` 按 **Address×Protocol**（`IPv4/IPv6` × `Tcp/Http`）索引；`DcType::{Regular, Temporary, MediaCluster, Cdn}`；`Environment::{Production, Test}`。

### 连接抽象与实现

```
AbstractConnection::Create(protocol)
  ├─ Tcp  → TcpConnection  → AbstractSocket::Create(secret)
  │                           ├─ secret[0]==0xEE 且长度≥21 → TlsSocket
  │                           └─ 否则 → TcpSocket
  └─ Http → HttpConnection（Qt QNetworkAccessManager）

ResolvingConnection → 包装 child，处理 domainResolved / 多 IP
```

（工厂行为摘自 `connection_abstract.cpp` 与 `details/mtproto_abstract_socket.cpp` 中可见分支，非完整状态机描述。）

相关文件：

- `connection_abstract.h` — `AbstractConnection`、`ConnectionPointer`、明文/密文包准备助手  
- `connection_tcp.h` / `connection_http.h`  
- `connection_resolving.h`  
- `details/mtproto_tcp_socket.*`、`mtproto_tls_socket.*`、`mtproto_web_proxy_socket.*`

### 发送门面

- `sender.h` — `MTP::Sender`：链式 `RequestBuilder`，Done/Fail 回调模板（偏主线程用法）。  
- `mtproto_concurrent_sender.h` — `ConcurrentSender`：可跨线程 `send()`，经 `weak_qptr<Instance>` + runner。  
- `facade.h` — DC shift 助手：`configDcId`、`downloadDcId`、`uploadDcId`、`updaterDcId`、`groupCallStreamDcId` 等；连接状态常量 `DisconnectedState` / `ConnectingState` / `ConnectedState`。

### 代理与配置旁路

- `mtproto_proxy_data.h` — `ProxyData::Type::{None, Socks5, Http, Mtproto, Web}`。  
- `proxy_check.*`、`config_loader.*`、`special_config_request.*` — 配置拉取 / 特殊端点。  
- `web_proxy/*` — Web 代理路径（`web_proxy_transport` 单文件约 64 KB）。

### 认证密钥

`AuthKey`（`mtproto_auth_key.h`）：`kSize = 256`（注释 2048 bits）；`Type::{Generated, Temporary, ReadFromFile, Local}`；提供 `prepareAES` / `prepareAES_oldmtp` 与 IGE 加解密包装。

## 异步与线程风格（头文件可见）

- **Qt 对象模型**：`Instance`、`Session`、`SessionPrivate`、`AbstractConnection` 均为 `QObject`；连接可 `moveToThread`。  
- **专用会话线程**：`Session` 构造接收 `not_null<QThread*>`；`SessionPrivate` 在该线程上工作；`SessionData` 用读写锁在主线程与会话线程间交接队列。  
- **回调 / 生产者**：广泛使用 `Fn<>`、`rpl::producer`（如 `mainDcIdValue`、`dcTemporaryKeyChanged`）；`AbstractSocket` 以 `rpl::event_stream` 暴露 `connected` / `readyRead` / `error`。  
- **时间**：`crl::time`、`base::Timer`（见 `Session` 的 `_sender` timer、`SessionPrivate` 重试/ping 定时器名）。  
- **并发发送**：`ConcurrentSender` 名称与 `with_instance` 模板表明请求可从非 Instance 主路径投递（具体 runner 实现未在本文展开）。

> **推测**：主线程负责 UI/回调解析，会话线程负责套接字与加密帧；与经典 tdesktop 线程模型叙述一致，但完整线程图需读 `mtp_instance.cpp` 中 session 创建路径。

## 逻辑分层示意（非运行时精确）

```
  SourceFiles/api/*  (业务 RPC / Updates)
           │  MTP* 生成类型 + Sender / ConcurrentSender
           ▼
  MTP::Instance  ──►  Session (per ShiftedDcId)
           │              │
           │              ▼
           │         SessionPrivate ── AbstractConnection
           │                              ├ Tcp / Http / Resolving
           │                              └ AbstractSocket (Tcp/Tls/WebProxy)
           ▼
  DcOptions / Config / AuthKey / Dcenter
           ▲
  td_scheme ← api.tl + mtproto.tl ← codegen_scheme.py ← lib_tl/generate_tl.py
```

## 外部协议文档指针

| 资源 | URL |
|---|---|
| MTProto 总览 | https://core.telegram.org/mtproto |
| （同站常见子页，未逐一核内容） | `/mtproto/description`、`/mtproto/auth_key`、`/mtproto/service_messages` 等 |
| Telegram API / TL | https://core.telegram.org/api 、 https://core.telegram.org/schema |

本仓实现应对齐官方语义，但 **以源码与当前 `api.tl` LAYER 为准**；官方页可能超前或滞后于桌面端所用 LAYER。

## 不确定 / 待后续章节

- `SessionPrivate` 内多连接竞速（`TestConnection` / `confirmBestConnection`）的完整策略。  
- Web Proxy（`web_proxy_transport`）与 `ProxyData::Type::Web` 的端到端路径。  
- `dedicated_file_loader` 与 media DC shift（`kBaseDownloadDcShift` 等）如何服务大文件。  
- `api_updates` / PTS 与 `Instance::setUpdatesHandler` 的接线（适合「API 层与更新机制」章）。  
- `tde2e` 与 MTProto 是否共享传输（02 章已列为待查）。

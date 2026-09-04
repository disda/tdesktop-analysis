# 01 · 历史时间线（可核验里程碑）

> 规则：下列日期/版本均来自本次拉取的 GitHub API、`changelog.txt`（`dev` raw）或上游 `README.md`。**未核验的传闻不写**；changelog 与 Releases 不一致处会标明。

时区说明：API 时间为 UTC；正文同时给出便于对照的 UTC 日期。用户本地为 Asia/Shanghai（UTC+8）时，可自行 +8 小时换算。

## 关键里程碑一览

| 日期 (UTC) | 事件 | 来源 |
|---|---|---|
| **2013-11-29**（changelog 记） | changelog 最早条目之一：`0.1.16 (29.11.13)` | `changelog.txt`（早于 GitHub 仓库创建；**非** GitHub Release） |
| **2014-05-02** | GitHub 仓库创建；同日 *Initial commit* `6d9ac2c` | `repos/... created_at`；`commits?until=2014-05-03` |
| **2015-09-13** | GitHub Releases 中较早可见条目之一：`v0.8.57` | `releases/tags/v0.8.57` → `2015-09-13T11:20:38Z` |
| **2015-09-24**（changelog） / **2015-10-04**（Release） | Channels；`0.9` / `v0.9.1` | changelog `0.9 stable (24.09.15)`；Release `v0.9.1` `2015-10-04T13:07:20Z` |
| **2017-01-11** | **v1.0.0**：Material 风格 UI、自定义主题、置顶对话等 | Release `2017-01-11T22:19:09Z`；changelog `1.0.0 (11.01.17)` |
| **2019-10-07** | **v1.8.15**：README 称其为 Win XP/Vista 等旧系统最后支持版 | Release `2019-10-07T16:50:06Z`；`README.md` |
| **2020-03-30** | **v2.0.0**：Chat Folders 等 | Release `2020-03-30T14:34:10Z`；changelog `2.0 (30.03.20)` |
| **2020-10-23** | **v2.4.4**：旧 macOS / Linux 32-bit 截止版（README） | Release `2020-10-23T20:17:43Z` |
| **2021-08-31** | **v3.0.0**：直播/屏幕分享到无限观众等 | Release `2021-08-31T22:24:21Z`；changelog `3.0 (31.08.21)` |
| **2022-06-21** | **v4.0.0**：Telegram Premium 相关能力集中出现 | Release `2022-06-21T08:40:08Z`；changelog `4.0 (22.06.22)` |
| **2023-09-19** | **v4.9.9**：旧 macOS 10.12 / 旧 glibc Linux 截止版（README） | Release `2023-09-19T13:10:18Z` |
| **2024-05-02** | **v5.0.0**：自定义字体、Frequent/Recent、频道列表等 | Release `2024-05-02T10:14:53Z`；changelog `5.0 (02.05.24)` |
| **2025-07-31** | **v6.0.0**：Public Post Search、Story Albums 等（changelog `6.0`） | Release `2025-07-31T20:18:47Z`；changelog `6.0 (31.07.25)` |
| **2026-07-14** | **v7.0.1**：GitHub 上 **首个 7.x Release tag**（无 `v7.0.0` tag） | Release `2026-07-14T16:16:01Z`；changelog 另有 `7.0 (11.07.26)` / `7.0.1` 条目 |
| **2026-08-22** | **v7.1.0** | Release `2026-08-22T10:03:34Z` |
| **2026-09-02** | **v7.1.5**（本次拉取时 Releases 列表最新正式版之一） | Release `2026-09-02T19:49:05Z` |

## 仓库元数据快照

拉取自 `GET /repos/telegramdesktop/tdesktop`：

- `created_at`: `2014-05-02T12:36:31Z`
- `pushed_at`（当时）: `2026-09-03T18:26:38Z`
- `default_branch`: `dev`
- `stargazers_count` / `forks_count`: `32825` / `7172`（瞬时）

首次提交（API）：

- SHA 前缀 `6d9ac2c`，message `Initial commit`，`2014-05-02T12:36:31Z`

## 大版本功能摘要（changelog，压缩引用）

仅摘录标题级要点，避免大段复制：

- **1.0.0**：新 Material 风格与动画；自定义主题；为所有人删除消息；置顶聊天；共同群组  
- **2.0**：Chat Folders；侧栏切换文件夹；骰子与一批动画 emoji  
- **3.0**：向无限观众直播视频/屏幕；转发时控制显示原发送者与说明文字等  
- **4.0**：Premium（大文件、更快下载、加倍限制、语音转写、更多表情反应等）  
- **5.0**：自定义字体族；聚焦搜索时 Frequent/Recent；频道列表与相似频道；投票中动画 emoji（Premium）等  
- **6.0**：Public Post Search；Story Albums；Gift Collections；Profile Rating  
- **7.0 / 7.0.1**：Rich messages / Rich Text Editor；Communities；Invisible Bot Messages 等  
- **7.1**：群组/频道欢迎语；富文本按钮与文件块；阅后即焚媒体；视频头像与视频编辑；WEB proxy 类型等  

## Releases vs changelog 注意点

1. **无 `v7.0.0` GitHub tag/Release**；7.x 线上 Release 从 **`v7.0.1`** 起。changelog 写有 `7.0 (11.07.26)`，属 changelog 记录，不等于存在同名 GitHub Release。  
2. `dev` 上 changelog 顶部可见 **`7.2.5 (03.09.26)`** 等，可能尚未全部反映在 `releases?per_page=20` 的最新 tag 中——写文档时区分「changelog 已记」与「GitHub 已发 Release」。  
3. 部分极早期 `v0.5.x` / `v0.8.x` Release 的 `published_at` 集中在 2017-03 批量上传，**不能**当作真实首次公开发布日；更宜结合 changelog 日期与仓库创建时间交叉理解。本表对早期条目已尽量选用较可信的 `v0.8.57` / `v0.9.1` 等。

## 近期 tag 样本（`tags?per_page=30`）

`v7.1.5` … `v6.7.5` 连续补丁线活跃；完整列表见 API，此处不枚举。

# 23 · 贡献与工程实践：Issues、Actions、`dev` vs `master`

> 材料：`.github/CONTRIBUTING.md`、`ISSUE_TEMPLATE/*`、`.github/workflows/*.yml`（Contents + raw）、`master_updater.yml`、`gh api` 默认分支元数据、`docs/building-*.md`。未统计历史 PR 合并率。

## 1. 贡献边界（CONTRIBUTING）

上游明确欢迎 / 拒绝：

| 接受 | **不合并** |
|---|---|
| Bug 修复 | 新功能实现 |
| 性能优化 | 新语言翻译（走 [translations.telegram.org](https://translations.telegram.org/)） |
| 文档修复 | 新的 UI 元素 / UX 改动 |

原因：tdesktop 是 Telegram 产品的一部分，功能与设计由 Telegram 团队按非公开路线图决定。

PR 实践要求：

1. **单问题**原子 PR；不要混拼写与重构
2. **squash** 成单 commit
3. 不要把 whitespace 清理混进逻辑改动
4. 标识符写全名（反对 `o` / `ph` / `mftdt` 式缩写）
5. 本地测过再提
6. commit message 讲清 *why*；修 issue 用 `Fix #N`

同步 fork：

```bash
git remote add upstream https://github.com/telegramdesktop/tdesktop.git
git fetch upstream master
git rebase upstream/master
# 再 force push 到自己的 fork 分支
```

（文档示例以 `master` 为上游稳定线；日常开发跟踪见下节 `dev`。）

## 2. 分支策略：`dev` vs `master`

| 分支 | 角色 |
|---|---|
| **`dev`** | GitHub **默认分支**；日常开发、CI 主目标、文档与 changelog 常领先 Releases |
| **`master`** | 稳定线；**Release 发布后**由 bot 从 `dev` 同步 |

自动化：`.github/workflows/master_updater.yml`

```yaml
on:
  release:
    types: released
jobs:
  User-agent:
    runs-on: ubuntu-slim
    steps:
      - uses: desktop-app/action_code_updater@master
        with:
          type: "dev-to-master"
```

含义：打出正式 GitHub Release 后，把 `dev` 变更推到 `master`，减少「默认分支超前、稳定分支落后」的人工合并。

其它长期分支（CI 特判）：

- `public-canary` / `private-canary` — 只跑 `canary.yml`，其它 workflow `branches-ignore`
- `nightly` — 平台 CI 倾向 Depot 大 runner 并上传产物

```mermaid
gitGraph
  commit id: "master stable"
  branch dev
  checkout dev
  commit id: "feat/fix"
  commit id: "ci green"
  checkout master
  commit id: "release tag" tag: "vX.Y.Z"
  checkout dev
  commit id: "post-release"
  checkout master
  merge dev id: "master_updater"
```

## 3. Issues 治理

模板（`.github/ISSUE_TEMPLATE/`）：

- `BUG_REPORT.yml`
- `FEATURE_REQUEST.yml`
- `config.yml`

配套 workflow（机器治理，而非编译）：

| Workflow | 用途 |
|---|---|
| `stale.yml` | 陈旧 issue 标记 |
| `lock.yml` | 锁定过期讨论 |
| `issue_closer.yml` | 自动关闭规则 |
| `cant-reproduce.yml` | 无法复现标签流 |
| `needs-user-action.yml` / `waiting-for-answer.yml` | 等待用户信息 |
| `changelog.yml` | changelog 相关校验/更新 |
| `copyright_year_updater.yml` | 版权年份 |
| `unused_styles_updater.yml` | 未用 style 清理 |
| `user_agent_updater.yml` | UA 字符串维护 |

Bug 跟踪入口亦写在 AppStream `bugtracker`：`github.com/telegramdesktop/tdesktop/issues`。

## 4. GitHub Actions 矩阵（构建向）

| Workflow | 平台 / 内容 | 触发摘要 |
|---|---|---|
| `win.yml` | Windows x64 / x86 / arm64 | push/PR；忽略它平台源与 snap |
| `mac.yml` | macOS | 同上；Depot `depot-macos-latest` 在 PR/tag/nightly |
| `mac_packaged.yml` | 打包变体 | 独立打包校验 |
| `linux.yml` | Rocky Linux 8 + Docker 环境 | 忽略 win/mac 源与 snap |
| `snap.yml` | Snap 构建 | 关注 `snap/**` |
| `docker.yml` | 推送 `centos_env` 镜像 | 仅默认分支 + docker 路径 |
| `canary.yml` | 三平台 LTO Release + 签名发布 | 仅 canary 分支 |
| `canary-bot-api.yml` | Bot API 镜像钉扎 | canary 基础设施 |
| `winget.yml` | 发布到 WinGet | `release` released/prereleased |
| `master_updater.yml` | `dev`→`master` | `release: released` |

路径过滤共性：改 `docs/**`、`*.md`、`LICENSE`、`LEGAL` **通常不触发**平台编译（Windows 对 `docs/building-win*.md` 有例外）。

Runner 策略：PR / tag / `nightly` 多用 **Depot** 赞助的大机（`depot-windows-latest-16`、`depot-macos-latest`、`depot-ubuntu-latest-16`）；普通 push 可用 `ubuntu-latest` 等标准机。README 致谢 Depot。

## 5. 本地构建入口

官方文档（仓内）：

- `docs/building-win.md`（及 win 变体）
- `docs/building-mac.md` / `docs/building-mas.md`（Mac App Store）
- `docs/building-linux.md`（Docker / centos_env）

与第 12 章 `configure.py` / `prepare.py` / cmake_helpers 衔接：贡献者先按平台文档拉依赖，再改代码。自建 API 凭证见第 24 章。

## 6. 实操建议（给分析者 / 外围贡献者）

1. **读 issue / PR 时以 `dev` 为准**看最新代码；对照 Releases 时再看 tag / `master`
2. 提修复前确认不属于「新 UI / 新功能」——否则即使代码质量好也不会合
3. 改平台文件时看对应 workflow 的 `paths-ignore`：只动 `platform/win` 不会跑 Linux CI
4. 安全/更新相关改动要意识到 Canary 与正式签名密钥分离（第 22 章）
5. 翻译、产品文案走 Translations 平台，不要开「加某语言」PR

**要点**：工程上 `dev` 是心跳，`master` 是发版镜像；CI 按平台拆分且大量路径忽略；社区治理靠模板 + stale/lock 机器人；代码贡献面刻意收窄到缺陷与性能，以保护统一产品体验。

# OpenCode v2 自动构建发布

每天自动检查上游 [`anomalyco/opencode` 的正式版 tag](https://github.com/anomalyco/opencode/tags)（`v2.0.x`，由上游 `publish.yml` 打出并同步发到 npm `@opencode/cli`），
若 tag 名或其指向的提交有变化，就**从该 tag 指向的提交**编译 **Windows 桌面版** 与 **Windows 终端版**，
以**同名 tag**（如 `v2.0.16`）发布到本仓库的 GitHub Release —— 产物版本串与官方完全一致，
并把 tag 与提交写回 `version.json` 作为下一轮的比对基准。

## 文件

| 文件 | 作用 |
| --- | --- |
| [`.github/workflows/daily-v2-release.yml`](.github/workflows/daily-v2-release.yml) | 定时检查 + 构建 + 发布 + 同步 |
| [`version.json`](version.json) | 记录「上一次已构建的官方 tag 与提交」，由 Action 自动提交更新 |

仓库里**不需要**放 opencode 源码：workflow 直接 checkout 上游 tag 指向的 commit 来构建。

> ⚠️ 官方 tag 指向的提交**不在 `v2` 分支线上**（上游 `publish.ts` 是 `git switch --detach` 后提交打 tag，
> 再单独把版本同步提交推回 `v2`，两条线是 diverged 关系），所以必须按 tag 解析出的 SHA checkout，
> 用 `ref: v2` 会拿到另一条线上的代码。

## 启用步骤

1. 把 `.github/workflows/daily-v2-release.yml` 和 `version.json` 提交到你自己的仓库默认分支。
2. **Settings → Actions → General → Workflow permissions** 选择 **Read and write permissions**
   （workflow 内已声明 `permissions: contents: write`，但组织级策略若限制为只读则必须在这里放开）。
3. 确认默认分支**没有开启**「禁止 github-actions 推送 / 必须 PR」的保护规则，
   否则最后一步 `sync-version` 提交 `version.json` 会失败（构建和发布本身不受影响）。
4. 想立刻验证，去 **Actions → daily-v2-release → Run workflow**，勾选 `force` 手动触发一次。

## 版本命名

**与官方完全对齐**，直接采用上游 release tag，不做自造格式：

| 用途 | 格式 | 例子 |
| --- | --- | --- |
| Release tag | `v<major>.<minor>.<patch>`（与官方 tag 同名） | `v2.0.16` |
| 产物版本串（`OPENCODE_VERSION`，烧进二进制、写进 package.json） | `<major>.<minor>.<patch>` | `2.0.16` |

- 版本号取自**上游最新正式版 tag**，也就是官方 `npm i -g @opencode/cli@latest` 的同一个号；
  `version.json` 里另记 `upstream_tag`（官方 tag 名）作为比对基准。
- tag 指向提交的 committer date 只用于 Release 标题展示，不参与命名。
- 同一 tag 强制重跑时，本仓库 Release tag 追加运行编号（`v2.0.16.42`），
  版本串挂到 prerelease 段（`2.0.16-mirror.42`，仍是合法 semver，electron-builder 可校验）；
  比对基准始终用 `upstream_tag`，不会因撞车后缀导致无限重建。
- **不再有 7 天节流**：每个官方 tag 都跟（官方约每天发一个）。
- 默认标记为 **prerelease**。想让最新一次构建显示为 Latest，把 workflow 顶部的 `PRERELEASE` 改成 `"false"`。

## 工作流程

```
check ──► build-cli ──► build-desktop ──► release ──► sync-version
（每日 08:00 北京时间检查一次；上游出新 tag 才真正构建）
```

### 1. `check` — 判定是否需要构建

取 `$UPSTREAM_BRANCH` 前缀（默认 `v2.`）下 semver 最大的正式版 tag，解析出它指向的提交 SHA，
与 `version.json` 的 `upstream_tag` + `commit` 比对：

| 情况 | 结果 |
| --- | --- |
| tag 名与 SHA 都相同 | 跳过（上游没有新 release） |
| 上游出现新 tag（如 `v2.0.16` → `v2.0.17`） | 构建并发布 |
| 同名 tag 被 force 重推（SHA 变了） | 构建并发布 |
| 手动勾选 `force` | 无视比对，直接构建 |
| `version.json` 是旧格式（无 `upstream_tag`） | 视为首次构建（迁移路径） |

跳过时整条链显示为 skipped（不算失败），Job summary 会写明原因。

`version.json` 里的 `desktop_version` / `cli_version` 从 **tag 提交**读取
（上游 `publish.ts` 打 tag 前已把所有 package.json 统一改写成该版本），
只写进 Release 正文做展示，不参与判定。

### 2. `build-cli` — 终端版

Linux runner 上用上游的 `packages/cli/script/build.ts` 交叉编译（与官方构建方式一致），
一次产出全部 12 个目标，只取 Windows 三个打包：

- `opencode-cli-windows-x64.zip`
- `opencode-cli-windows-x64-baseline.zip`（不支持 AVX2 的老 CPU）
- `opencode-cli-windows-arm64.zip`

`OPENCODE_VERSION` 注入的正是官方版本号（如 `2.0.16`），解压后是 `bin/opencode.exe`。

### 3. `build-desktop` — 桌面版

Windows runner 上按上游 `publish.yml` 的 electron 流程：

1. 把上一步的 CLI 产物放回 `packages/cli/dist`
2. **冒烟测试**：直接执行 Linux 交叉编译出来的 Windows 二进制（自动探测 `bin/*.exe`）`--version`，
   确认能在真 Windows 上运行且烧进去的版本号与注入值一致（放在 `bun install` 之前，坏了能早退）
3. `bun ./scripts/prepare.ts`（内嵌 `opencode-cli.exe`、复制 prod 图标、写入版本号）
4. `bun run build`（electron-vite）
5. `npx electron-builder --win --x64 --publish never`

产物：`opencode-desktop-win-x64.exe`（NSIS 安装程序，约 200 MB）+ `.blockmap`。
注意 electron-builder 的 `${os}` 在 Windows 下展开为 `win` 而非 `windows`，所以文件名是 `win-x64`。

### 4. `release` — 发布

用内置 `GITHUB_TOKEN` 创建 Release，上传桌面版安装程序与三个终端版 zip。
正文注明对应的官方 tag / npm 版本，并用 GitHub compare API 列出**距上次构建之间的上游 commit**（首次构建则列最近 30 条）。

### 5. `sync-version` — 同步

把本轮的官方 tag（`upstream_tag`）、版本串、Release 链接写回 `version.json` 并提交推送。

## 实测记录（2026-09-05，ming-14/OpenCodev2-Release）

> ⚠️ 下面是**旧版（按 commit 日期打 tag `v2-<日期>-<sha>`、7 天节流）**的实测记录，
> 已被「对齐官方 `v2.0.x` tag」的新方案取代，仅作历史参考。

| Run | 触发 | 结果 | 验证了什么 |
| --- | --- | --- | --- |
| #1 | push | failure | workflow 解析失败：job 级 `env:` 里用了 `env` 上下文（只有 step 级可用） |
| #2 | dispatch | **success** | 首次构建全链路：12 目标交叉编译 → Windows 打包 → 5 个资产上传 → `version.json` 回写 |
| #3 | dispatch | **success** | 跳过路径：`decision: 上游无新提交` → 下游 4 个任务全 skipped，无重复 Release |
| #4 | dispatch `force` | **success** | force 无视节流 + tag 撞车退化出 `v2-20260905-7a4ad68.4` + CLI 冒烟测试 |

关键实测数据：

- `check` ~40 秒；`build-cli` ~3 分钟；`build-desktop` ~13 分钟；全程 ~17 分钟。
- 上游 `setup-bun` 复合 action 在跨仓库 checkout 下可正常引用（`uses: ./.github/actions/setup-bun`）。
- 交叉编译的 Windows 二进制在真 Windows runner 上可直接运行并校验版本号。
- 未签名构建可行：`Skipping Windows signing because Azure Artifact Signing is not configured`，electron-builder 正常出包。
- 产物：桌面版 202 MB + blockmap，三个终端版 zip 各 88–93 MB，一轮 Release 约 475 MB。

## 可调项（都在 workflow 顶部的 `env`）

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `UPSTREAM_REPO` / `UPSTREAM_BRANCH` | `anomalyco/opencode` / `v2` | 上游来源；`UPSTREAM_BRANCH` 还决定跟哪条 tag 线（前缀 `<branch>.`）与变更列表 |
| `OPENCODE_CHANNEL` | `prod` | `prod` → 产品名 "OpenCode"、appId `ai.opencode.desktop`；改 `beta` → "OpenCode Beta"、appId `ai.opencode.desktop.beta` |
| `PRERELEASE` | `"true"` | 构建是否标记为预发布 |
| `BUN_COMPILE_RELEASE` | `bun-v1.4.2` | 编译进产物的 Bun 运行时版本，需与上游 `packageManager` 匹配 |
| `on.schedule.cron` | `0 0 * * *` | UTC 00:00 = 北京时间 08:00 |

- 上游官方约**每天**打一个 `v2.0.x` tag，所以本仓库默认也是每个版本都跟，不再节流。
  注意官方 tag 是 `--force-with-lease` 推的，理论上可能被重推，届时会检测到 SHA 变化并重建。

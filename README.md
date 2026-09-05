# OpenCode v2 自动构建发布

每天自动检查上游 [`anomalyco/opencode` 的 `v2` 分支](https://github.com/anomalyco/opencode/tree/v2)，
若 HEAD commit 前进且**距上次发布已满 7 天**，就从该 commit 编译 **Windows 桌面版** 与 **Windows 终端版**，
发布到本仓库的 GitHub Release，并把 commit 与时间写回 `version.json` 作为下一轮的比对基准。

## 文件

| 文件 | 作用 |
| --- | --- |
| [`.github/workflows/daily-v2-release.yml`](.github/workflows/daily-v2-release.yml) | 定时检查 + 构建 + 发布 + 同步 |
| [`version.json`](version.json) | 记录「上一次已构建的上游 commit」，由 Action 自动提交更新 |

仓库里**不需要**放 opencode 源码：workflow 直接 checkout 上游 `v2` 的对应 commit 来构建。

## 启用步骤

1. 把 `.github/workflows/daily-v2-release.yml` 和 `version.json` 提交到你自己的仓库默认分支。
2. **Settings → Actions → General → Workflow permissions** 选择 **Read and write permissions**
   （workflow 内已声明 `permissions: contents: write`，但组织级策略若限制为只读则必须在这里放开）。
3. 确认默认分支**没有开启**「禁止 github-actions 推送 / 必须 PR」的保护规则，
   否则最后一步 `sync-version` 提交 `version.json` 会失败（构建和发布本身不受影响）。
4. 想立刻验证，去 **Actions → daily-v2-release → Run workflow**，勾选 `force` 手动触发一次。

## 版本命名

两个串各有用途，都由上游 commit 决定：

| 用途 | 格式 | 例子 |
| --- | --- | --- |
| Release tag | `v2-<commit日期>-<短sha>` | `v2-20260905-7a4ad68` |
| 产物版本串（`OPENCODE_VERSION`，烧进二进制、写进 package.json） | `0.0.0-v2.<commit日期>.<短sha>` | `0.0.0-v2.20260905.7a4ad68` |

- 日期取**上游 commit 的 committer date**（UTC），不是构建时间 —— 这样同一个 commit 重跑得到同一个 tag，可复现。
- 产物版本串必须是合法 semver（electron-builder 会校验 NSIS 安装程序的版本号），
  所以不能直接用 tag 名，两者格式不同。
- 同一天对同一 commit 强制重跑时，tag 撞车会自动追加运行编号：`v2-20260905-7a4ad68.42`。
- 默认标记为 **prerelease**（周期性镜像不宜抢「Latest」位）。
  想让最新一次构建显示为 Latest，把 workflow 顶部的 `PRERELEASE` 改成 `"false"`。

## 工作流程

```
check ──► build-cli ──► build-desktop ──► release ──► sync-version
(每日 08:00 北京时间检查一次；满足条件才真正构建)
```

### 1. `check` — 判定是否需要构建

取上游 `v2` 的 HEAD commit SHA，与 `version.json` 里记录的比较，两道门都要过：

| 情况 | 结果 |
| --- | --- |
| 上游 SHA == 已记录 SHA | 跳过（上游无新提交） |
| 有新提交，但距 `version.json` 的 `updated_at` 不足 `MIN_INTERVAL_DAYS`（默认 7）天 | 跳过（节流） |
| 有新提交**且**已满 7 天 | 构建并发布 |
| 手动勾选 `force` | 无视上面两条，直接构建 |

保持 cron 每天跑、用「最小间隔」而不是「每周一固定跑」，效果是 Release 落在**满 7 天的那一天**，
而不是钉死在某个星期几；上游若连续 7 天没动，也不会发出一个内容与上次完全相同的 Release。

跳过时整条链显示为 skipped（不算失败），Job summary 会写明原因，含「距上次发布 N 天 / 最小间隔 7 天」。

上游 `packages/desktop/package.json` 与 `packages/cli/package.json` 的版本号也会被读取，
但**只写进 Release 正文做参考**，不参与判定 —— v2 分支上这两个号长期不同步
（当前 desktop `1.18.15` / cli `1.18.4`，而官方已发布到 `v1.18.29`），拿它们当触发条件会漏掉大量更新。

### 2. `build-cli` — 终端版

Linux runner 上用上游的 `packages/cli/script/build.ts` 交叉编译（与官方构建方式一致），
一次产出全部 12 个目标，只取 Windows 三个打包：

- `opencode-cli-windows-x64.zip`
- `opencode-cli-windows-x64-baseline.zip`（不支持 AVX2 的老 CPU）
- `opencode-cli-windows-arm64.zip`

产物命令名为 `opencode2`，解压后是 `bin/opencode2.exe`。

### 3. `build-desktop` — 桌面版

Windows runner 上按上游 `publish.yml` 的 electron 流程：

1. 把上一步的 CLI 产物放回 `packages/cli/dist`
2. `bun ./scripts/prepare.ts`（内嵌 `opencode-cli.exe`、复制 prod 图标、写入版本号）
3. `bun run build`（electron-vite）
4. `npx electron-builder --win --x64 --publish never`

产物：`opencode-desktop-windows-x64.exe`（NSIS 安装程序）+ `.blockmap`。

### 4. `release` — 发布

用内置 `GITHUB_TOKEN` 创建 Release，上传桌面版安装程序与三个终端版 zip。
正文用 GitHub compare API 列出**距上次构建之间的上游 commit**（首次构建则列最近 30 条）。

### 5. `sync-version` — 同步

把本轮的上游 commit、版本串、Release 链接写回 `version.json` 并提交推送。

## 可调项（都在 workflow 顶部的 `env`）

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `UPSTREAM_REPO` / `UPSTREAM_BRANCH` | `anomalyco/opencode` / `v2` | 上游来源 |
| `OPENCODE_CHANNEL` | `prod` | `prod` → 产品名 "OpenCode"、appId `ai.opencode.desktop`；改 `beta` → "OpenCode Beta"、appId `ai.opencode.desktop.beta` |
| `PRERELEASE` | `"true"` | 构建是否标记为预发布 |
| `MIN_INTERVAL_DAYS` | `"7"` | 距上次成功发布的最小间隔（天）；设 `0` 等于不节流，有新提交就发 |
| `BUN_COMPILE_RELEASE` | `bun-v1.4.2` | 编译进产物的 Bun 运行时版本，需与上游 `packageManager` 匹配 |
| `on.schedule.cron` | `0 0 * * *` | UTC 00:00 = 北京时间 08:00 |

## 已知限制

- **未签名。** 没有 Apple / Azure Trusted Signing 证书，Windows 安装程序没有 Authenticode 签名，
  SmartScreen 首次运行会弹警告。上游的 `script/sign-windows.ps1` 在缺少 Azure 相关环境变量时会自动跳过，
  所以无需改动源码。
- **桌面版自动更新指向官方仓库。** `prod` 通道的 electron-builder 配置里 `publish` 写死了
  `anomalyco/opencode`，因此应用内的自动更新会去查官方 Release 而不是本仓库。
  本 workflow 用 `--publish never` 且**不上传** `latest.yml`，不会污染官方更新流。
  介意这一点的话把 `OPENCODE_CHANNEL` 改成 `beta`。
- **只构建 Windows x64 桌面版。** 需要 macOS / Linux 或 Windows ARM64，
  参照上游 `publish.yml` 的 `build-electron` matrix 增加条目即可
  （macOS 还需加 `--config.mac.identity=null --config.mac.notarize=false` 才能免签名构建）。
- **单次构建约 20–40 分钟**，大头是 monorepo 的 `bun install` 和 CLI 的 12 目标交叉编译。
  上游用的是 blacksmith 加速 runner（`blacksmith-4vcpu-*`），本仓库改用官方 `ubuntu-latest` / `windows-latest`，
  公开仓库有免费额度；私有仓库按 7 天间隔算，每月约 4–5 轮 ≈ 100–200 分钟，
  另外每天一次的 `check` 只是几个 API 调用，几十秒，基本不消耗额度。
- **每个 Release 覆盖 7 天的上游变更**，不是「最新」构建。上游 `v2` 工作日通常有 5–15 个 commit，
  所以一轮 Release 的正文里一般会列出几十条变更。需要更及时就把 `MIN_INTERVAL_DAYS` 调小，
  设 `0` 即回到「每天有新提交就发一次」。

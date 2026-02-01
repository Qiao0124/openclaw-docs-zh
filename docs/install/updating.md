---
summary: "安全更新 OpenClaw（全局安装或源码）以及回滚策略"
read_when:
  - 更新 OpenClaw
  - 更新后出现问题
title: "更新（Updating）"
---

# 更新（Updating）

OpenClaw 仍在快速演进（“1.0” 前）。把更新当作发布基础设施：更新 → 检查 → 重启（或用 `openclaw update` 自动重启） → 验证。

## 推荐：重新运行官网安装器（原地升级）

**推荐**的更新路径是重新运行官网安装器。它会检测已有安装、原地升级，并在需要时运行 `openclaw doctor`。

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

说明：

- 若不想再次运行引导，添加 `--no-onboard`。
- 对于 **源码安装**，使用：
  ```bash
  curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --no-onboard
  ```
  仅当仓库干净时，安装器才会 `git pull --rebase`。
- 对于 **全局安装**，脚本底层会执行 `npm install -g openclaw@latest`。
- 兼容提示：`openclaw` 仍作为兼容层存在。

## 更新前（Before you update）

- 明确你的安装方式：**全局**（npm/pnpm）还是 **源码**（git clone）。
- 明确网关运行方式：**前台终端** 还是 **守护服务**（launchd/systemd）。
- 备份你的定制：
  - 配置：`~/.openclaw/openclaw.json`
  - 凭据：`~/.openclaw/credentials/`
  - 工作区：`~/.openclaw/workspace`

## 更新（全局安装）

全局更新（任选其一）：

```bash
npm i -g openclaw@latest
```

```bash
pnpm add -g openclaw@latest
```

**不建议**用 Bun 作为网关运行时（WhatsApp/Telegram 有问题）。

切换更新通道（git + npm 安装均适用）：

```bash
openclaw update --channel beta
openclaw update --channel dev
openclaw update --channel stable
```

可用 `--tag <dist-tag|version>` 指定一次性的 tag/版本。

通道语义与发布说明：见 [Development channels](/install/development-channels)。

注意：npm 安装时，网关启动会输出更新提示（检查当前通道 tag）。可通过 `update.checkOnStart: false` 关闭。

然后：

```bash
openclaw doctor
openclaw gateway restart
openclaw health
```

说明：

- 如果网关由服务管理，推荐 `openclaw gateway restart`，不要手动杀 PID。
- 如果你固定在某个版本，见下文“回滚 / 固定版本”。

## 更新（`openclaw update`）

对于 **源码安装**（git checkout），优先：

```bash
openclaw update
```

它执行一个相对安全的更新流程：

- 要求工作树干净。
- 切换到选定通道（tag 或分支）。
- 对配置的上游（dev 通道）执行 fetch + rebase。
- 安装依赖、构建、构建 Control UI，并运行 `openclaw doctor`。
- 默认重启网关（可用 `--no-restart` 跳过）。

如果你通过 **npm/pnpm** 安装（无 git 元数据），`openclaw update` 会尝试用包管理器更新；若无法识别安装方式，请使用“更新（全局安装）”。

## 更新（Control UI / RPC）

Control UI 提供 **Update & Restart**（RPC：`update.run`）：

1. 执行与 `openclaw update` 相同的源码更新流程（仅 git checkout）。
2. 写入重启标记并附带结构化报告（stdout/stderr 尾部）。
3. 重启网关，并将报告发送给最近活跃会话。

若 rebase 失败，网关会中止并在不应用更新的情况下重启。

## 更新（从源码）

在仓库里：

推荐：

```bash
openclaw update
```

手动（近似等价）：

```bash
git pull
pnpm install
pnpm build
pnpm ui:build # auto-installs UI deps on first run
openclaw doctor
openclaw health
```

说明：

- 当你运行打包后的 `openclaw` 二进制（[`openclaw.mjs`](https://github.com/openclaw/openclaw/blob/main/openclaw.mjs)）或用 Node 跑 `dist/` 时，`pnpm build` 很关键。
- 如果你在仓库里运行且没有全局安装，请使用 `pnpm openclaw ...` 执行 CLI。
- 若直接从 TypeScript 运行（`pnpm openclaw ...`），通常无需重建，但 **配置迁移仍需执行** → 运行 doctor。
- 在全局安装与 git 安装之间切换很容易：安装另一种方式，再运行 `openclaw doctor`，网关服务入口会被重写到当前安装。

## 务必运行：`openclaw doctor`

Doctor 是“安全更新”命令。它刻意保持朴素：修复 + 迁移 + 警告。

注意：如果你是 **源码安装**（git checkout），`openclaw doctor` 会先提示你运行 `openclaw update`。

典型操作包括：

- 迁移废弃配置键 / 老的配置文件路径。
- 审计私聊策略，对风险 “open” 设置发出警告。
- 检查网关健康，并可提示重启。
- 检测并迁移旧的网关服务（launchd/systemd；旧 schtasks）到当前 OpenClaw 服务。
- Linux 上确保 systemd user lingering（网关在登出后仍可运行）。

详情：[Doctor](/gateway/doctor)

## 启动 / 停止 / 重启网关（Gateway）

CLI（跨 OS 通用）：

```bash
openclaw gateway status
openclaw gateway stop
openclaw gateway restart
openclaw gateway --port 18789
openclaw logs --follow
```

如果由服务管理：

- macOS launchd（应用内置 LaunchAgent）：`launchctl kickstart -k gui/$UID/bot.molt.gateway`（用 `bot.molt.<profile>`；旧 `com.openclaw.*` 仍可用）
- Linux systemd 用户服务：`systemctl --user restart openclaw-gateway[-<profile>].service`
- Windows（WSL2）：`systemctl --user restart openclaw-gateway[-<profile>].service`
  - `launchctl`/`systemctl` 仅在已安装服务时可用，否则请运行 `openclaw gateway install`。

运行手册 + 精确服务名：[Gateway runbook](/gateway)

## 回滚 / 固定版本（出问题时）

### 固定版本（全局安装）

安装已知可用版本（用最后可用版本替换 `<version>`）：

```bash
npm i -g openclaw@<version>
```

```bash
pnpm add -g openclaw@<version>
```

提示：查看当前发布版本可运行 `npm view openclaw version`。

然后重启并运行 doctor：

```bash
openclaw doctor
openclaw gateway restart
```

### 按日期固定（源码）

选择某日期的 commit（示例：“2026-01-01 的 main 状态”）：

```bash
git fetch origin
git checkout "$(git rev-list -n 1 --before=\"2026-01-01\" origin/main)"
```

然后重新安装依赖并重启：

```bash
pnpm install
pnpm build
openclaw gateway restart
```

如需回到最新：

```bash
git checkout main
git pull
```

## 如果卡住了

- 再次运行 `openclaw doctor` 并仔细阅读输出（通常会给出修复提示）。
- 查看：[Troubleshooting](/gateway/troubleshooting)
- 在 Discord 提问：https://discord.gg/clawd

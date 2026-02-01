---
summary: "设置指南：保持个性化的同时持续更新"
read_when:
  - 设置一台新机器
  - 你想要“最新 + 最好”同时不破坏个人配置
title: "设置（Setup）"
---

# 设置（Setup）

最后更新：2026-01-01

## 摘要（TL;DR）

- **个性化配置放在仓库外：** `~/.openclaw/workspace`（工作区） + `~/.openclaw/openclaw.json`（配置）。
- **稳定工作流：** 安装 macOS 应用；让它运行内置网关。
- **前沿工作流：** 使用 `pnpm gateway:watch` 自行运行网关，然后让 macOS 应用以本地模式连接。

## 前置条件（从源码 / Prereqs）

- Node `>=22`
- `pnpm`
- Docker（可选；仅用于容器化/端到端 — 参见 [Docker](/install/docker)）

## 定制策略（避免更新伤害 / Tailoring strategy）

如果你想要“100% 贴合自己”且易于更新，请把自定义放在：

- **配置：** `~/.openclaw/openclaw.json`（JSON/JSON5 风格）
- **工作区：** `~/.openclaw/workspace`（技能、提示词、记忆；建议建成私有 git 仓库）

初始化一次：

```bash
openclaw setup
```

在本仓库内使用本地 CLI 入口：

```bash
openclaw setup
```

如果你还没有全局安装，请通过 `pnpm openclaw setup` 运行。

## 稳定工作流（macOS 应用优先 / Stable workflow）

1. 安装并启动 **OpenClaw.app**（菜单栏）。
2. 完成引导/权限清单（TCC 提示）。
3. 确保网关为 **Local** 且已运行（由应用管理）。
4. 连接渠道（示例：WhatsApp）：

```bash
openclaw channels login
```

5. 健康检查：

```bash
openclaw health
```

如果你的构建没有引导功能：

- 运行 `openclaw setup`，然后 `openclaw channels login`，最后手动启动网关（`openclaw gateway`）。

## 前沿工作流（终端运行网关 / Bleeding edge workflow）

目标：开发 TypeScript 网关，获得热重载，同时保持 macOS 应用 UI 连接。

### 0) （可选）macOS 应用也从源码运行

如果你也想让 macOS 应用使用前沿版本：

```bash
./scripts/restart-mac.sh
```

### 1) 启动开发网关

```bash
pnpm install
pnpm gateway:watch
```

`gateway:watch` 会以监听模式运行网关，并在 TypeScript 变化时自动重载。

### 2) 让 macOS 应用连接到你运行中的网关

在 **OpenClaw.app** 中：

- Connection Mode：**Local**
  应用会连接到配置端口上的运行中网关。

### 3) 验证

- 应用内网关状态应显示 **“Using existing gateway …”**
- 或通过 CLI：

```bash
openclaw health
```

### 常见坑（Common footguns）

- **端口不一致：** 网关 WS 默认 `ws://127.0.0.1:18789`；应用与 CLI 要使用同一端口。
- **状态位置：**
  - 凭据：`~/.openclaw/credentials/`
  - 会话：`~/.openclaw/agents/<agentId>/sessions/`
  - 日志：`/tmp/openclaw/`

## 凭据存储映射（Credential storage map）

调试认证或决定备份范围时使用：

- **WhatsApp**：`~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **Telegram 机器人令牌**：config/env 或 `channels.telegram.tokenFile`
- **Discord 机器人令牌**：config/env（暂不支持令牌文件）
- **Slack tokens**：config/env（`channels.slack.*`）
- **配对允许列表**：`~/.openclaw/credentials/<channel>-allowFrom.json`
- **模型认证配置**：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **旧版 OAuth 导入**：`~/.openclaw/credentials/oauth.json`
  更多细节：[Security](/gateway/security#credential-storage-map)。

## 更新（不破坏配置 / Updating）

- 将 `~/.openclaw/workspace` 与 `~/.openclaw/` 视为“你的内容”；不要把个人提示词/配置放在 `openclaw` 仓库中。
- 更新源码：`git pull` + `pnpm install`（当 lockfile 有变化时）+ 继续使用 `pnpm gateway:watch`。

## Linux（systemd 用户服务 / systemd user service）

Linux 安装使用 systemd **用户**服务。默认情况下，systemd 会在注销/空闲时停止用户服务，从而终止网关。
引导会尝试为你开启 lingering（可能提示 sudo）。若仍未开启，请运行：

```bash
sudo loginctl enable-linger $USER
```

对需要常驻或多用户的服务器，考虑使用 **系统**服务替代用户服务（无需 lingering）。
关于 systemd 说明见 [Gateway runbook](/gateway)。

## 相关文档（Related docs）

- [Gateway runbook](/gateway)（参数、守护、端口）
- [Gateway configuration](/gateway/configuration)（配置 schema + 示例）
- [Discord](/channels/discord) 与 [Telegram](/channels/telegram)（回复标签 + replyToMode 设置）
- [OpenClaw assistant setup](/start/openclaw)
- [macOS app](/platforms/macos)（网关生命周期）

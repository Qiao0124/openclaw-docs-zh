---
summary: "运行 OpenClaw 作为个人助手的端到端指南（含安全提示）"
read_when:
  - 引导一个新的助手实例
  - 回顾安全与权限影响
title: "个人助手配置（Personal Assistant Setup）"
---

# 使用 OpenClaw 构建个人助手（Building a personal assistant with OpenClaw）

OpenClaw 是面向 **Pi** 代理的 WhatsApp + Telegram + Discord + iMessage 网关。插件可添加 Mattermost。本指南是“个人助手”配置：一个专用 WhatsApp 号码，表现为你的常驻代理。

## ⚠️ 安全第一（Safety first）

你正在让代理拥有以下能力：

- 在你的机器上执行命令（取决于你的 Pi 工具配置）
- 读写你的工作区文件
- 通过 WhatsApp/Telegram/Discord/Mattermost（插件）向外发送消息

请从保守设置开始：

- 始终设置 `channels.whatsapp.allowFrom`（不要在个人 Mac 上开放给所有人）。
- 为助手使用专用的 WhatsApp 号码。
- 心跳默认每 30 分钟一次。先将 `agents.defaults.heartbeat.every: "0m"` 设为禁用，待信任后再开启。

## 前置条件（Prerequisites）

- Node **22+**
- OpenClaw 已在 PATH 中（推荐全局安装）
- 为助手准备第二个手机号（SIM/eSIM/预付费）

```bash
npm install -g openclaw@latest
# or: pnpm add -g openclaw@latest
```

从源码（开发）：

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build # auto-installs UI deps on first run
pnpm build
pnpm link --global
```

## 双手机方案（推荐 / The two-phone setup）

你期望的结构：

```
Your Phone (personal)          Second Phone (assistant)
┌─────────────────┐           ┌─────────────────┐
│  Your WhatsApp  │  ──────▶  │  Assistant WA   │
│  +1-555-YOU     │  message  │  +1-555-ASSIST  │
└─────────────────┘           └────────┬────────┘
                                       │ linked via QR
                                       ▼
                              ┌─────────────────┐
                              │  Your Mac       │
                              │  (openclaw)      │
                              │    Pi agent     │
                              └─────────────────┘
```

如果把你的个人 WhatsApp 连接到 OpenClaw，每条发给你的消息都会变成“代理输入”。这通常并非你想要的。

## 5 分钟快速开始（5-minute quick start）

1. 配对 WhatsApp Web（显示二维码；用助手手机扫码）：

```bash
openclaw channels login
```

2. 启动网关（保持运行）：

```bash
openclaw gateway --port 18789
```

3. 在 `~/.openclaw/openclaw.json` 中放入最小配置：

```json5
{
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

现在用已允许的手机给助手号码发送消息。

引导完成后，我们会自动打开带网关令牌的 Dashboard，并打印带令牌的链接。之后可用：`openclaw dashboard`。

## 为代理配置工作区（AGENTS）

OpenClaw 会从工作区目录读取运行说明与“记忆”。

默认情况下，OpenClaw 使用 `~/.openclaw/workspace` 作为代理工作区，并在设置/首次运行时自动创建（含 `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`）。`BOOTSTRAP.md` 只会在全新工作区时创建（删除后不会再出现）。

提示：把该文件夹当作 OpenClaw 的“记忆”，并建成 git 仓库（最好私有），这样你的 `AGENTS.md` + 记忆文件会被备份。如果已安装 git，新工作区会自动初始化。

```bash
openclaw setup
```

完整工作区布局 + 备份指南：[Agent workspace](/concepts/agent-workspace)
记忆工作流：[Memory](/concepts/memory)

可选：用 `agents.defaults.workspace` 指定不同工作区（支持 `~`）。

```json5
{
  agent: {
    workspace: "~/.openclaw/workspace",
  },
}
```

如果你已经从仓库分发自己的工作区文件，可以完全禁用引导文件创建：

```json5
{
  agent: {
    skipBootstrap: true,
  },
}
```

## 将其变成“助手”的配置（The config that turns it into “an assistant”）

OpenClaw 默认已是不错的助手配置，但你通常会调整：

- `SOUL.md` 中的角色/指令
- 思考默认值（如需要）
- 心跳（等你信任后再开启）

示例：

```json5
{
  logging: { level: "info" },
  agent: {
    model: "anthropic/claude-opus-4-5",
    workspace: "~/.openclaw/workspace",
    thinkingDefault: "high",
    timeoutSeconds: 1800,
    // Start with 0; enable later.
    heartbeat: { every: "0m" },
  },
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: {
        "*": { requireMention: true },
      },
    },
  },
  routing: {
    groupChat: {
      mentionPatterns: ["@openclaw", "openclaw"],
    },
  },
  session: {
    scope: "per-sender",
    resetTriggers: ["/new", "/reset"],
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 10080,
    },
  },
}
```

## 会话与记忆（Sessions and memory）

- 会话文件：`~/.openclaw/agents/<agentId>/sessions/{{SessionId}}.jsonl`
- 会话元数据（token 使用、上次路由等）：`~/.openclaw/agents/<agentId>/sessions/sessions.json`（旧版：`~/.openclaw/sessions/sessions.json`）
- `/new` 或 `/reset` 会为该聊天开启新会话（通过 `resetTriggers` 可配置）。若单独发送，代理会回复一个简短问候确认重置。
- `/compact [instructions]` 会压缩会话上下文并报告剩余上下文预算。

## 心跳（主动模式 / Heartbeats）

默认情况下，OpenClaw 每 30 分钟运行一次心跳，提示词为：
`Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
设定 `agents.defaults.heartbeat.every: "0m"` 可禁用。

- 若 `HEARTBEAT.md` 存在但基本为空（仅空行或 `# Heading` 等 markdown 标题），OpenClaw 会跳过心跳以节省 API 调用。
- 若文件缺失，心跳仍会运行并由模型决定操作。
- 若代理回复 `HEARTBEAT_OK`（可选短填充；见 `agents.defaults.heartbeat.ackMaxChars`），OpenClaw 会抑制该次心跳的外发消息。
- 心跳是完整代理回合 —— 间隔越短，消耗的 token 越多。

```json5
{
  agent: {
    heartbeat: { every: "30m" },
  },
}
```

## 媒体输入输出（Media in and out）

入站附件（图片/音频/文档）可通过模板字段提供给命令：

- `{{MediaPath}}`（本地临时文件路径）
- `{{MediaUrl}}`（伪 URL）
- `{{Transcript}}`（若启用音频转写）

代理的出站附件：在独立一行写 `MEDIA:<path-or-url>`（不含空格）。示例：

```
Here’s the screenshot.
MEDIA:https://example.com/screenshot.png
```

OpenClaw 会提取这些并与文本一起发送为媒体。

## 运维清单（Operations checklist）

```bash
openclaw status          # local status (creds, sessions, queued events)
openclaw status --all    # full diagnosis (read-only, pasteable)
openclaw status --deep   # adds gateway health probes (Telegram + Discord)
openclaw health --json   # gateway health snapshot (WS)
```

日志位于 `/tmp/openclaw/`（默认：`openclaw-YYYY-MM-DD.log`）。

## 下一步（Next steps）

- WebChat：[WebChat](/web/webchat)
- 网关运维：[Gateway runbook](/gateway)
- Cron + 唤醒：[Cron jobs](/automation/cron-jobs)
- macOS 菜单栏伴随应用：[OpenClaw macOS app](/platforms/macos)
- iOS 节点应用：[iOS app](/platforms/ios)
- Android 节点应用：[Android app](/platforms/android)
- Windows 状态：[Windows (WSL2)](/platforms/windows)
- Linux 状态：[Linux app](/platforms/linux)
- 安全：[Security](/gateway/security)

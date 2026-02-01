---
summary: "OpenClaw 的整体概览、核心能力与用途"
read_when:
  - 向新同学介绍 OpenClaw
title: "OpenClaw"
---

# OpenClaw 🦞

> _"蜕壳！蜕壳！"_ — 一只太空龙虾，可能吧

<p align="center">
    <img
        src="/assets/openclaw-logo-text-dark.png"
        alt="OpenClaw"
        width="500"
        class="dark:hidden"
    />
    <img
        src="/assets/openclaw-logo-text.png"
        alt="OpenClaw"
        width="500"
        class="hidden dark:block"
    />
</p>

<p align="center">
  <strong>任意系统 + WhatsApp/Telegram/Discord/iMessage 的 AI 代理网关（Pi）。</strong><br />
  插件可添加 Mattermost 等渠道。
  发一条消息，口袋里就能得到代理回复。
</p>

<p align="center">
  <a href="https://github.com/openclaw/openclaw">GitHub</a> ·
  <a href="https://github.com/openclaw/openclaw/releases">版本</a> ·
  <a href="/">文档</a> ·
  <a href="/start/openclaw">OpenClaw 助手设置</a>
</p>

OpenClaw 连接 WhatsApp（通过 WhatsApp Web / Baileys）、Telegram（Bot API / grammY）、Discord（Bot API / channels.discord.js）以及 iMessage（imsg CLI），把它们桥接到像 [Pi](https://github.com/badlogic/pi-mono) 这样的编程代理。插件还可添加 Mattermost（Bot API + WebSocket）等更多渠道。OpenClaw 也驱动 OpenClaw Assistant。

## 从这里开始

- **全新安装从零开始：** [入门](/start/getting-started)
- **引导式设置（推荐）：** [向导](/start/wizard) (`openclaw onboard`)
- **打开仪表盘（本地网关）：** http://127.0.0.1:18789/（或 http://localhost:18789/）

如果网关在同一台电脑上运行，上面的链接会直接打开浏览器控制台。如果无法访问，先启动网关：`openclaw gateway`。

## 仪表盘（浏览器控制台）

仪表盘是浏览器中的控制台，用于聊天、配置、节点、会话等。
本地默认地址：http://127.0.0.1:18789/
远程访问：[Web 界面](/web) 与 [Tailscale](/gateway/tailscale)

<p align="center">
  <img src="whatsapp-openclaw.jpg" alt="OpenClaw" width="420" />
</p>

## 工作原理

```
WhatsApp / Telegram / Discord / iMessage (+ plugins)
        │
        ▼
  ┌───────────────────────────┐
  │          Gateway          │  ws://127.0.0.1:18789 (loopback-only)
  │     (single source)       │
  │                           │  http://<gateway-host>:18793
  │                           │    /__openclaw__/canvas/ (Canvas host)
  └───────────┬───────────────┘
              │
              ├─ Pi agent (RPC)
              ├─ CLI (openclaw …)
              ├─ Chat UI (SwiftUI)
              ├─ macOS app (OpenClaw.app)
              ├─ iOS node via Gateway WS + pairing
              └─ Android node via Gateway WS + pairing
```

大多数操作都通过 **网关**（`openclaw gateway`）流转。网关是一个长期运行的进程，负责渠道连接与 WebSocket 控制平面。

## 网络模型

- **每台主机一个网关（推荐）**：它是唯一允许持有 WhatsApp Web 会话的进程。如需救援机器人或严格隔离，可运行多个网关并使用独立配置与端口；参见 [多网关](/gateway/multiple-gateways)。
- **优先回环地址**：网关 WS 默认 `ws://127.0.0.1:18789`。
  - 向导现在默认生成网关令牌（即使是回环地址）。
  - Tailnet 访问时运行 `openclaw gateway --bind tailnet --token ...`（非回环绑定必须带 token）。
- **节点**：通过网关 WebSocket 连接（按需使用 LAN/tailnet/SSH）；旧的 TCP bridge 已弃用/移除。
- **Canvas host**：HTTP 文件服务位于 `canvasHost.port`（默认 `18793`），提供 `/__openclaw__/canvas/` 供节点 WebView 使用；参见 [网关配置](/gateway/configuration)（`canvasHost`）。
- **远程使用**：SSH 隧道或 tailnet/VPN；参见 [远程访问](/gateway/remote) 与 [发现](/gateway/discovery)。

## 主要特性（概览）

- 📱 **WhatsApp 集成** — 使用 Baileys 处理 WhatsApp Web 协议
- ✈️ **Telegram 机器人** — grammY 支持私聊与群聊
- 🎮 **Discord 机器人** — channels.discord.js 支持私聊与服务器频道
- 🧩 **Mattermost 机器人（插件）** — Bot Token + WebSocket 事件
- 💬 **iMessage** — macOS 上的本地 imsg CLI 集成
- 🤖 **代理桥接** — Pi（RPC 模式）+ 工具流式输出
- ⏱️ **流式输出 + 分块** — Block streaming + Telegram 草稿流细节（[/concepts/streaming](/concepts/streaming)）
- 🧠 **多代理路由** — 将模型账号/同伴路由到隔离的代理（workspace + per-agent sessions）
- 🔐 **订阅授权** — Anthropic（Claude Pro/Max）+ OpenAI（ChatGPT/Codex）OAuth
- 💬 **会话** — 私聊默认合并到 `main`；群聊独立
- 👥 **群聊支持** — 默认基于@提及；群主可切换 `/activation always|mention`
- 📎 **媒体支持** — 发送与接收图片、音频、文档
- 🎤 **语音消息** — 可选转写 hook
- 🖥️ **WebChat + macOS 应用** — 本地 UI + 菜单栏伴侣，支持运维与语音唤醒
- 📱 **iOS 节点** — 作为节点配对并提供 Canvas 界面
- 📱 **Android 节点** — 作为节点配对并提供 Canvas + Chat + Camera

注意：旧的 Claude/Codex/Gemini/Opencode 路径已移除；Pi 是唯一的编程代理路径。

## 快速开始

运行环境要求：**Node ≥ 22**。

```bash
# Recommended: global install (npm/pnpm)
npm install -g openclaw@latest
# or: pnpm add -g openclaw@latest

# Onboard + install the service (launchd/systemd user service)
openclaw onboard --install-daemon

# Pair WhatsApp Web (shows QR)
openclaw channels login

# Gateway runs via the service after onboarding; manual run is still possible:
openclaw gateway --port 18789
```

后续在 npm 与 git 安装之间切换也很容易：安装另一种方式并运行 `openclaw doctor` 更新网关服务入口。

从源码（开发模式）：

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build # auto-installs UI deps on first run
pnpm build
openclaw onboard --install-daemon
```

如果尚未全局安装，可在仓库内通过 `pnpm openclaw ...` 运行引导步骤。

多实例快速开始（可选）：

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

发送测试消息（需要网关已在运行）：

```bash
openclaw message send --target +15555550123 --message "Hello from OpenClaw"
```

## 配置（可选）

配置文件位于 `~/.openclaw/openclaw.json`。

- 如果你 **什么都不做**，OpenClaw 会在 RPC 模式下使用内置的 Pi 二进制，并为每个发送者维护会话。
- 如果你想收紧权限，从 `channels.whatsapp.allowFrom` 与（群聊）提及规则开始。

示例：

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  messages: { groupChat: { mentionPatterns: ["@openclaw"] } },
}
```

## 文档

- 从这里开始：
  - [文档索引（全链接）](/start/hubs)
  - [帮助](/help) ← _常见修复 + 故障排查_
  - [配置](/gateway/configuration)
  - [配置示例](/gateway/configuration-examples)
  - [Slash 命令](/tools/slash-commands)
  - [多代理路由](/concepts/multi-agent)
  - [更新 / 回滚](/install/updating)
  - [配对（私聊 + 节点）](/start/pairing)
  - [Nix 模式](/install/nix)
  - [OpenClaw 助手设置](/start/openclaw)
  - [技能](/tools/skills)
  - [技能配置](/tools/skills-config)
  - [Workspace 模板](/reference/templates/AGENTS)
  - [RPC 适配器](/reference/rpc)
  - [网关运行手册](/gateway)
  - [节点（iOS/Android）](/nodes)
  - [Web 界面（Control UI）](/web)
  - [发现与传输](/gateway/discovery)
  - [远程访问](/gateway/remote)
- 渠道与体验：
  - [WebChat](/web/webchat)
  - [Control UI（浏览器）](/web/control-ui)
  - [Telegram](/channels/telegram)
  - [Discord](/channels/discord)
  - [Mattermost（插件）](/channels/mattermost)
  - [iMessage](/channels/imessage)
  - [群组](/concepts/groups)
  - [WhatsApp 群消息](/concepts/group-messages)
  - [媒体：图片](/nodes/images)
  - [媒体：音频](/nodes/audio)
- 配套应用：
  - [macOS 应用](/platforms/macos)
  - [iOS 应用](/platforms/ios)
  - [Android 应用](/platforms/android)
  - [Windows（WSL2）](/platforms/windows)
  - [Linux 应用](/platforms/linux)
- 运维与安全：
  - [会话](/concepts/session)
  - [Cron 任务](/automation/cron-jobs)
  - [Webhooks](/automation/webhook)
  - [Gmail hooks（Pub/Sub）](/automation/gmail-pubsub)
  - [安全](/gateway/security)
  - [故障排查](/gateway/troubleshooting)

## 名称由来

**OpenClaw = CLAW + TARDIS** —— 因为每只太空龙虾都需要一台时空机器。

---

_"我们都只是在玩自己的提示词。"_ — 一位可能在高 token 状态的 AI

## 致谢

- **Peter Steinberger** ([@steipete](https://x.com/steipete)) — 创作者，龙虾低语者
- **Mario Zechner** ([@badlogicc](https://x.com/badlogicgames)) — Pi 创建者，安全渗透测试者
- **Clawd** — 要求更好名字的太空龙虾

## 核心贡献者

- **Maxim Vovshin** (@Hyaxia, 36747317+Hyaxia@users.noreply.github.com) — Blogwatcher 技能
- **Nacho Iacovino** (@nachoiacovino, nacho.iacovino@gmail.com) — 位置解析（Telegram + WhatsApp）

## 许可证

MIT — 像海里的龙虾一样自由 🦞

---

_"我们都只是在玩自己的提示词。"_ — 一位 AI，可能在高 token 状态

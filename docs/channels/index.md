---
summary: "OpenClaw 可以连接的即时通讯平台"
read_when:
  - 你想为 OpenClaw 选择一个聊天频道
  - 你需要支持的即时通讯平台的快速概览
title: "聊天频道"
---

# 聊天频道

OpenClaw 可以在你已经在使用的任何聊天应用上与你对话。每个频道通过网关连接。
所有频道都支持文本；媒体和反应因频道而异。

## 支持的频道

- [WhatsApp](/channels/whatsapp) — 最受欢迎；使用 Baileys 并需要 QR 配对。
- [Telegram](/channels/telegram) — 通过 grammY 的 Bot API；支持群组。
- [Discord](/channels/discord) — Discord Bot API + Gateway；支持服务器、频道和 DM。
- [Slack](/channels/slack) — Bolt SDK；工作空间应用。
- [Google Chat](/channels/googlechat) — 通过 HTTP webhook 的 Google Chat API 应用。
- [Mattermost](/channels/mattermost) — Bot API + WebSocket；频道、群组、DM（插件，单独安装）。
- [Signal](/channels/signal) — signal-cli；注重隐私。
- [BlueBubbles](/channels/bluebubbles) — **推荐用于 iMessage**；使用 BlueBubbles macOS 服务器 REST API，完整功能支持（编辑、取消发送、效果、反应、群组管理 — 编辑目前在 macOS 26 Tahoe 上损坏）。
- [iMessage](/channels/imessage) — 仅 macOS；通过 imsg 的原生集成（传统，新设置请考虑 BlueBubbles）。
- [Microsoft Teams](/channels/msteams) — Bot Framework；企业支持（插件，单独安装）。
- [LINE](/channels/line) — LINE Messaging API 机器人（插件，单独安装）。
- [Nextcloud Talk](/channels/nextcloud-talk) — 通过 Nextcloud Talk 的自托管聊天（插件，单独安装）。
- [Matrix](/channels/matrix) — Matrix 协议（插件，单独安装）。
- [Nostr](/channels/nostr) — 通过 NIP-04 的去中心化 DM（插件，单独安装）。
- [Tlon](/channels/tlon) — 基于 Urbit 的通讯工具（插件，单独安装）。
- [Twitch](/channels/twitch) — 通过 IRC 连接的 Twitch 聊天（插件，单独安装）。
- [Zalo](/channels/zalo) — Zalo Bot API；越南流行的通讯工具（插件，单独安装）。
- [Zalo Personal](/channels/zalouser) — 通过 QR 登录的 Zalo 个人账户（插件，单独安装）。
- [WebChat](/web/webchat) — 通过 WebSocket 的网关 WebChat UI。

## 说明

- 频道可以同时运行；配置多个，OpenClaw 将按聊天路由。
- 最快的设置通常是 **Telegram**（简单的机器人令牌）。WhatsApp 需要 QR 配对并在磁盘上存储更多状态。
- 群组行为因频道而异；参见 [群组](/concepts/groups)。
- DM 配对和 allowlist 为安全而强制执行；参见 [安全](/gateway/security)。
- Telegram 内部机制：[grammY 说明](/channels/grammy)。
- 故障排除：[频道故障排除](/channels/troubleshooting)。
- 模型提供商单独记录；参见 [模型提供商](/providers/models)。

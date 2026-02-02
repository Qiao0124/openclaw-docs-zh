---
summary: "Slack 的 Socket 模式或 HTTP Webhook 模式设置"
read_when: "设置 Slack 或调试 Slack Socket/HTTP 模式"
title: "Slack"
---

# Slack

## Socket 模式（默认）

### 快速设置（初学者）

1. 创建一个 Slack 应用并启用 **Socket Mode**。
2. 创建一个 **App Token** (`xapp-...`) 和 **Bot Token** (`xoxb-...`)。
3. 为 OpenClaw 设置令牌并启动网关。

最小配置：

```json5
{
  channels: {
    slack: {
      enabled: true,
      appToken: "xapp-...",
      botToken: "xoxb-...",
    },
  },
}
```

### 设置

1. 在 https://api.slack.com/apps 创建一个 Slack 应用（从头开始）。
2. **Socket Mode** → 开启。然后进入 **Basic Information** → **App-Level Tokens** → **Generate Token and Scopes**，选择范围 `connections:write`。复制 **App Token** (`xapp-...`)。
3. **OAuth & Permissions** → 添加 bot token 范围（使用下面的 manifest）。点击 **Install to Workspace**。复制 **Bot User OAuth Token** (`xoxb-...`)。
4. 可选：**OAuth & Permissions** → 添加 **User Token Scopes**（参见下面的只读列表）。重新安装应用并复制 **User OAuth Token** (`xoxp-...`)。
5. **Event Subscriptions** → 启用事件并订阅：
   - `message.*`（包括编辑/删除/线程广播）
   - `app_mention`
   - `reaction_added`, `reaction_removed`
   - `member_joined_channel`, `member_left_channel`
   - `channel_rename`
   - `pin_added`, `pin_removed`
6. 邀请机器人加入你希望它读取的频道。
7. Slash Commands → 如果你使用 `channels.slack.slashCommand`，创建 `/openclaw`。如果你启用原生命令，为每个内置命令添加一个 slash 命令（名称与 `/help` 相同）。原生命令默认对 Slack 关闭，除非你设置 `channels.slack.commands.native: true`（全局 `commands.native` 是 `"auto"`，这会使 Slack 保持关闭）。
8. App Home → 启用 **Messages Tab**，以便用户可以 DM 机器人。

使用下面的 manifest 以确保范围和事件保持同步。

多账户支持：使用 `channels.slack.accounts`，包含每个账户的令牌和可选的 `name`。参见 [`gateway/configuration`](/gateway/configuration#telegramaccounts--discordaccounts--slackaccounts--signalaccounts--imessageaccounts) 了解共享模式。

### OpenClaw 配置（最小）

通过环境变量设置令牌（推荐）：

- `SLACK_APP_TOKEN=xapp-...`
- `SLACK_BOT_TOKEN=xoxb-...`

或通过配置：

```json5
{
  channels: {
    slack: {
      enabled: true,
      appToken: "xapp-...",
      botToken: "xoxb-...",
    },
  },
}
```

### User Token（可选）

OpenClaw 可以使用 Slack user token (`xoxp-...`) 进行读取操作（历史记录、置顶、反应、表情符号、成员信息）。默认情况下这是只读的：读取优先使用 user token（如果存在），写入仍然使用 bot token，除非你明确选择加入。即使设置了 `userTokenReadOnly: false`，当 bot token 可用时，它仍然优先用于写入。

User tokens 在配置文件中配置（不支持环境变量）。对于多账户，设置 `channels.slack.accounts.<id>.userToken`。

使用 bot + app + user tokens 的示例：

```json5
{
  channels: {
    slack: {
      enabled: true,
      appToken: "xapp-...",
      botToken: "xoxb-...",
      userToken: "xoxp-...",
    },
  },
}
```

显式设置 userTokenReadOnly 的示例（允许 user token 写入）：

```json5
{
  channels: {
    slack: {
      enabled: true,
      appToken: "xapp-...",
      botToken: "xoxb-...",
      userToken: "xoxp-...",
      userTokenReadOnly: false,
    },
  },
}
```

#### Token 使用

- 读取操作（历史记录、反应列表、置顶列表、表情符号列表、成员信息、搜索）优先使用配置的 user token，否则使用 bot token。
- 写入操作（发送/编辑/删除消息、添加/删除反应、置顶/取消置顶、文件上传）默认使用 bot token。如果 `userTokenReadOnly: false` 且没有可用的 bot token，OpenClaw 会回退到 user token。

### 历史上下文

- `channels.slack.historyLimit`（或 `channels.slack.accounts.*.historyLimit`）控制有多少最近的频道/群组消息被包装到 prompt 中。
- 回退到 `messages.groupChat.historyLimit`。设置为 `0` 以禁用（默认 50）。

## HTTP 模式（Events API）

当你的网关可以通过 HTTPS 被 Slack 访问时（典型的服务器部署），使用 HTTP webhook 模式。HTTP 模式使用 Events API + Interactivity + Slash Commands，共享一个请求 URL。

### 设置

1. 创建一个 Slack 应用并**禁用 Socket Mode**（如果你只使用 HTTP，这是可选的）。
2. **Basic Information** → 复制 **Signing Secret**。
3. **OAuth & Permissions** → 安装应用并复制 **Bot User OAuth Token** (`xoxb-...`)。
4. **Event Subscriptions** → 启用事件并将 **Request URL** 设置为你的网关 webhook 路径（默认 `/slack/events`）。
5. **Interactivity & Shortcuts** → 启用并设置相同的 **Request URL**。
6. **Slash Commands** → 为你的命令设置相同的 **Request URL**。

请求 URL 示例：
`https://gateway-host/slack/events`

### OpenClaw 配置（最小）

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: "xoxb-...",
      signingSecret: "your-signing-secret",
      webhookPath: "/slack/events",
    },
  },
}
```

多账户 HTTP 模式：设置 `channels.slack.accounts.<id>.mode = "http"` 并为每个账户提供一个唯一的 `webhookPath`，以便每个 Slack 应用可以指向自己的 URL。

### Manifest（可选）

使用这个 Slack 应用 manifest 快速创建应用（根据需要调整名称/命令）。如果你计划配置 user token，请包含 user 范围。

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw",
      "always_online": false
    },
    "app_home": {
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Send a message to OpenClaw",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "chat:write",
        "channels:history",
        "channels:read",
        "groups:history",
        "groups:read",
        "groups:write",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "users:read",
        "app_mentions:read",
        "reactions:read",
        "reactions:write",
        "pins:read",
        "pins:write",
        "emoji:read",
        "commands",
        "files:read",
        "files:write"
      ],
      "user": [
        "channels:history",
        "channels:read",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "users:read",
        "reactions:read",
        "pins:read",
        "emoji:read",
        "search:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "reaction_added",
        "reaction_removed",
        "member_joined_channel",
        "member_left_channel",
        "channel_rename",
        "pin_added",
        "pin_removed"
      ]
    }
  }
}
```

如果你启用原生命令，为每个你想暴露的命令添加一个 `slash_commands` 条目（匹配 `/help` 列表）。使用 `channels.slack.commands.native` 覆盖。

## Scopes（当前 vs 可选）

Slack 的 Conversations API 是按类型划分范围的：你只需要为你实际接触的对话类型（channels、groups、im、mpim）设置范围。概述参见 https://docs.slack.dev/apis/web-api/using-the-conversations-api/。

### Bot token scopes（必需）

- `chat:write`（通过 `chat.postMessage` 发送/更新/删除消息）
  https://docs.slack.dev/reference/methods/chat.postMessage
- `im:write`（通过 `conversations.open` 打开 DM 以进行用户 DM）
  https://docs.slack.dev/reference/methods/conversations.open
- `channels:history`, `groups:history`, `im:history`, `mpim:history`
  https://docs.slack.dev/reference/methods/conversations.history
- `channels:read`, `groups:read`, `im:read`, `mpim:read`
  https://docs.slack.dev/reference/methods/conversations.info
- `users:read`（用户查找）
  https://docs.slack.dev/reference/methods/users.info
- `reactions:read`, `reactions:write`（`reactions.get` / `reactions.add`）
  https://docs.slack.dev/reference/methods/reactions.get
  https://docs.slack.dev/reference/methods/reactions.add
- `pins:read`, `pins:write`（`pins.list` / `pins.add` / `pins.remove`）
  https://docs.slack.dev/reference/scopes/pins.read
  https://docs.slack.dev/reference/scopes/pins.write
- `emoji:read`（`emoji.list`）
  https://docs.slack.dev/reference/scopes/emoji.read
- `files:write`（通过 `files.uploadV2` 上传）
  https://docs.slack.dev/messaging/working-with-files/#upload

### User token scopes（可选，默认只读）

如果你配置 `channels.slack.userToken`，在 **User Token Scopes** 下添加这些。

- `channels:history`, `groups:history`, `im:history`, `mpim:history`
- `channels:read`, `groups:read`, `im:read`, `mpim:read`
- `users:read`
- `reactions:read`
- `pins:read`
- `emoji:read`
- `search:read`

### 目前不需要（但可能未来需要）

- `mpim:write`（仅当我们添加 group-DM 打开/DM 启动通过 `conversations.open`）
- `groups:write`（仅当我们添加私有频道管理：创建/重命名/邀请/归档）
- `chat:write.public`（仅当我们想发布到机器人不在的频道）
  https://docs.slack.dev/reference/scopes/chat.write.public
- `users:read.email`（仅当我们需要从 `users.info` 获取 email 字段）
  https://docs.slack.dev/changelog/2017-04-narrowing-email-access
- `files:read`（仅当我们开始列出/读取文件元数据）

## 配置

Slack 仅使用 Socket 模式（无 HTTP webhook 服务器）。提供两个令牌：

```json
{
  "slack": {
    "enabled": true,
    "botToken": "xoxb-...",
    "appToken": "xapp-...",
    "groupPolicy": "allowlist",
    "dm": {
      "enabled": true,
      "policy": "pairing",
      "allowFrom": ["U123", "U456", "*"],
      "groupEnabled": false,
      "groupChannels": ["G123"],
      "replyToMode": "all"
    },
    "channels": {
      "C123": { "allow": true, "requireMention": true },
      "#general": {
        "allow": true,
        "requireMention": true,
        "users": ["U123"],
        "skills": ["search", "docs"],
        "systemPrompt": "Keep answers short."
      }
    },
    "reactionNotifications": "own",
    "reactionAllowlist": ["U123"],
    "replyToMode": "off",
    "actions": {
      "reactions": true,
      "messages": true,
      "pins": true,
      "memberInfo": true,
      "emojiList": true
    },
    "slashCommand": {
      "enabled": true,
      "name": "openclaw",
      "sessionPrefix": "slack:slash",
      "ephemeral": true
    },
    "textChunkLimit": 4000,
    "mediaMaxMb": 20
  }
}
```

令牌也可以通过环境变量提供：

- `SLACK_BOT_TOKEN`
- `SLACK_APP_TOKEN`

确认反应通过 `messages.ackReaction` + `messages.ackReactionScope` 全局控制。使用 `messages.removeAckAfterReply` 在机器人回复后清除确认反应。

## 限制

- 出站文本被分块到 `channels.slack.textChunkLimit`（默认 4000）。
- 可选的换行分块：设置 `channels.slack.chunkMode="newline"` 以在空行（段落边界）处分割，然后进行长度分块。
- 媒体上传受 `channels.slack.mediaMaxMb` 限制（默认 20）。

## 回复线程

默认情况下，OpenClaw 在主频道中回复。使用 `channels.slack.replyToMode` 控制自动线程：

| 模式    | 行为                                                                                                                                                            |
| ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `off`   | **默认。** 在主频道中回复。仅当触发消息已经在线程中时才使用线程。                                                                                                  |
| `first` | 第一次回复进入线程（在触发消息下），后续回复进入主频道。有助于在避免线程混乱的同时保持上下文可见。                                                              |
| `all`   | 所有回复都进入线程。保持对话集中，但可能降低可见性。                                                                                                            |

该模式适用于自动回复和 agent 工具调用（`slack sendMessage`）。

### 按聊天类型的线程

你可以通过设置 `channels.slack.replyToModeByChatType` 为每种聊天类型配置不同的线程行为：

```json5
{
  channels: {
    slack: {
      replyToMode: "off", // 频道的默认值
      replyToModeByChatType: {
        direct: "all", // DM 始终使用线程
        group: "first", // 群组 DM/MPIM 第一次回复使用线程
      },
    },
  },
}
```

支持的聊天类型：

- `direct`: 1:1 DM（Slack `im`）
- `group`: 群组 DM / MPIM（Slack `mpim`）
- `channel`: 标准频道（公开/私有）

优先级：

1. `replyToModeByChatType.<chatType>`
2. `replyToMode`
3. Provider 默认值（`off`）

当没有设置聊天类型覆盖时，传统的 `channels.slack.dm.replyToMode` 仍可作为 `direct` 的回退接受。

示例：

仅线程 DM：

```json5
{
  channels: {
    slack: {
      replyToMode: "off",
      replyToModeByChatType: { direct: "all" },
    },
  },
}
```

线程群组 DM 但保持频道在根目录：

```json5
{
  channels: {
    slack: {
      replyToMode: "off",
      replyToModeByChatType: { group: "first" },
    },
  },
}
```

使频道线程化，保持 DM 在根目录：

```json5
{
  channels: {
    slack: {
      replyToMode: "first",
      replyToModeByChatType: { direct: "off", group: "off" },
    },
  },
}
```

### 手动线程标签

对于细粒度控制，在 agent 响应中使用这些标签：

- `[[reply_to_current]]` — 回复触发消息（开始/继续线程）。
- `[[reply_to:<id>]]` — 回复特定的消息 id。

## Sessions + 路由

- DM 共享 `main` session（类似 WhatsApp/Telegram）。
- 频道映射到 `agent:<agentId>:slack:channel:<channelId>` sessions。
- Slash 命令使用 `agent:<agentId>:slack:slash:<userId>` sessions（前缀可通过 `channels.slack.slashCommand.sessionPrefix` 配置）。
- 如果 Slack 没有提供 `channel_type`，OpenClaw 从频道 ID 前缀（`D`、`C`、`G`）推断，并默认为 `channel` 以保持 session key 稳定。
- 原生命令注册使用 `commands.native`（全局默认 `"auto"` → Slack 关闭），可以使用 `channels.slack.commands.native` 按工作空间覆盖。文本命令需要独立的 `/...` 消息，可以使用 `commands.text: false` 禁用。Slack slash 命令在 Slack 应用中管理，不会自动移除。使用 `commands.useAccessGroups: false` 绕过命令的访问组检查。
- 完整的命令列表 + 配置：[Slash 命令](/tools/slash-commands)

## DM 安全（配对）

- 默认：`channels.slack.dm.policy="pairing"` — 未知的 DM 发送者获得一个配对码（1 小时后过期）。
- 通过以下方式批准：`openclaw pairing approve slack <code>`。
- 允许任何人：设置 `channels.slack.dm.policy="open"` 和 `channels.slack.dm.allowFrom=["*"]`。
- `channels.slack.dm.allowFrom` 接受用户 ID、@handles 或 emails（在令牌允许时在启动时解析）。向导在令牌允许时在设置期间接受用户名并解析为 ID。

## 群组策略

- `channels.slack.groupPolicy` 控制频道处理（`open|disabled|allowlist`）。
- `allowlist` 要求频道列在 `channels.slack.channels` 中。
- 如果你只设置 `SLACK_BOT_TOKEN`/`SLACK_APP_TOKEN` 而从不创建 `channels.slack` 部分，运行时默认将 `groupPolicy` 设置为 `open`。添加 `channels.slack.groupPolicy`、`channels.defaults.groupPolicy` 或频道 allowlist 以锁定它。
- 配置向导接受 `#channel` 名称并在可能时解析为 ID（公开 + 私有）；如果存在多个匹配，它优先选择活动频道。
- 在启动时，OpenClaw 在令牌允许时将 allowlist 中的频道/用户名解析为 ID，并记录映射；未解析的条目保持原样。
- 要**不允许任何频道**，设置 `channels.slack.groupPolicy: "disabled"`（或保持空的 allowlist）。

频道选项（`channels.slack.channels.<id>` 或 `channels.slack.channels.<name>`）：

- `allow`：当 `groupPolicy="allowlist"` 时允许/拒绝频道。
- `requireMention`：频道的提及门控。
- `tools`：可选的每频道工具策略覆盖（`allow`/`deny`/`alsoAllow`）。
- `toolsBySender`：频道内可选的每发送者工具策略覆盖（key 是发送者 id/@handles/emails；支持 `"*"` 通配符）。
- `allowBots`：允许此频道中机器人撰写的消息（默认：false）。
- `users`：可选的每频道用户 allowlist。
- `skills`：技能过滤器（省略 = 所有技能，空 = 无）。
- `systemPrompt`：频道的额外系统 prompt（与 topic/purpose 组合）。
- `enabled`：设置为 `false` 以禁用频道。

## 投递目标

将这些与 cron/CLI 发送一起使用：

- `user:<id>` 用于 DM
- `channel:<id>` 用于频道

## 工具操作

Slack 工具操作可以使用 `channels.slack.actions.*` 进行门控：

| 操作组 | 默认 | 说明                  |
| ------------ | ------- | ---------------------- |
| reactions    | enabled | 反应 + 列出反应 |
| messages     | enabled | 读取/发送/编辑/删除  |
| pins         | enabled | 置顶/取消置顶/列出         |
| memberInfo   | enabled | 成员信息            |
| emojiList    | enabled | 自定义表情符号列表      |

## 安全说明

- 写入默认使用 bot token，因此状态更改操作保持在应用的 bot 权限和身份范围内。
- 设置 `userTokenReadOnly: false` 允许 user token 在 bot token 不可用时用于写入操作，这意味着操作以安装用户的访问权限运行。将 user token 视为高度特权，并保持操作门控和 allowlist 严格。
- 如果你启用 user-token 写入，请确保 user token 包含你期望的写入范围（`chat:write`、`reactions:write`、`pins:write`、`files:write`），否则这些操作将失败。

## 说明

- 提及门控通过 `channels.slack.channels` 控制（设置 `requireMention` 为 `true`）；`agents.list[].groupChat.mentionPatterns`（或 `messages.groupChat.mentionPatterns`）也计入提及。
- 多 agent 覆盖：在 `agents.list[].groupChat.mentionPatterns` 上设置每 agent 模式。
- 反应通知遵循 `channels.slack.reactionNotifications`（将 `reactionAllowlist` 与模式 `allowlist` 一起使用）。
- 机器人撰写的消息默认被忽略；通过 `channels.slack.allowBots` 或 `channels.slack.channels.<id>.allowBots` 启用。
- 警告：如果你允许回复其他机器人（`channels.slack.allowBots=true` 或 `channels.slack.channels.<id>.allowBots=true`），使用 `requireMention`、`channels.slack.channels.<id>.users` allowlist 和/或 `AGENTS.md` 和 `SOUL.md` 中的清晰 guardrails 防止机器人到机器人的回复循环。
- 对于 Slack 工具，反应移除语义在 [/tools/reactions](/tools/reactions) 中。
- 附件在允许时下载到媒体存储，并在大小限制内。

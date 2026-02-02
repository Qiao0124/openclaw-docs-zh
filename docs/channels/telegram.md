---
summary: "Telegram 机器人支持状态、功能和配置"
read_when:
  - 开发 Telegram 功能或 Webhook
title: "Telegram"
---

# Telegram（Bot API）

状态：通过 grammY 支持机器人私信和群组的生产环境。默认长轮询；可选 Webhook。

## 快速设置（初学者）

1. 使用 **@BotFather** 创建机器人并复制令牌。
2. 设置令牌：
   - 环境变量：`TELEGRAM_BOT_TOKEN=...`
   - 或配置：`channels.telegram.botToken: "..."`。
   - 如果两者都设置，配置优先（环境变量仅作为默认账户的回退）。
3. 启动网关。
4. 私信访问默认使用配对模式；首次联系时批准配对码。

最小配置：

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
    },
  },
}
```

## 功能概述

- 由网关拥有的 Telegram Bot API 频道。
- 确定性路由：回复返回 Telegram；模型不会选择频道。
- 私信共享代理的主会话；群组保持隔离（`agent:<agentId>:telegram:group:<chatId>`）。

## 设置（快速路径）

### 1) 创建机器人令牌（BotFather）

1. 打开 Telegram 并与 **@BotFather** 聊天。
2. 运行 `/newbot`，然后按照提示操作（名称 + 以 `bot` 结尾的用户名）。
3. 复制令牌并安全保存。

可选的 BotFather 设置：

- `/setjoingroups` — 允许/禁止将机器人添加到群组。
- `/setprivacy` — 控制机器人是否看到所有群组消息。

### 2) 配置令牌（环境变量或配置）

示例：

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

环境变量选项：`TELEGRAM_BOT_TOKEN=...`（适用于默认账户）。
如果环境变量和配置都设置，配置优先。

多账户支持：使用 `channels.telegram.accounts` 配合每个账户的令牌和可选的 `name`。共享模式参见 [`gateway/configuration`](/gateway/configuration#telegramaccounts--discordaccounts--slackaccounts--signalaccounts--imessageaccounts)。

3. 启动网关。当令牌解析完成时 Telegram 启动（配置优先，环境变量回退）。
4. 私信访问默认为配对模式。机器人首次被联系时批准代码。
5. 对于群组：添加机器人，决定隐私/管理员行为（如下），然后设置 `channels.telegram.groups` 以控制提及门控和允许列表。

## 令牌 + 隐私 + 权限（Telegram 端）

### 令牌创建（BotFather）

- `/newbot` 创建机器人并返回令牌（请保密）。
- 如果令牌泄露，通过 @BotFather 撤销/重新生成并更新配置。

### 群组消息可见性（隐私模式）

Telegram 机器人默认启用**隐私模式**，这限制它们接收的群组消息。
如果您的机器人必须看到*所有*群组消息，您有两个选项：

- 使用 `/setprivacy` 禁用隐私模式**或**
- 将机器人添加为群组**管理员**（管理员机器人接收所有消息）。

**注意：** 切换隐私模式时，Telegram 要求从每个群组中移除并重新添加机器人，更改才能生效。

### 群组权限（管理员权限）

管理员状态在群组内设置（Telegram UI）。管理员机器人始终接收所有群组消息，因此如果您需要完全可见性，请使用管理员。

## 工作原理（行为）

- 入站消息被规范化为共享频道信封，包含回复上下文和媒体占位符。
- 群组回复默认需要提及（原生 @提及或 `agents.list[].groupChat.mentionPatterns` / `messages.groupChat.mentionPatterns`）。
- 多代理覆盖：在 `agents.list[].groupChat.mentionPatterns` 上设置每个代理的模式。
- 回复始终路由回同一个 Telegram 聊天。
- 长轮询使用 grammY runner 进行每聊天排序；整体并发受 `agents.defaults.maxConcurrent` 限制。
- Telegram Bot API 不支持已读回执；没有 `sendReadReceipts` 选项。

## 草稿流式传输

OpenClaw 可以使用 `sendMessageDraft` 在 Telegram 私信中流式传输部分回复。

要求：

- 在 @BotFather 中为机器人启用话题模式（论坛主题模式）。
- 仅限私人聊天话题（Telegram 在入站消息上包含 `message_thread_id`）。
- `channels.telegram.streamMode` 未设置为 `"off"`（默认：`"partial"`，`"block"` 启用分块草稿更新）。

草稿流式传输仅限私信；Telegram 在群组或频道中不支持。

## 格式化（Telegram HTML）

- 出站 Telegram 文本使用 `parse_mode: "HTML"`（Telegram 支持的标签子集）。
- 类 Markdown 输入被渲染为 **Telegram 安全 HTML**（粗体/斜体/删除线/代码/链接）；块元素被扁平化为带换行符/项目符号的文本。
- 来自模型的原始 HTML 被转义以避免 Telegram 解析错误。
- 如果 Telegram 拒绝 HTML 负载，OpenClaw 以纯文本重试相同消息。

## 命令（原生 + 自定义）

OpenClaw 在启动时向 Telegram 的机器人菜单注册原生命令（如 `/status`、`/reset`、`/model`）。
您可以通过配置将自定义命令添加到菜单：

```json5
{
  channels: {
    telegram: {
      customCommands: [
        { command: "backup", description: "Git 备份" },
        { command: "generate", description: "创建图片" },
      ],
    },
  },
}
```

## 故障排除

- 日志中的 `setMyCommands failed` 通常意味着出站 HTTPS/DNS 被阻止访问 `api.telegram.org`。
- 如果您看到 `sendMessage` 或 `sendChatAction` 失败，请检查 IPv6 路由和 DNS。

更多帮助：[频道故障排除](/channels/troubleshooting)。

注意：

- 自定义命令**仅是菜单条目**；除非您在别处处理它们，否则 OpenClaw 不会实现它们。
- 命令名称被规范化（去除前导 `/`，小写）且必须匹配 `a-z`、`0-9`、`_`（1-32 个字符）。
- 自定义命令**不能覆盖原生命令**。冲突被忽略并记录。
- 如果 `commands.native` 被禁用，仅注册自定义命令（或如果没有则清除）。

## 限制

- 出站文本被分块为 `channels.telegram.textChunkLimit`（默认 4000）。
- 可选换行分块：设置 `channels.telegram.chunkMode="newline"` 以在长度分块之前在空行（段落边界）处分割。
- 媒体下载/上传受 `channels.telegram.mediaMaxMb` 限制（默认 5）。
- Telegram Bot API 请求在 `channels.telegram.timeoutSeconds` 后超时（默认通过 grammY 为 500）。设置较低以避免长时间挂起。
- 群组历史上下文使用 `channels.telegram.historyLimit`（或 `channels.telegram.accounts.*.historyLimit`），回退到 `messages.groupChat.historyLimit`。设置为 `0` 以禁用（默认 50）。
- 私信历史可以使用 `channels.telegram.dmHistoryLimit` 限制（用户轮次）。每个用户的覆盖：`channels.telegram.dms["<user_id>"].historyLimit`。

## 群组激活模式

默认情况下，机器人仅响应群组中的提及（`@botname` 或 `agents.list[].groupChat.mentionPatterns` 中的模式）。要更改此行为：

### 通过配置（推荐）

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": { requireMention: false }, // 在此群组中始终响应
      },
    },
  },
}
```

**重要：** 设置 `channels.telegram.groups` 会创建**允许列表** - 仅列出的群组（或 `"*"`）会被接受。
论坛主题继承其父群组配置（allowFrom、requireMention、skills、prompts），除非您在 `channels.telegram.groups.<groupId>.topics.<topicId>` 下添加每个主题的覆盖。

允许所有群组并始终响应：

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: false }, // 所有群组，始终响应
      },
    },
  },
}
```

对所有群组保持仅提及（默认行为）：

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: true }, // 或完全省略 groups
      },
    },
  },
}
```

### 通过命令（会话级别）

在群组中发送：

- `/activation always` - 响应所有消息
- `/activation mention` - 需要提及（默认）

**注意：** 命令仅更新会话状态。要在重启后保持持久行为，请使用配置。

### 获取群组聊天 ID

将群组中的任何消息转发给 Telegram 上的 `@userinfobot` 或 `@getidsbot` 以查看聊天 ID（负数如 `-1001234567890`）。

**提示：** 对于您自己的用户 ID，私信机器人，它会回复您的用户 ID（配对消息），或在命令启用后使用 `/whoami`。

**隐私注意：** `@userinfobot` 是第三方机器人。如果您愿意，将机器人添加到群组，发送消息，并使用 `openclaw logs --follow` 读取 `chat.id`，或使用 Bot API `getUpdates`。

## 配置写入

默认情况下，允许 Telegram 写入由频道事件或 `/config set|unset` 触发的配置更新。

这发生在：

- 群组升级为超级群组且 Telegram 发出 `migrate_to_chat_id`（聊天 ID 更改）。OpenClaw 可以自动迁移 `channels.telegram.groups`。
- 您在 Telegram 聊天中运行 `/config set` 或 `/config unset`（需要 `commands.config: true`）。

禁用：

```json5
{
  channels: { telegram: { configWrites: false } },
}
```

## 主题（论坛超级群组）

Telegram 论坛主题为每条消息包含一个 `message_thread_id`。OpenClaw：

- 将 `:topic:<threadId>` 附加到 Telegram 群组会话键，因此每个主题是隔离的。
- 发送打字指示器和带 `message_thread_id` 的回复，因此响应保留在主题中。
- 一般主题（线程 id `1`）是特殊的：消息发送省略 `message_thread_id`（Telegram 会拒绝它），但打字指示器仍然包含它。
- 在模板上下文中公开 `MessageThreadId` + `IsForum` 用于路由/模板。
- 主题特定配置在 `channels.telegram.groups.<chatId>.topics.<threadId>` 下可用（skills、允许列表、自动回复、系统提示、禁用）。
- 主题配置继承群组设置（requireMention、允许列表、skills、prompts、enabled），除非按主题覆盖。

私人聊天在某些边缘情况下可以包含 `message_thread_id`。OpenClaw 保持私信会话键不变，但在存在时仍使用线程 id 进行回复/草稿流式传输。

## 内联按钮

Telegram 支持带回调按钮的内联键盘。

```json5
{
  channels: {
    telegram: {
      capabilities: {
        inlineButtons: "allowlist",
      },
    },
  },
}
```

对于每个账户的配置：

```json5
{
  channels: {
    telegram: {
      accounts: {
        main: {
          capabilities: {
            inlineButtons: "allowlist",
          },
        },
      },
    },
  },
}
```

范围：

- `off` — 内联按钮禁用
- `dm` — 仅私信（群组目标被阻止）
- `group` — 仅群组（私信目标被阻止）
- `all` — 私信 + 群组
- `allowlist` — 私信 + 群组，但仅允许 `allowFrom`/`groupAllowFrom` 允许的发送者（与控制命令规则相同）

默认：`allowlist`。
遗留：`capabilities: ["inlineButtons"]` = `inlineButtons: "all"`。

### 发送按钮

使用带 `buttons` 参数的消息工具：

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "选择一个选项：",
  buttons: [
    [
      { text: "是", callback_data: "yes" },
      { text: "否", callback_data: "no" },
    ],
    [{ text: "取消", callback_data: "cancel" }],
  ],
}
```

当用户点击按钮时，回调数据以格式 `callback_data: value` 作为消息发送回代理。

### 配置选项

Telegram 功能可以在两个级别配置（上面显示对象形式；遗留字符串数组仍受支持）：

- `channels.telegram.capabilities`：全局默认功能配置，应用于所有 Telegram 账户，除非被覆盖。
- `channels.telegram.accounts.<account>.capabilities`：每个账户的功能，覆盖该特定账户的全局默认值。

当所有 Telegram 机器人/账户应该行为相同时使用全局设置。当不同机器人需要不同行为时使用每个账户配置（例如，一个账户仅处理私信，而另一个允许在群组中）。

## 访问控制（私信 + 群组）

### 私信访问

- 默认：`channels.telegram.dmPolicy = "pairing"`。未知发送者接收配对码；消息被忽略直到批准（代码在 1 小时后过期）。
- 通过以下方式批准：
  - `openclaw pairing list telegram`
  - `openclaw pairing approve telegram <CODE>`
- 配对是 Telegram 私信使用的默认令牌交换。详情：[配对](/start/pairing)
- `channels.telegram.allowFrom` 接受数字用户 ID（推荐）或 `@username` 条目。它**不是**机器人用户名；使用人类发送者的 ID。向导接受 `@username` 并在可能时解析为数字 ID。

#### 查找您的 Telegram 用户 ID

更安全（无第三方机器人）：

1. 启动网关并向您的机器人发送私信。
2. 运行 `openclaw logs --follow` 并查找 `from.id`。

替代（官方 Bot API）：

1. 私信您的机器人。
2. 使用您的机器人令牌获取更新并读取 `message.from.id`：
   ```bash
   curl "https://api.telegram.org/bot<bot_token>/getUpdates"
   ```

第三方（隐私较低）：

- 私信 `@userinfobot` 或 `@getidsbot` 并使用返回的用户 id。

### 群组访问

两个独立的控制：

**1. 允许哪些群组**（通过 `channels.telegram.groups` 的群组允许列表）：

- 没有 `groups` 配置 = 允许所有群组
- 有 `groups` 配置 = 仅允许列出的群组或 `"*"`
- 示例：`"groups": { "-1001234567890": {}, "*": {} }` 允许所有群组

**2. 允许哪些发送者**（通过 `channels.telegram.groupPolicy` 的发送者过滤）：

- `"open"` = 允许群组中的所有发送者可以发送消息
- `"allowlist"` = 仅 `channels.telegram.groupAllowFrom` 中的发送者可以发送消息
- `"disabled"` = 完全不接受群组消息
  默认是 `groupPolicy: "allowlist"`（除非您添加 `groupAllowFrom`，否则被阻止）。

大多数用户想要：`groupPolicy: "allowlist"` + `groupAllowFrom` + 在 `channels.telegram.groups` 中列出的特定群组

## 长轮询 vs Webhook

- 默认：长轮询（不需要公共 URL）。
- Webhook 模式：设置 `channels.telegram.webhookUrl`（可选 `channels.telegram.webhookSecret` + `channels.telegram.webhookPath`）。
  - 本地监听器绑定到 `0.0.0.0:8787` 并默认服务 `POST /telegram-webhook`。
  - 如果您的公共 URL 不同，请使用反向代理并将 `channels.telegram.webhookUrl` 指向公共端点。

## 回复线程

Telegram 支持通过标签的可选线程回复：

- `[[reply_to_current]]` -- 回复触发消息。
- `[[reply_to:<id>]]` -- 回复特定消息 id。

由 `channels.telegram.replyToMode` 控制：

- `first`（默认）、`all`、`off`。

## 音频消息（语音 vs 文件）

Telegram 区分**语音笔记**（圆形气泡）和**音频文件**（元数据卡片）。
OpenClaw 默认为音频文件以保持向后兼容。

要在代理回复中强制使用语音笔记气泡，请在回复中的任何位置包含此标签：

- `[[audio_as_voice]]` — 将音频作为语音笔记而不是文件发送。

该标签从传递的文本中剥离。其他频道忽略此标签。

对于消息工具发送，使用语音兼容音频 `media` URL 设置 `asVoice: true`
（当媒体存在时 `message` 是可选的）：

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

## 贴纸

OpenClaw 支持接收和发送 Telegram 贴纸，具有智能缓存。

### 接收贴纸

当用户发送贴纸时，OpenClaw 根据贴纸类型处理：

- **静态贴纸（WEBP）：** 下载并通过视觉处理。贴纸在消息内容中显示为 `<media:sticker>` 占位符。
- **动画贴纸（TGS）：** 跳过（不支持 Lottie 格式处理）。
- **视频贴纸（WEBM）：** 跳过（不支持视频格式处理）。

接收贴纸时可用的模板参数字段：

- `Sticker` — 对象，包含：
  - `emoji` — 与贴纸关联的表情符号
  - `setName` — 贴纸集名称
  - `fileId` — Telegram 文件 ID（发送相同贴纸回来）
  - `fileUniqueId` — 用于缓存查找的稳定 ID
  - `cachedDescription` — 可用时的缓存视觉描述

### 贴纸缓存

贴纸通过 AI 的视觉功能处理以生成描述。由于相同的贴纸经常被重复发送，OpenClaw 缓存这些描述以避免冗余 API 调用。

**工作原理：**

1. **首次遇到：** 贴纸图像被发送到 AI 进行视觉分析。AI 生成描述（例如，"一只热情挥手的卡通猫"）。
2. **缓存存储：** 描述与贴纸的文件 ID、表情符号和集名称一起保存。
3. **后续遇到：** 当再次看到相同的贴纸时，直接使用缓存的描述。图像不会发送到 AI。

**缓存位置：** `~/.openclaw/telegram/sticker-cache.json`

**缓存条目格式：**

```json
{
  "fileId": "CAACAgIAAxkBAAI...",
  "fileUniqueId": "AgADBAADb6cxG2Y",
  "emoji": "👋",
  "setName": "CoolCats",
  "description": "一只热情挥手的卡通猫",
  "cachedAt": "2026-01-15T10:30:00.000Z"
}
```

**好处：**

- 通过避免对相同贴纸的重复视觉调用降低 API 成本
- 缓存贴纸的响应时间更快（无视觉处理延迟）
- 支持基于缓存描述的贴纸搜索功能

缓存随着贴纸接收自动填充。不需要手动缓存管理。

### 发送贴纸

代理可以使用 `sticker` 和 `sticker-search` 操作发送和搜索贴纸。这些默认禁用，必须在配置中启用：

```json5
{
  channels: {
    telegram: {
      actions: {
        sticker: true,
      },
    },
  },
}
```

**发送贴纸：**

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

参数：

- `fileId`（必需）— 贴纸的 Telegram 文件 ID。从接收贴纸时的 `Sticker.fileId` 获取，或从 `sticker-search` 结果获取。
- `replyTo`（可选）— 要回复的消息 ID。
- `threadId`（可选）— 论坛主题的消息线程 ID。

**搜索贴纸：**

代理可以通过描述、表情符号或集名称搜索缓存的贴纸：

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "cat waving",
  limit: 5,
}
```

从缓存返回匹配的贴纸：

```json5
{
  ok: true,
  count: 2,
  stickers: [
    {
      fileId: "CAACAgIAAxkBAAI...",
      emoji: "👋",
      description: "一只热情挥手的卡通猫",
      setName: "CoolCats",
    },
  ],
}
```

搜索使用描述文本、表情符号字符和集名称的模糊匹配。

**带线程的示例：**

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "-1001234567890",
  fileId: "CAACAgIAAxkBAAI...",
  replyTo: 42,
  threadId: 123,
}
```

## 流式传输（草稿）

Telegram 可以在代理生成响应时流式传输**草稿气泡**。
OpenClaw 使用 Bot API `sendMessageDraft`（不是真实消息），然后发送最终回复作为普通消息。

要求（Telegram Bot API 9.3+）：

- **启用主题的私人聊天**（机器人的论坛主题模式）。
- 入站消息必须包含 `message_thread_id`（私人话题线程）。
- 流式传输对群组/超级群组/频道被忽略。

配置：

- `channels.telegram.streamMode: "off" | "partial" | "block"`（默认：`partial`）
  - `partial`：用最新的流式文本更新草稿气泡。
  - `block`：用更大的块（分块）更新草稿气泡。
  - `off`：禁用草稿流式传输。
- 可选（仅用于 `streamMode: "block"`）：
  - `channels.telegram.draftChunk: { minChars?, maxChars?, breakPreference? }`
    - 默认值：`minChars: 200`，`maxChars: 800`，`breakPreference: "paragraph"`（限制为 `channels.telegram.textChunkLimit`）。

注意：草稿流式传输与**块流式传输**（频道消息）是分开的。
块流式传输默认关闭，如果您想要早期 Telegram 消息而不是草稿更新，需要 `channels.telegram.blockStreaming: true`。

推理流（仅限 Telegram）：

- `/reasoning stream` 在生成回复时将推理流式传输到草稿气泡，然后发送不带推理的最终答案。
- 如果 `channels.telegram.streamMode` 是 `off`，推理流被禁用。
  更多上下文：[流式传输 + 分块](/concepts/streaming)。

## 重试策略

出站 Telegram API 调用在瞬态网络/429 错误上使用指数退避和抖动重试。通过 `channels.telegram.retry` 配置。参见 [重试策略](/concepts/retry)。

## 代理工具（消息 + 反应）

- 工具：`telegram` 带 `sendMessage` 操作（`to`、`content`、可选 `mediaUrl`、`replyToMessageId`、`messageThreadId`）。
- 工具：`telegram` 带 `react` 操作（`chatId`、`messageId`、`emoji`）。
- 工具：`telegram` 带 `deleteMessage` 操作（`chatId`、`messageId`）。
- 反应移除语义：参见 [/tools/reactions](/tools/reactions)。
- 工具门控：`channels.telegram.actions.reactions`、`channels.telegram.actions.sendMessage`、`channels.telegram.actions.deleteMessage`（默认：启用），和 `channels.telegram.actions.sticker`（默认：禁用）。

## 反应通知

**反应工作原理：**
Telegram 反应作为**单独的 `message_reaction` 事件**到达，而不是消息负载中的属性。当用户添加反应时，OpenClaw：

1. 从 Telegram API 接收 `message_reaction` 更新
2. 将其转换为**系统事件**，格式为：`"Telegram reaction added: {emoji} by {user} on msg {id}"`
3. 使用与常规消息**相同的会话键**将系统事件入队
4. 当该对话中的下一条消息到达时，系统事件被排出并添加到代理的上下文前面

代理将反应视为对话历史中的**系统通知**，而不是消息元数据。

**配置：**

- `channels.telegram.reactionNotifications`：控制哪些反应触发通知
  - `"off"` — 忽略所有反应
  - `"own"` — 当用户反应到机器人消息时通知（尽力而为；内存中）（默认）
  - `"all"` — 通知所有反应

- `channels.telegram.reactionLevel`：控制代理的反应能力
  - `"off"` — 代理不能对消息反应
  - `"ack"` — 机器人发送确认反应（处理时的 👀）（默认）
  - `"minimal"` — 代理可以谨慎反应（指南：每 5-10 次交换 1 次）
  - `"extensive"` — 代理可以在适当时自由反应

**论坛群组：** 论坛群组中的反应包含 `message_thread_id` 并使用类似 `agent:main:telegram:group:{chatId}:topic:{threadId}` 的会话键。这确保同一主题中的反应和消息保持在一起。

**示例配置：**

```json5
{
  channels: {
    telegram: {
      reactionNotifications: "all", // 查看所有反应
      reactionLevel: "minimal", // 代理可以谨慎反应
    },
  },
}
```

**要求：**

- Telegram 机器人必须显式请求 `allowed_updates` 中的 `message_reaction`（由 OpenClaw 自动配置）
- 对于 Webhook 模式，反应包含在 Webhook `allowed_updates` 中
- 对于轮询模式，反应包含在 `getUpdates` `allowed_updates` 中

## 传递目标（CLI/cron）

- 使用聊天 id（`123456789`）或用户名（`@name`）作为目标。
- 示例：`openclaw message send --channel telegram --target 123456789 --message "hi"`。

## 故障排除

**机器人在群组中不响应非提及消息：**

- 如果您设置 `channels.telegram.groups.*.requireMention=false`，Telegram 的 Bot API **隐私模式**必须禁用。
  - BotFather：`/setprivacy` → **Disable**（然后从群组中移除 + 重新添加机器人）
- `openclaw channels status` 显示配置期望未提及群组消息时的警告。
- `openclaw channels status --probe` 可以额外检查显式数字群组 ID 的成员资格（它无法审计通配符 `"*"` 规则）。
- 快速测试：`/activation always`（仅会话；使用配置保持持久性）

**机器人完全看不到群组消息：**

- 如果设置了 `channels.telegram.groups`，群组必须被列出或使用 `"*"`
- 检查 @BotFather 中的隐私设置 → "Group Privacy" 应该是 **OFF**
- 验证机器人确实是成员（不仅仅是管理员但没有读取权限）
- 检查网关日志：`openclaw logs --follow`（查找 "skipping group message"）

**机器人响应提及但不响应 `/activation always`：**

- `/activation` 命令更新会话状态但不持久保存到配置
- 对于持久行为，将群组添加到 `channels.telegram.groups` 并设置 `requireMention: false`

**`/status` 等命令不起作用：**

- 确保您的 Telegram 用户 ID 已授权（通过配对或 `channels.telegram.allowFrom`）
- 命令需要授权，即使在 `groupPolicy: "open"` 的群组中也是如此

**长轮询在 Node 22+ 上立即中止（通常使用代理/自定义 fetch）：**

- Node 22+ 对 `AbortSignal` 实例更严格；外部信号可以立即中止 `fetch` 调用。
- 升级到规范化中止信号的 OpenClaw 构建，或在可以升级之前在 Node 20 上运行网关。

**机器人启动，然后静默停止响应（或日志显示 `HttpError: Network request ... failed`）：**

- 某些主机将 `api.telegram.org` 解析为 IPv6 优先。如果您的服务器没有可用的 IPv6 出口，grammY 可能会卡在仅 IPv6 的请求上。
- 通过启用 IPv6 出口**或**强制 `api.telegram.org` 的 IPv4 解析来修复（例如，使用 IPv4 A 记录添加 `/etc/hosts` 条目，或在您的操作系统 DNS 堆栈中优先使用 IPv4），然后重启网关。
- 快速检查：`dig +short api.telegram.org A` 和 `dig +short api.telegram.org AAAA` 确认 DNS 返回什么。

## 配置参考（Telegram）

完整配置：[配置](/gateway/configuration)

提供商选项：

- `channels.telegram.enabled`：启用/禁用频道启动。
- `channels.telegram.botToken`：机器人令牌（BotFather）。
- `channels.telegram.tokenFile`：从文件路径读取令牌。
- `channels.telegram.dmPolicy`：`pairing | allowlist | open | disabled`（默认：pairing）。
- `channels.telegram.allowFrom`：私信允许列表（id/用户名）。`open` 需要 `"*"`。
- `channels.telegram.groupPolicy`：`open | allowlist | disabled`（默认：allowlist）。
- `channels.telegram.groupAllowFrom`：群组发送者允许列表（id/用户名）。
- `channels.telegram.groups`：每个群组的默认值 + 允许列表（使用 `"*"` 作为全局默认值）。
  - `channels.telegram.groups.<id>.requireMention`：提及门控默认值。
  - `channels.telegram.groups.<id>.skills`：技能过滤器（省略 = 所有技能，空 = 无）。
  - `channels.telegram.groups.<id>.allowFrom`：每个群组的发送者允许列表覆盖。
  - `channels.telegram.groups.<id>.systemPrompt`：群组的额外系统提示。
  - `channels.telegram.groups.<id>.enabled`：设置为 `false` 时禁用群组。
  - `channels.telegram.groups.<id>.topics.<threadId>.*`：每个主题的覆盖（与群组相同字段）。
  - `channels.telegram.groups.<id>.topics.<threadId>.requireMention`：每个主题的提及门控覆盖。
- `channels.telegram.capabilities.inlineButtons`：`off | dm | group | all | allowlist`（默认：allowlist）。
- `channels.telegram.accounts.<account>.capabilities.inlineButtons`：每个账户的覆盖。
- `channels.telegram.replyToMode`：`off | first | all`（默认：`first`）。
- `channels.telegram.textChunkLimit`：出站块大小（字符）。
- `channels.telegram.chunkMode`：`length`（默认）或 `newline` 以在空行（段落边界）处分割，然后进行长度分块。
- `channels.telegram.linkPreview`：切换出站消息的链接预览（默认：true）。
- `channels.telegram.streamMode`：`off | partial | block`（草稿流式传输）。
- `channels.telegram.mediaMaxMb`：入站/出站媒体上限（MB）。
- `channels.telegram.retry`：出站 Telegram API 调用的重试策略（尝试次数、minDelayMs、maxDelayMs、抖动）。
- `channels.telegram.network.autoSelectFamily`：覆盖 Node autoSelectFamily（true=启用，false=禁用）。默认在 Node 22 上禁用以避免 Happy Eyeballs 超时。
- `channels.telegram.proxy`：Bot API 调用的代理 URL（SOCKS/HTTP）。
- `channels.telegram.webhookUrl`：启用 Webhook 模式。
- `channels.telegram.webhookSecret`：Webhook 密钥（可选）。
- `channels.telegram.webhookPath`：本地 Webhook 路径（默认 `/telegram-webhook`）。
- `channels.telegram.actions.reactions`：门控 Telegram 工具反应。
- `channels.telegram.actions.sendMessage`：门控 Telegram 工具消息发送。
- `channels.telegram.actions.deleteMessage`：门控 Telegram 工具消息删除。
- `channels.telegram.actions.sticker`：门控 Telegram 贴纸操作 — 发送和搜索（默认：false）。
- `channels.telegram.reactionNotifications`：`off | own | all` — 控制哪些反应触发系统事件（默认：未设置时为 `own`）。
- `channels.telegram.reactionLevel`：`off | ack | minimal | extensive` — 控制代理的反应能力（默认：未设置时为 `minimal`）。

相关全局选项：

- `agents.list[].groupChat.mentionPatterns`（提及门控模式）。
- `messages.groupChat.mentionPatterns`（全局回退）。
- `commands.native`（默认为 `"auto"` → Telegram/Discord 开启，Slack 关闭），`commands.text`，`commands.useAccessGroups`（命令行为）。使用 `channels.telegram.commands.native` 覆盖。
- `messages.responsePrefix`、`messages.ackReaction`、`messages.ackReactionScope`、`messages.removeAckAfterReply`。

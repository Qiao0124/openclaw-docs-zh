---
summary: "Discord 机器人支持状态、功能和配置"
read_when:
  - 开发 Discord 频道功能
title: "Discord"
---

# Discord（Bot API）

状态：通过官方 Discord 机器人网关支持私信和公会文字频道。

## 快速设置（初学者）

1. 创建 Discord 机器人并复制机器人令牌。
2. 在 Discord 应用设置中，启用**消息内容意图**（如果您计划使用允许列表或名称查找，还需要**服务器成员意图**）。
3. 为 OpenClaw 设置令牌：
   - 环境变量：`DISCORD_BOT_TOKEN=...`
   - 或配置：`channels.discord.token: "..."`。
   - 如果两者都设置，配置优先（环境变量回退仅为默认账户）。
4. 邀请机器人到您的服务器并授予消息权限（如果只需要私信，请创建私人服务器）。
5. 启动网关。
6. 私信访问默认使用配对；首次联系时批准配对码。

最小配置：

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN",
    },
  },
}
```

## 目标

- 通过 Discord 私信或公会频道与 OpenClaw 对话。
- 直接聊天折叠到代理的主会话（默认 `agent:main:main`）；公会频道保持隔离为 `agent:<agentId>:discord:channel:<channelId>`（显示名称使用 `discord:<guildSlug>#<channelSlug>`）。
- 群组私信默认被忽略；通过 `channels.discord.dm.groupEnabled` 启用，并通过 `channels.discord.dm.groupChannels` 可选限制。
- 保持路由确定性：回复始终返回到接收消息的频道。

## 工作原理

1. 创建 Discord 应用 → 机器人，启用您需要的意图（私信 + 公会消息 + 消息内容），并获取机器人令牌。
2. 邀请机器人到您的服务器，授予在您想使用它的地方读取/发送消息所需的权限。
3. 使用 `channels.discord.token`（或作为回退的 `DISCORD_BOT_TOKEN`）配置 OpenClaw。
4. 运行网关；当令牌可用时自动启动 Discord 频道（配置优先，环境变量回退）且 `channels.discord.enabled` 不为 `false`。
   - 如果您更喜欢环境变量，请设置 `DISCORD_BOT_TOKEN`（配置块是可选的）。
5. 直接聊天：传递时使用 `user:<id>`（或 `<@id>` 提及）；所有轮次都落在共享的 `main` 会话中。裸数字 ID 是模糊的并被拒绝。
6. 公会频道：传递时使用 `channel:<channelId>`。默认需要提及，可以按公会或按频道设置。
7. 直接聊天：通过 `channels.discord.dm.policy` 默认安全（默认：`"pairing"`）。未知发送者获得配对码（1 小时后过期）；通过 `openclaw pairing approve discord <code>` 批准。
   - 保持旧的"对任何人开放"行为：设置 `channels.discord.dm.policy="open"` 和 `channels.discord.dm.allowFrom=["*"]`。
   - 硬允许列表：设置 `channels.discord.dm.policy="allowlist"` 并在 `channels.discord.dm.allowFrom` 中列出发送者。
   - 忽略所有私信：设置 `channels.discord.dm.enabled=false` 或 `channels.discord.dm.policy="disabled"`。
8. 群组私信默认被忽略；通过 `channels.discord.dm.groupEnabled` 启用，并通过 `channels.discord.dm.groupChannels` 可选限制。
9. 可选公会规则：按公会 id（首选）或 slug 设置 `channels.discord.guilds`，带有每个频道的规则。
10. 可选原生命令：`commands.native` 默认为 `"auto"`（Discord/Telegram 开启，Slack 关闭）。使用 `channels.discord.commands.native: true|false|"auto"` 覆盖；`false` 清除先前注册的命令。文本命令由 `commands.text` 控制，必须作为独立的 `/...` 消息发送。使用 `commands.useAccessGroups: false` 绕过命令的访问组检查。
    - 完整命令列表 + 配置：[斜杠命令](/tools/slash-commands)
11. 可选公会上下文历史：设置 `channels.discord.historyLimit`（默认 20，回退到 `messages.groupChat.historyLimit`）以在回复提及时包含最后 N 条公会消息作为上下文。设置为 `0` 以禁用。
12. 反应：代理可以通过 `discord` 工具触发反应（由 `channels.discord.actions.*` 门控）。
    - 反应移除语义：参见 [/tools/reactions](/tools/reactions)。
    - 仅当当前频道是 Discord 时才公开 `discord` 工具。
13. 原生命令使用隔离的会话键（`agent:<agentId>:discord:slash:<userId>`）而不是共享的 `main` 会话。

注意：名称 → id 解析使用公会成员搜索并需要服务器成员意图；如果机器人无法搜索成员，请使用 id 或 `<@id>` 提及。
注意：Slugs 是小写，空格替换为 `-`。频道名称被 slug 化，不带前导 `#`。
注意：公会上下文 `[from:]` 行包含 `author.tag` + `id` 以使可 ping 回复变得容易。

## 配置写入

默认情况下，允许 Discord 写入由 `/config set|unset` 触发的配置更新（需要 `commands.config: true`）。

禁用：

```json5
{
  channels: { discord: { configWrites: false } },
}
```

## 如何创建您自己的机器人

这是在服务器（公会）频道如 `#help` 中运行 OpenClaw 的"Discord 开发者门户"设置。

### 1) 创建 Discord 应用 + 机器人用户

1. Discord 开发者门户 → **Applications** → **New Application**
2. 在您的应用中：
   - **Bot** → **Add Bot**
   - 复制 **Bot Token**（这是您放入 `DISCORD_BOT_TOKEN` 的内容）

### 2) 启用 OpenClaw 需要的网关意图

Discord 阻止"特权意图"，除非您显式启用它们。

在 **Bot** → **Privileged Gateway Intents** 中，启用：

- **Message Content Intent**（需要在大多数公会中读取消息文本；没有它您会看到"Used disallowed intents"或机器人将连接但不响应消息）
- **Server Members Intent**（推荐；需要某些成员/用户查找和公会中的允许列表匹配）

您通常**不需要****Presence Intent**。

### 3) 生成邀请 URL（OAuth2 URL 生成器）

在您的应用中：**OAuth2** → **URL Generator**

**Scopes**

- ✅ `bot`
- ✅ `applications.commands`（原生命令需要）

**Bot Permissions**（最小基线）

- ✅ View Channels
- ✅ Send Messages
- ✅ Read Message History
- ✅ Embed Links
- ✅ Attach Files
- ✅ Add Reactions（可选但推荐）
- ✅ Use External Emojis / Stickers（可选；仅在需要时使用）

除非您在调试并完全信任机器人，否则避免**Administrator**。

复制生成的 URL，打开它，选择您的服务器，并安装机器人。

### 4) 获取 id（公会/用户/频道）

Discord 到处使用数字 id；OpenClaw 配置更喜欢 id。

1. Discord（桌面/网页）→ **User Settings** → **Advanced** → 启用 **Developer Mode**
2. 右键点击：
   - 服务器名称 → **Copy Server ID**（公会 id）
   - 频道（例如 `#help`）→ **Copy Channel ID**
   - 您的用户 → **Copy User ID**

### 5) 配置 OpenClaw

#### 令牌

通过环境变量设置机器人令牌（服务器上推荐）：

- `DISCORD_BOT_TOKEN=...`

或通过配置：

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN",
    },
  },
}
```

多账户支持：使用 `channels.discord.accounts` 配合每个账户的令牌和可选的 `name`。共享模式参见 [`gateway/configuration`](/gateway/configuration#telegramaccounts--discordaccounts--slackaccounts--signalaccounts--imessageaccounts)。

#### 允许列表 + 频道路由

示例"单服务器，仅允许我，仅允许 #help"：

```json5
{
  channels: {
    discord: {
      enabled: true,
      dm: { enabled: false },
      guilds: {
        YOUR_GUILD_ID: {
          users: ["YOUR_USER_ID"],
          requireMention: true,
          channels: {
            help: { allow: true, requireMention: true },
          },
        },
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

注意：

- `requireMention: true` 意味着机器人仅在被提及时回复（共享频道推荐）。
- `agents.list[].groupChat.mentionPatterns`（或 `messages.groupChat.mentionPatterns`）也算作公会消息的提及。
- 多代理覆盖：在 `agents.list[].groupChat.mentionPatterns` 上设置每个代理的模式。
- 如果 `channels` 存在，任何未列出的频道默认被拒绝。
- 使用 `"*"` 频道条目将默认值应用于所有频道；显式频道条目覆盖通配符。
- 线程继承父频道配置（允许列表、`requireMention`、skills、prompts 等），除非您显式添加线程频道 id。
- 机器人撰写的消息默认被忽略；设置 `channels.discord.allowBots=true` 以允许它们（自己的消息保持过滤）。
- 警告：如果您允许回复其他机器人（`channels.discord.allowBots=true`），使用 `requireMention`、`channels.discord.guilds.*.channels.<id>.users` 允许列表，和/或 `AGENTS.md` 和 `SOUL.md` 中的清晰护栏防止机器人到机器人回复循环。

### 6) 验证它工作

1. 启动网关。
2. 在您的服务器频道中，发送：`@Krill hello`（或您的机器人名称）。
3. 如果没有反应：检查下面的**故障排除**。

### 故障排除

- 首先：运行 `openclaw doctor` 和 `openclaw channels status --probe`（可操作的警告 + 快速审计）。
- **"Used disallowed intents"**：在开发者门户中启用**Message Content Intent**（和可能的**Server Members Intent**），然后重启网关。
- **机器人连接但在公会频道中从不回复**：
  - 缺少**Message Content Intent**，或
  - 机器人缺少频道权限（查看/发送/读取历史），或
  - 您的配置需要提及而您没有提及它，或
  - 您的公会/频道允许列表拒绝频道/用户。
- **`requireMention: false` 但仍然没有回复**：
- `channels.discord.groupPolicy` 默认为**allowlist**；将其设置为 `"open"` 或在 `channels.discord.guilds` 下添加公会条目（可选地在 `channels.discord.guilds.<id>.channels` 下列出频道以限制）。
  - 如果您只设置 `DISCORD_BOT_TOKEN` 而从未创建 `channels.discord` 部分，运行时
    默认 `groupPolicy` 为 `open`。添加 `channels.discord.groupPolicy`、
    `channels.defaults.groupPolicy` 或公会/频道允许列表以锁定它。
- `requireMention` 必须位于 `channels.discord.guilds` 下（或特定频道）。顶层的 `channels.discord.requireMention` 被忽略。
- **权限审计**（`channels status --probe`）仅检查数字频道 ID。如果您使用 slugs/名称作为 `channels.discord.guilds.*.channels` 键，审计无法验证权限。
- **私信不起作用**：`channels.discord.dm.enabled=false`、`channels.discord.dm.policy="disabled"`，或您尚未被批准（`channels.discord.dm.policy="pairing"`）。

## 功能和限制

- 私信和公会文字频道（线程被视为单独频道；不支持语音）。
- 打字指示器尽力发送；消息分块使用 `channels.discord.textChunkLimit`（默认 2000）并按行数分割高回复（`channels.discord.maxLinesPerMessage`，默认 17）。
- 可选换行分块：设置 `channels.discord.chunkMode="newline"` 以在空行（段落边界）处分割，然后进行长度分块。
- 文件上传支持高达配置的 `channels.discord.mediaMaxMb`（默认 8 MB）。
- 默认提及门控公会回复以避免嘈杂的机器人。
- 当消息引用另一条消息时注入回复上下文（引用内容 + id）。
- 原生回复线程**默认关闭**；使用 `channels.discord.replyToMode` 和回复标签启用。

## 重试策略

出站 Discord API 调用在速率限制（429）上使用 Discord `retry_after`（可用时）重试，使用指数退避和抖动。通过 `channels.discord.retry` 配置。参见 [重试策略](/concepts/retry)。

## 配置

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "abc.123",
      groupPolicy: "allowlist",
      guilds: {
        "*": {
          channels: {
            general: { allow: true },
          },
        },
      },
      mediaMaxMb: 8,
      actions: {
        reactions: true,
        stickers: true,
        emojiUploads: true,
        stickerUploads: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        channels: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off",
      dm: {
        enabled: true,
        policy: "pairing", // pairing | allowlist | open | disabled
        allowFrom: ["123456789012345678", "steipete"],
        groupEnabled: false,
        groupChannels: ["openclaw-dm"],
      },
      guilds: {
        "*": { requireMention: true },
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          reactionNotifications: "own",
          users: ["987654321098765432", "steipete"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["search", "docs"],
              systemPrompt: "Keep answers short.",
            },
          },
        },
      },
    },
  },
}
```

确认反应通过 `messages.ackReaction` +
`messages.ackReactionScope` 全局控制。使用 `messages.removeAckAfterReply` 在
机器人回复后清除确认反应。

- `dm.enabled`：设置为 `false` 以忽略所有私信（默认 `true`）。
- `dm.policy`：私信访问控制（推荐 `pairing`）。`"open"` 需要 `dm.allowFrom=["*"]`。
- `dm.allowFrom`：私信允许列表（用户 id 或名称）。由 `dm.policy="allowlist"` 使用，并用于 `dm.policy="open"` 验证。向导接受用户名并在机器人可以搜索成员时解析为 id。
- `dm.groupEnabled`：启用群组私信（默认 `false`）。
- `dm.groupChannels`：群组私信频道 id 或 slugs 的可选允许列表。
- `groupPolicy`：控制公会频道处理（`open|disabled|allowlist`）；`allowlist` 需要频道允许列表。
- `guilds`：按公会 id（首选）或 slug 的每个公会规则。
- `guilds."*"`：当没有显式条目时应用的默认每个公会设置。
- `guilds.<id>.slug`：用于显示名称的可选友好 slug。
- `guilds.<id>.users`：可选的每个公会用户允许列表（id 或名称）。
- `guilds.<id>.tools`：可选的每个公会工具策略覆盖（`allow`/`deny`/`alsoAllow`），当频道覆盖缺失时使用。
- `guilds.<id>.toolsBySender`：可选的公会级别每个发送者工具策略覆盖（当频道覆盖缺失时应用；支持 `"*"` 通配符）。
- `guilds.<id>.channels.<channel>.allow`：当 `groupPolicy="allowlist"` 时允许/拒绝频道。
- `guilds.<id>.channels.<channel>.requireMention`：频道的提及门控。
- `guilds.<id>.channels.<channel>.tools`：可选的每个频道工具策略覆盖（`allow`/`deny`/`alsoAllow`）。
- `guilds.<id>.channels.<channel>.toolsBySender`：可选的频道内每个发送者工具策略覆盖（支持 `"*"` 通配符）。
- `guilds.<id>.channels.<channel>.users`：可选的每个频道用户允许列表。
- `guilds.<id>.channels.<channel>.skills`：技能过滤器（省略 = 所有技能，空 = 无）。
- `guilds.<id>.channels.<channel>.systemPrompt`：频道的额外系统提示（与频道主题组合）。
- `guilds.<id>.channels.<channel>.enabled`：设置为 `false` 以禁用频道。
- `guilds.<id>.channels`：频道规则（键是频道 slugs 或 id）。
- `guilds.<id>.requireMention`：每个公会的提及要求（可按频道覆盖）。
- `guilds.<id>.reactionNotifications`：反应系统事件模式（`off`、`own`、`all`、`allowlist`）。
- `textChunkLimit`：出站文本块大小（字符）。默认：2000。
- `chunkMode`：`length`（默认）仅在超过 `textChunkLimit` 时分割；`newline` 在空行（段落边界）处分割，然后进行长度分块。
- `maxLinesPerMessage`：每条消息的软最大行数。默认：17。
- `mediaMaxMb`：限制保存到磁盘的入站媒体。
- `historyLimit`：回复提及时要包含作为上下文的最近公会消息数（默认 20；回退到 `messages.groupChat.historyLimit`；`0` 禁用）。
- `dmHistoryLimit`：私信历史限制（用户轮次）。每个用户的覆盖：`dms["<user_id>"].historyLimit`。
- `retry`：出站 Discord API 调用的重试策略（尝试次数、minDelayMs、maxDelayMs、抖动）。
- `pluralkit`：解析 PluralKit 代理消息，使系统成员显示为不同的发送者。
- `actions`：每个操作的工具门控；省略以允许所有（设置为 `false` 以禁用）。
  - `reactions`（涵盖 react + 读取反应）
  - `stickers`、`emojiUploads`、`stickerUploads`、`polls`、`permissions`、`messages`、`threads`、`pins`、`search`
  - `memberInfo`、`roleInfo`、`channelInfo`、`voiceStatus`、`events`
  - `channels`（创建/编辑/删除频道 + 分类 + 权限）
  - `roles`（角色添加/移除，默认 `false`）
  - `moderation`（超时/踢出/禁止，默认 `false`）

反应通知使用 `guilds.<id>.reactionNotifications`：

- `off`：无反应事件。
- `own`：对机器人自己消息的反应（默认）。
- `all`：对所有消息的所有反应。
- `allowlist`：来自 `guilds.<id>.users` 对所有消息的反应（空列表禁用）。

### PluralKit (PK) 支持

启用 PK 查找，使代理消息解析到底层系统 + 成员。
启用时，OpenClaw 使用成员身份进行允许列表，并将
发送者标记为 `Member (PK:System)` 以避免意外的 Discord ping。

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // 可选；私有系统需要
      },
    },
  },
}
```

允许列表注意（启用 PK）：

- 在 `dm.allowFrom`、`guilds.<id>.users` 或每个频道 `users` 中使用 `pk:<memberId>`。
- 成员显示名称也通过名称/slug 匹配。
- 查找使用**原始** Discord 消息 ID（代理前消息），因此
  PK API 仅在其 30 分钟窗口内解析它。
- 如果 PK 查找失败（例如，没有令牌的私有系统），代理消息
  被视为机器人消息，除非 `channels.discord.allowBots=true`，否则被丢弃。

### 工具操作默认值

| 操作组         | 默认     | 说明                              |
| -------------- | -------- | --------------------------------- |
| reactions      | enabled  | React + 列出反应 + emojiList      |
| stickers       | enabled  | 发送贴纸                          |
| emojiUploads   | enabled  | 上传表情符号                      |
| stickerUploads | enabled  | 上传贴纸                          |
| polls          | enabled  | 创建投票                          |
| permissions    | enabled  | 频道权限快照                      |
| messages       | enabled  | 读取/发送/编辑/删除               |
| threads        | enabled  | 创建/列出/回复                    |
| pins           | enabled  | 置顶/取消置顶/列出                |
| search         | enabled  | 消息搜索（预览功能）              |
| memberInfo     | enabled  | 成员信息                          |
| roleInfo       | enabled  | 角色列表                          |
| channelInfo    | enabled  | 频道信息 + 列表                   |
| channels       | enabled  | 频道/分类管理                     |
| voiceStatus    | enabled  | 语音状态查找                      |
| events         | enabled  | 列出/创建计划事件                 |
| roles          | disabled | 角色添加/移除                     |
| moderation     | disabled | 超时/踢出/禁止                    |

- `replyToMode`：`off`（默认）、`first` 或 `all`。仅在模型包含回复标签时应用。

## 回复标签

要请求线程回复，模型可以在其输出中包含一个标签：

- `[[reply_to_current]]` — 回复触发 Discord 消息。
- `[[reply_to:<id>]]` — 回复来自上下文/历史的特定消息 id。
  当前消息 id 作为 `[message_id: …]` 附加到提示；历史条目已包含 id。

由 `channels.discord.replyToMode` 控制：

- `off`：忽略标签。
- `first`：仅第一个出站块/附件是回复。
- `all`：每个出站块/附件都是回复。

允许列表匹配注意：

- `allowFrom`/`users`/`groupChannels` 接受 id、名称、标签或类似 `<@id>` 的提及。
- 支持类似 `discord:`/`user:`（用户）和 `channel:`（群组私信）的前缀。
- 使用 `*` 允许任何发送者/频道。
- 当 `guilds.<id>.channels` 存在时，未列出的频道默认被拒绝。
- 当 `guilds.<id>.channels` 省略时，允许列表中的公会中的所有频道都被允许。
- 要**不允许任何频道**，设置 `channels.discord.groupPolicy: "disabled"`（或保持空允许列表）。
- 配置向导接受 `Guild/Channel` 名称（公共 + 私有）并在可能时解析为 ID。
- 启动时，OpenClaw 在机器人可以搜索成员时将允许列表中的频道/用户名称解析为 ID
  并记录映射；未解析的条目保持原样。

原生命令注意：

- 注册的命令镜像 OpenClaw 的聊天命令。
- 原生命令遵守与私信/公会消息相同的允许列表（`channels.discord.dm.allowFrom`、`channels.discord.guilds`、每个频道规则）。
- 斜杠命令仍可能对未列入允许列表的用户在 Discord UI 中可见；OpenClaw 在执行时强制执行允许列表并回复"not authorized"。

## 工具操作

代理可以调用 `discord` 执行操作，如：

- `react` / `reactions`（添加或列出反应）
- `sticker`、`poll`、`permissions`
- `readMessages`、`sendMessage`、`editMessage`、`deleteMessage`
- 读取/搜索/置顶工具负载包含规范化的 `timestampMs`（UTC epoch ms）和 `timestampUtc` 以及原始 Discord `timestamp`。
- `threadCreate`、`threadList`、`threadReply`
- `pinMessage`、`unpinMessage`、`listPins`
- `searchMessages`、`memberInfo`、`roleInfo`、`roleAdd`、`roleRemove`、`emojiList`
- `channelInfo`、`channelList`、`voiceStatus`、`eventList`、`eventCreate`
- `timeout`、`kick`、`ban`

Discord 消息 id 在注入的上下文中显示（`[discord message id: …]` 和历史行），因此代理可以定位它们。
表情符号可以是 unicode（例如 `✅`）或自定义表情符号语法如 `<:party_blob:1234567890>`。

## 安全与运维

- 将机器人令牌视为密码；在受监督的主机上优先使用 `DISCORD_BOT_TOKEN` 环境变量或锁定配置文件权限。
- 仅授予机器人需要的权限（通常是读取/发送消息）。
- 如果机器人卡住或速率受限，在确认没有其他进程拥有 Discord 会话后重启网关（`openclaw gateway --force`）。

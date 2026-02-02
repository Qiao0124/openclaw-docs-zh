---
summary: "~/.openclaw/openclaw.json 的所有配置选项及示例"
read_when:
  - 添加或修改配置字段
title: "配置"
---

# 配置 🔧

OpenClaw 从 `~/.openclaw/openclaw.json` 读取可选的 **JSON5** 配置（允许注释和尾随逗号）。

如果文件缺失，OpenClaw 使用安全的默认设置（嵌入式 Pi agent + 按发送者隔离的 session + 工作目录 `~/.openclaw/workspace`）。通常只有在以下情况才需要配置：

- 限制谁可以触发机器人（`channels.whatsapp.allowFrom`、`channels.telegram.allowFrom` 等）
- 控制群组白名单和提及行为（`channels.whatsapp.groups`、`channels.telegram.groups`、`channels.discord.guilds`、`channels.googlechat.groups`、`agents.list[].groupChat`）
- 自定义消息前缀（`messages`）
- 设置 agent 的工作目录（`agents.defaults.workspace` 或 `agents.list[].workspace`）
- 调整嵌入式 agent 默认值（`agents.defaults`）和 session 行为（`session`）
- 设置每个 agent 的身份（`agents.list[].identity`）

> **初次接触配置？** 查看 [配置示例](/gateway/configuration-examples) 指南获取完整示例和详细说明！

## 严格的配置验证

OpenClaw 只接受完全符合 schema 的配置。
未知的键、格式错误的类型或无效的值会导致 Gateway **拒绝启动**，以确保安全。

验证失败时：

- Gateway 不会启动。
- 只允许诊断命令（例如：`openclaw doctor`、`openclaw logs`、`openclaw health`、`openclaw status`、`openclaw service`、`openclaw help`）。
- 运行 `openclaw doctor` 查看具体问题。
- 运行 `openclaw doctor --fix`（或 `--yes`）应用迁移/修复。

Doctor 只有在您明确选择 `--fix`/`--yes` 时才会写入更改。

## Schema + UI 提示

Gateway 通过 `config.schema` 暴露配置的 JSON Schema 表示，供 UI 编辑器使用。
Control UI 从此 schema 渲染表单，并提供 **Raw JSON** 编辑器作为备用方案。

Channel 插件和扩展可以为它们的配置注册 schema 和 UI 提示，因此 channel 设置
可以在不同应用中保持 schema 驱动，无需硬编码表单。

提示（标签、分组、敏感字段）随 schema 一起提供，因此客户端可以渲染
更好的表单，无需硬编码配置知识。

## Apply + restart (RPC)

使用 `config.apply` 验证并写入完整配置，然后一步重启 Gateway。
它会写入重启标记文件，并在 Gateway 恢复后 ping 最后活动的 session。

警告：`config.apply` 会替换**整个配置**。如果只想更改几个键，
请使用 `config.patch` 或 `openclaw config set`。请备份 `~/.openclaw/openclaw.json`。

参数：

- `raw` (string) — 整个配置的 JSON5 负载
- `baseHash` (可选) — 来自 `config.get` 的配置哈希（当配置已存在时需要）
- `sessionKey` (可选) — 用于唤醒 ping 的最后活动 session 键
- `note` (可选) — 包含在重启标记文件中的备注
- `restartDelayMs` (可选) — 重启前的延迟（默认 2000）

示例（通过 `gateway call`）：

```bash
openclaw gateway call config.get --params '{}' # 捕获 payload.hash
openclaw gateway call config.apply --params '{
  "raw": "{\\n  agents: { defaults: { workspace: \\"~/.openclaw/workspace\\" } }\\n}\\n",
  "baseHash": "<hash-from-config.get>",
  "sessionKey": "agent:main:whatsapp:dm:+15555550123",
  "restartDelayMs": 1000
}'
```

## 部分更新 (RPC)

使用 `config.patch` 将部分更新合并到现有配置，而不会覆盖
不相关的键。它应用 JSON merge patch 语义：

- 对象递归合并
- `null` 删除键
- 数组替换
  与 `config.apply` 类似，它会验证、写入配置、存储重启标记文件，并安排
  Gateway 重启（当提供 `sessionKey` 时可选唤醒）。

参数：

- `raw` (string) — 仅包含要更改的键的 JSON5 负载
- `baseHash` (必需) — 来自 `config.get` 的配置哈希
- `sessionKey` (可选) — 用于唤醒 ping 的最后活动 session 键
- `note` (可选) — 包含在重启标记文件中的备注
- `restartDelayMs` (可选) — 重启前的延迟（默认 2000）

示例：

```bash
openclaw gateway call config.get --params '{}' # 捕获 payload.hash
openclaw gateway call config.patch --params '{
  "raw": "{\\n  channels: { telegram: { groups: { \\"*\\": { requireMention: false } } } }\\n}\\n",
  "baseHash": "<hash-from-config.get>",
  "sessionKey": "agent:main:whatsapp:dm:+15555550123",
  "restartDelayMs": 1000
}'
```

## 最小配置（推荐起点）

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

使用以下命令一次性构建默认镜像：

```bash
scripts/sandbox-setup.sh
```

## 自聊模式（推荐用于群组控制）

要防止机器人在 WhatsApp 群组中响应 @-提及（仅响应特定文本触发器）：

```json5
{
  agents: {
    defaults: { workspace: "~/.openclaw/workspace" },
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["@openclaw", "reisponde"] },
      },
    ],
  },
  channels: {
    whatsapp: {
      // 白名单仅限私信；包含您自己的号码可启用自聊模式。
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
}
```

## 配置包含 (`$include`)

使用 `$include` 指令将配置拆分为多个文件。这适用于：

- 组织大型配置（例如，按客户端的 agent 定义）
- 跨环境共享通用设置
- 将敏感配置分开保存

### 基本用法

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },

  // 包含单个文件（替换键的值）
  agents: { $include: "./agents.json5" },

  // 包含多个文件（按顺序深度合并）
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

```json5
// ~/.openclaw/agents.json5
{
  defaults: { sandbox: { mode: "all", scope: "session" } },
  list: [{ id: "main", workspace: "~/.openclaw/workspace" }],
}
```

### 合并行为

- **单个文件**：替换包含 `$include` 的对象
- **文件数组**：按顺序深度合并文件（后面的文件覆盖前面的）
- **与兄弟键共存**：兄弟键在包含后合并（覆盖包含的值）
- **兄弟键 + 数组/基本类型**：不支持（包含的内容必须是对象）

```json5
// 兄弟键覆盖包含的值
{
  $include: "./base.json5", // { a: 1, b: 2 }
  b: 99, // 结果: { a: 1, b: 99 }
}
```

### 嵌套包含

包含的文件本身可以包含 `$include` 指令（最多 10 层深度）：

```json5
// clients/mueller.json5
{
  agents: { $include: "./mueller/agents.json5" },
  broadcast: { $include: "./mueller/broadcast.json5" },
}
```

### 路径解析

- **相对路径**：相对于包含文件解析
- **绝对路径**：按原样使用
- **父目录**：`../` 引用按预期工作

```json5
{ "$include": "./sub/config.json5" }      // 相对
{ "$include": "/etc/openclaw/base.json5" } // 绝对
{ "$include": "../shared/common.json5" }   // 父目录
```

### 错误处理

- **文件缺失**：清晰的错误并显示解析后的路径
- **解析错误**：显示哪个包含的文件失败
- **循环包含**：检测并报告包含链

### 示例：多客户端法律设置

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789, auth: { token: "secret" } },

  // 通用 agent 默认值
  agents: {
    defaults: {
      sandbox: { mode: "all", scope: "session" },
    },
    // 合并所有客户端的 agent 列表
    list: { $include: ["./clients/mueller/agents.json5", "./clients/schmidt/agents.json5"] },
  },

  // 合并广播配置
  broadcast: {
    $include: ["./clients/mueller/broadcast.json5", "./clients/schmidt/broadcast.json5"],
  },

  channels: { whatsapp: { groupPolicy: "allowlist" } },
}
```

```json5
// ~/.openclaw/clients/mueller/agents.json5
[
  { id: "mueller-transcribe", workspace: "~/clients/mueller/transcribe" },
  { id: "mueller-docs", workspace: "~/clients/mueller/docs" },
]
```

```json5
// ~/.openclaw/clients/mueller/broadcast.json5
{
  "120363403215116621@g.us": ["mueller-transcribe", "mueller-docs"],
}
```

## 常用选项

### 环境变量 + `.env`

OpenClaw 从父进程（shell、launchd/systemd、CI 等）读取环境变量。

此外，它加载：

- 当前工作目录的 `.env`（如果存在）
- 全局备用 `.env` 从 `~/.openclaw/.env`（即 `$OPENCLAW_STATE_DIR/.env`）

两个 `.env` 文件都不会覆盖现有的环境变量。

您还可以在配置中提供内联环境变量。这些仅在
进程环境缺少该键时应用（相同的非覆盖规则）：

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
  },
}
```

完整优先级和来源请参阅 [/environment](/environment)。

### `env.shellEnv` (可选)

选择加入的便利功能：如果启用且尚未设置任何预期键，OpenClaw 会运行您的登录 shell 并仅导入缺失的预期键（从不覆盖）。
这实际上会 source 您的 shell 配置文件。

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

环境变量等效项：

- `OPENCLAW_LOAD_SHELL_ENV=1`
- `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`

### 配置中的环境变量替换

您可以使用 `${VAR_NAME}` 语法在任何配置字符串值中直接引用环境变量。变量在配置加载时、验证之前替换。

```json5
{
  models: {
    providers: {
      "vercel-gateway": {
        apiKey: "${VERCEL_GATEWAY_API_KEY}",
      },
    },
  },
  gateway: {
    auth: {
      token: "${OPENCLAW_GATEWAY_TOKEN}",
    },
  },
}
```

**规则：**

- 仅匹配大写环境变量名：`[A-Z_][A-Z0-9_]*`
- 缺失或空的环境变量在配置加载时抛出错误
- 使用 `$${VAR}` 转义以输出字面量 `${VAR}`
- 与 `$include` 一起工作（包含的文件也会进行替换）

**内联替换：**

```json5
{
  models: {
    providers: {
      custom: {
        baseUrl: "${CUSTOM_API_BASE}/v1", // → "https://api.example.com/v1"
      },
    },
  },
}
```

### 认证存储 (OAuth + API keys)

OpenClaw 将**每个 agent** 的认证配置文件（OAuth + API keys）存储在：

- `<agentDir>/auth-profiles.json`（默认：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`）

另请参阅：[/concepts/oauth](/concepts/oauth)

旧版 OAuth 导入：

- `~/.openclaw/credentials/oauth.json`（或 `$OPENCLAW_STATE_DIR/credentials/oauth.json`）

嵌入式 Pi agent 在以下位置维护运行时缓存：

- `<agentDir>/auth.json`（自动管理；请勿手动编辑）

旧版 agent 目录（多 agent 之前）：

- `~/.openclaw/agent/*`（由 `openclaw doctor` 迁移到 `~/.openclaw/agents/<defaultAgentId>/agent/*`）

覆盖：

- OAuth 目录（仅旧版导入）：`OPENCLAW_OAUTH_DIR`
- Agent 目录（默认 agent 根目录覆盖）：`OPENCLAW_AGENT_DIR`（推荐），`PI_CODING_AGENT_DIR`（旧版）

首次使用时，OpenClaw 将 `oauth.json` 条目导入 `auth-profiles.json`。

### `auth`

认证配置文件的可选元数据。这**不**存储密钥；它映射
配置文件 ID 到提供商 + 模式（和可选电子邮件）并定义用于故障转移的提供商
轮换顺序。

```json5
{
  auth: {
    profiles: {
      "anthropic:me@example.com": { provider: "anthropic", mode: "oauth", email: "me@example.com" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
    },
    order: {
      anthropic: ["anthropic:me@example.com", "anthropic:work"],
    },
  },
}
```

### `agents.list[].identity`

用于默认值和 UX 的可选每个 agent 身份。这由 macOS 入门助手写入。

如果设置，OpenClaw 派生默认值（仅当您未显式设置它们时）：

- `messages.ackReaction` 来自**活动 agent**的 `identity.emoji`（回退到 👀）
- `agents.list[].groupChat.mentionPatterns` 来自 agent 的 `identity.name`/`identity.emoji`（因此 "@Samantha" 在 Telegram/Slack/Discord/Google Chat/iMessage/WhatsApp 的群组中有效）
- `identity.avatar` 接受相对于工作目录的图片路径或远程 URL/data URL。本地文件必须位于 agent 工作目录内。

`identity.avatar` 接受：

- 相对于工作目录的路径（必须位于 agent 工作目录内）
- `http(s)` URL
- `data:` URI

```json5
{
  agents: {
    list: [
      {
        id: "main",
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
      },
    ],
  },
}
```

### `wizard`

CLI 向导（`onboard`、`configure`、`doctor`）写入的元数据。

```json5
{
  wizard: {
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
  },
}
```

### `logging`

- 默认日志文件：`/tmp/openclaw/openclaw-YYYY-MM-DD.log`
- 如果需要稳定的路径，将 `logging.file` 设置为 `/tmp/openclaw/openclaw.log`。
- 控制台输出可通过以下方式单独调整：
  - `logging.consoleLevel`（默认为 `info`，`--verbose` 时提升为 `debug`）
  - `logging.consoleStyle`（`pretty` | `compact` | `json`）
- 工具摘要可以编辑以避免泄露密钥：
  - `logging.redactSensitive`（`off` | `tools`，默认：`tools`）
  - `logging.redactPatterns`（正则表达式字符串数组；覆盖默认值）

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty",
    redactSensitive: "tools",
    redactPatterns: [
      // 示例：用您自己的规则覆盖默认值。
      "\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1",
      "/\\bsk-[A-Za-z0-9_-]{8,}\\b/gi",
    ],
  },
}
```

### `channels.whatsapp.dmPolicy`

控制 WhatsApp 私信（DMs）的处理方式：

- `"pairing"` (默认)：未知发送者获得配对码；所有者必须批准
- `"allowlist"`：仅允许 `channels.whatsapp.allowFrom`（或配对允许存储）中的发送者
- `"open"`：允许所有入站私信（**需要** `channels.whatsapp.allowFrom` 包含 `"*"`）
- `"disabled"`：忽略所有入站私信

配对码在 1 小时后过期；机器人仅在新请求创建时发送配对码。待处理的私信配对请求默认限制为**每个 channel 3 个**。

配对批准：

- `openclaw pairing list whatsapp`
- `openclaw pairing approve whatsapp <code>`

### `channels.whatsapp.allowFrom`

允许触发 WhatsApp 自动回复的 E.164 电话号码白名单（**仅限私信**）。
如果为空且 `channels.whatsapp.dmPolicy="pairing"`，未知发送者将收到配对码。
对于群组，使用 `channels.whatsapp.groupPolicy` + `channels.whatsapp.groupAllowFrom`。

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000, // 可选出站分块大小（字符）
      chunkMode: "length", // 可选分块模式（length | newline）
      mediaMaxMb: 50, // 可选入站媒体上限（MB）
    },
  },
}
```

### `channels.whatsapp.sendReadReceipts`

控制入站 WhatsApp 消息是否标记为已读（蓝色对勾）。默认：`true`。

自聊模式始终跳过已读回执，即使已启用。

每个账户覆盖：`channels.whatsapp.accounts.<id>.sendReadReceipts`。

```json5
{
  channels: {
    whatsapp: { sendReadReceipts: false },
  },
}
```

### `channels.whatsapp.accounts` (多账户)

在一个 gateway 中运行多个 WhatsApp 账户：

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {}, // 可选；保持默认 id 稳定
        personal: {},
        biz: {
          // 可选覆盖。默认：~/.openclaw/credentials/whatsapp/biz
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

注意：

- 出站命令默认使用账户 `default`（如果存在）；否则使用第一个配置的账户 id（排序后）。
- 旧版单账户 Baileys 认证目录由 `openclaw doctor` 迁移到 `whatsapp/default`。

### `channels.telegram.accounts` / `channels.discord.accounts` / `channels.googlechat.accounts` / `channels.slack.accounts` / `channels.mattermost.accounts` / `channels.signal.accounts` / `channels.imessage.accounts`

每个 channel 运行多个账户（每个账户有自己的 `accountId` 和可选的 `name`）：

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

注意：

- 省略 `accountId` 时使用 `default`（CLI + 路由）。
- 环境变量 token 仅适用于**默认**账户。
- 基本 channel 设置（群组策略、提及门控等）适用于所有账户，除非每个账户覆盖。
- 使用 `bindings[].match.accountId` 将每个账户路由到不同的 agents.defaults。

### 群组聊天提及门控 (`agents.list[].groupChat` + `messages.groupChat`)

群组消息默认**需要提及**（元数据提及或正则表达式模式）。适用于 WhatsApp、Telegram、Discord、Google Chat 和 iMessage 群组聊天。

**提及类型：**

- **元数据提及**：原生平台 @-提及（例如，WhatsApp 点击提及）。在 WhatsApp 自聊模式下忽略（参见 `channels.whatsapp.allowFrom`）。
- **文本模式**：`agents.list[].groupChat.mentionPatterns` 中定义的正则表达式模式。始终检查，无论自聊模式如何。
- 仅在可以检测提及时才强制执行提及门控（原生提及或至少一个 `mentionPattern`）。

```json5
{
  messages: {
    groupChat: { historyLimit: 50 },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

`messages.groupChat.historyLimit` 设置群组历史上下文的全局默认值。Channel 可以用 `channels.<channel>.historyLimit`（或多账户的 `channels.<channel>.accounts.*.historyLimit`）覆盖。设置为 `0` 禁用历史包装。

#### 私信历史限制

私信对话使用 agent 管理的基于 session 的历史。您可以限制每个私信 session 保留的用户轮次数量：

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30, // 将私信 session 限制为 30 个用户轮次
      dms: {
        "123456789": { historyLimit: 50 }, // 每个用户覆盖（用户 ID）
      },
    },
  },
}
```

解析顺序：

1. 每个私信覆盖：`channels.<provider>.dms[userId].historyLimit`
2. 提供商默认：`channels.<provider>.dmHistoryLimit`
3. 无限制（保留所有历史）

支持的提供商：`telegram`、`whatsapp`、`discord`、`slack`、`signal`、`imessage`、`msteams`。

每个 agent 覆盖（设置时优先，即使是 `[]`）：

```json5
{
  agents: {
    list: [
      { id: "work", groupChat: { mentionPatterns: ["@workbot", "\\+15555550123"] } },
      { id: "personal", groupChat: { mentionPatterns: ["@homebot", "\\+15555550999"] } },
    ],
  },
}
```

提及门控默认值位于每个 channel（`channels.whatsapp.groups`、`channels.telegram.groups`、`channels.imessage.groups`、`channels.discord.guilds`）。设置 `*.groups` 时，它也充当群组白名单；包含 `"*"` 以允许所有群组。

仅响应**特定**文本触发器（忽略原生 @-提及）：

```json5
{
  channels: {
    whatsapp: {
      // 包含您自己的号码以启用自聊模式（忽略原生 @-提及）。
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          // 只有这些文本模式会触发响应
          mentionPatterns: ["reisponde", "@openclaw"],
        },
      },
    ],
  },
}
```

### 群组策略（每个 channel）

使用 `channels.*.groupPolicy` 控制是否接受群组/房间消息：

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["tg:123456789", "@alice"],
    },
    signal: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["chat_id:123"],
    },
    msteams: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["user@org.com"],
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        GUILD_ID: {
          channels: { help: { allow: true } },
        },
      },
    },
    slack: {
      groupPolicy: "allowlist",
      channels: { "#general": { allow: true } },
    },
  },
}
```

注意：

- `"open"`：群组绕过白名单；提及门控仍然适用。
- `"disabled"`：阻止所有群组/房间消息。
- `"allowlist"`：仅允许匹配配置白名单的群组/房间。
- `channels.defaults.groupPolicy` 在提供商的 `groupPolicy` 未设置时设置默认值。
- WhatsApp/Telegram/Signal/iMessage/Microsoft Teams 使用 `groupAllowFrom`（回退：显式 `allowFrom`）。
- Discord/Slack 使用 channel 白名单（`channels.discord.guilds.*.channels`、`channels.slack.channels`）。
- 群组私信（Discord/Slack）仍由 `dm.groupEnabled` + `dm.groupChannels` 控制。
- 默认为 `groupPolicy: "allowlist"`（除非被 `channels.defaults.groupPolicy` 覆盖）；如果未配置白名单，群组消息被阻止。

### 多 agent 路由 (`agents.list` + `bindings`)

在一个 Gateway 中运行多个隔离的 agent（独立的工作目录、`agentDir`、session）。
入站消息通过绑定路由到 agent。

- `agents.list[]`：每个 agent 的覆盖。
  - `id`：稳定的 agent id（必需）。
  - `default`：可选；当设置多个时，第一个获胜并记录警告。
    如果未设置，列表中的**第一个条目**是默认 agent。
  - `name`：agent 的显示名称。
  - `workspace`：默认 `~/.openclaw/workspace-<agentId>`（对于 `main`，回退到 `agents.defaults.workspace`）。
  - `agentDir`：默认 `~/.openclaw/agents/<agentId>/agent`。
  - `model`：每个 agent 的默认模型，覆盖该 agent 的 `agents.defaults.model`。
    - 字符串形式：`"provider/model"`，仅覆盖 `agents.defaults.model.primary`
    - 对象形式：`{ primary, fallbacks }`（fallbacks 覆盖 `agents.defaults.model.fallbacks`；`[]` 禁用该 agent 的全局 fallbacks）
  - `identity`：每个 agent 的名称/主题/表情符号（用于提及模式和确认反应）。
  - `groupChat`：每个 agent 的提及门控（`mentionPatterns`）。
  - `sandbox`：每个 agent 的 sandbox 配置（覆盖 `agents.defaults.sandbox`）。
    - `mode`：`"off"` | `"non-main"` | `"all"`
    - `workspaceAccess`：`"none"` | `"ro"` | `"rw"`
    - `scope`：`"session"` | `"agent"` | `"shared"`
    - `workspaceRoot`：自定义 sandbox 工作目录根
    - `docker`：每个 agent 的 docker 覆盖（例如 `image`、`network`、`env`、`setupCommand`、limits；`scope: "shared"` 时忽略）
    - `browser`：每个 agent 的 sandbox 浏览器覆盖（`scope: "shared"` 时忽略）
    - `prune`：每个 agent 的 sandbox 清理覆盖（`scope: "shared"` 时忽略）
  - `subagents`：每个 agent 的子 agent 默认值。
    - `allowAgents`：允许从此 agent 进行 `sessions_spawn` 的 agent id 白名单（`["*"]` = 允许任何；默认：仅相同 agent）
  - `tools`：每个 agent 的工具限制（在 sandbox 工具策略之前应用）。
    - `profile`：基础工具配置文件（在允许/拒绝之前应用）
    - `allow`：允许的工具名称数组
    - `deny`：拒绝的工具名称数组（拒绝优先）
- `agents.defaults`：共享的 agent 默认值（模型、工作目录、sandbox 等）。
- `bindings[]`：将入站消息路由到 `agentId`。
  - `match.channel`（必需）
  - `match.accountId`（可选；`*` = 任何账户；省略 = 默认账户）
  - `match.peer`（可选；`{ kind: dm|group|channel, id }`）
  - `match.guildId` / `match.teamId`（可选；特定于 channel）

确定性匹配顺序：

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId`（精确，无 peer/guild/team）
5. `match.accountId: "*"`（全 channel，无 peer/guild/team）
6. 默认 agent（`agents.list[].default`，否则列表中的第一个条目，否则 `"main"`）

在每个匹配层级内，`bindings` 中的第一个匹配条目获胜。

#### 每个 agent 的访问配置文件（多 agent）

每个 agent 可以携带自己的 sandbox + 工具策略。使用此功能在一个 gateway 中混合
访问级别：

- **完全访问**（个人 agent）
- **只读**工具 + 工作目录
- **无文件系统访问**（仅消息/session 工具）

有关优先级和更多示例，请参阅 [多 Agent Sandbox 和工具](/multi-agent-sandbox-tools)。

完全访问（无 sandbox）：

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

只读工具 + 只读工作目录：

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "ro",
        },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

无文件系统访问（启用消息/session 工具）：

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: {
          mode: "all",
          scope: "agent",
          workspaceAccess: "none",
        },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

示例：两个 WhatsApp 账户 → 两个 agent：

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
  channels: {
    whatsapp: {
      accounts: {
        personal: {},
        biz: {},
      },
    },
  },
}
```

### `tools.agentToAgent` (可选)

Agent 间消息传递是选择加入的：

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `messages.queue`

控制当 agent 运行已处于活动状态时入站消息的行为。

```json5
{
  messages: {
    queue: {
      mode: "collect", // steer | followup | collect | steer-backlog (steer+backlog ok) | interrupt (queue=steer legacy)
      debounceMs: 1000,
      cap: 20,
      drop: "summarize", // old | new | summarize
      byChannel: {
        whatsapp: "collect",
        telegram: "collect",
        discord: "collect",
        imessage: "collect",
        webchat: "collect",
      },
    },
  },
}
```

### `messages.inbound`

去抖动来自**同一发送者**的快速入站消息，使多个连续
消息成为单个 agent 轮次。去抖动按 channel + 对话范围，并使用最新消息进行回复线程/ID。

```json5
{
  messages: {
    inbound: {
      debounceMs: 2000, // 0 禁用
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
        discord: 1500,
      },
    },
  },
}
```

注意：

- 去抖动批量处理**仅文本**消息；媒体/附件立即刷新。
- 控制命令（例如 `/queue`、`/new`）绕过去抖动，因此它们保持独立。

### `commands`（聊天命令处理）

控制如何在连接器中启用聊天命令。

```json5
{
  commands: {
    native: "auto", // 支持时注册原生命令（auto）
    text: true, // 在聊天消息中解析斜杠命令
    bash: false, // 允许 !（别名：/bash）（仅限主机；需要 tools.elevated 白名单）
    bashForegroundMs: 2000, // bash 前台窗口（0 立即后台）
    config: false, // 允许 /config（写入磁盘）
    debug: false, // 允许 /debug（仅运行时覆盖）
    restart: false, // 允许 /restart + gateway 重启工具
    useAccessGroups: true, // 对命令强制执行访问组白名单/策略
  },
}
```

注意：

- 文本命令必须作为**独立**消息发送，并使用前导 `/`（无纯文本别名）。
- `commands.text: false` 禁用解析聊天消息中的命令。
- `commands.native: "auto"`（默认）为 Discord/Telegram 开启原生命令，Slack 保持关闭；不支持的 channel 保持仅文本。
- 设置 `commands.native: true|false` 以强制全部，或通过 `channels.discord.commands.native`、`channels.telegram.commands.native`、`channels.slack.commands.native`（bool 或 `"auto"`）按 channel 覆盖。`false` 在启动时清除 Discord/Telegram 上先前注册的命令；Slack 命令在 Slack 应用中管理。
- `channels.telegram.customCommands` 添加额外的 Telegram 机器人菜单条目。名称被规范化；与原生命令的冲突被忽略。
- `commands.bash: true` 启用 `! <cmd>` 运行主机 shell 命令（`/bash <cmd>` 也作为别名工作）。需要 `tools.elevated.enabled` 并在 `tools.elevated.allowFrom.<channel>` 中将发送者列入白名单。
- `commands.bashForegroundMs` 控制 bash 在后台运行前等待的时间。当 bash 作业运行时，新的 `! <cmd>` 请求被拒绝（一次一个）。
- `commands.config: true` 启用 `/config`（读取/写入 `openclaw.json`）。
- `channels.<provider>.configWrites` 控制该 channel 发起的配置变更（默认：true）。这适用于 `/config set|unset` 加上提供商特定的自动迁移（Telegram 超级组 ID 变更、Slack channel ID 变更）。
- `commands.debug: true` 启用 `/debug`（仅运行时覆盖）。
- `commands.restart: true` 启用 `/restart` 和 gateway 工具重启操作。
- `commands.useAccessGroups: false` 允许命令绕过访问组白名单/策略。
- 斜杠命令和指令仅对**授权发送者**有效。授权源自
  channel 白名单/配对加上 `commands.useAccessGroups`。

### `web`（WhatsApp web channel 运行时）

WhatsApp 通过 gateway 的 web channel（Baileys Web）运行。当存在链接的 session 时自动启动。
设置 `web.enabled: false` 以默认保持关闭。

```json5
{
  web: {
    enabled: true,
    heartbeatSeconds: 60,
    reconnect: {
      initialMs: 2000,
      maxMs: 120000,
      factor: 1.4,
      jitter: 0.2,
      maxAttempts: 0,
    },
  },
}
```

### `channels.telegram`（机器人传输）

OpenClaw 仅在存在 `channels.telegram` 配置部分时启动 Telegram。机器人 token 从 `channels.telegram.botToken`（或 `channels.telegram.tokenFile`）解析，`TELEGRAM_BOT_TOKEN` 作为默认账户的备用。
设置 `channels.telegram.enabled: false` 以禁用自动启动。
多账户支持位于 `channels.telegram.accounts` 下（参见上面的多账户部分）。环境变量 token 仅适用于默认账户。
设置 `channels.telegram.configWrites: false` 以阻止 Telegram 发起的配置写入（包括超级组 ID 迁移和 `/config set|unset`）。

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["tg:123456789"], // 可选；"open" 需要 ["*"]
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "Stay on topic.",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
      historyLimit: 50, // 包含最后 N 条群组消息作为上下文（0 禁用）
      replyToMode: "first", // off | first | all
      linkPreview: true, // 切换出站链接预览
      streamMode: "partial", // off | partial | block（草稿流式传输；与块流式传输分开）
      draftChunk: {
        // 可选；仅用于 streamMode=block
        minChars: 200,
        maxChars: 800,
        breakPreference: "paragraph", // paragraph | newline | sentence
      },
      actions: { reactions: true, sendMessage: true }, // 工具操作门控（false 禁用）
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 5,
      retry: {
        // 出站重试策略
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        // 传输覆盖
        autoSelectFamily: false,
      },
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

草稿流式传输说明：

- 使用 Telegram `sendMessageDraft`（草稿气泡，不是真实消息）。
- 需要**私信主题**（私信中的 message_thread_id；机器人已启用主题）。
- `/reasoning stream` 将推理流式传输到草稿中，然后发送最终答案。
  重试策略默认和行为记录在 [重试策略](/concepts/retry) 中。

### `channels.discord`（机器人传输）

通过设置机器人 token 和可选门控配置 Discord 机器人：
多账户支持位于 `channels.discord.accounts` 下（参见上面的多账户部分）。环境变量 token 仅适用于默认账户。

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 8, // 限制入站媒体大小
      allowBots: false, // 允许机器人撰写的消息
      actions: {
        // 工具操作门控（false 禁用）
        reactions: true,
        stickers: true,
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
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all
      dm: {
        enabled: true, // false 时禁用所有私信
        policy: "pairing", // pairing | allowlist | open | disabled
        allowFrom: ["1234567890", "steipete"], // 可选私信白名单（"open" 需要 ["*"]）
        groupEnabled: false, // 启用群组私信
        groupChannels: ["openclaw-dm"], // 可选群组私信白名单
      },
      guilds: {
        "123456789012345678": {
          // 公会 id（推荐）或 slug
          slug: "friends-of-openclaw",
          requireMention: false, // 每个公会默认
          reactionNotifications: "own", // off | own | all | allowlist
          users: ["987654321098765432"], // 可选每个公会用户白名单
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "Short answers only.",
            },
          },
        },
      },
      historyLimit: 20, // 包含最后 N 条公会消息作为上下文
      textChunkLimit: 2000, // 可选出站文本分块大小（字符）
      chunkMode: "length", // 可选分块模式（length | newline）
      maxLinesPerMessage: 17, // 每条消息的软最大行数（Discord UI 裁剪）
      retry: {
        // 出站重试策略
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

OpenClaw 仅在存在 `channels.discord` 配置部分时启动 Discord。token 从 `channels.discord.token` 解析，`DISCORD_BOT_TOKEN` 作为默认账户的备用（除非 `channels.discord.enabled` 为 `false`）。为 cron/CLI 命令指定传递目标时使用 `user:<id>`（私信）或 `channel:<id>`（公会 channel）；裸数字 ID 有歧义并被拒绝。
公会 slug 是小写，空格替换为 `-`；channel 键使用 slugged channel 名称（无前导 `#`）。首选公会 id 作为键以避免重命名歧义。
默认情况下忽略机器人撰写的消息。使用 `channels.discord.allowBots` 启用（自己的消息仍被过滤以防止自回复循环）。
反应通知模式：

- `off`：无反应事件。
- `own`：对机器人自己消息的反应（默认）。
- `all`：对所有消息的所有反应。
- `allowlist`：来自 `guilds.<id>.users` 对所有消息的反应（空列表禁用）。
  出站文本按 `channels.discord.textChunkLimit`（默认 2000）分块。设置 `channels.discord.chunkMode="newline"` 以在长度分块之前在空行（段落边界）处分割。Discord 客户端可以裁剪非常高的消息，因此 `channels.discord.maxLinesPerMessage`（默认 17）即使在少于 2000 字符时也会分割长的多行回复。
  重试策略默认和行为记录在 [重试策略](/concepts/retry) 中。

### `channels.googlechat`（Chat API webhook）

Google Chat 通过应用级认证（服务账户）的 HTTP webhook 运行。
多账户支持位于 `channels.googlechat.accounts` 下（参见上面的多账户部分）。环境变量仅适用于默认账户。

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890", // 可选；改进提及检测
      dm: {
        enabled: true,
        policy: "pairing", // pairing | allowlist | open | disabled
        allowFrom: ["users/1234567890"], // 可选；"open" 需要 ["*"]
      },
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

注意：

- 服务账户 JSON 可以是内联（`serviceAccount`）或基于文件（`serviceAccountFile`）。
- 默认账户的环境变量备用：`GOOGLE_CHAT_SERVICE_ACCOUNT` 或 `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`。
- `audienceType` + `audience` 必须匹配 Chat 应用的 webhook 认证配置。
- 设置传递目标时使用 `spaces/<spaceId>` 或 `users/<userId|email>`。

### `channels.slack`（socket 模式）

Slack 在 Socket 模式下运行，需要机器人 token 和应用 token：

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      dm: {
        enabled: true,
        policy: "pairing", // pairing | allowlist | open | disabled
        allowFrom: ["U123", "U456", "*"], // 可选；"open" 需要 ["*"]
        groupEnabled: false,
        groupChannels: ["G123"],
      },
      channels: {
        C123: { allow: true, requireMention: true, allowBots: false },
        "#general": {
          allow: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "Short answers only.",
        },
      },
      historyLimit: 50, // 包含最后 N 条 channel/群组消息作为上下文（0 禁用）
      allowBots: false,
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["U123"],
      replyToMode: "off", // off | first | all
      thread: {
        historyScope: "thread", // thread | channel
        inheritParent: false,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      textChunkLimit: 4000,
      chunkMode: "length",
      mediaMaxMb: 20,
    },
  },
}
```

多账户支持位于 `channels.slack.accounts` 下（参见上面的多账户部分）。环境变量 token 仅适用于默认账户。

OpenClaw 在提供商启用且两个 token 都设置时启动 Slack（通过配置或 `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN`）。为 cron/CLI 命令指定传递目标时使用 `user:<id>`（私信）或 `channel:<id>`。
设置 `channels.slack.configWrites: false` 以阻止 Slack 发起的配置写入（包括 channel ID 迁移和 `/config set|unset`）。

默认情况下忽略机器人撰写的消息。使用 `channels.slack.allowBots` 或 `channels.slack.channels.<id>.allowBots` 启用。

反应通知模式：

- `off`：无反应事件。
- `own`：对机器人自己消息的反应（默认）。
- `all`：对所有消息的所有反应。
- `allowlist`：来自 `channels.slack.reactionAllowlist` 对所有消息的反应（空列表禁用）。

Thread session 隔离：

- `channels.slack.thread.historyScope` 控制 thread 历史是每个 thread（`thread`，默认）还是在 channel 中共享（`channel`）。
- `channels.slack.thread.inheritParent` 控制新 thread session 是否继承父 channel 记录（默认：false）。

Slack 操作组（门控 `slack` 工具操作）：
| 操作组 | 默认 | 说明 |
| --- | --- | --- |
| reactions | enabled | 反应 + 列出反应 |
| messages | enabled | 读取/发送/编辑/删除 |
| pins | enabled | 置顶/取消置顶/列表 |
| memberInfo | enabled | 成员信息 |
| emojiList | enabled | 自定义表情符号列表 |

### `channels.mattermost`（机器人 token）

Mattermost 作为插件提供，不与核心安装捆绑。
首先安装：`openclaw plugins install @openclaw/mattermost`（或 git checkout 中的 `./extensions/mattermost`）。

Mattermost 需要机器人 token 加上服务器的 base URL：

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

OpenClaw 在账户配置（机器人 token + base URL）并启用时启动 Mattermost。token + base URL 从 `channels.mattermost.botToken` + `channels.mattermost.baseUrl` 或默认账户的 `MATTERMOST_BOT_TOKEN` + `MATTERMOST_URL` 解析（除非 `channels.mattermost.enabled` 为 `false`）。

聊天模式：

- `oncall` (默认)：仅当 @mentioned 时响应 channel 消息。
- `onmessage`：响应每个 channel 消息。
- `onchar`：当消息以触发前缀开头时响应（`channels.mattermost.oncharPrefixes`，默认 `[">", "!"]`）。

访问控制：

- 默认私信：`channels.mattermost.dmPolicy="pairing"`（未知发送者获得配对码）。
- 公开私信：`channels.mattermost.dmPolicy="open"` 加上 `channels.mattermost.allowFrom=["*"]`。
- 群组：默认 `channels.mattermost.groupPolicy="allowlist"`（提及门控）。使用 `channels.mattermost.groupAllowFrom` 限制发送者。

多账户支持位于 `channels.mattermost.accounts` 下（参见上面的多账户部分）。环境变量仅适用于默认账户。
指定传递目标时使用 `channel:<id>` 或 `user:<id>`（或 `@username`）；裸 id 被视为 channel id。

### `channels.signal`（signal-cli）

Signal 反应可以发出系统事件（共享反应工具）：

```json5
{
  channels: {
    signal: {
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50, // 包含最后 N 条群组消息作为上下文（0 禁用）
    },
  },
}
```

反应通知模式：

- `off`：无反应事件。
- `own`：对机器人自己消息的反应（默认）。
- `all`：对所有消息的所有反应。
- `allowlist`：来自 `channels.signal.reactionAllowlist` 对所有消息的反应（空列表禁用）。

### `channels.imessage`（imsg CLI）

OpenClaw 生成 `imsg rpc`（JSON-RPC over stdio）。不需要守护进程或端口。

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host", // 使用 SSH wrapper 时用于远程附件的 SCP
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50, // 包含最后 N 条群组消息作为上下文（0 禁用）
      includeAttachments: false,
      mediaMaxMb: 16,
      service: "auto",
      region: "US",
    },
  },
}
```

多账户支持位于 `channels.imessage.accounts` 下（参见上面的多账户部分）。

注意：

- 需要对 Messages DB 的完全磁盘访问权限。
- 首次发送将提示 Messages 自动化权限。
- 首选 `chat_id:<id>` 目标。使用 `imsg chats --limit 20` 列出聊天。
- `channels.imessage.cliPath` 可以指向 wrapper 脚本（例如 `ssh` 到另一台运行 `imsg rpc` 的 Mac）；使用 SSH 密钥避免密码提示。
- 对于远程 SSH wrapper，设置 `channels.imessage.remoteHost` 以在启用 `includeAttachments` 时通过 SCP 获取附件。

示例 wrapper：

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

### `agents.defaults.workspace`

设置 agent 用于文件操作的**单一全局工作目录**。

默认：`~/.openclaw/workspace`。

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

如果启用 `agents.defaults.sandbox`，非主 session 可以用 `agents.defaults.sandbox.workspaceRoot` 下的自己的每个范围工作目录覆盖此设置。

### `agents.defaults.repoRoot`

可选的仓库根目录，显示在系统提示的 Runtime 行中。如果未设置，OpenClaw
尝试通过从工作目录（和当前工作目录）向上遍历来检测 `.git` 目录。路径必须存在才能使用。

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skipBootstrap`

禁用自动创建工作目录引导文件（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md` 和 `BOOTSTRAP.md`）。

用于工作目录文件来自仓库的预置部署。

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.bootstrapMaxChars`

在截断之前注入系统提示的每个工作目录引导文件的最大字符数。默认：`20000`。

当文件超过此限制时，OpenClaw 记录警告并注入截断的
头/尾并带标记。

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.userTimezone`

设置**系统提示上下文**的用户时区（不用于消息信封中的时间戳）。如果未设置，OpenClaw 在运行时使用主机时区。

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

控制系统提示的当前日期和时间部分中显示的**时间格式**。
默认：`auto`（OS 偏好）。

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

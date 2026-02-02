---
summary: "多 Agent 路由：隔离的 Agent、频道账户和绑定"
title: 多 Agent 路由
read_when: "你想在一个 Gateway 进程中运行多个隔离的 Agent（工作区 + 认证）。"
status: active
---

# 多 Agent 路由

目标：多个*隔离*的 Agent（独立工作区 + `agentDir` + 会话），加上多个频道账户（例如两个 WhatsApp）在一个运行的 Gateway 中。入站通过绑定路由到 Agent。

## 什么是"一个 Agent"？

一个 **Agent** 是一个完全限定的大脑，拥有自己的：

- **工作区**（文件、AGENTS.md/SOUL.md/USER.md、本地笔记、人设规则）。
- **状态目录**（`agentDir`），用于认证配置文件、模型注册表和每个 Agent 的配置。
- **会话存储**（聊天历史 + 路由状态）位于 `~/.openclaw/agents/<agentId>/sessions`。

认证配置文件是**每个 Agent**的。每个 Agent 从自己的位置读取：

```
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

主 Agent 凭证**不会**自动共享。永远不要跨 Agent 重用 `agentDir`
（它会导致认证/会话冲突）。如果你想共享凭证，
将 `auth-profiles.json` 复制到另一个 Agent 的 `agentDir` 中。

Skills 通过每个工作区的 `skills/` 文件夹按 Agent 划分，共享 skills
可从 `~/.openclaw/skills` 获得。参见 [Skills：每个 Agent 与共享](/tools/skills#per-agent-vs-shared-skills)。

Gateway 可以托管**一个 Agent**（默认）或并排托管**多个 Agent**。

**工作区注意：**每个 Agent 的工作区是**默认 cwd**，而不是硬
沙盒。相对路径在工作区内解析，但绝对路径可以
到达其他主机位置，除非启用了沙盒。参见
[沙盒](/gateway/sandboxing)。

## 路径（快速映射）

- 配置：`~/.openclaw/openclaw.json`（或 `OPENCLAW_CONFIG_PATH`）
- 状态目录：`~/.openclaw`（或 `OPENCLAW_STATE_DIR`）
- 工作区：`~/.openclaw/workspace`（或 `~/.openclaw/workspace-<agentId>`）
- Agent 目录：`~/.openclaw/agents/<agentId>/agent`（或 `agents.list[].agentDir`）
- 会话：`~/.openclaw/agents/<agentId>/sessions`

### 单 Agent 模式（默认）

如果你什么都不做，OpenClaw 运行单个 Agent：

- `agentId` 默认为 **`main`**。
- 会话键为 `agent:main:<mainKey>`。
- 工作区默认为 `~/.openclaw/workspace`（或当设置 `OPENCLAW_PROFILE` 时为 `~/.openclaw/workspace-<profile>`）。
- 状态默认为 `~/.openclaw/agents/main/agent`。

## Agent 助手

使用 Agent 向导添加新的隔离 Agent：

```bash
openclaw agents add work
```

然后添加 `bindings`（或让向导完成）以路由入站消息。

使用以下命令验证：

```bash
openclaw agents list --bindings
```

## 多个 Agent = 多个人、多种个性

使用**多个 Agent**，每个 `agentId` 成为一个**完全隔离的人设**：

- **不同的电话号码/账户**（每个频道 `accountId`）。
- **不同的个性**（每个 Agent 的工作区文件如 `AGENTS.md` 和 `SOUL.md`）。
- **独立的认证 + 会话**（除非显式启用，否则没有串扰）。

这让**多个人**共享一个 Gateway 服务器，同时保持他们的 AI "大脑"和数据隔离。

## 一个 WhatsApp 号码，多个人（DM 分割）

你可以将**不同的 WhatsApp DM** 路由到不同的 Agent，同时保持在**一个 WhatsApp 账户**上。使用发送者 E.164（如 `+15551234567`）匹配 `peer.kind: "dm"`。回复仍来自同一个 WhatsApp 号码（没有每个 Agent 的发送者身份）。

重要细节：直接聊天折叠到 Agent 的**主会话键**，因此真正的隔离需要**每个人一个 Agent**。

示例：

```json5
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  bindings: [
    { agentId: "alex", match: { channel: "whatsapp", peer: { kind: "dm", id: "+15551230001" } } },
    { agentId: "mia", match: { channel: "whatsapp", peer: { kind: "dm", id: "+15551230002" } } },
  ],
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551230001", "+15551230002"],
    },
  },
}
```

注意：

- DM 访问控制是**每个 WhatsApp 账户全局**的（配对/允许列表），不是每个 Agent。
- 对于共享群组，将群组绑定到一个 Agent 或使用[广播群组](/broadcast-groups)。

## 路由规则（消息如何选择 Agent）

绑定是**确定性的**，**最具体的获胜**：

1. `peer` 匹配（精确的 DM/群组/频道 ID）
2. `guildId`（Discord）
3. `teamId`（Slack）
4. 频道的 `accountId` 匹配
5. 频道级匹配（`accountId: "*"`）
6. 回退到默认 Agent（`agents.list[].default`，否则第一个列表条目，默认：`main`）

## 多个账户 / 电话号码

支持**多个账户**的频道（例如 WhatsApp）使用 `accountId` 来识别
每次登录。每个 `accountId` 可以路由到不同的 Agent，因此一个服务器可以托管
多个电话号码而不会混合会话。

## 概念

- `agentId`：一个"大脑"（工作区、每个 Agent 认证、每个 Agent 会话存储）。
- `accountId`：一个频道账户实例（例如 WhatsApp 账户 `"personal"` 与 `"biz"`）。
- `binding`：通过 `(channel, accountId, peer)` 和可选的公会/团队 ID 将入站消息路由到 `agentId`。
- 直接聊天折叠到 `agent:<agentId>:<mainKey>`（每个 Agent "主"；`session.mainKey`）。

## 示例：两个 WhatsApp → 两个 Agent

`~/.openclaw/openclaw.json`（JSON5）：

```js
{
  agents: {
    list: [
      {
        id: "home",
        default: true,
        name: "Home",
        workspace: "~/.openclaw/workspace-home",
        agentDir: "~/.openclaw/agents/home/agent",
      },
      {
        id: "work",
        name: "Work",
        workspace: "~/.openclaw/workspace-work",
        agentDir: "~/.openclaw/agents/work/agent",
      },
    ],
  },

  // 确定性路由：第一个匹配获胜（最具体的优先）。
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

    // 可选的每个对等覆盖（示例：将特定群组发送到工作 Agent）。
    {
      agentId: "work",
      match: {
        channel: "whatsapp",
        accountId: "personal",
        peer: { kind: "group", id: "1203630...@g.us" },
      },
    },
  ],

  // 默认关闭：Agent 到 Agent 的消息传递必须显式启用 + 允许列表。
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },

  channels: {
    whatsapp: {
      accounts: {
        personal: {
          // 可选覆盖。默认：~/.openclaw/credentials/whatsapp/personal
          // authDir: "~/.openclaw/credentials/whatsapp/personal",
        },
        biz: {
          // 可选覆盖。默认：~/.openclaw/credentials/whatsapp/biz
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

## 示例：WhatsApp 日常聊天 + Telegram 深度工作

按频道分割：将 WhatsApp 路由到快速日常 Agent，将 Telegram 路由到 Opus Agent。

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-5",
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-5",
      },
    ],
  },
  bindings: [
    { agentId: "chat", match: { channel: "whatsapp" } },
    { agentId: "opus", match: { channel: "telegram" } },
  ],
}
```

注意：

- 如果你有频道的多个账户，请向绑定添加 `accountId`（例如 `{ channel: "whatsapp", accountId: "personal" }`）。
- 要将单个 DM/群组路由到 Opus 同时保持其余在 chat 上，请为该对等添加 `match.peer` 绑定；对等匹配总是胜过频道范围规则。

## 示例：同一频道，一个对等路由到 Opus

将 WhatsApp 保持在快速 Agent 上，但将一个 DM 路由到 Opus：

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-5",
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-5",
      },
    ],
  },
  bindings: [
    { agentId: "opus", match: { channel: "whatsapp", peer: { kind: "dm", id: "+15551234567" } } },
    { agentId: "chat", match: { channel: "whatsapp" } },
  ],
}
```

对等绑定总是获胜，因此将它们保持在频道范围规则之上。

## 绑定到 WhatsApp 群组的家庭 Agent

将专用的家庭 Agent 绑定到单个 WhatsApp 群组，带有提及门控
和更严格的工具策略：

```json5
{
  agents: {
    list: [
      {
        id: "family",
        name: "Family",
        workspace: "~/.openclaw/workspace-family",
        identity: { name: "Family Bot" },
        groupChat: {
          mentionPatterns: ["@family", "@familybot", "@Family Bot"],
        },
        sandbox: {
          mode: "all",
          scope: "agent",
        },
        tools: {
          allow: [
            "exec",
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "browser", "canvas", "nodes", "cron"],
        },
      },
    ],
  },
  bindings: [
    {
      agentId: "family",
      match: {
        channel: "whatsapp",
        peer: { kind: "group", id: "120363999999999999@g.us" },
      },
    },
  ],
}
```

注意：

- 工具允许/拒绝列表是**工具**，不是 skills。如果 skill 需要运行
  二进制文件，确保 `exec` 被允许且二进制文件存在于沙盒中。
- 对于更严格的门控，设置 `agents.list[].groupChat.mentionPatterns` 并保留
  频道的群组允许列表启用。

## 每个 Agent 的沙盒和工具配置

从 v2026.1.6 开始，每个 Agent 可以有自己的沙盒和工具限制：

```js
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // 个人 Agent 没有沙盒
        },
        // 没有工具限制 - 所有工具可用
      },
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // 始终沙盒化
          scope: "agent",  // 每个 Agent 一个容器
          docker: {
            // 容器创建后的可选一次性设置
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        tools: {
          allow: ["read"],                    // 仅读取工具
          deny: ["exec", "write", "edit", "apply_patch"],    // 拒绝其他工具
        },
      },
    ],
  },
}
```

注意：`setupCommand` 位于 `sandbox.docker` 下，在容器创建时运行一次。
当解析的范围是 `"shared"` 时，每个 Agent 的 `sandbox.docker.*` 覆盖被忽略。

**好处：**

- **安全隔离**：为不受信任的 Agent 限制工具
- **资源控制**：沙盒特定 Agent 同时保持其他在主机上
- **灵活策略**：每个 Agent 的不同权限

注意：`tools.elevated` 是**全局**的且基于发送者；它不是每个 Agent 可配置的。
如果你需要每个 Agent 的边界，使用 `agents.list[].tools` 拒绝 `exec`。
对于群组定位，使用 `agents.list[].groupChat.mentionPatterns`，以便 @提及干净地映射到预期的 Agent。

参见 [多 Agent 沙盒和工具](/multi-agent-sandbox-tools) 以获取详细示例。

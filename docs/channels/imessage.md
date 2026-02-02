---
summary: "通过 imsg（stdio 上的 JSON-RPC）实现 iMessage 支持，设置和 chat_id 路由"
read_when:
  - 设置 iMessage 支持
  - 调试 iMessage 发送/接收
title: iMessage
---

# iMessage（imsg）

状态：外部 CLI 集成。网关生成 `imsg rpc`（stdio 上的 JSON-RPC）。

## 快速设置（初学者）

1. 确保 Messages 在此 Mac 上已登录。
2. 安装 `imsg`：
   - `brew install steipete/tap/imsg`
3. 使用 `channels.imessage.cliPath` 和 `channels.imessage.dbPath` 配置 OpenClaw。
4. 启动网关并批准任何 macOS 提示（自动化 + 完全磁盘访问）。

最小配置：

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/<you>/Library/Messages/chat.db",
    },
  },
}
```

## 功能概述

- 由 macOS 上的 `imsg` 支持的 iMessage 频道。
- 确定性路由：回复始终返回 iMessage。
- 私信共享代理的主会话；群组是隔离的（`agent:<agentId>:imessage:group:<chat_id>`）。
- 如果多参与者线程以 `is_group=false` 到达，您仍然可以使用 `channels.imessage.groups` 按 `chat_id` 隔离它（参见下面的"类群组线程"）。

## 配置写入

默认情况下，允许 iMessage 写入由 `/config set|unset` 触发的配置更新（需要 `commands.config: true`）。

禁用：

```json5
{
  channels: { imessage: { configWrites: false } },
}
```

## 要求

- 已登录 Messages 的 macOS。
- OpenClaw + `imsg` 的完全磁盘访问（Messages DB 访问）。
- 发送时的自动化权限。
- `channels.imessage.cliPath` 可以指向任何代理 stdin/stdout 的命令（例如，一个包装脚本，通过 SSH 连接到另一台 Mac 并运行 `imsg rpc`）。

## 设置（快速路径）

1. 确保 Messages 在此 Mac 上已登录。
2. 配置 iMessage 并启动网关。

### 专用机器人 macOS 用户（用于隔离身份）

如果您希望机器人从**单独的 iMessage 身份**发送（并保持您的个人 Messages 干净），请使用专用 Apple ID + 专用 macOS 用户。

1. 创建专用 Apple ID（示例：`my-cool-bot@icloud.com`）。
   - Apple 可能需要电话号码进行验证 / 2FA。
2. 创建一个 macOS 用户（示例：`openclawhome`）并登录。
3. 在该 macOS 用户中打开 Messages 并使用机器人 Apple ID 登录 iMessage。
4. 启用远程登录（系统设置 → 通用 → 共享 → 远程登录）。
5. 安装 `imsg`：
   - `brew install steipete/tap/imsg`
6. 设置 SSH，使 `ssh <bot-macos-user>@localhost true` 无需密码即可工作。
7. 将 `channels.imessage.accounts.bot.cliPath` 指向以机器人用户身份运行 `imsg` 的 SSH 包装器。

首次运行注意：发送/接收可能需要_机器人 macOS 用户_中的 GUI 批准（自动化 + 完全磁盘访问）。如果 `imsg rpc` 看起来卡住或退出，登录该用户（屏幕共享有帮助），运行一次性的 `imsg chats --limit 1` / `imsg send ...`，批准提示，然后重试。

示例包装器（`chmod +x`）。将 `<bot-macos-user>` 替换为您的实际 macOS 用户名：

```bash
#!/usr/bin/env bash
set -euo pipefail

# 首先运行一次交互式 SSH 以接受主机密钥：
#   ssh <bot-macos-user>@localhost true
exec /usr/bin/ssh -o BatchMode=yes -o ConnectTimeout=5 -T <bot-macos-user>@localhost \
  "/usr/local/bin/imsg" "$@"
```

示例配置：

```json5
{
  channels: {
    imessage: {
      enabled: true,
      accounts: {
        bot: {
          name: "Bot",
          enabled: true,
          cliPath: "/path/to/imsg-bot",
          dbPath: "/Users/<bot-macos-user>/Library/Messages/chat.db",
        },
      },
    },
  },
}
```

对于单账户设置，使用平面选项（`channels.imessage.cliPath`、`channels.imessage.dbPath`）而不是 `accounts` 映射。

### 远程/SSH 变体（可选）

如果您想在另一台 Mac 上使用 iMessage，请将 `channels.imessage.cliPath` 设置为通过 SSH 在远程 macOS 主机上运行 `imsg` 的包装器。OpenClaw 只需要 stdio。

示例包装器：

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

**远程附件：** 当 `cliPath` 通过 SSH 指向远程主机时，Messages 数据库中的附件路径引用远程机器上的文件。OpenClaw 可以通过设置 `channels.imessage.remoteHost` 自动通过 SCP 获取这些文件：

```json5
{
  channels: {
    imessage: {
      cliPath: "~/imsg-ssh", // 到远程 Mac 的 SSH 包装器
      remoteHost: "user@gateway-host", // 用于 SCP 文件传输
      includeAttachments: true,
    },
  },
}
```

如果未设置 `remoteHost`，OpenClaw 尝试通过解析包装脚本中的 SSH 命令自动检测它。建议显式配置以确保可靠性。

#### 通过 Tailscale 的远程 Mac（示例）

如果网关在 Linux 主机/VM 上运行，但 iMessage 必须在 Mac 上运行，Tailscale 是最简单的桥接：网关通过 tailnet 与 Mac 通信，通过 SSH 运行 `imsg`，并通过 SCP 返回附件。

架构：

```
┌──────────────────────────────┐          SSH (imsg rpc)          ┌──────────────────────────┐
│ 网关主机 (Linux/VM)          │──────────────────────────────────▶│ 带 Messages + imsg 的 Mac │
│ - openclaw gateway           │          SCP (附件)               │ - Messages 已登录          │
│ - channels.imessage.cliPath  │◀──────────────────────────────────│ - 远程登录已启用           │
└──────────────────────────────┘                                   └──────────────────────────┘
              ▲
              │ Tailscale tailnet (主机名或 100.x.y.z)
              ▼
        user@gateway-host
```

具体配置示例（Tailscale 主机名）：

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
      includeAttachments: true,
      dbPath: "/Users/bot/Library/Messages/chat.db",
    },
  },
}
```

示例包装器（`~/.openclaw/scripts/imsg-ssh`）：

```bash
#!/usr/bin/env bash
exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
```

注意：

- 确保 Mac 已登录 Messages，且远程登录已启用。
- 使用 SSH 密钥，使 `ssh bot@mac-mini.tailnet-1234.ts.net` 无需提示即可工作。
- `remoteHost` 应匹配 SSH 目标，以便 SCP 可以获取附件。

多账户支持：使用 `channels.imessage.accounts` 配合每个账户的配置和可选的 `name`。共享模式参见 [`gateway/configuration`](/gateway/configuration#telegramaccounts--discordaccounts--slackaccounts--signalaccounts--imessageaccounts)。不要提交 `~/.openclaw/openclaw.json`（它通常包含令牌）。

## 访问控制（私信 + 群组）

私信：

- 默认：`channels.imessage.dmPolicy = "pairing"`。
- 未知发送者接收配对码；消息被忽略直到批准（代码在 1 小时后过期）。
- 通过以下方式批准：
  - `openclaw pairing list imessage`
  - `openclaw pairing approve imessage <CODE>`
- 配对是 iMessage 私信的默认令牌交换。详情：[配对](/start/pairing)

群组：

- `channels.imessage.groupPolicy = open | allowlist | disabled`。
- 当设置 `allowlist` 时，`channels.imessage.groupAllowFrom` 控制谁可以在群组中触发。
- 提及门控使用 `agents.list[].groupChat.mentionPatterns`（或 `messages.groupChat.mentionPatterns`），因为 iMessage 没有原生提及元数据。
- 多代理覆盖：在 `agents.list[].groupChat.mentionPatterns` 上设置每个代理的模式。

## 工作原理（行为）

- `imsg` 流式传输消息事件；网关将它们规范化为共享频道信封。
- 回复始终路由回相同的聊天 id 或句柄。

## 类群组线程（`is_group=false`）

某些 iMessage 线程可以有多个参与者，但根据 Messages 存储聊天标识符的方式，仍然以 `is_group=false` 到达。

如果您在 `channels.imessage.groups` 下显式配置 `chat_id`，OpenClaw 将该线程视为"群组"用于：

- 会话隔离（单独的 `agent:<agentId>:imessage:group:<chat_id>` 会话键）
- 群组允许列表 / 提及门控行为

示例：

```json5
{
  channels: {
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123"],
      groups: {
        "42": { requireMention: false },
      },
    },
  },
}
```

当您想要为特定线程设置隔离的个性/模型时，这很有用（参见 [多代理路由](/concepts/multi-agent)）。对于文件系统隔离，参见 [沙盒](/gateway/sandboxing)。

## 媒体 + 限制

- 通过 `channels.imessage.includeAttachments` 可选摄取附件。
- 通过 `channels.imessage.mediaMaxMb` 限制媒体。

## 限制

- 出站文本被分块为 `channels.imessage.textChunkLimit`（默认 4000）。
- 可选换行分块：设置 `channels.imessage.chunkMode="newline"` 以在空行（段落边界）处分割，然后进行长度分块。
- 媒体上传受 `channels.imessage.mediaMaxMb` 限制（默认 16）。

## 寻址 / 传递目标

优先使用 `chat_id` 进行稳定路由：

- `chat_id:123`（首选）
- `chat_guid:...`
- `chat_identifier:...`
- 直接句柄：`imessage:+1555` / `sms:+1555` / `user@example.com`

列出聊天：

```
imsg chats --limit 20
```

## 配置参考（iMessage）

完整配置：[配置](/gateway/configuration)

提供商选项：

- `channels.imessage.enabled`：启用/禁用频道启动。
- `channels.imessage.cliPath`：`imsg` 的路径。
- `channels.imessage.dbPath`：Messages DB 路径。
- `channels.imessage.remoteHost`：当 `cliPath` 指向远程 Mac 时用于 SCP 附件传输的 SSH 主机（例如 `user@gateway-host`）。如果未设置，从 SSH 包装器自动检测。
- `channels.imessage.service`：`imessage | sms | auto`。
- `channels.imessage.region`：SMS 区域。
- `channels.imessage.dmPolicy`：`pairing | allowlist | open | disabled`（默认：pairing）。
- `channels.imessage.allowFrom`：私信允许列表（句柄、电子邮件、E.164 号码或 `chat_id:*`）。`open` 需要 `"*"`。iMessage 没有用户名；使用句柄或聊天目标。
- `channels.imessage.groupPolicy`：`open | allowlist | disabled`（默认：allowlist）。
- `channels.imessage.groupAllowFrom`：群组发送者允许列表。
- `channels.imessage.historyLimit` / `channels.imessage.accounts.*.historyLimit`：作为上下文包含的最大群组消息数（0 禁用）。
- `channels.imessage.dmHistoryLimit`：私信历史限制（用户轮次）。每个用户的覆盖：`channels.imessage.dms["<handle>"].historyLimit`。
- `channels.imessage.groups`：每个群组的默认值 + 允许列表（使用 `"*"` 作为全局默认值）。
- `channels.imessage.includeAttachments`：将附件摄取到上下文中。
- `channels.imessage.mediaMaxMb`：入站/出站媒体上限（MB）。
- `channels.imessage.textChunkLimit`：出站块大小（字符）。
- `channels.imessage.chunkMode`：`length`（默认）或 `newline` 以在空行（段落边界）处分割，然后进行长度分块。

相关全局选项：

- `agents.list[].groupChat.mentionPatterns`（或 `messages.groupChat.mentionPatterns`）。
- `messages.responsePrefix`。

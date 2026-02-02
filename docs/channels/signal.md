---
summary: "通过 signal-cli 的 Signal 支持（JSON-RPC + SSE）、设置和号码模型"
read_when:
  - 设置 Signal 支持
  - 调试 Signal 发送/接收
title: "Signal"
---

# Signal（signal-cli）

状态：外部 CLI 集成。网关通过 HTTP JSON-RPC + SSE 与 `signal-cli` 通信。

## 快速设置（初学者）

1. 为机器人使用**单独的 Signal 号码**（推荐）。
2. 安装 `signal-cli`（需要 Java）。
3. 链接机器人设备并启动守护进程：
   - `signal-cli link -n "OpenClaw"`
4. 配置 OpenClaw 并启动网关。

最小配置：

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

## 它是什么

- 通过 `signal-cli` 的 Signal 频道（非嵌入式 libsignal）。
- 确定性路由：回复始终返回到 Signal。
- DM 共享 agent 的 main session；群组是隔离的（`agent:<agentId>:signal:group:<groupId>`）。

## 配置写入

默认情况下，Signal 允许由 `/config set|unset` 触发的配置更新（需要 `commands.config: true`）。

禁用：

```json5
{
  channels: { signal: { configWrites: false } },
}
```

## 号码模型（重要）

- 网关连接到一个 **Signal 设备**（`signal-cli` 账户）。
- 如果你在**个人 Signal 账户**上运行机器人，它将忽略你自己的消息（循环保护）。
- 对于"我发短信给机器人，它回复"，使用**单独的机器人号码**。

## 设置（快速路径）

1. 安装 `signal-cli`（需要 Java）。
2. 链接一个机器人账户：
   - `signal-cli link -n "OpenClaw"` 然后在 Signal 中扫描 QR 码。
3. 配置 Signal 并启动网关。

示例：

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

多账户支持：使用 `channels.signal.accounts`，包含每账户配置和可选的 `name`。参见 [`gateway/configuration`](/gateway/configuration#telegramaccounts--discordaccounts--slackaccounts--signalaccounts--imessageaccounts) 了解共享模式。

## 外部守护进程模式（httpUrl）

如果你想自己管理 `signal-cli`（缓慢的 JVM 冷启动、容器初始化或共享 CPU），单独运行守护进程并指向 OpenClaw：

```json5
{
  channels: {
    signal: {
      httpUrl: "http://127.0.0.1:8080",
      autoStart: false,
    },
  },
}
```

这跳过 OpenClaw 内部的自动生成和启动等待。对于自动生成时的缓慢启动，设置 `channels.signal.startupTimeoutMs`。

## 访问控制（DM + 群组）

DM：

- 默认：`channels.signal.dmPolicy = "pairing"`。
- 未知发送者收到配对码；消息在批准前被忽略（码在 1 小时后过期）。
- 通过以下方式批准：
  - `openclaw pairing list signal`
  - `openclaw pairing approve signal <CODE>`
- 配对是 Signal DM 的默认令牌交换。详情：[配对](/start/pairing)
- 仅 UUID 的发送者（来自 `sourceUuid`）在 `channels.signal.allowFrom` 中存储为 `uuid:<id>`。

群组：

- `channels.signal.groupPolicy = open | allowlist | disabled`。
- 当设置 `allowlist` 时，`channels.signal.groupAllowFrom` 控制谁可以在群组中触发。

## 工作原理（行为）

- `signal-cli` 作为守护进程运行；网关通过 SSE 读取事件。
- 入站消息被规范化为共享的频道 envelope。
- 回复始终路由回相同的号码或群组。

## 媒体 + 限制

- 出站文本被分块到 `channels.signal.textChunkLimit`（默认 4000）。
- 可选的换行分块：设置 `channels.signal.chunkMode="newline"` 以在空行（段落边界）处分割，然后进行长度分块。
- 支持附件（从 `signal-cli` 获取 base64）。
- 默认媒体上限：`channels.signal.mediaMaxMb`（默认 8）。
- 使用 `channels.signal.ignoreAttachments` 跳过下载媒体。
- 群组历史上下文使用 `channels.signal.historyLimit`（或 `channels.signal.accounts.*.historyLimit`），回退到 `messages.groupChat.historyLimit`。设置为 `0` 以禁用（默认 50）。

## 输入 + 已读回执

- **输入指示器**：OpenClaw 通过 `signal-cli sendTyping` 发送输入信号，并在回复运行时刷新它们。
- **已读回执**：当 `channels.signal.sendReadReceipts` 为 true 时，OpenClaw 转发允许 DM 的已读回执。
- Signal-cli 不暴露群组的已读回执。

## 反应（消息工具）

- 使用 `message action=react` 和 `channel=signal`。
- 目标：发送者 E.164 或 UUID（从配对输出中使用 `uuid:<id>`；裸 UUID 也可以）。
- `messageId` 是你正在反应的 Signal 消息的时间戳。
- 群组反应需要 `targetAuthor` 或 `targetAuthorUuid`。

示例：

```
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

配置：

- `channels.signal.actions.reactions`：启用/禁用反应操作（默认 true）。
- `channels.signal.reactionLevel`：`off | ack | minimal | extensive`。
  - `off`/`ack` 禁用 agent 反应（消息工具 `react` 将报错）。
  - `minimal`/`extensive` 启用 agent 反应并设置指导级别。
- 每账户覆盖：`channels.signal.accounts.<id>.actions.reactions`、`channels.signal.accounts.<id>.reactionLevel`。

## 投递目标（CLI/cron）

- DM：`signal:+15551234567`（或纯 E.164）。
- UUID DM：`uuid:<id>`（或裸 UUID）。
- 群组：`signal:group:<groupId>`。
- 用户名：`username:<name>`（如果你的 Signal 账户支持）。

## 配置参考（Signal）

完整配置：[配置](/gateway/configuration)

Provider 选项：

- `channels.signal.enabled`：启用/禁用频道启动。
- `channels.signal.account`：机器人账户的 E.164。
- `channels.signal.cliPath`：`signal-cli` 的路径。
- `channels.signal.httpUrl`：完整的守护进程 URL（覆盖 host/port）。
- `channels.signal.httpHost`、`channels.signal.httpPort`：守护进程绑定（默认 127.0.0.1:8080）。
- `channels.signal.autoStart`：自动生成守护进程（如果 `httpUrl` 未设置，默认 true）。
- `channels.signal.startupTimeoutMs`：启动等待超时（毫秒）（上限 120000）。
- `channels.signal.receiveMode`：`on-start | manual`。
- `channels.signal.ignoreAttachments`：跳过附件下载。
- `channels.signal.ignoreStories`：忽略来自守护进程的故事。
- `channels.signal.sendReadReceipts`：转发已读回执。
- `channels.signal.dmPolicy`：`pairing | allowlist | open | disabled`（默认：pairing）。
- `channels.signal.allowFrom`：DM allowlist（E.164 或 `uuid:<id>`）。`open` 需要 `"*"`。Signal 没有用户名；使用 phone/UUID id。
- `channels.signal.groupPolicy`：`open | allowlist | disabled`（默认：allowlist）。
- `channels.signal.groupAllowFrom`：群组发送者 allowlist。
- `channels.signal.historyLimit`：作为上下文包含的最大群组消息数（0 禁用）。
- `channels.signal.dmHistoryLimit`：用户轮次中的 DM 历史限制。每用户覆盖：`channels.signal.dms["<phone_or_uuid>"].historyLimit`。
- `channels.signal.textChunkLimit`：出站块大小（字符）。
- `channels.signal.chunkMode`：`length`（默认）或 `newline` 以在空行（段落边界）处分割，然后进行长度分块。
- `channels.signal.mediaMaxMb`：入站/出站媒体上限（MB）。

相关全局选项：

- `agents.list[].groupChat.mentionPatterns`（Signal 不支持原生提及）。
- `messages.groupChat.mentionPatterns`（全局回退）。
- `messages.responsePrefix`。

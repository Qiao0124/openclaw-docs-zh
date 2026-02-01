---
summary: "Webhook 入口，用于唤醒与隔离代理运行"
read_when:
  - 新增或修改 webhook 端点
  - 将外部系统接入 OpenClaw
title: "Webhooks"
---

# Webhooks

网关可以暴露一个小型 HTTP webhook 端点，用于外部触发。

## 启用

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

说明：

- `hooks.token` 在 `hooks.enabled=true` 时必填。
- `hooks.path` 默认为 `/hooks`。

## 认证

每个请求必须携带 hook token。推荐使用 header：

- `Authorization: Bearer <token>`（推荐）
- `x-openclaw-token: <token>`
- `?token=<token>`（已弃用；会记录警告并将在未来大版本移除）

## 端点

### `POST /hooks/wake`

Payload：

```json
{ "text": "System line", "mode": "now" }
```

- `text` **必填**（string）：事件描述（如 "New email received"）。
- `mode` 可选（`now` | `next-heartbeat`）：立即触发心跳（默认 `now`）或等待下一次周期检查。

效果：

- 将系统事件入队到 **主** 会话
- 若 `mode=now`，立即触发一次心跳

### `POST /hooks/agent`

Payload：

```json
{
  "message": "Run this",
  "name": "Email",
  "sessionKey": "hook:email:msg-123",
  "wakeMode": "now",
  "deliver": true,
  "channel": "last",
  "to": "+15551234567",
  "model": "openai/gpt-5.2-mini",
  "thinking": "low",
  "timeoutSeconds": 120
}
```

- `message` **必填**（string）：代理需要处理的提示或消息。
- `name` 可选（string）：hook 的可读名称（如 "GitHub"），用于会话摘要前缀。
- `sessionKey` 可选（string）：用于标识代理会话的 key。默认随机 `hook:<uuid>`。
  使用稳定的 key 可在 hook 上下文里持续多轮对话。
- `wakeMode` 可选（`now` | `next-heartbeat`）：是否立即触发心跳（默认 `now`）或等待下一次周期检查。
- `deliver` 可选（boolean）：`true` 时将代理响应发送到消息渠道。默认 `true`。
  仅心跳确认类回复会自动跳过投递。
- `channel` 可选（string）：投递渠道。可选值：`last`、`whatsapp`、`telegram`、`discord`、`slack`、`mattermost`（插件）、`signal`、`imessage`、`msteams`。默认 `last`。
- `to` 可选（string）：渠道收件人标识（WhatsApp/Signal 为手机号，Telegram 为 chat id，Discord/Slack/Mattermost 为 channel id，MS Teams 为 conversation id）。默认使用主会话的上次收件人。
- `model` 可选（string）：模型覆盖（如 `anthropic/claude-3-5-sonnet` 或别名）。若有限制，必须在允许列表中。
- `thinking` 可选（string）：思考级别覆盖（如 `low`、`medium`、`high`）。
- `timeoutSeconds` 可选（number）：代理运行的最大时长（秒）。

效果：

- 运行 **隔离** 的代理回合（独立会话 key）
- 总会在 **主** 会话发布摘要
- 若 `wakeMode=now`，立即触发一次心跳

### `POST /hooks/<name>`（映射）

自定义 hook 名称通过 `hooks.mappings` 解析（见配置）。映射可将任意 payload
转换为 `wake` 或 `agent` 动作，并支持模板或代码转换。

映射选项（摘要）：

- `hooks.presets: ["gmail"]` 启用内置 Gmail 映射。
- `hooks.mappings` 让你在配置中定义 `match`、`action` 与模板。
- `hooks.transformsDir` + `transform.module` 加载 JS/TS 模块实现自定义逻辑。
- 使用 `match.source` 可保留通用入口（基于 payload 路由）。
- TS transform 运行时需要 TS loader（如 `bun` 或 `tsx`），或预编译 `.js`。
- 在映射中设置 `deliver: true` + `channel`/`to` 可把回复路由到聊天渠道
  （`channel` 默认 `last`，回退到 WhatsApp）。
- `allowUnsafeExternalContent: true` 会为该 hook 关闭外部内容安全包裹
  （危险，仅用于可信内部源）。
- `openclaw webhooks gmail setup` 会写入 `hooks.gmail` 配置供 `openclaw webhooks gmail run` 使用。
  Gmail watch 完整流程见 [Gmail Pub/Sub](/automation/gmail-pubsub)。

## 响应

- `/hooks/wake` 返回 `200`
- `/hooks/agent` 返回 `202`（异步运行已开始）
- 认证失败返回 `401`
- payload 无效返回 `400`
- payload 过大返回 `413`

## 示例

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake   -H "Authorization: Bearer SECRET"   -H "Content-Type: application/json"   -d "{"text":"New email received","mode":"now"}"
```

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent   -H "x-openclaw-token: SECRET"   -H "Content-Type: application/json"   -d "{"message":"Summarize inbox","name":"Email","wakeMode":"next-heartbeat"}"
```

### 使用不同模型

在 agent payload（或映射）中添加 `model` 以覆盖本次运行的模型：

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent   -H "x-openclaw-token: SECRET"   -H "Content-Type: application/json"   -d "{"message":"Summarize inbox","name":"Email","model":"openai/gpt-5.2-mini"}"
```

如果你启用了 `agents.defaults.models`，请确保覆盖模型包含在列表中。

```bash
curl -X POST http://127.0.0.1:18789/hooks/gmail   -H "Authorization: Bearer SECRET"   -H "Content-Type: application/json"   -d "{"source":"gmail","messages":[{"from":"Ada","subject":"Hello","snippet":"Hi"}]}"
```

## 安全

- 将 hook 端点放在回环、tailnet 或可信反向代理之后。
- 使用独立的 hook token；不要复用网关认证 token。
- 避免在 webhook 日志中包含敏感原始 payload。
- hook payload 默认视为不可信并包裹安全边界。
  若必须为某个 hook 关闭，请在该 hook 映射中设置 `allowUnsafeExternalContent: true`
  （危险）。

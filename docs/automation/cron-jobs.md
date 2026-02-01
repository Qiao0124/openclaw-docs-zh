---
summary: "网关调度器的 Cron 任务与唤醒"
read_when:
  - 安排后台任务或唤醒
  - 编排需要随心跳运行的自动化
  - 在心跳与 cron 之间做选择
title: "Cron 任务"
---

# Cron 任务（网关调度器）

> **Cron 还是 Heartbeat？** 何时使用各自，请参见 [Cron vs Heartbeat](/automation/cron-vs-heartbeat)。

Cron 是网关内置的调度器。它会持久化任务、在正确时间唤醒代理，并可选地
把输出投递回聊天渠道。

如果你想要 _“每天早上运行”_ 或 _“20 分钟后提醒我”_，
cron 就是这个机制。

## 要点

- Cron **运行在网关内部**（不在模型里）。
- 任务持久化在 `~/.openclaw/cron/`，重启不会丢失。
- 两种执行风格：
  - **主会话**：入队系统事件，在下一次心跳执行。
  - **隔离**：在 `cron:<jobId>` 中运行专用代理回合，可选投递输出。
- 唤醒是一等公民：任务可选择“立刻唤醒”或“下一次心跳”。

## 快速开始（可直接用）

创建一次性提醒，确认任务存在，并立即运行：

```bash
openclaw cron add \
  --name "Reminder" \
  --at "2026-02-01T16:00:00Z" \
  --session main \
  --system-event "Reminder: check the cron docs draft" \
  --wake now \
  --delete-after-run

openclaw cron list
openclaw cron run <job-id> --force
openclaw cron runs --id <job-id>
```

安排一个可投递的重复隔离任务：

```bash
openclaw cron add \
  --name "Morning brief" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize overnight updates." \
  --deliver \
  --channel slack \
  --to "channel:C1234567890"
```

## 工具调用等价形式（Gateway cron 工具）

可用的 JSON 形状与示例，请见
[工具调用 JSON Schema](/automation/cron-jobs#json-schema-for-tool-calls)。

## Cron 任务存储位置

Cron 任务默认存储在网关主机的 `~/.openclaw/cron/jobs.json`。
网关会将其加载到内存，并在变更时回写，因此只有在网关停止时，
手动编辑才是安全的。修改请优先使用 `openclaw cron add/edit`
或 cron 工具调用 API。

## 新手友好概览

把一个 cron 任务理解为：**何时运行** + **做什么**。

1. **选择计划**
   - 一次性提醒 → `schedule.kind = "at"`（CLI：`--at`）
   - 周期性任务 → `schedule.kind = "every"` 或 `schedule.kind = "cron"`
   - 如果 ISO 时间戳未包含时区，会被视为 **UTC**。

2. **选择运行位置**
   - `sessionTarget: "main"` → 在下一次心跳用主会话上下文执行。
   - `sessionTarget: "isolated"` → 在 `cron:<jobId>` 中运行专用代理回合。

3. **选择负载内容**
   - 主会话 → `payload.kind = "systemEvent"`
   - 隔离会话 → `payload.kind = "agentTurn"`

可选：`deleteAfterRun: true` 会在一次性任务成功后从存储中删除。

## 概念

### 任务（Jobs）

一个 cron 任务是一个持久化记录，包含：

- **计划**（何时运行）
- **负载**（做什么）
- 可选 **投递**（输出发往哪里）
- 可选 **代理绑定**（`agentId`）：让任务在特定代理下运行；
  若缺失或未知，网关会回退到默认代理。

任务通过稳定的 `jobId` 标识（CLI/Gateway API 使用）。
在代理工具调用中，`jobId` 为规范字段；兼容性仍接受旧的 `id`。
一次性任务可用 `deleteAfterRun: true` 在成功后自动删除。

### 计划（Schedules）

Cron 支持三种计划类型：

- `at`：一次性时间戳（毫秒）。网关接受 ISO 8601 并强制转为 UTC。
- `every`：固定间隔（毫秒）。
- `cron`：5 字段 cron 表达式 + 可选 IANA 时区。

Cron 表达式使用 `croner`。如果未指定时区，使用网关主机的本地时区。

### 主会话 vs 隔离执行

#### 主会话任务（系统事件）

主会话任务会入队系统事件，并可选唤醒心跳执行。
必须使用 `payload.kind = "systemEvent"`。

- `wakeMode: "next-heartbeat"`（默认）：事件等待下一次心跳。
- `wakeMode: "now"`：立即触发一次心跳运行。

当你需要使用常规心跳提示词 + 主会话上下文时，这是最合适的方式。
参见 [Heartbeat](/gateway/heartbeat)。

#### 隔离任务（专用 cron 会话）

隔离任务会在 `cron:<jobId>` 中运行专用代理回合。

关键行为：

- 提示词会前缀 `[cron:<jobId> <job name>]`，便于追踪。
- 每次运行都会使用 **全新的会话 id**（无历史对话继承）。
- 会在主会话里发布一条摘要（前缀 `Cron`，可配置）。
- `wakeMode: "now"` 会在发布摘要后立即触发心跳。
- 如果 `payload.deliver: true`，输出会投递到渠道；否则仅内部保存。

适用于噪声较大、频繁或“后台杂务”类任务，避免污染主聊天历史。

### 负载形态（运行什么）

支持两种负载类型：

- `systemEvent`：仅主会话，通过心跳提示词处理。
- `agentTurn`：仅隔离会话，运行专用代理回合。

`agentTurn` 常见字段：

- `message`：必填提示文本。
- `model` / `thinking`：可选覆盖（见下文）。
- `timeoutSeconds`：可选超时覆盖。
- `deliver`：`true` 时将输出投递到渠道。
- `channel`：`last` 或指定渠道。
- `to`：渠道的目标标识（手机号/聊天/频道 id）。
- `bestEffortDeliver`：投递失败时不让任务失败。

隔离选项（仅 `session=isolated`）：

- `postToMainPrefix`（CLI：`--post-prefix`）：主会话系统事件前缀。
- `postToMainMode`：`summary`（默认）或 `full`。
- `postToMainMaxChars`：`postToMainMode=full` 时的最大字符数（默认 8000）。

### 模型与思考级别覆盖

隔离任务（`agentTurn`）可以覆盖模型与思考级别：

- `model`：提供方/模型字符串（如 `anthropic/claude-sonnet-4-20250514`）或别名（如 `opus`）
- `thinking`：思考级别（`off`、`minimal`、`low`、`medium`、`high`、`xhigh`；仅 GPT-5.2 + Codex 模型）

注意：主会话任务也可以设置 `model`，但它会改变主会话共享模型。
我们建议只在隔离任务上使用模型覆盖，以避免意外的上下文切换。

优先级：

1. 任务负载覆盖（最高）
2. Hook 特定默认值（如 `hooks.gmail.model`）
3. 代理配置默认值

### 投递（渠道 + 目标）

隔离任务可将输出投递到渠道。任务负载可指定：

- `channel`：`whatsapp` / `telegram` / `discord` / `slack` / `mattermost`（插件） / `signal` / `imessage` / `last`
- `to`：渠道特定的收件人目标

如果未指定 `channel` 或 `to`，cron 可回退到主会话的“上次路由”
（代理上一次回复的地点）。

投递注意事项：

- 如果设置了 `to`，即使未写 `deliver`，cron 也会自动投递最终输出。
- 如果你希望“上次路由投递”，但没有显式 `to`，请用 `deliver: true`。
- 若你希望即使存在 `to` 也只在内部保存，请用 `deliver: false`。

目标格式提醒：

- Slack/Discord/Mattermost（插件）目标应使用显式前缀（如 `channel:<id>`、`user:<id>`），避免歧义。
- Telegram 话题请使用 `:topic:` 格式（见下文）。

#### Telegram 投递目标（话题 / 论坛线程）

Telegram 通过 `message_thread_id` 支持论坛话题。用于 cron 投递时，你可以把
话题/线程编码进 `to` 字段：

- `-1001234567890`（仅 chat id）
- `-1001234567890:topic:123`（推荐：显式话题标记）
- `-1001234567890:123`（简写：数字后缀）

也支持带前缀的目标，如 `telegram:...` / `telegram:group:...`：

- `telegram:group:-1001234567890:topic:123`

## 工具调用 JSON Schema {#json-schema-for-tool-calls}

直接调用 Gateway `cron.*` 工具时使用以下形状（代理工具调用或 RPC）。
CLI 参数支持 `20m` 这类人类友好时长，但工具调用使用毫秒时间戳：
`atMs` 与 `everyMs`（`at` 也接受 ISO 时间）。

### cron.add 参数

一次性主会话任务（系统事件）：

```json
{
  "name": "Reminder",
  "schedule": { "kind": "at", "atMs": 1738262400000 },
  "sessionTarget": "main",
  "wakeMode": "now",
  "payload": { "kind": "systemEvent", "text": "Reminder text" },
  "deleteAfterRun": true
}
```

带投递的重复隔离任务：

```json
{
  "name": "Morning brief",
  "schedule": { "kind": "cron", "expr": "0 7 * * *", "tz": "America/Los_Angeles" },
  "sessionTarget": "isolated",
  "wakeMode": "next-heartbeat",
  "payload": {
    "kind": "agentTurn",
    "message": "Summarize overnight updates.",
    "deliver": true,
    "channel": "slack",
    "to": "channel:C1234567890",
    "bestEffortDeliver": true
  },
  "isolation": { "postToMainPrefix": "Cron", "postToMainMode": "summary" }
}
```

说明：

- `schedule.kind`：`at`（`atMs`）、`every`（`everyMs`）或 `cron`（`expr`、可选 `tz`）。
- `atMs` 与 `everyMs` 均为 epoch 毫秒。
- `sessionTarget` 必须是 `"main"` 或 `"isolated"`，且必须匹配 `payload.kind`。
- 可选字段：`agentId`、`description`、`enabled`、`deleteAfterRun`、`isolation`。
- 省略时 `wakeMode` 默认 `"next-heartbeat"`。

### cron.update 参数

```json
{
  "jobId": "job-123",
  "patch": {
    "enabled": false,
    "schedule": { "kind": "every", "everyMs": 3600000 }
  }
}
```

说明：

- `jobId` 为规范字段；`id` 为兼容字段。
- 用 `agentId: null` 可清除代理绑定。

### cron.run 与 cron.remove 参数

```json
{ "jobId": "job-123", "mode": "force" }
```

```json
{ "jobId": "job-123" }
```

## 存储与历史

- 任务存储：`~/.openclaw/cron/jobs.json`（网关管理的 JSON）。
- 运行历史：`~/.openclaw/cron/runs/<jobId>.jsonl`（JSONL，自动裁剪）。
- 覆盖存储路径：配置 `cron.store`。

## 配置

```json5
{
  cron: {
    enabled: true, // default true
    store: "~/.openclaw/cron/jobs.json",
    maxConcurrentRuns: 1, // default 1
  },
}
```

完全禁用 cron：

- `cron.enabled: false`（配置）
- `OPENCLAW_SKIP_CRON=1`（环境变量）

## CLI 快速上手

一次性提醒（UTC ISO，成功后自动删除）：

```bash
openclaw cron add \
  --name "Send reminder" \
  --at "2026-01-12T18:00:00Z" \
  --session main \
  --system-event "Reminder: submit expense report." \
  --wake now \
  --delete-after-run
```

一次性提醒（主会话，立即唤醒）：

```bash
openclaw cron add \
  --name "Calendar check" \
  --at "20m" \
  --session main \
  --system-event "Next heartbeat: check calendar." \
  --wake now
```

重复隔离任务（投递到 WhatsApp）：

```bash
openclaw cron add \
  --name "Morning status" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize inbox + calendar for today." \
  --deliver \
  --channel whatsapp \
  --to "+15551234567"
```

重复隔离任务（投递到 Telegram 话题）：

```bash
openclaw cron add \
  --name "Nightly summary (topic)" \
  --cron "0 22 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize today; send to the nightly topic." \
  --deliver \
  --channel telegram \
  --to "-1001234567890:topic:123"
```

带模型与思考覆盖的隔离任务：

```bash
openclaw cron add \
  --name "Deep analysis" \
  --cron "0 6 * * 1" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Weekly deep analysis of project progress." \
  --model "opus" \
  --thinking high \
  --deliver \
  --channel whatsapp \
  --to "+15551234567"
```

代理选择（多代理场景）：

```bash
# Pin a job to agent "ops" (falls back to default if that agent is missing)
openclaw cron add --name "Ops sweep" --cron "0 6 * * *" --session isolated --message "Check ops queue" --agent ops

# Switch or clear the agent on an existing job
openclaw cron edit <jobId> --agent ops
openclaw cron edit <jobId> --clear-agent
```

手动运行（调试）：

```bash
openclaw cron run <jobId> --force
```

编辑已有任务（补丁字段）：

```bash
openclaw cron edit <jobId> \
  --message "Updated prompt" \
  --model "opus" \
  --thinking low
```

运行历史：

```bash
openclaw cron runs --id <jobId> --limit 50
```

无需创建任务的即时系统事件：

```bash
openclaw system event --mode now --text "Next heartbeat: check battery."
```

## Gateway API 接口面

- `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`
- `cron.run`（force 或 due）、`cron.runs`
  如需不创建任务的即时系统事件，请使用 [`openclaw system event`](/cli/system)。

## 故障排查

### “什么都没运行”

- 确认已启用 cron：`cron.enabled` 与 `OPENCLAW_SKIP_CRON`。
- 确认网关持续运行（cron 运行在网关进程内）。
- 对于 `cron` 计划：检查 `--tz` 时区与主机时区。

### Telegram 投递到错误位置

- 对论坛话题，使用 `-100…:topic:<id>`，明确且无歧义。
- 日志或存储的“last route”里出现 `telegram:...` 前缀是正常的；
  cron 会接受这些格式并正确解析话题 ID。

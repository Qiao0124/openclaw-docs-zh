---
summary: "Agent 运行时（嵌入式 pi-mono）、工作区契约和会话引导"
read_when:
  - 修改 Agent 运行时、工作区引导或会话行为
title: "Agent 运行时"
---

# Agent 运行时 🤖

OpenClaw 运行一个从 **pi-mono** 衍生的单一嵌入式 Agent 运行时。

## 工作区（必需）

OpenClaw 使用单一 Agent 工作区目录（`agents.defaults.workspace`）作为 Agent 的**唯一**工作目录（`cwd`），用于工具和上下文。

建议：如果缺失，使用 `openclaw setup` 创建 `~/.openclaw/openclaw.json` 并初始化工作区文件。

完整工作区布局 + 备份指南：[Agent 工作区](/concepts/agent-workspace)

如果启用了 `agents.defaults.sandbox`，非主会话可以在 `agents.defaults.sandbox.workspaceRoot` 下使用每个会话的工作区进行覆盖（参见 [Gateway 配置](/gateway/configuration)）。

## 引导文件（注入）

在 `agents.defaults.workspace` 内，OpenClaw 期望以下用户可编辑的文件：

- `AGENTS.md` — 操作指令 + "记忆"
- `SOUL.md` — 人设、边界、语气
- `TOOLS.md` — 用户维护的工具说明（例如 `imsg`、`sag`、约定）
- `BOOTSTRAP.md` — 一次性首次运行仪式（完成后删除）
- `IDENTITY.md` — Agent 名称/氛围/表情符号
- `USER.md` — 用户资料 + 首选称呼

在新会话的第一轮，OpenClaw 将这些文件的内容直接注入到 Agent 上下文中。

空文件会被跳过。大文件会被修剪和截断并带有标记，以保持提示精简（读取文件以获取完整内容）。

如果文件缺失，OpenClaw 会注入一行"缺失文件"标记（`openclaw setup` 将创建一个安全的默认模板）。

`BOOTSTRAP.md` 仅针对**全新的工作区**创建（不存在其他引导文件）。如果你在完成仪式后删除它，在后续重启时不应重新创建。

要完全禁用引导文件创建（用于预置工作区），请设置：

```json5
{ agent: { skipBootstrap: true } }
```

## 内置工具

核心工具（read/exec/edit/write 和相关系统工具）始终可用，
受工具策略约束。`apply_patch` 是可选的，由
`tools.exec.applyPatch` 控制。`TOOLS.md`**不**控制哪些工具存在；它是关于*你*希望如何使用它们的指导。

## Skills

OpenClaw 从三个位置加载 skills（工作区在名称冲突时获胜）：

- 捆绑的（随安装一起提供）
- 托管/本地：`~/.openclaw/skills`
- 工作区：`<workspace>/skills`

Skills 可以由配置/环境控制（参见 [Gateway 配置](/gateway/configuration) 中的 `skills`）。

## pi-mono 集成

OpenClaw 重用 pi-mono 代码库的部分内容（模型/工具），但**会话管理、发现和工具连接由 OpenClaw 拥有**。

- 没有 pi-coding Agent 运行时。
- 不会参考 `~/.pi/agent` 或 `<workspace>/.pi` 设置。

## 会话

会话记录以 JSONL 格式存储在：

- `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`

会话 ID 是稳定的，由 OpenClaw 选择。
旧版 Pi/Tau 会话文件夹**不**会被读取。

## 流式传输中的引导

当队列模式为 `steer` 时，入站消息被注入到当前运行中。
队列在**每次工具调用后**检查；如果存在排队消息，
则跳过当前助手消息的剩余工具调用（错误工具
结果带有"由于排队的用户消息而跳过。"），然后在下一次助手响应之前注入排队的用户消息。

当队列模式为 `followup` 或 `collect` 时，入站消息被保留直到
当前轮次结束，然后以排队的负载开始新的 Agent 轮次。有关模式 + 去抖动/上限行为，请参见 [队列](/concepts/queue)。

块流式传输在完成后立即发送已完成的助手块；它**默认关闭**（`agents.defaults.blockStreamingDefault: "off"`）。
通过 `agents.defaults.blockStreamingBreak` 调整边界（`text_end` 与 `message_end`；默认为 text_end）。
使用 `agents.defaults.blockStreamingChunk` 控制软块分块（默认为
800-1200 字符；优先选择段落分隔符，然后是换行符；最后是句子）。
使用 `agents.defaults.blockStreamingCoalesce` 合并流式块以减少
单行垃圾信息（发送前基于空闲的合并）。非 Telegram 频道需要
显式设置 `*.blockStreaming: true` 才能启用块回复。
详细工具摘要在工具开始时发出（没有去抖动）；当可用时，Control UI
通过 Agent 事件流式传输工具输出。
更多详情：[流式传输 + 分块](/concepts/streaming)。

## 模型引用

配置中的模型引用（例如 `agents.defaults.model` 和 `agents.defaults.models`）通过分割**第一个** `/` 来解析。

- 配置模型时使用 `provider/model`。
- 如果模型 ID 本身包含 `/`（OpenRouter 风格），请包含提供商前缀（示例：`openrouter/moonshotai/kimi-k2`）。
- 如果省略提供商，OpenClaw 将输入视为别名或**默认提供商**的模型（仅当模型 ID 中没有 `/` 时才有效）。

## 配置（最小化）

至少设置：

- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom`（强烈建议）

---

_下一篇：[群聊](/concepts/group-messages)_ 🦞

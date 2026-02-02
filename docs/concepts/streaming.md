---
summary: "流式传输 + 分块行为（块回复、草稿流式传输、限制）"
read_when:
  - 解释频道上流式传输或分块的工作原理
  - 更改块流式传输或频道分块行为
  - 调试重复/过早的块回复或草稿流式传输
title: "流式传输和分块"
---

# 流式传输 + 分块

OpenClaw 有两个独立的"流式传输"层：

- **块流式传输（频道）：** 在助手写入时发出已完成的**块**。这些是普通的频道消息（不是令牌增量）。
- **令牌式流式传输（仅限 Telegram）：** 在生成时用部分文本更新**草稿气泡**；最终消息在结束时发送。

今天**没有真正的令牌流式传输**到外部频道消息。Telegram 草稿流式传输是唯一的部分流表面。

## 块流式传输（频道消息）

块流式传输在助手输出可用时以粗略的块发送。

```
模型输出
  └─ text_delta/事件
       ├─ (blockStreamingBreak=text_end)
       │    └─ 分块器在缓冲区增长时发出块
       └─ (blockStreamingBreak=message_end)
            └─ 分块器在 message_end 时刷新
                   └─ 频道发送（块回复）
```

图例：

- `text_delta/事件`：模型流事件（对于非流式模型可能稀疏）。
- `分块器`：`EmbeddedBlockChunker` 应用最小/最大边界 + 中断偏好。
- `频道发送`：实际的出站消息（块回复）。

**控制：**

- `agents.defaults.blockStreamingDefault`：`"on"`/`"off"`（默认关闭）。
- 频道覆盖：`*.blockStreaming`（和每个账户变体）以在每个频道强制 `"on"`/`"off"`。
- `agents.defaults.blockStreamingBreak`：`"text_end"` 或 `"message_end"`。
- `agents.defaults.blockStreamingChunk`：`{ minChars, maxChars, breakPreference? }`。
- `agents.defaults.blockStreamingCoalesce`：`{ minChars?, maxChars?, idleMs? }`（发送前合并流式块）。
- 频道硬上限：`*.textChunkLimit`（例如 `channels.whatsapp.textChunkLimit`）。
- 频道分块模式：`*.chunkMode`（`length` 默认，`newline` 在长度分块之前在空行（段落边界）上分割）。
- Discord 软上限：`channels.discord.maxLinesPerMessage`（默认 17）将高回复分割以避免 UI 裁剪。

**边界语义：**

- `text_end`：分块器发出时立即流式传输块；在每个 `text_end` 时刷新。
- `message_end`：等待助手消息完成，然后刷新缓冲的输出。

如果缓冲的文本超过 `maxChars`，`message_end` 仍然使用分块器，因此它可以在结束时发出多个块。

## 分块算法（低/高边界）

块分块由 `EmbeddedBlockChunker` 实现：

- **下界：** 直到缓冲区 >= `minChars`（除非强制）才发出。
- **上界：** 优先在 `maxChars` 之前分割；如果强制，在 `maxChars` 处分割。
- **中断偏好：** `paragraph` → `newline` → `sentence` → `whitespace` → 硬中断。
- **代码围栏：** 从不在围栏内分割；当在 `maxChars` 处强制时，关闭 + 重新打开围栏以保持 Markdown 有效。

`maxChars` 被限制在频道 `textChunkLimit`，因此你不能超过每个频道的上限。

## 合并（合并流式块）

当启用块流式传输时，OpenClaw 可以**合并连续的块块**
在发送之前。这减少了"单行垃圾信息"，同时仍然提供
渐进输出。

- 合并在刷新前等待**空闲间隙**（`idleMs`）。
- 缓冲区由 `maxChars` 限制，如果超过将刷新。
- `minChars` 防止在累积足够文本之前发送微小片段
  （最终刷新总是发送剩余文本）。
- 连接符派生自 `blockStreamingChunk.breakPreference`
  （`paragraph` → `\n\n`，`newline` → `\n`，`sentence` → 空格）。
- 频道覆盖可通过 `*.blockStreamingCoalesce` 获得（包括每个账户配置）。
- 除非覆盖，否则 Signal/Slack/Discord 的默认合并 `minChars` 提升到 1500。

## 块之间类似人类的间隔

当启用块流式传输时，你可以在
块回复（第一个块之后）之间添加**随机暂停**。这使多气泡响应感觉
更自然。

- 配置：`agents.defaults.humanDelay`（通过 `agents.list[].humanDelay` 覆盖每个 Agent）。
- 模式：`off`（默认）、`natural`（800-2500ms）、`custom`（`minMs`/`maxMs`）。
- 仅适用于**块回复**，不适用于最终回复或工具摘要。

## "流式传输块或全部"

这映射到：

- **流式传输块：** `blockStreamingDefault: "on"` + `blockStreamingBreak: "text_end"`（随走随发）。非 Telegram 频道还需要 `*.blockStreaming: true`。
- **在结束时流式传输全部：** `blockStreamingBreak: "message_end"`（刷新一次，如果非常长可能多个块）。
- **没有块流式传输：** `blockStreamingDefault: "off"`（仅最终回复）。

**频道注意：**对于非 Telegram 频道，除非显式设置 `*.blockStreaming` 为 `true`，否则块流式传输是**关闭的**。Telegram 可以流式传输草稿
（`channels.telegram.streamMode`）而没有块回复。

配置位置提醒：`blockStreaming*` 默认值位于
`agents.defaults` 下，而不是根配置下。

## Telegram 草稿流式传输（令牌式）

Telegram 是唯一具有草稿流式传输的频道：

- 在**带有主题的私人聊天**中使用 Bot API `sendMessageDraft`。
- `channels.telegram.streamMode: "partial" | "block" | "off"`。
  - `partial`：草稿用最新的流文本更新。
  - `block`：草稿以分块更新（相同的分块器规则）。
  - `off`：没有草稿流式传输。
- 草稿分块配置（仅用于 `streamMode: "block"`）：`channels.telegram.draftChunk`（默认值：`minChars: 200`，`maxChars: 800`）。
- 草稿流式传输与块流式传输分开；块回复默认关闭，仅在非 Telegram 频道上通过 `*.blockStreaming: true` 启用。
- 最终回复仍然是普通消息。
- `/reasoning stream` 将推理写入草稿气泡（仅限 Telegram）。

当草稿流式传输激活时，OpenClaw 禁用该回复的块流式传输以避免双重流式传输。

```
Telegram（私人 + 主题）
  └─ sendMessageDraft（草稿气泡）
       ├─ streamMode=partial → 更新最新文本
       └─ streamMode=block   → 分块器更新草稿
  └─ 最终回复 → 普通消息
```

图例：

- `sendMessageDraft`：Telegram 草稿气泡（不是真正的消息）。
- `最终回复`：普通 Telegram 消息发送。

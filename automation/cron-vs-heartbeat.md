---
summary: "选择在自动化中使用心跳还是 cron 的指南"
read_when:
  - 决定如何调度周期性任务
  - 设置后台监控或通知
  - 优化周期检查的 token 使用
title: "Cron 与 Heartbeat"
---

# Cron 与 Heartbeat：何时使用

心跳（heartbeat）与 cron 都可以按计划运行任务。本指南帮助你在具体场景中做选择。

## 快速决策指南

| 场景 | 推荐 | 原因 |
| --- | --- | --- |
| 每 30 分钟检查收件箱 | Heartbeat | 可与其他检查批处理，且具备上下文 |
| 早上 9 点准时发送日报 | Cron（隔离） | 需要精确时间 |
| 监控日历即将到来的事件 | Heartbeat | 适合周期性意识维护 |
| 每周深度分析 | Cron（隔离） | 独立任务，可用不同模型 |
| 20 分钟后提醒我 | Cron（主会话，`--at`） | 一次性且时间精确 |
| 后台项目健康检查 | Heartbeat | 可与现有周期合并 |

## Heartbeat：周期性意识

心跳会在 **主会话** 中按固定间隔运行（默认 30 分钟）。它用于让代理主动检查并提示重要事项。

### 何时用心跳

- **多个周期检查**：与其设 5 个 cron 分别检查收件箱、日历、天气、通知、项目状态，不如用一次心跳批处理。
- **上下文驱动**：代理具备完整主会话上下文，能判断紧急程度。
- **对话连续性**：心跳共享同一会话，能自然跟进最近对话。
- **低开销监控**：一个心跳替代多个小轮询任务。

### 心跳优势

- **批处理多个检查**：一次代理回合可同时查看收件箱、日历和通知。
- **减少 API 调用**：一次心跳比 5 个隔离 cron 便宜。
- **上下文感知**：代理了解你在做什么并能优先级排序。
- **智能抑制**：若无关注事项，代理返回 `HEARTBEAT_OK`，不会投递消息。
- **自然时间**：会随着队列负载略有漂移，但对大多数监控无影响。

### 心跳示例：HEARTBEAT.md 清单

```md
# 心跳检查清单

- 检查邮箱中的紧急邮件
- 查看未来 2 小时内的日程
- 如后台任务完成，汇总结果
- 若 8+ 小时无活动，发送简短问候
```

代理会在每次心跳时读取这份清单，并在一次回合内处理全部事项。

### 配置心跳

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 间隔
        target: "last", // 投递位置
        activeHours: { start: "08:00", end: "22:00" }, // 可选
      },
    },
  },
}
```

完整配置见 [Heartbeat](/gateway/heartbeat)。

## Cron：精确调度

Cron 任务在 **精确时间** 运行，并可在隔离会话中执行，不影响主会话上下文。

### 何时用 cron

- **需要精确时间**：比如“每周一 9:00 发送”。
- **独立任务**：不需要对话上下文。
- **不同模型/思考**：重分析可用更强模型。
- **一次性提醒**：用 `--at`。
- **噪声高/频繁任务**：避免污染主会话历史。
- **外部触发**：任务需要独立于代理活跃状态运行。

### Cron 优势

- **精确时间**：5 字段 cron 表达式 + 时区支持。
- **会话隔离**：在 `cron:<jobId>` 中运行，不污染主历史。
- **模型覆盖**：每个任务可选更便宜或更强模型。
- **投递控制**：可直接投递到渠道；默认仍会向主会话发摘要（可配置）。
- **无需主会话上下文**：即使主会话闲置或已压缩也能运行。
- **一次性支持**：`--at` 指定精确未来时间戳。

### Cron 示例：每日早报

```bash
openclaw cron add   --name "Morning briefing"   --cron "0 7 * * *"   --tz "America/New_York"   --session isolated   --message "Generate todays briefing: weather, calendar, top emails, news summary."   --model opus   --deliver   --channel whatsapp   --to "+15551234567"
```

这会在纽约时间 7:00 准时运行，使用 Opus 保证质量，并直接投递到 WhatsApp。

### Cron 示例：一次性提醒

```bash
openclaw cron add   --name "Meeting reminder"   --at "20m"   --session main   --system-event "Reminder: standup meeting starts in 10 minutes."   --wake now   --delete-after-run
```

完整 CLI 参考见 [Cron jobs](/automation/cron-jobs)。

## 决策流程图

```
是否需要在精确时间运行？
  是 -> 使用 cron
  否 -> 继续...

是否需要与主会话隔离？
  是 -> 使用 cron（隔离）
  否 -> 继续...

能与其他周期性检查合并吗？
  是 -> 使用心跳（加入 HEARTBEAT.md）
  否 -> 使用 cron

是否为一次性提醒？
  是 -> 用 cron + --at
  否 -> 继续...

是否需要不同模型或思考级别？
  是 -> 使用 cron（隔离）并设置 --model/--thinking
  否 -> 使用心跳
```

## 组合使用

最高效的配置往往是 **两者搭配**：

1. **Heartbeat** 处理常规监控（收件箱、日历、通知），每 30 分钟一次合并处理。
2. **Cron** 处理精确调度（日报、周报）与一次性提醒。

### 示例：高效自动化配置

**HEARTBEAT.md**（每 30 分钟检查）：

```md
# 心跳检查清单

- 扫描收件箱中的紧急邮件
- 查看未来 2 小时日程
- 回顾待办
- 若 8+ 小时无消息，轻量问候
```

**Cron 任务**（精确时间）：

```bash
# 每天 7 点晨间简报
openclaw cron add --name "Morning brief" --cron "0 7 * * *" --session isolated --message "..." --deliver

# 每周一 9 点项目复盘
openclaw cron add --name "Weekly review" --cron "0 9 * * 1" --session isolated --message "..." --model opus

# 一次性提醒
openclaw cron add --name "Call back" --at "2h" --session main --system-event "Call back the client" --wake now
```

## Lobster：带审批的确定性工作流

Lobster 是用于 **多步骤工具流水线** 的工作流运行时，强调确定性执行与显式审批。
当任务不止一个代理回合、且需要可恢复流程与人工检查点时使用它。

### Lobster 适用场景

- **多步自动化**：需要固定工具调用流水线，而不是一次性提示。
- **审批关卡**：有副作用的步骤需要你确认后继续。
- **可恢复运行**：暂停后继续执行，无需重跑前面步骤。

### 它与心跳/cron 的配合

- **Heartbeat/cron** 决定 _何时_ 触发。
- **Lobster** 定义 _触发后执行哪些步骤_。

定时流程可用 cron 或 heartbeat 触发一个调用 Lobster 的代理回合。
临时流程可直接调用 Lobster。

### 运行说明（来自代码）

- Lobster 以 **本地子进程**（`lobster` CLI）在工具模式下运行，并返回 **JSON envelope**。
- 工具返回 `needs_approval` 时，用 `resumeToken` + `approve` 继续。
- 该工具为 **可选插件**；建议通过 `tools.alsoAllow: ["lobster"]` 启用。
- 如果传入 `lobsterPath`，必须是 **绝对路径**。

详见 [Lobster](/tools/lobster)。

## 主会话 vs 隔离会话

心跳与 cron 都会与主会话产生交互，但方式不同：

| | Heartbeat | Cron（主会话） | Cron（隔离） |
| --- | --- | --- | --- |
| 会话 | 主会话 | 主会话（系统事件） | `cron:<jobId>` |
| 历史 | 共享 | 共享 | 每次新建 |
| 上下文 | 完整 | 完整 | 无（全新） |
| 模型 | 主会话模型 | 主会话模型 | 可覆盖 |
| 输出 | 非 `HEARTBEAT_OK` 时投递 | 心跳提示词 + 事件 | 向主会话发摘要 |

### 何时使用主会话 cron

在你希望以下行为时，使用 `--session main` + `--system-event`：

- 提醒/事件出现在主会话上下文中
- 代理在下一次心跳里用完整上下文处理
- 不产生独立隔离运行

```bash
openclaw cron add   --name "Check project"   --every "4h"   --session main   --system-event "Time for a project health check"   --wake now
```

### 何时使用隔离 cron

当你需要：

- 无历史上下文的干净环境
- 不同的模型或思考设置
- 输出直接投递到渠道（默认仍会向主会话发摘要）
- 不污染主会话历史的运行记录

```bash
openclaw cron add   --name "Deep analysis"   --cron "0 6 * * 0"   --session isolated   --message "Weekly codebase analysis..."   --model opus   --thinking high   --deliver
```

## 成本考量

| 机制 | 成本特征 |
| --- | --- |
| Heartbeat | 每 N 分钟一个回合；随 HEARTBEAT.md 大小线性增长 |
| Cron（主会话） | 向下一次心跳添加事件（无隔离回合） |
| Cron（隔离） | 每个任务完整代理回合；可用更便宜模型 |

**建议**：

- 保持 `HEARTBEAT.md` 简洁，降低 token 开销。
- 把类似检查合并进心跳，而非多个 cron 任务。
- 若只需内部处理，心跳可用 `target: "none"`。
- 日常任务可用更便宜模型的隔离 cron。

## 相关内容

- [Heartbeat](/gateway/heartbeat) - 完整心跳配置
- [Cron jobs](/automation/cron-jobs) - 完整 cron CLI 与 API 参考
- [System](/cli/system) - 系统事件与心跳控制

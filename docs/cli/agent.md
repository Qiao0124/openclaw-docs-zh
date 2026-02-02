---
summary: "`openclaw agent` 的 CLI 参考（通过 Gateway 发送一次 agent 轮次）"
read_when:
  - 你想从脚本运行一次 agent 轮次（可选择传递回复）
title: "agent"
---

# `openclaw agent`

通过 Gateway 运行一次 agent 轮次（使用 `--local` 进行嵌入式运行）。
使用 `--agent <id>` 直接定位已配置的 agent。

相关文档：

- Agent 发送工具：[Agent send](/tools/agent-send)

## 示例

```bash
openclaw agent --to +15555550123 --message "status update" --deliver
openclaw agent --agent ops --message "Summarize logs"
openclaw agent --session-id 1234 --message "Summarize inbox" --thinking medium
openclaw agent --agent ops --message "Generate report" --deliver --reply-channel slack --reply-to "#reports"
```

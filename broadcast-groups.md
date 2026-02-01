---
summary: "将一条 WhatsApp 消息广播给多个代理"
read_when:
  - 配置广播组
  - 排查 WhatsApp 多代理回复问题
status: experimental
title: "广播组"
---

# 广播组

**状态：** 实验性  
**版本：** 2026.1.9 新增

## 概览

广播组允许多个代理同时处理并响应同一条消息。你可以在一个 WhatsApp 群组或私聊中
组成专门的代理团队，仍然只使用一个手机号。

当前范围：**仅 WhatsApp**（Web 渠道）。

广播组在渠道 allowlist 与群组激活规则之后评估。对 WhatsApp 群组而言，
这意味着只有在 OpenClaw 原本会回复时才会广播（例如：按提及触发，取决于群设置）。

## 使用场景

### 1. 专业化代理团队

部署多个有明确分工的代理：

```
Group: "Development Team"
Agents:
  - CodeReviewer (reviews code snippets)
  - DocumentationBot (generates docs)
  - SecurityAuditor (checks for vulnerabilities)
  - TestGenerator (suggests test cases)
```

每个代理处理同一条消息，并提供各自的专业视角。

### 2. 多语言支持

```
Group: "International Support"
Agents:
  - Agent_EN (responds in English)
  - Agent_DE (responds in German)
  - Agent_ES (responds in Spanish)
```

### 3. 质量保障流程

```
Group: "Customer Support"
Agents:
  - SupportAgent (provides answer)
  - QAAgent (reviews quality, only responds if issues found)
```

### 4. 任务自动化

```
Group: "Project Management"
Agents:
  - TaskTracker (updates task database)
  - TimeLogger (logs time spent)
  - ReportGenerator (creates summaries)
```

## 配置

### 基础配置

添加顶层 `broadcast` 配置（与 `bindings` 同级）。键为 WhatsApp peer id：

- 群聊：群组 JID（如 `120363403215116621@g.us`）
- 私聊：E.164 电话号码（如 `+15551234567`）

```json
{
  "broadcast": {
    "120363403215116621@g.us": ["alfred", "baerbel", "assistant3"]
  }
}
```

**结果：** 当 OpenClaw 在该聊天中会回复时，会运行三个代理。

### 处理策略

控制代理处理消息的方式：

#### 并行（默认）

所有代理同时处理：

```json
{
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"]
  }
}
```

#### 串行

代理按顺序处理（前一个完成后再处理下一个）：

```json
{
  "broadcast": {
    "strategy": "sequential",
    "120363403215116621@g.us": ["alfred", "baerbel"]
  }
}
```

### 完整示例

```json
{
  "agents": {
    "list": [
      {
        "id": "code-reviewer",
        "name": "Code Reviewer",
        "workspace": "/path/to/code-reviewer",
        "sandbox": { "mode": "all" }
      },
      {
        "id": "security-auditor",
        "name": "Security Auditor",
        "workspace": "/path/to/security-auditor",
        "sandbox": { "mode": "all" }
      },
      {
        "id": "docs-generator",
        "name": "Documentation Generator",
        "workspace": "/path/to/docs-generator",
        "sandbox": { "mode": "all" }
      }
    ]
  },
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": ["code-reviewer", "security-auditor", "docs-generator"],
    "120363424282127706@g.us": ["support-en", "support-de"],
    "+15555550123": ["assistant", "logger"]
  }
}
```

## 工作原理

### 消息流

1. **收到消息**：进入 WhatsApp 群组的消息
2. **广播检查**：系统检查 peer ID 是否在 `broadcast`
3. **在广播列表中**：
   - 列表中的所有代理都会处理该消息
   - 每个代理有独立会话 key 与隔离上下文
   - 代理并行（默认）或串行处理
4. **不在广播列表中**：
   - 使用常规路由（第一个匹配的 binding）

注意：广播组不会绕过渠道 allowlist 或群组激活规则（提及/命令等）。
它只改变 **在消息可处理时运行哪些代理**。

### 会话隔离

广播组中的每个代理完全隔离：

- **会话 key**（`agent:alfred:whatsapp:group:120363...` vs `agent:baerbel:whatsapp:group:120363...`）
- **对话历史**（代理看不到其他代理的消息）
- **工作区**（如配置了 sandbox，则各自隔离）
- **工具访问**（不同的 allow/deny 列表）
- **记忆/上下文**（独立的 IDENTITY.md、SOUL.md 等）
- **群组上下文缓冲**（用于上下文的最近群消息）按 peer 共享，因此所有广播代理看到相同上下文

这让每个代理都可以拥有：

- 不同人格
- 不同工具权限（如只读 vs 读写）
- 不同模型（如 opus vs sonnet）
- 不同已安装技能

### 示例：隔离会话

在群组 `120363403215116621@g.us` 中，代理为 `["alfred", "baerbel"]`：

**Alfred 上下文：**

```
Session: agent:alfred:whatsapp:group:120363403215116621@g.us
History: [user message, alfred previous responses]
Workspace: /Users/user/openclaw-alfred/
Tools: read, write, exec
```

**Bärbel 上下文：**

```
Session: agent:baerbel:whatsapp:group:120363403215116621@g.us
History: [user message, baerbel previous responses]
Workspace: /Users/user/openclaw-baerbel/
Tools: read only
```

## 最佳实践

### 1. 保持代理专注

为每个代理设计单一、清晰的职责：

```json
{
  "broadcast": {
    "DEV_GROUP": ["formatter", "linter", "tester"]
  }
}
```

✅ **好：** 每个代理只做一件事  
❌ **坏：** 一个泛化的 “dev-helper” 代理

### 2. 使用描述性命名

让代理职责一目了然：

```json
{
  "agents": {
    "security-scanner": { "name": "Security Scanner" },
    "code-formatter": { "name": "Code Formatter" },
    "test-generator": { "name": "Test Generator" }
  }
}
```

### 3. 配置不同工具权限

为代理只开放所需工具：

```json
{
  "agents": {
    "reviewer": {
      "tools": { "allow": ["read", "exec"] } // 只读
    },
    "fixer": {
      "tools": { "allow": ["read", "write", "edit", "exec"] } // 读写
    }
  }
}
```

### 4. 关注性能

代理数量较多时，建议：

- 使用 `"strategy": "parallel"`（默认）以提升速度
- 将每个广播组限制在 5-10 个代理
- 对简单代理使用更快模型

### 5. 优雅处理失败

代理独立失败，一个代理的错误不会阻塞其他代理：

```
Message → [Agent A ✓, Agent B ✗ error, Agent C ✓]
Result: Agent A and C respond, Agent B logs error
```

## 兼容性

### 提供方

广播组当前支持：

- ✅ WhatsApp（已实现）
- 🚧 Telegram（计划中）
- 🚧 Discord（计划中）
- 🚧 Slack（计划中）

### 路由

广播组可与现有路由共存：

```json
{
  "bindings": [
    {
      "match": { "channel": "whatsapp", "peer": { "kind": "group", "id": "GROUP_A" } },
      "agentId": "alfred"
    }
  ],
  "broadcast": {
    "GROUP_B": ["agent1", "agent2"]
  }
}
```

- `GROUP_A`：只有 alfred 回复（常规路由）
- `GROUP_B`：agent1 与 agent2 都回复（广播）

**优先级：** `broadcast` 优先于 `bindings`。

## 故障排查

### 代理不回复

**检查：**

1. `agents.list` 中存在对应 agent id
2. peer id 格式正确（如 `120363403215116621@g.us`）
3. 代理未被 deny 列表屏蔽

**调试：**

```bash
tail -f ~/.openclaw/logs/gateway.log | grep broadcast
```

### 只有一个代理回复

**原因：** peer id 可能在 `bindings` 中，但不在 `broadcast` 中。

**解决：** 将其加入 broadcast 配置，或从 bindings 中移除。

### 性能问题

**多代理较慢时：**

- 减少每组代理数量
- 使用轻量模型（sonnet 而非 opus）
- 检查 sandbox 启动时间

## 示例

### 示例 1：代码审查团队

```json
{
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": [
      "code-formatter",
      "security-scanner",
      "test-coverage",
      "docs-checker"
    ]
  },
  "agents": {
    "list": [
      {
        "id": "code-formatter",
        "workspace": "~/agents/formatter",
        "tools": { "allow": ["read", "write"] }
      },
      {
        "id": "security-scanner",
        "workspace": "~/agents/security",
        "tools": { "allow": ["read", "exec"] }
      },
      {
        "id": "test-coverage",
        "workspace": "~/agents/testing",
        "tools": { "allow": ["read", "exec"] }
      },
      { "id": "docs-checker", "workspace": "~/agents/docs", "tools": { "allow": ["read"] } }
    ]
  }
}
```

**用户发送：** 代码片段  
**回复：**

- code-formatter："修复了缩进并添加了类型提示"
- security-scanner："⚠️ 第 12 行存在 SQL 注入漏洞"
- test-coverage："覆盖率为 45%，缺少错误场景测试"
- docs-checker："函数 `process_data` 缺少 docstring"

### 示例 2：多语言支持

```json
{
  "broadcast": {
    "strategy": "sequential",
    "+15555550123": ["detect-language", "translator-en", "translator-de"]
  },
  "agents": {
    "list": [
      { "id": "detect-language", "workspace": "~/agents/lang-detect" },
      { "id": "translator-en", "workspace": "~/agents/translate-en" },
      { "id": "translator-de", "workspace": "~/agents/translate-de" }
    ]
  }
}
```

## API 参考

### 配置 Schema

```typescript
interface OpenClawConfig {
  broadcast?: {
    strategy?: "parallel" | "sequential";
    [peerId: string]: string[];
  };
}
```

### 字段

- `strategy`（可选）：代理处理方式
  - `"parallel"`（默认）：所有代理并行处理
  - `"sequential"`：代理按数组顺序处理
- `[peerId]`：WhatsApp 群组 JID、E.164 电话号码或其他 peer ID
  - 值：需要处理消息的代理 ID 数组

## 限制

1. **最大代理数：** 无硬限制，但 10+ 代理可能变慢
2. **共享上下文：** 代理看不到彼此的回复（设计如此）
3. **消息顺序：** 并行回复可能无序到达
4. **速率限制：** 所有代理都计入 WhatsApp 速率限制

## 未来增强

规划中的特性：

- [ ] 共享上下文模式（代理可看到彼此回复）
- [ ] 代理协作（代理之间可相互通知）
- [ ] 动态代理选择（根据消息内容选择代理）
- [ ] 代理优先级（部分代理更早回复）

## 相关内容

- [多代理配置](/multi-agent-sandbox-tools)
- [路由配置](/concepts/channel-routing)
- [会话管理](/concepts/sessions)

---
summary: "OpenClaw 内存工作原理（工作区文件 + 自动内存刷新）"
read_when:
  - 你需要内存文件布局和工作流程
  - 你想调整自动预压缩内存刷新
title: "内存"
---

# 内存

OpenClaw 内存是 **Agent 工作区中的纯 Markdown**。文件是
事实来源；模型只"记住"写入磁盘的内容。

内存搜索工具由活动的内存插件提供（默认：
`memory-core`）。使用 `plugins.slots.memory = "none"` 禁用内存插件。

## 内存文件（Markdown）

默认工作区布局使用两个内存层：

- `memory/YYYY-MM-DD.md`
  - 每日日志（仅追加）。
  - 在会话开始时读取今天 + 昨天的内容。
- `MEMORY.md`（可选）
  - 策划的长期内存。
  - **仅在主私有会话中加载**（不在群组上下文中）。

这些文件位于工作区下（`agents.defaults.workspace`，默认
`~/.openclaw/workspace`）。完整布局请参见 [Agent 工作区](/concepts/agent-workspace)。

## 何时写入内存

- 决策、偏好和持久事实写入 `MEMORY.md`。
- 日常笔记和运行上下文写入 `memory/YYYY-MM-DD.md`。
- 如果有人说"记住这个"，请写下来（不要保留在 RAM 中）。
- 这个领域仍在发展中。提醒模型存储内存会有帮助；它会知道该怎么做。
- 如果你想让某些内容持久化，**让机器人将其写入**内存。

## 自动内存刷新（预压缩 ping）

当会话**接近自动压缩**时，OpenClaw 会触发一个**静默的
Agent 轮次**，提醒模型在上下文被压缩**之前**将持久内存写入磁盘。默认提示明确说明模型*可以回复*，但通常 `NO_REPLY` 是正确的响应，以便用户永远看不到这个轮次。

这由 `agents.defaults.compaction.memoryFlush` 控制：

```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store.",
        },
      },
    },
  },
}
```

详情：

- **软阈值**：当会话令牌估计值超过
  `contextWindow - reserveTokensFloor - softThresholdTokens` 时触发刷新。
- 默认**静默**：提示包含 `NO_REPLY`，因此不会传递任何内容。
- **两个提示**：用户提示加上系统提示追加提醒。
- **每个压缩周期一次刷新**（在 `sessions.json` 中跟踪）。
- **工作区必须可写**：如果会话以
  `workspaceAccess: "ro"` 或 `"none"` 运行沙盒，则跳过刷新。

有关完整的压缩生命周期，请参见
[会话管理 + 压缩](/reference/session-management-compaction)。

## 向量内存搜索

OpenClaw 可以在 `MEMORY.md` 和 `memory/*.md` 上构建一个小型向量索引（加上
你选择加入的任何额外目录或文件），以便语义查询可以找到相关的
笔记，即使措辞不同。

默认值：

- 默认启用。
- 监视内存文件的变化（去抖动）。
- 默认使用远程嵌入。如果未设置 `memorySearch.provider`，OpenClaw 自动选择：
  1. 如果配置了 `memorySearch.local.modelPath` 且文件存在，则为 `local`。
  2. 如果可以解析 OpenAI 密钥，则为 `openai`。
  3. 如果可以解析 Gemini 密钥，则为 `gemini`。
  4. 否则内存搜索保持禁用直到配置。
- 本地模式使用 node-llama-cpp，可能需要 `pnpm approve-builds`。
- 在 SQLite 内部使用 sqlite-vec 加速向量搜索。

远程嵌入**需要**嵌入提供商的 API 密钥。OpenClaw
从认证配置文件、`models.providers.*.apiKey` 或环境
变量解析密钥。Codex OAuth 仅涵盖聊天/补全，**不**满足
内存搜索的嵌入。对于 Gemini，使用 `GEMINI_API_KEY` 或
`models.providers.google.apiKey`。使用自定义 OpenAI 兼容端点时，
设置 `memorySearch.remote.apiKey`（和可选的 `memorySearch.remote.headers`）。

### 额外的内存路径

如果你想索引默认工作区布局之外的 Markdown 文件，请添加
显式路径：

```json5
agents: {
  defaults: {
    memorySearch: {
      extraPaths: ["../team-docs", "/srv/shared-notes/overview.md"]
    }
  }
}
```

注意：

- 路径可以是绝对路径或相对于工作区的路径。
- 目录会递归扫描 `.md` 文件。
- 仅索引 Markdown 文件。
- 符号链接被忽略（文件或目录）。

### Gemini 嵌入（原生）

将提供商设置为 `gemini` 以直接使用 Gemini 嵌入 API：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "gemini",
      model: "gemini-embedding-001",
      remote: {
        apiKey: "YOUR_GEMINI_API_KEY"
      }
    }
  }
}
```

注意：

- `remote.baseUrl` 是可选的（默认为 Gemini API 基础 URL）。
- `remote.headers` 允许你在需要时添加额外的头。
- 默认模型：`gemini-embedding-001`。

如果你想使用**自定义 OpenAI 兼容端点**（OpenRouter、vLLM 或代理），
你可以将 `remote` 配置与 OpenAI 提供商一起使用：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_OPENAI_COMPAT_API_KEY",
        headers: { "X-Custom-Header": "value" }
      }
    }
  }
}
```

如果你不想设置 API 密钥，请使用 `memorySearch.provider = "local"` 或设置
`memorySearch.fallback = "none"`。

回退：

- `memorySearch.fallback` 可以是 `openai`、`gemini`、`local` 或 `none`。
- 仅当主嵌入提供商失败时才使用回退提供商。

批量索引（OpenAI + Gemini）：

- 默认为 OpenAI 和 Gemini 嵌入启用。设置 `agents.defaults.memorySearch.remote.batch.enabled = false` 以禁用。
- 默认行为等待批量完成；如果需要，调整 `remote.batch.wait`、`remote.batch.pollIntervalMs` 和 `remote.batch.timeoutMinutes`。
- 设置 `remote.batch.concurrency` 以控制我们并行提交的批量作业数量（默认：2）。
- 当 `memorySearch.provider = "openai"` 或 `"gemini"` 并使用相应的 API 密钥时，应用批量模式。
- Gemini 批量作业使用异步嵌入批量端点，需要 Gemini Batch API 可用。

为什么 OpenAI 批量快速 + 便宜：

- 对于大型回填，OpenAI 通常是我们支持的最快选项，因为我们可以将许多嵌入请求提交到单个批量作业中，让 OpenAI 异步处理它们。
- OpenAI 为 Batch API 工作负载提供折扣定价，因此大型索引运行通常比同步发送相同请求更便宜。
- 有关详细信息，请参见 OpenAI Batch API 文档和定价：
  - https://platform.openai.com/docs/api-reference/batch
  - https://platform.openai.com/pricing

配置示例：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      model: "text-embedding-3-small",
      fallback: "openai",
      remote: {
        batch: { enabled: true, concurrency: 2 }
      },
      sync: { watch: true }
    }
  }
}
```

工具：

- `memory_search` — 返回带有文件 + 行范围的片段。
- `memory_get` — 按路径读取内存文件内容。

本地模式：

- 设置 `agents.defaults.memorySearch.provider = "local"`。
- 提供 `agents.defaults.memorySearch.local.modelPath`（GGUF 或 `hf:` URI）。
- 可选：设置 `agents.defaults.memorySearch.fallback = "none"` 以避免远程回退。

### 内存工具的工作原理

- `memory_search` 语义搜索来自 `MEMORY.md` + `memory/**/*.md` 的 Markdown 块（~400 令牌目标，80 令牌重叠）。它返回片段文本（上限 ~700 字符）、文件路径、行范围、分数、提供商/模型，以及我们是否从本地 → 远程嵌入回退。不返回完整文件负载。
- `memory_get` 读取特定的内存 Markdown 文件（相对于工作区），可选从起始行开始读取 N 行。只有当路径明确列在 `memorySearch.extraPaths` 中时，才允许 `MEMORY.md` / `memory/` 之外的路径。
- 仅当 `memorySearch.enabled` 为 Agent 解析为 true 时，这两个工具才会启用。

### 什么被索引（以及何时）

- 文件类型：仅 Markdown（`MEMORY.md`、`memory/**/*.md`，以及 `memorySearch.extraPaths` 下的任何 `.md` 文件）。
- 索引存储：每个 Agent 的 SQLite 位于 `~/.openclaw/memory/<agentId>.sqlite`（可通过 `agents.defaults.memorySearch.store.path` 配置，支持 `{agentId}` 令牌）。
- 新鲜度：`MEMORY.md`、`memory/` 和 `memorySearch.extraPaths` 的监视器将索引标记为脏（去抖动 1.5 秒）。同步在会话启动时、搜索时或按间隔调度，并异步运行。会话记录使用增量阈值触发后台同步。
- 重新索引触发器：索引存储嵌入**提供商/模型 + 端点指纹 + 分块参数**。如果其中任何一个发生变化，OpenClaw 会自动重置并重新索引整个存储。

### 混合搜索（BM25 + 向量）

启用时，OpenClaw 结合：

- **向量相似度**（语义匹配，措辞可以不同）
- **BM25 关键字相关性**（精确令牌如 ID、环境变量、代码符号）

如果你的平台无法使用全文搜索，OpenClaw 会回退到仅向量搜索。

#### 为什么使用混合？

向量搜索擅长"这意味着相同的事情"：

- "Mac Studio gateway host" 与 "the machine running the gateway"
- "debounce file updates" 与 "avoid indexing on every write"

但它在精确、高信号的令牌上可能较弱：

- ID（`a828e60`、`b3b9895a…`）
- 代码符号（`memorySearch.query.hybrid`）
- 错误字符串（"sqlite-vec unavailable"）

BM25（全文）正好相反：在精确令牌上强，在释义上弱。
混合搜索是务实的中间地带：**使用两种检索信号**，以便你获得
"自然语言"查询和"大海捞针"查询的良好结果。

#### 我们如何合并结果（当前设计）

实现草图：

1. 从两侧检索候选池：

- **向量**：按余弦相似度的前 `maxResults * candidateMultiplier`。
- **BM25**：按 FTS5 BM25 排名的前 `maxResults * candidateMultiplier`（越低越好）。

2. 将 BM25 排名转换为 0..1 左右的分数：

- `textScore = 1 / (1 + max(0, bm25Rank))`

3. 按块 ID 合并候选并计算加权分数：

- `finalScore = vectorWeight * vectorScore + textWeight * textScore`

注意：

- `vectorWeight` + `textWeight` 在配置解析中归一化为 1.0，因此权重表现为百分比。
- 如果嵌入不可用（或提供商返回零向量），我们仍运行 BM25 并返回关键字匹配。
- 如果无法创建 FTS5，我们保持仅向量搜索（没有硬失败）。

这不是"IR 理论完美"，但它简单、快速，并且倾向于改善真实笔记的召回率/精确率。
如果我们以后想变得更花哨，常见的下一步是倒数排名融合（RRF）或分数归一化
（最小/最大或 z 分数）然后再混合。

配置：

```json5
agents: {
  defaults: {
    memorySearch: {
      query: {
        hybrid: {
          enabled: true,
          vectorWeight: 0.7,
          textWeight: 0.3,
          candidateMultiplier: 4
        }
      }
    }
  }
}
```

### 嵌入缓存

OpenClaw 可以在 SQLite 中缓存**块嵌入**，因此重新索引和频繁更新（特别是会话记录）不会重新嵌入未更改的文本。

配置：

```json5
agents: {
  defaults: {
    memorySearch: {
      cache: {
        enabled: true,
        maxEntries: 50000
      }
    }
  }
}
```

### 会话内存搜索（实验性）

你可以选择性地索引**会话记录**并通过 `memory_search` 展示它们。
这由一个实验性标志控制。

```json5
agents: {
  defaults: {
    memorySearch: {
      experimental: { sessionMemory: true },
      sources: ["memory", "sessions"]
    }
  }
}
```

注意：

- 会话索引是**选择加入**（默认关闭）。
- 会话更新是去抖动的，一旦它们超过增量阈值就会**异步索引**（尽力而为）。
- `memory_search` 从不阻塞索引；在后台同步完成之前，结果可能略微过时。
- 结果仍然只包含片段；`memory_get` 仍限于内存文件。
- 会话索引按 Agent 隔离（仅索引该 Agent 的会话日志）。
- 会话日志保存在磁盘上（`~/.openclaw/agents/<agentId>/sessions/*.jsonl`）。任何具有文件系统访问权限的进程/用户都可以读取它们，因此将磁盘访问视为信任边界。对于更严格的隔离，在单独的 OS 用户或主机下运行 Agent。

增量阈值（显示的默认值）：

```json5
agents: {
  defaults: {
    memorySearch: {
      sync: {
        sessions: {
          deltaBytes: 100000,   // ~100 KB
          deltaMessages: 50     // JSONL 行
        }
      }
    }
  }
}
```

### SQLite 向量加速（sqlite-vec）

当 sqlite-vec 扩展可用时，OpenClaw 将嵌入存储在
SQLite 虚拟表（`vec0`）中，并在数据库中执行向量距离查询。这使搜索保持快速，而无需将每个嵌入加载到 JS 中。

配置（可选）：

```json5
agents: {
  defaults: {
    memorySearch: {
      store: {
        vector: {
          enabled: true,
          extensionPath: "/path/to/sqlite-vec"
        }
      }
    }
  }
}
```

注意：

- `enabled` 默认为 true；禁用时，搜索回退到存储嵌入的进程中余弦相似度。
- 如果 sqlite-vec 扩展缺失或加载失败，OpenClaw 记录错误并继续使用 JS 回退（没有向量表）。
- `extensionPath` 覆盖捆绑的 sqlite-vec 路径（对自定义构建或非标准安装位置有用）。

### 本地嵌入自动下载

- 默认本地嵌入模型：`hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf`（~0.6 GB）。
- 当 `memorySearch.provider = "local"` 时，`node-llama-cpp` 解析 `modelPath`；如果 GGUF 缺失，它会**自动下载**到缓存（或如果设置了 `local.modelCacheDir`），然后加载它。下载在重试时恢复。
- 原生构建要求：运行 `pnpm approve-builds`，选择 `node-llama-cpp`，然后 `pnpm rebuild node-llama-cpp`。
- 回退：如果本地设置失败且 `memorySearch.fallback = "openai"`，我们会自动切换到远程嵌入（`openai/text-embedding-3-small`，除非覆盖）并记录原因。

### 自定义 OpenAI 兼容端点示例

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_REMOTE_API_KEY",
        headers: {
          "X-Organization": "org-id",
          "X-Project": "project-id"
        }
      }
    }
  }
}
```

注意：

- `remote.*` 优先于 `models.providers.openai.*`。
- `remote.headers` 与 OpenAI 头合并；在键冲突时远程获胜。省略 `remote.headers` 以使用 OpenAI 默认值。

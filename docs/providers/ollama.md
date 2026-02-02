---
summary: "使用 Ollama (本地 LLM 运行时) 运行 OpenClaw"
read_when:
  - 你想通过 Ollama 使用本地模型运行 OpenClaw
  - 你需要 Ollama 设置和配置指导
title: "Ollama"
---

# Ollama

Ollama 是一个本地 LLM 运行时，让你可以轻松地在机器上运行开源模型。OpenClaw 集成 Ollama 的 OpenAI 兼容 API，并且当你使用 `OLLAMA_API_KEY`（或认证配置文件）选择加入且未定义显式的 `models.providers.ollama` 条目时，可以**自动发现支持工具的模型**。

## 快速开始

1. 安装 Ollama: https://ollama.ai

2. 拉取模型:

```bash
ollama pull llama3.3
# 或
ollama pull qwen2.5-coder:32b
# 或
ollama pull deepseek-r1:32b
```

3. 为 OpenClaw 启用 Ollama (任何值都可以; Ollama 不需要真实的密钥):

```bash
# 设置环境变量
export OLLAMA_API_KEY="ollama-local"

# 或在配置文件中配置
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

4. 使用 Ollama 模型:

```json5
{
  agents: {
    defaults: {
      model: { primary: "ollama/llama3.3" },
    },
  },
}
```

## 模型发现 (隐式 provider)

当你设置了 `OLLAMA_API_KEY`（或认证配置文件）且**未**定义 `models.providers.ollama` 时，OpenClaw 会从本地 Ollama 实例 `http://127.0.0.1:11434` 发现模型:

- 查询 `/api/tags` 和 `/api/show`
- 仅保留报告 `tools` 能力的模型
- 当模型报告 `thinking` 时标记 `reasoning`
- 从 `model_info["<arch>.context_length"]` 读取 `contextWindow`（如果可用）
- 将 `maxTokens` 设置为 context window 的 10 倍
- 将所有成本设置为 `0`

这避免了手动模型条目，同时保持目录与 Ollama 的能力同步。

查看可用模型:

```bash
ollama list
openclaw models list
```

添加新模型，只需用 Ollama 拉取:

```bash
ollama pull mistral
```

新模型将被自动发现并可用。

如果你显式设置了 `models.providers.ollama`，自动发现将被跳过，你必须手动定义模型（见下文）。

## 配置

### 基本设置 (隐式发现)

启用 Ollama 的最简单方式是通过环境变量:

```bash
export OLLAMA_API_KEY="ollama-local"
```

### 显式设置 (手动模型)

在以下情况下使用显式配置:

- Ollama 运行在其他主机/端口。
- 你想强制指定特定的 context window 或模型列表。
- 你想包含不报告工具支持的模型。

```json5
{
  models: {
    providers: {
      ollama: {
        // 使用包含 /v1 的主机以兼容 OpenAI API
        baseUrl: "http://ollama-host:11434/v1",
        apiKey: "ollama-local",
        api: "openai-completions",
        models: [
          {
            id: "llama3.3",
            name: "Llama 3.3",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192 * 10
          }
        ]
      }
    }
  }
}
```

如果设置了 `OLLAMA_API_KEY`，你可以在 provider 条目中省略 `apiKey`，OpenClaw 将填充它用于可用性检查。

### 自定义 base URL (显式配置)

如果 Ollama 运行在不同的主机或端口（显式配置禁用自动发现，因此需要手动定义模型）:

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434/v1",
      },
    },
  },
}
```

### 模型选择

配置完成后，所有 Ollama 模型都可用:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/llama3.3",
        fallback: ["ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## 高级

### Reasoning 模型

当 Ollama 在 `/api/show` 中报告 `thinking` 时，OpenClaw 将模型标记为支持 reasoning:

```bash
ollama pull deepseek-r1:32b
```

### 模型成本

Ollama 是免费的本地运行，因此所有模型成本都设置为 $0。

### Context windows

对于自动发现的模型，OpenClaw 使用 Ollama 报告的 context window（如果可用），否则默认为 `8192`。你可以在显式 provider 配置中覆盖 `contextWindow` 和 `maxTokens`。

## 故障排除

### Ollama 未检测到

确保 Ollama 正在运行且你设置了 `OLLAMA_API_KEY`（或认证配置文件），并且你**没有**定义显式的 `models.providers.ollama` 条目:

```bash
ollama serve
```

并确保 API 可访问:

```bash
curl http://localhost:11434/api/tags
```

### 没有可用模型

OpenClaw 仅自动发现报告工具支持的模型。如果你的模型未列出，请:

- 拉取支持工具的模型，或
- 在 `models.providers.ollama` 中显式定义模型。

添加模型:

```bash
ollama list  # 查看已安装的内容
ollama pull llama3.3  # 拉取模型
```

### 连接被拒绝

检查 Ollama 是否在正确的端口上运行:

```bash
# 检查 Ollama 是否正在运行
ps aux | grep ollama

# 或重启 Ollama
ollama serve
```

## 另请参阅

- [Model Providers](/concepts/model-providers) - 所有 provider 概览
- [Model Selection](/concepts/models) - 如何选择模型
- [Configuration](/gateway/configuration) - 完整配置参考

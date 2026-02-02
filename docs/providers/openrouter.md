---
summary: "在 OpenClaw 中使用 OpenRouter 的统一 API 访问多种模型"
read_when:
  - 你想用一个 API 密钥访问多种 LLM
  - 你想在 OpenClaw 中通过 OpenRouter 运行模型
title: "OpenRouter"
---

# OpenRouter

OpenRouter 提供**统一 API**，将请求路由到单一端点和 API 密钥背后的多种模型。它兼容 OpenAI，因此大多数 OpenAI SDK 只需切换 base URL 即可工作。

## CLI 设置

```bash
openclaw onboard --auth-choice apiKey --token-provider openrouter --token "$OPENROUTER_API_KEY"
```

## 配置片段

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/anthropic/claude-sonnet-4-5" },
    },
  },
}
```

## 注意事项

- 模型引用格式为 `openrouter/<provider>/<model>`。
- 更多模型/provider 选项，参见 [/concepts/model-providers](/concepts/model-providers)。
- OpenRouter 底层使用带有你的 API 密钥的 Bearer token。

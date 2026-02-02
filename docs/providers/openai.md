---
summary: "在 OpenClaw 中通过 API 密钥或 Codex 订阅使用 OpenAI"
read_when:
  - 你想在 OpenClaw 中使用 OpenAI 模型
  - 你想使用 Codex 订阅认证而非 API 密钥
title: "OpenAI"
---

# OpenAI

OpenAI 为 GPT 模型提供开发者 API。Codex 支持 **ChatGPT 登录**用于订阅访问，或 **API 密钥**登录用于按量访问。Codex cloud 需要 ChatGPT 登录。

## 选项 A: OpenAI API 密钥 (OpenAI Platform)

**最适合：** 直接 API 访问和按量计费。
从 OpenAI 仪表板获取你的 API 密钥。

### CLI 设置

```bash
openclaw onboard --auth-choice openai-api-key
# 或非交互式
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### 配置片段

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.2" } } },
}
```

## 选项 B: OpenAI Code (Codex) 订阅

**最适合：** 使用 ChatGPT/Codex 订阅访问而非 API 密钥。
Codex cloud 需要 ChatGPT 登录，而 Codex CLI 支持 ChatGPT 或 API 密钥登录。

### CLI 设置

```bash
# 在向导中运行 Codex OAuth
openclaw onboard --auth-choice openai-codex

# 或直接运行 OAuth
openclaw models auth login --provider openai-codex
```

### 配置片段

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.2" } } },
}
```

## 注意事项

- 模型引用始终使用 `provider/model` 格式（参见 [/concepts/models](/concepts/models)）。
- 认证详情 + 重用规则参见 [/concepts/oauth](/concepts/oauth)。

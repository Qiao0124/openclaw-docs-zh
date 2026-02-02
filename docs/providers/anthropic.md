---
summary: "在 OpenClaw 中通过 API 密钥或 setup-token 使用 Anthropic Claude"
read_when:
  - 你想在 OpenClaw 中使用 Anthropic 模型
  - 你想使用 setup-token 而非 API 密钥
title: "Anthropic"
---

# Anthropic (Claude)

Anthropic 构建了 **Claude** 模型家族，并通过 API 提供访问。
在 OpenClaw 中，你可以使用 API 密钥或 **setup-token** 进行认证。

## 选项 A: Anthropic API 密钥

**最适合：** 标准 API 访问和按量计费。
在 Anthropic Console 中创建你的 API 密钥。

### CLI 设置

```bash
openclaw onboard
# 选择: Anthropic API key

# 或非交互式
openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
```

### 配置片段

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-5" } } },
}
```

## Prompt 缓存 (Anthropic API)

OpenClaw **不会**覆盖 Anthropic 的默认缓存 TTL，除非你进行设置。
这仅适用于 **API**；订阅认证不支持 TTL 设置。

要为每个模型设置 TTL，请在模型 `params` 中使用 `cacheControlTtl`：

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-5": {
          params: { cacheControlTtl: "5m" }, // 或 "1h"
        },
      },
    },
  },
}
```

OpenClaw 为 Anthropic API 请求包含 `extended-cache-ttl-2025-04-11` beta 标志；
如果你覆盖 provider headers，请保留它（参见 [/gateway/configuration](/gateway/configuration)）。

## 选项 B: Claude setup-token

**最适合：** 使用你的 Claude 订阅。

### 如何获取 setup-token

Setup-tokens 由 **Claude Code CLI** 创建，而非 Anthropic Console。你可以在**任何机器**上运行：

```bash
claude setup-token
```

将 token 粘贴到 OpenClaw 中（向导：**Anthropic token (paste setup-token)**），或在 gateway 主机上运行：

```bash
openclaw models auth setup-token --provider anthropic
```

如果你在不同的机器上生成了 token，请粘贴它：

```bash
openclaw models auth paste-token --provider anthropic
```

### CLI 设置

```bash
# 在 onboarding 过程中粘贴 setup-token
openclaw onboard --auth-choice setup-token
```

### 配置片段

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-5" } } },
}
```

## 注意事项

- 使用 `claude setup-token` 生成 setup-token 并粘贴，或在 gateway 主机上运行 `openclaw models auth setup-token`。
- 如果你在 Claude 订阅上看到 "OAuth token refresh failed ..."，请使用 setup-token 重新认证。参见 [/gateway/troubleshooting#oauth-token-refresh-failed-anthropic-claude-subscription](/gateway/troubleshooting#oauth-token-refresh-failed-anthropic-claude-subscription)。
- 认证详情 + 重用规则参见 [/concepts/oauth](/concepts/oauth)。

## 故障排除

**401 错误 / token 突然失效**

- Claude 订阅认证可能会过期或被撤销。重新运行 `claude setup-token` 并将其粘贴到 **gateway 主机**。
- 如果 Claude CLI 登录在不同的机器上，请在 gateway 主机上使用 `openclaw models auth paste-token --provider anthropic`。

**No API key found for provider "anthropic"**

- 认证是**每个 agent**独立的。新 agent 不会继承主 agent 的密钥。
- 为该 agent 重新运行 onboarding，或在 gateway 主机上粘贴 setup-token / API 密钥，然后使用 `openclaw models status` 验证。

**No credentials found for profile `anthropic:default`**

- 运行 `openclaw models status` 查看哪个认证配置文件处于活动状态。
- 重新运行 onboarding，或为该配置文件粘贴 setup-token / API 密钥。

**No available auth profile (all in cooldown/unavailable)**

- 检查 `openclaw models status --json` 中的 `auth.unusableProfiles`。
- 添加另一个 Anthropic 配置文件或等待冷却期结束。

更多内容：[/gateway/troubleshooting](/gateway/troubleshooting) 和 [/help/faq](/help/faq)。

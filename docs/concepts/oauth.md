---
summary: "OpenClaw 中的 OAuth：令牌交换、存储和多账户模式"
read_when:
  - 您想了解端到端的 OpenClaw OAuth
  - 您遇到令牌失效/注销问题
  - 您想要 setup-token 或 OAuth 认证流程
  - 您想要多个账户或配置文件路由
title: "OAuth"
---

# OAuth

OpenClaw 通过 OAuth 支持提供该功能的提供商的"订阅认证"（特别是 **OpenAI Codex (ChatGPT OAuth)**）。对于 Anthropic 订阅，请使用 **setup-token** 流程。本页解释：

- OAuth **令牌交换**的工作原理（PKCE）
- 令牌的**存储**位置（以及原因）
- 如何处理**多个账户**（配置文件 + 每会话覆盖）

OpenClaw 还支持自带 OAuth 或 API 密钥流程的**提供商插件**。通过以下方式运行：

```bash
openclaw models auth login --provider <id>
```

## 令牌池（存在的原因）

OAuth 提供商通常在登录/刷新流程中生成**新的刷新令牌**。某些提供商（或 OAuth 客户端）可以在为同一用户/应用颁发新令牌时使旧的刷新令牌失效。

实际症状：

- 您通过 OpenClaw _和_ Claude Code / Codex CLI 登录 → 其中一个稍后随机"注销"

为了减少这种情况，OpenClaw 将 `auth-profiles.json` 视为**令牌池**：

- 运行时从**一处**读取凭据
- 我们可以保留多个配置文件并确定性路由

## 存储（令牌存储位置）

机密按**代理**存储：

- 认证配置文件（OAuth + API 密钥）：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- 运行时缓存（自动管理；请勿编辑）：`~/.openclaw/agents/<agentId>/agent/auth.json`

传统仅导入文件（仍支持，但不是主要存储）：

- `~/.openclaw/credentials/oauth.json`（首次使用时导入到 `auth-profiles.json`）

以上所有也尊重 `$OPENCLAW_STATE_DIR`（状态目录覆盖）。完整参考：[/gateway/configuration](/gateway/configuration#auth-storage-oauth--api-keys)

## Anthropic setup-token（订阅认证）

在任何机器上运行 `claude setup-token`，然后粘贴到 OpenClaw：

```bash
openclaw models auth setup-token --provider anthropic
```

如果您在其他地方生成了令牌，请手动粘贴：

```bash
openclaw models auth paste-token --provider anthropic
```

验证：

```bash
openclaw models status
```

## OAuth 交换（登录工作原理）

OpenClaw 的交互式登录流程在 `@mariozechner/pi-ai` 中实现，并连接到向导/命令。

### Anthropic（Claude Pro/Max）setup-token

流程形状：

1. 运行 `claude setup-token`
2. 将令牌粘贴到 OpenClaw
3. 存储为令牌认证配置文件（无刷新）

向导路径是 `openclaw onboard` → 认证选择 `setup-token`（Anthropic）。

### OpenAI Codex（ChatGPT OAuth）

流程形状（PKCE）：

1. 生成 PKCE 验证器/质询 + 随机 `state`
2. 打开 `https://auth.openai.com/oauth/authorize?...`
3. 尝试在 `http://127.0.0.1:1455/auth/callback` 捕获回调
4. 如果回调无法绑定（或您处于远程/无头环境），粘贴重定向 URL/代码
5. 在 `https://auth.openai.com/oauth/token` 交换
6. 从访问令牌中提取 `accountId` 并存储 `{ access, refresh, expires, accountId }`

向导路径是 `openclaw onboard` → 认证选择 `openai-codex`。

## 刷新 + 过期

配置文件存储 `expires` 时间戳。

运行时：

- 如果 `expires` 在未来 → 使用存储的访问令牌
- 如果已过期 → 在文件锁下刷新并覆盖存储的凭据

刷新流程是自动的；您通常不需要手动管理令牌。

## 多个账户（配置文件）+ 路由

两种模式：

### 1）首选：分离代理

如果您希望"个人"和"工作"永不交互，请使用隔离代理（分离的会话 + 凭据 + 工作空间）：

```bash
openclaw agents add work
openclaw agents add personal
```

然后为每个代理配置认证（向导）并将聊天路由到正确的代理。

### 2）高级：一个代理中的多个配置文件

`auth-profiles.json` 支持同一提供商的多个配置文件 ID。

选择使用哪个配置文件：

- 通过配置排序全局（`auth.order`）
- 通过每会话 `/model ...@<profileId>`

示例（会话覆盖）：

- `/model Opus@anthropic:work`

如何查看存在的配置文件 ID：

- `openclaw channels list --json`（显示 `auth[]`）

相关文档：

- [/concepts/model-failover](/concepts/model-failover)（轮换 + 冷却规则）
- [/tools/slash-commands](/tools/slash-commands)（命令界面）

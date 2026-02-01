---
summary: "OpenClaw 首次运行引导流程（macOS 应用）"
read_when:
  - 设计 macOS 引导助手
  - 实现认证或身份设置
title: "引导（Onboarding）"
---

# 引导（Onboarding, macOS app）

本文描述**当前**首次运行的引导流程。目标是提供顺滑的“第 0 天”体验：选择网关运行位置、连接认证、运行向导，并让代理自举。

## 页面顺序（当前 / Page order）

1. 欢迎 + 安全提示
2. **网关选择**（本地 / 远程 / 稍后配置）
3. **认证（Anthropic OAuth）** — 仅本地
4. **设置向导**（由网关驱动）
5. **权限**（TCC 提示）
6. **CLI**（可选）
7. **引导聊天**（独立会话）
8. 就绪

## 1) 本地 vs 远程（Local vs Remote）

**网关**运行在哪里？

- **本地（此 Mac）：** 引导可在本地执行 OAuth 并写入凭据。
- **远程（SSH/Tailnet）：** 引导**不会**在本地执行 OAuth；凭据必须存在于网关主机。
- **稍后配置：** 跳过设置，应用保持未配置状态。

网关认证提示：

- 向导现在即使在回环地址也会生成 **token**，因此本地 WS 客户端必须认证。
- 若禁用认证，任何本地进程都可连接；仅在完全可信的机器上使用。
- 多机访问或非回环绑定请使用 **token**。

## 2) 仅本地认证（Anthropic OAuth）

macOS 应用支持 Anthropic OAuth（Claude Pro/Max）。流程如下：

- 打开浏览器进行 OAuth（PKCE）
- 提示用户粘贴 `code#state`
- 将凭据写入 `~/.openclaw/credentials/oauth.json`

其他提供方（OpenAI、自定义 API）目前通过环境变量或配置文件来设置。

## 3) 设置向导（由网关驱动 / Setup Wizard）

应用可以运行与 CLI 相同的设置向导。这样引导流程与网关侧行为保持一致，并避免在 SwiftUI 中重复实现逻辑。

## 4) 权限（Permissions）

引导会请求以下 TCC 权限：

- 通知
- 辅助功能
- 屏幕录制
- 麦克风 / 语音识别
- 自动化（AppleScript）

## 5) CLI（可选 / CLI）

应用可通过 npm/pnpm 安装全局 `openclaw` CLI，使终端工作流与 launchd 任务开箱即用。

## 6) 引导聊天（独立会话 / Onboarding chat）

完成设置后，应用会打开一个专用的引导聊天会话，让代理自我介绍并引导后续步骤。这样首轮引导与正常对话分离。

## 代理自举仪式（Agent bootstrap ritual）

首次运行时，OpenClaw 会自举工作区（默认 `~/.openclaw/workspace`）：

- 写入 `AGENTS.md`、`BOOTSTRAP.md`、`IDENTITY.md`、`USER.md`
- 执行简短问答仪式（一次一个问题）
- 将身份与偏好写入 `IDENTITY.md`、`USER.md`、`SOUL.md`
- 完成后移除 `BOOTSTRAP.md`，确保只运行一次

## 可选：Gmail hooks（手动 / Optional）

Gmail Pub/Sub 当前仍需手动配置。使用：

```bash
openclaw webhooks gmail setup --account you@gmail.com
```

详情见 [/automation/gmail-pubsub](/automation/gmail-pubsub)。

## 远程模式说明（Remote mode notes）

当网关运行在另一台机器上时，凭据与工作区文件都**在该主机上**。如果你需要在远程模式下使用 OAuth，请在网关主机上创建：

- `~/.openclaw/credentials/oauth.json`
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`

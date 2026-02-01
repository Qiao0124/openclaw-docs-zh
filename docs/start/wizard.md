---
summary: "CLI 引导向导：引导配置网关、工作区、渠道与技能"
read_when:
  - 运行或配置引导向导
  - 设置一台新机器
title: "引导向导（Onboarding Wizard）"
---

# 引导向导（Onboarding Wizard, CLI）

引导向导是在 macOS、Linux 或 Windows（通过 WSL2；强烈推荐）上设置 OpenClaw 的**推荐**方式。
它会在一个引导流程中配置本地网关或远程网关连接，以及渠道、技能和工作区默认值。

主要入口：

```bash
openclaw onboard
```

最快的第一条聊天：打开 Control UI（无需配置渠道）。运行
`openclaw dashboard` 并在浏览器中聊天。文档：[Dashboard](/web/dashboard)。

后续重新配置：

```bash
openclaw configure
```

推荐：设置 Brave Search API key 以便代理使用 `web_search`
（`web_fetch` 无需 key）。最简单路径：`openclaw configure --section web`
（保存到 `tools.web.search.apiKey`）。文档：[Web tools](/tools/web)。

## 快速开始 vs 高级（QuickStart vs Advanced）

向导以 **QuickStart**（默认值）与 **Advanced**（完全控制）开始。

**QuickStart** 会保留默认值：

- 本地网关（回环地址）
- 工作区默认值（或已有工作区）
- 网关端口 **18789**
- 网关认证 **Token**（自动生成，即使是回环）
- Tailscale 暴露 **关闭**
- Telegram + WhatsApp 私信默认 **允许列表**（会提示输入手机号）

**Advanced** 会展示所有步骤（模式、工作区、网关、渠道、守护进程、技能）。

## 向导会做什么（What the wizard does）

**本地模式（默认）** 会引导你完成：

- 模型/认证（OpenAI Code（Codex）订阅 OAuth、Anthropic API key（推荐）或 setup-token（粘贴），以及 MiniMax/GLM/Moonshot/AI Gateway 选项）
- 工作区位置 + 引导文件
- 网关设置（端口/绑定/认证/Tailscale）
- 提供方（Telegram、WhatsApp、Discord、Google Chat、Mattermost（插件）、Signal）
- 守护进程安装（LaunchAgent / systemd 用户单元）
- 健康检查
- 技能（推荐）

**远程模式** 仅配置本地客户端去连接远端网关。
它**不会**在远端主机上安装或修改任何内容。

要添加更多隔离代理（独立工作区 + 会话 + 认证），使用：

```bash
openclaw agents add <name>
```

提示：`--json` **不**表示非交互模式。脚本中请使用 `--non-interactive`（以及 `--workspace`）。

## 流程细节（本地 / local）

1. **检测已有配置（Existing config detection）**
   - 如果存在 `~/.openclaw/openclaw.json`，可选择 **Keep / Modify / Reset**。
   - 重新运行向导**不会**清空任何内容，除非你显式选择 **Reset**（或传入 `--reset`）。
   - 如果配置无效或包含旧版键，向导会停止并提示先运行 `openclaw doctor`。
   - Reset 使用 `trash`（绝不使用 `rm`），并提供范围：
     - 仅配置
     - 配置 + 凭据 + 会话
     - 完全重置（同时移除工作区）

2. **模型/认证（Model/Auth）**
   - **Anthropic API key（推荐）**：如果存在 `ANTHROPIC_API_KEY` 则使用，否则提示输入并为守护进程保存。
   - **Anthropic OAuth（Claude Code CLI）**：macOS 会检查钥匙串项 “Claude Code-credentials”（建议选择 “Always Allow” 以免 launchd 启动阻塞）；Linux/Windows 若存在 `~/.claude/.credentials.json` 则复用。
   - **Anthropic token（粘贴 setup-token）**：在任意机器运行 `claude setup-token`，再粘贴 token（可命名；留空为默认）。
   - **OpenAI Code（Codex）订阅（Codex CLI）**：若存在 `~/.codex/auth.json`，向导可复用。
   - **OpenAI Code（Codex）订阅（OAuth）**：浏览器流程；粘贴 `code#state`。
     - 当模型未设置或为 `openai/*` 时，将 `agents.defaults.model` 设为 `openai-codex/gpt-5.2`。
   - **OpenAI API key**：若存在 `OPENAI_API_KEY` 则使用，否则提示输入并保存到 `~/.openclaw/.env`，以便 launchd 读取。
   - **OpenCode Zen（多模型代理）**：提示输入 `OPENCODE_API_KEY`（或 `OPENCODE_ZEN_API_KEY`，获取地址 https://opencode.ai/auth）。
   - **API key**：为你保存 key。
   - **Vercel AI Gateway（多模型代理）**：提示输入 `AI_GATEWAY_API_KEY`。
   - 详情：[Vercel AI Gateway](/providers/vercel-ai-gateway)
   - **MiniMax M2.1**：自动写入配置。
   - 详情：[MiniMax](/providers/minimax)
   - **Synthetic（Anthropic 兼容）**：提示输入 `SYNTHETIC_API_KEY`。
   - 详情：[Synthetic](/providers/synthetic)
   - **Moonshot（Kimi K2）**：自动写入配置。
   - **Kimi Coding**：自动写入配置。
   - 详情：[Moonshot AI（Kimi + Kimi Coding）](/providers/moonshot)
   - **跳过（Skip）**：暂不配置认证。
   - 从检测到的选项中选择默认模型（或手动输入提供方/模型）。
   - 向导会运行模型检查，并在模型未知或缺少认证时提示警告。

- OAuth 凭据存放在 `~/.openclaw/credentials/oauth.json`；认证配置存放在 `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（API key + OAuth）。
- 详情：[/concepts/oauth](/concepts/oauth)

3. **工作区（Workspace）**
   - 默认 `~/.openclaw/workspace`（可配置）。
   - 写入代理引导所需的工作区文件。
   - 完整布局与备份指南：[Agent workspace](/concepts/agent-workspace)

4. **网关（Gateway）**
   - 端口、绑定、认证模式、Tailscale 暴露。
   - 认证建议：即使是回环也保留 **Token**，这样本地 WS 客户端必须认证。
   - 仅在你完全信任所有本地进程时才禁用认证。
   - 非回环绑定仍要求认证。

5. **渠道（Channels）**
   - [WhatsApp](/channels/whatsapp)：可选二维码登录。
   - [Telegram](/channels/telegram)：机器人令牌。
   - [Discord](/channels/discord)：机器人令牌。
   - [Google Chat](/channels/googlechat)：服务账号 JSON + webhook audience。
   - [Mattermost](/channels/mattermost)（插件）：机器人令牌 + base URL。
   - [Signal](/channels/signal)：可选安装 `signal-cli` + 账号配置。
   - [iMessage](/channels/imessage)：本地 `imsg` CLI 路径 + DB 访问。
   - DM 安全：默认是配对。首条 DM 会发送短码；用 `openclaw pairing approve <channel> <code>` 批准或使用允许列表。

6. **守护进程安装（Daemon install）**
   - macOS：LaunchAgent
     - 需要已登录用户会话；无头场景使用自定义 LaunchDaemon（未内置）。
   - Linux（以及通过 WSL2 的 Windows）：systemd 用户单元
     - 向导会尝试 `loginctl enable-linger <user>` 以便网关在注销后仍保持运行。
     - 可能会提示 sudo（写入 `/var/lib/systemd/linger`）；会先尝试无 sudo。
   - **运行时选择（Runtime selection）：** Node（推荐；WhatsApp/Telegram 必需）。不推荐 Bun。

7. **健康检查（Health check）**
   - 启动网关（如需）并运行 `openclaw health`。
   - 提示：`openclaw status --deep` 会在状态输出中加入网关健康探测（需要可达网关）。

8. **技能（推荐）（Skills）**
   - 读取可用技能并检查依赖。
   - 选择节点管理器：**npm / pnpm**（不推荐 bun）。
   - 安装可选依赖（部分在 macOS 使用 Homebrew）。

9. **完成（Finish）**
   - 汇总 + 后续步骤（包含 iOS/Android/macOS 应用以获取更多功能）。

- 若未检测到 GUI，向导会打印 Control UI 的 SSH 端口转发说明，而不是打开浏览器。
- 若 Control UI 资源缺失，向导会尝试构建；回退命令为 `pnpm ui:build`（首次运行会自动安装 UI 依赖）。

## 远程模式（Remote mode）

远程模式会配置本地客户端去连接远端网关。

你将设置：

- 远程网关 URL（`ws://...`）
- 若远端网关需要认证，则设置 Token（推荐）

备注：

- 不会执行远端安装或守护进程更改。
- 若网关仅监听回环地址，请使用 SSH 隧道或 tailnet。
- 发现提示：
  - macOS：Bonjour（`dns-sd`）
  - Linux：Avahi（`avahi-browse`）

## 添加另一个代理（Add another agent）

使用 `openclaw agents add <name>` 创建独立代理，拥有各自的工作区、会话与认证配置。未传 `--workspace` 时会启动向导。

它会设置：

- `agents.list[].name`
- `agents.list[].workspace`
- `agents.list[].agentDir`

备注：

- 默认工作区为 `~/.openclaw/workspace-<agentId>`。
- 添加 `bindings` 以路由入站消息（向导可完成）。
- 非交互参数：`--model`、`--agent-dir`、`--bind`、`--non-interactive`。

## 非交互模式（Non-interactive mode）

使用 `--non-interactive` 进行自动化或脚本化引导：

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

添加 `--json` 可输出机器可读的摘要。

Gemini 示例（Gemini example）：

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

Z.AI 示例（Z.AI example）：

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice zai-api-key \
  --zai-api-key "$ZAI_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

Vercel AI Gateway 示例（Vercel AI Gateway example）：

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice ai-gateway-api-key \
  --ai-gateway-api-key "$AI_GATEWAY_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

Moonshot 示例（Moonshot example）：

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice moonshot-api-key \
  --moonshot-api-key "$MOONSHOT_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

Synthetic 示例（Synthetic example）：

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice synthetic-api-key \
  --synthetic-api-key "$SYNTHETIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

OpenCode Zen 示例（OpenCode Zen example）：

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice opencode-zen \
  --opencode-zen-api-key "$OPENCODE_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

添加代理（非交互）示例（Add agent, non-interactive）：

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.2 \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

## 网关向导 RPC（Gateway wizard RPC）

网关通过 RPC 暴露向导流程（`wizard.start`、`wizard.next`、`wizard.cancel`、`wizard.status`）。
客户端（macOS 应用、Control UI）可渲染步骤而无需重新实现引导逻辑。

## Signal 配置（signal-cli）

向导可以从 GitHub Releases 安装 `signal-cli`：

- 下载对应的发布资产。
- 存放在 `~/.openclaw/tools/signal-cli/<version>/`。
- 将 `channels.signal.cliPath` 写入配置。

备注：

- JVM 构建要求 **Java 21**。
- 可用时优先使用 Native 构建。
- Windows 使用 WSL2；signal-cli 安装会在 WSL 内按照 Linux 流程进行。

## 向导会写入哪些内容（What the wizard writes）

`~/.openclaw/openclaw.json` 的典型字段：

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers`（若选择 Minimax）
- `gateway.*`（模式、绑定、认证、Tailscale）
- `channels.telegram.botToken`、`channels.discord.token`、`channels.signal.*`、`channels.imessage.*`
- 当你在提示中选择启用时，渠道允许列表（Slack/Discord/Matrix/Microsoft Teams）（名称会尽量解析为 ID）。
- `skills.install.nodeManager`
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

`openclaw agents add` 会写入 `agents.list[]` 以及可选的 `bindings`。

WhatsApp 凭据存放在 `~/.openclaw/credentials/whatsapp/<accountId>/`。
会话存放在 `~/.openclaw/agents/<agentId>/sessions/`。

部分渠道以插件形式提供。在引导中选择后，向导会提示安装它（npm 或本地路径），随后才能配置。

## 相关文档（Related docs）

- macOS 应用引导：[Onboarding](/start/onboarding)
- 配置参考：[Gateway configuration](/gateway/configuration)
- 提供方：[WhatsApp](/channels/whatsapp)、[Telegram](/channels/telegram)、[Discord](/channels/discord)、[Google Chat](/channels/googlechat)、[Signal](/channels/signal)、[iMessage](/channels/imessage)
- 技能：[Skills](/tools/skills)、[Skills config](/tools/skills-config)

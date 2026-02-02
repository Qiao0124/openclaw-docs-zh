---
summary: "OpenClaw 设置、配置和使用的常见问题解答"
title: "FAQ"
---

# 常见问题解答 (FAQ)

针对实际场景（本地开发、VPS、多 Agent、OAuth/API 密钥、模型故障转移）的快速解答和深入故障排除。运行时诊断请参见[故障排除](/gateway/troubleshooting)。完整配置参考请参见[配置](/gateway/configuration)。

## 目录

- [快速开始和首次运行设置](#快速开始和首次运行设置)
- [什么是 OpenClaw？](#什么是-openclaw)
- [技能和自动化](#技能和自动化)
- [沙盒和内存](#沙盒和内存)
- [磁盘上的文件位置](#磁盘上的文件位置)
- [配置基础](#配置基础)
- [远程网关和节点](#远程网关和节点)
- [环境变量和 .env 加载](#环境变量和-env-加载)
- [会话和多聊天](#会话和多聊天)
- [模型：默认值、选择、别名、切换](#模型默认值选择别名切换)
- [模型故障转移和"所有模型都失败"](#模型故障转移和所有模型都失败)
- [认证配置文件](#认证配置文件)
- [网关：端口、"已在运行"和远程模式](#网关端口已在运行和远程模式)
- [日志和调试](#日志和调试)
- [媒体和附件](#媒体和附件)
- [安全和访问控制](#安全和访问控制)
- [聊天命令、中止任务和"它不会停止"](#聊天命令中止任务和它不会停止)

---

## 快速开始和首次运行设置

### 我卡住了，最快的解决方法是什么？

使用一个可以**看到你的机器**的本地 AI Agent。这比在 Discord 中询问要有效得多，因为大多数"我卡住了"的情况都是**本地配置或环境问题**，远程助手无法检查。

- **Claude Code**: https://www.anthropic.com/claude-code/
- **OpenAI Codex**: https://openai.com/codex/

这些工具可以读取仓库、运行命令、检查日志，并帮助修复你的机器级设置（PATH、服务、权限、认证文件）。通过可修改的（git）安装方式给它们**完整的源码检出**：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
```

这会从 git 检出安装 OpenClaw，因此 Agent 可以读取代码 + 文档并推理你正在运行的确切版本。你可以随时通过重新运行不带 `--install-method git` 的安装程序切换回稳定版。

提示：要求 Agent **规划和监督**修复（分步），然后只执行必要的命令。这样更改更小，更容易审计。

如果你发现真正的错误或修复，请提交 GitHub issue 或发送 PR：
https://github.com/openclaw/openclaw/issues
https://github.com/openclaw/openclaw/pulls

从以下命令开始（寻求帮助时分享输出）：

```bash
openclaw status
openclaw models status
openclaw doctor
```

它们的作用：

- `openclaw status`：网关/Agent 健康 + 基本配置的快速快照
- `openclaw models status`：检查提供商认证 + 模型可用性
- `openclaw doctor`：验证和修复常见配置/状态问题

其他有用的 CLI 检查：`openclaw status --all`、`openclaw logs --follow`、
`openclaw gateway status`、`openclaw health --verbose`。

快速调试循环：[出问题时前 60 秒](#出问题时前-60-秒)。
安装文档：[安装](/install)、[安装程序标志](/install/installer)、[更新](/install/updating)。

### 安装和设置 OpenClaw 的推荐方法是什么？

仓库推荐从源码运行并使用引导向导：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw onboard --install-daemon
```

向导还可以自动构建 UI 资源。引导完成后，你通常在端口 **18789** 上运行网关。

从源码安装（贡献者/开发者）：

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm build
pnpm ui:build # 首次运行时自动安装 UI 依赖
openclaw onboard
```

如果你还没有全局安装，请通过 `pnpm openclaw onboard` 运行。

### 引导后如何打开仪表板？

向导现在会在引导后立即用你的浏览器打开带令牌的仪表板 URL，并在摘要中打印完整链接（带令牌）。保持该标签页打开；如果没有启动，请在同一台机器上复制/粘贴打印的 URL。令牌保留在你的主机本地——不会从浏览器获取任何内容。

### 如何在 localhost 与远程上认证仪表板（令牌）？

**本地主机（同一台机器）：**

- 打开 `http://127.0.0.1:18789/`。
- 如果要求认证，运行 `openclaw dashboard` 并使用带令牌的链接（`?token=...`）。
- 令牌与 `gateway.auth.token`（或 `OPENCLAW_GATEWAY_TOKEN`）的值相同，UI 在首次加载后存储它。

**不在本地主机上：**

- **Tailscale Serve**（推荐）：保持绑定环回，运行 `openclaw gateway --tailscale serve`，打开 `https://<magicdns>/`。如果 `gateway.auth.allowTailscale` 为 `true`，身份标头满足认证（无需令牌）。
- **Tailnet 绑定**：运行 `openclaw gateway --bind tailnet --token "<token>"`，打开 `http://<tailscale-ip>:18789/`，在仪表板设置中粘贴令牌。
- **SSH 隧道**：`ssh -N -L 18789:127.0.0.1:18789 user@host` 然后从 `openclaw dashboard` 打开 `http://127.0.0.1:18789/?token=...`。

参见[仪表板](/web/dashboard)和[Web 界面](/web)了解绑定模式和认证详情。

### 我需要什么运行时？

需要 Node **>= 22**。推荐 `pnpm`。Bun **不推荐**用于网关。

### 它能在树莓派上运行吗？

可以。网关是轻量级的——文档列出 **512MB-1GB RAM**、**1 核**和约 **500MB** 磁盘足以供个人使用，并指出**树莓派 4 可以运行它**。

如果你想要额外的余量（日志、媒体、其他服务），**推荐 2GB**，但这不是硬性最低要求。

提示：小型 Pi/VPS 可以托管网关，你可以在笔记本电脑/手机上配对**节点**以进行本地屏幕/摄像头/画布或命令执行。参见[节点](/nodes)。

### 树莓派安装有什么技巧吗？

简而言之：它可以工作，但预期会有一些问题。

- 使用 **64 位**操作系统并保持 Node >= 22。
- 首选**可修改的（git）安装**以便查看日志和快速更新。
- 从没有频道/技能开始，然后逐个添加。
- 如果遇到奇怪的二进制问题，通常是 **ARM 兼容性**问题。

文档：[Linux](/platforms/linux)、[安装](/install)。

### 它卡在"wake up my friend" / 引导无法孵化。现在怎么办？

该屏幕取决于网关是否可达和已认证。TUI 也会在首次孵化时自动发送"Wake up, my friend!"。如果你看到该行**没有回复**且令牌保持为 0，Agent 从未运行。

1. 重启网关：

```bash
openclaw gateway restart
```

2. 检查状态 + 认证：

```bash
openclaw status
openclaw models status
openclaw logs --follow
```

3. 如果仍然卡住，运行：

```bash
openclaw doctor
```

如果网关是远程的，确保隧道/Tailscale 连接已启动且 UI 指向正确的网关。参见[远程访问](/gateway/remote)。

### 我可以在不重做引导的情况下将我的设置迁移到新机器（Mac mini）吗？

可以。复制**状态目录**和**工作区**，然后运行一次 Doctor。这会让你的机器人保持"完全相同"（内存、会话历史、认证和频道状态），只要你复制**两个**位置：

1. 在新机器上安装 OpenClaw。
2. 从旧机器复制 `$OPENCLAW_STATE_DIR`（默认：`~/.openclaw`）。
3. 复制你的工作区（默认：`~/.openclaw/workspace`）。
4. 运行 `openclaw doctor` 并重启网关服务。

这会保留配置、认证配置文件、WhatsApp 凭证、会话和内存。如果你在远程模式下，记住网关主机拥有会话存储和工作区。

**重要：**如果你只将工作区提交/推送到 GitHub，你备份的是**内存 + 引导文件**，但**不是**会话历史或认证。这些位于 `~/.openclaw/` 下（例如 `~/.openclaw/agents/<agentId>/sessions/`）。

相关：[迁移](/install/migrating)、[磁盘上的文件位置](#openclaw-在哪里存储其数据)、
[Agent 工作区](/concepts/agent-workspace)、[Doctor](/gateway/doctor)、
[远程模式](/gateway/remote)。

### 我在哪里可以看到最新版本的新内容？

查看 GitHub 更新日志：
https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md

最新条目在顶部。如果顶部部分标记为**未发布**，下一个日期部分是最新发布的版本。条目按**亮点**、**更改**和**修复**分组（加上文档/其他部分，需要时）。

### 我无法访问 docs.openclaw.ai（SSL 错误）。现在怎么办？

一些 Comcast/Xfinity 连接通过 Xfinity 高级安全错误地阻止 `docs.openclaw.ai`。禁用它或将 `docs.openclaw.ai` 加入允许列表，然后重试。更多详情：[故障排除](/help/troubleshooting#docsopenclawai-显示-ssl-错误-comcastxfinity)。
请通过此处帮助我们解除阻止：https://spa.xfinity.com/check_url_status。

如果你仍然无法访问该网站，文档在 GitHub 上有镜像：
https://github.com/openclaw/openclaw/tree/main/docs

### 稳定版和测试版有什么区别？

**稳定版**和**测试版**是 **npm 分发标签**，不是单独的代码线：

- `latest` = 稳定版
- `beta` = 早期测试版本

我们将构建版本发布到 **beta**，测试它们，一旦构建稳定，我们就**将相同版本提升到 `latest`**。这就是为什么测试版和稳定版可以指向**相同版本**。

查看更改内容：
https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md

### 如何安装测试版，测试版和开发版有什么区别？

**测试版**是 npm 分发标签 `beta`（可能与 `latest` 匹配）。
**开发版**是 `main` 的移动头部（git）；发布时，它使用 npm 分发标签 `dev`。

单行命令（macOS/Linux）：

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
```

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
```

Windows 安装程序（PowerShell）：
https://openclaw.ai/install.ps1

更多详情：[开发频道](/install/development-channels)和[安装程序标志](/install/installer)。

### 安装和引导通常需要多长时间？

大致指南：

- **安装：** 2-5 分钟
- **引导：** 5-15 分钟，取决于你配置多少个频道/模型

如果卡住，使用[安装程序卡住](#安装程序卡住如何获得更多反馈)和[我卡住了](#我卡住了最快的解决方法是什么)中的快速调试循环。

### 安装程序卡住？如何获得更多反馈？

使用**详细输出**重新运行安装程序：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
```

测试版详细安装：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
```

对于可修改的（git）安装：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
```

更多选项：[安装程序标志](/install/installer)。

### Windows 安装显示找不到 git 或无法识别 openclaw

两个常见的 Windows 问题：

**1) npm 错误 spawn git / 找不到 git**

- 安装**Git for Windows**并确保 `git` 在你的 PATH 上。
- 关闭并重新打开 PowerShell，然后重新运行安装程序。

**2) 安装后无法识别 openclaw**

- 你的 npm 全局 bin 文件夹不在 PATH 上。
- 检查路径：
  ```powershell
  npm config get prefix
  ```
- 确保 `<prefix>\\bin` 在 PATH 上（在大多数系统上是 `%AppData%\\npm`）。
- 更新 PATH 后关闭并重新打开 PowerShell。

如果你想要最顺畅的 Windows 设置，请使用 **WSL2** 而不是原生 Windows。
文档：[Windows](/platforms/windows)。

### 文档没有回答我的问题——如何获得更好的答案？

使用**可修改的（git）安装**，以便你在本地拥有完整的源代码和文档，然后向你的机器人（或 Claude/Codex）询问*从该文件夹*，以便它可以读取仓库并精确回答。

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
```

更多详情：[安装](/install)和[安装程序标志](/install/installer)。

### 如何在 Linux 上安装 OpenClaw？

简而言之：按照 Linux 指南操作，然后运行引导向导。

- Linux 快速路径 + 服务安装：[Linux](/platforms/linux)。
- 完整演练：[入门](/start/getting-started)。
- 安装程序 + 更新：[安装和更新](/install/updating)。

### 如何在 VPS 上安装 OpenClaw？

任何 Linux VPS 都可以。在服务器上安装，然后使用 SSH/Tailscale 访问网关。

指南：[exe.dev](/platforms/exe-dev)、[Hetzner](/platforms/hetzner)、[Fly.io](/platforms/fly)。
远程访问：[网关远程](/gateway/remote)。

### 云/VPS 安装指南在哪里？

我们维护一个**托管中心**，包含常见提供商。选择一个并按照指南操作：

- [VPS 托管](/vps)（所有提供商集中在一处）
- [Fly.io](/platforms/fly)
- [Hetzner](/platforms/hetzner)
- [exe.dev](/platforms/exe-dev)

云端工作原理：**网关在服务器上运行**，你通过控制 UI（或 Tailscale/SSH）从笔记本电脑/手机访问它。你的状态 + 工作区在服务器上，因此将主机视为真相来源并备份它。

你可以将**节点**（Mac/iOS/Android/无头）配对到该云网关，以访问本地屏幕/摄像头/画布或在笔记本电脑上运行命令，同时将网关保留在云端。

中心：[平台](/platforms)。远程访问：[网关远程](/gateway/remote)。
节点：[节点](/nodes)、[节点 CLI](/cli/nodes)。

### 我可以要求 OpenClaw 自我更新吗？

简而言之：**可能，不推荐**。更新流程可以重启网关（这会断开活动会话），可能需要干净的 git 检出，并可能提示确认。更安全：以操作员身份从 shell 运行更新。

使用 CLI：

```bash
openclaw update
openclaw update status
openclaw update --channel stable|beta|dev
openclaw update --tag <dist-tag|version>
openclaw update --no-restart
```

如果必须从 Agent 自动化：

```bash
openclaw update --yes --no-restart
openclaw gateway restart
```

文档：[更新](/cli/update)、[更新](/install/updating)。

### 引导向导实际上做什么？

`openclaw onboard` 是推荐的设置路径。在**本地模式**下，它会引导你完成：

- **模型/认证设置**（推荐 Claude 订阅使用 Anthropic **setup-token**，支持 OpenAI Codex OAuth，可选 API 密钥，支持 LM Studio 本地模型）
- **工作区**位置 + 引导文件
- **网关设置**（绑定/端口/认证/tailscale）
- **提供商**（WhatsApp、Telegram、Discord、Mattermost（插件）、Signal、iMessage）
- **守护程序安装**（macOS 上的 LaunchAgent；Linux/WSL2 上的 systemd 用户单元）
- **健康检查**和**技能**选择

如果配置的模型未知或缺少认证，它还会发出警告。

### 我需要 Claude 或 OpenAI 订阅才能运行这个吗？

不需要。你可以使用 **API 密钥**（Anthropic/OpenAI/其他）或**仅限本地模型**运行 OpenClaw，以便你的数据保留在你的设备上。订阅（Claude Pro/Max 或 OpenAI Codex）是认证这些提供商的可选方式。

文档：[Anthropic](/providers/anthropic)、[OpenAI](/providers/openai)、
[本地模型](/gateway/local-models)、[模型](/concepts/models)。

### 我可以在没有 API 密钥的情况下使用 Claude Max 订阅吗？

可以。你可以使用 **setup-token** 而不是 API 密钥进行认证。这是订阅路径。

Claude Pro/Max 订阅**不包含 API 密钥**，因此这是订阅帐户的正确方法。重要：你必须向 Anthropic 确认此使用在其订阅政策和条款下是允许的。如果你想要最明确、受支持的路径，请使用 Anthropic API 密钥。

### Anthropic setup-token 认证如何工作？

`claude setup-token` 通过 Claude Code CLI 生成**令牌字符串**（在 Web 控制台中不可用）。你可以在**任何机器**上运行它。在向导中选择 **Anthropic 令牌（粘贴 setup-token）**或使用 `openclaw models auth paste-token --provider anthropic` 粘贴它。令牌存储为 **anthropic** 提供商的认证配置文件，并像 API 密钥一样使用（无自动刷新）。更多详情：[OAuth](/concepts/oauth)。

### 我在哪里可以找到 Anthropic setup-token？

它**不在** Anthropic 控制台中。setup-token 由**任何机器**上的 **Claude Code CLI** 生成：

```bash
claude setup-token
```

复制它打印的令牌，然后在向导中选择 **Anthropic 令牌（粘贴 setup-token）**。如果你想在网关主机上运行它，请使用 `openclaw models auth setup-token --provider anthropic`。如果你在别处运行了 `claude setup-token`，请在网关主机上使用 `openclaw models auth paste-token --provider anthropic` 粘贴它。参见 [Anthropic](/providers/anthropic)。

### 你们支持 Claude 订阅认证（Claude Pro/Max）吗？

支持——通过 **setup-token**。OpenClaw 不再重用 Claude Code CLI OAuth 令牌；使用 setup-token 或 Anthropic API 密钥。在任何地方生成令牌并粘贴到网关主机上。参见 [Anthropic](/providers/anthropic) 和 [OAuth](/concepts/oauth)。

注意：Claude 订阅访问受 Anthropic 条款约束。对于生产或多用户工作负载，API 密钥通常是更安全的选择。

### 为什么我看到来自 Anthropic 的 `HTTP 429: rate_limit_error`？

这意味着你当前窗口的 **Anthropic 配额/速率限制**已耗尽。如果你使用 **Claude 订阅**（setup-token 或 Claude Code OAuth），等待窗口重置或升级你的计划。如果你使用 **Anthropic API 密钥**，请检查 Anthropic 控制台的使用/计费并根据需要提高限制。

提示：设置**备用模型**，以便 OpenClaw 在提供商速率受限时继续回复。
参见[模型](/cli/models)和[OAuth](/concepts/oauth)。

### 支持 AWS Bedrock 吗？

支持——通过 pi-ai 的 **Amazon Bedrock (Converse)** 提供商进行**手动配置**。你必须在网关主机上提供 AWS 凭证/区域，并在模型配置中添加 Bedrock 提供商条目。参见 [Amazon Bedrock](/bedrock) 和[模型提供商](/providers/models)。如果你更喜欢托管密钥流程，Bedrock 前面的 OpenAI 兼容代理仍然是一个有效选项。

### Codex 认证如何工作？

OpenClaw 通过 OAuth（ChatGPT 登录）支持 **OpenAI Code (Codex)**。向导可以运行 OAuth 流程，并在适当时将默认模型设置为 `openai-codex/gpt-5.2`。参见[模型提供商](/concepts/model-providers)和[向导](/start/wizard)。

### 你们支持 OpenAI 订阅认证（Codex OAuth）吗？

支持。OpenClaw 完全支持 **OpenAI Code (Codex) 订阅 OAuth**。引导向导可以为你运行 OAuth 流程。

参见 [OAuth](/concepts/oauth)、[模型提供商](/concepts/model-providers)和[向导](/start/wizard)。

### 如何设置 Gemini CLI OAuth？

Gemini CLI 使用**插件认证流程**，而不是 `openclaw.json` 中的客户端 ID 或密钥。

步骤：

1. 启用插件：`openclaw plugins enable google-gemini-cli-auth`
2. 登录：`openclaw models auth login --provider google-gemini-cli --set-default`

这将 OAuth 令牌存储在网关主机上的认证配置文件中。详情：[模型提供商](/concepts/model-providers)。

### 本地模型适合随意聊天吗？

通常不适合。OpenClaw 需要大上下文 + 强安全性；小模型会截断和泄漏。如果必须，请在本地运行你能运行的**最大** MiniMax M2.1 构建（LM Studio）并参见 [/gateway/local-models](/gateway/local-models)。较小/量化的模型会增加提示注入风险——参见[安全](/gateway/security)。

### 如何将托管模型流量保持在特定区域？

选择区域固定端点。OpenRouter 为 MiniMax、Kimi 和 GLM 提供美国托管选项；选择美国托管变体以将数据保留在区域内。你仍然可以通过使用 `models.mode: "merge"` 列出 Anthropic/OpenAI，以便在遵守你选择的区域提供商的同时保持备用可用。

### 我必须购买 Mac Mini 才能安装这个吗？

不需要。OpenClaw 在 macOS 或 Linux（通过 WSL2 的 Windows）上运行。Mac mini 是可选的——有些人购买它作为始终在线的主机，但小型 VPS、家庭服务器或树莓派级盒子也可以工作。

你只需要 Mac **用于 macOS 专用工具**。对于 iMessage，你可以将网关保留在 Linux 上，并通过将 `channels.imessage.cliPath` 指向 SSH 包装器在任何 Mac 上通过 SSH 运行 `imsg`。如果你想要其他 macOS 专用工具，请在 Mac 上运行网关或配对 macOS 节点。

文档：[iMessage](/channels/imessage)、[节点](/nodes)、[Mac 远程模式](/platforms/mac/remote)。

### 我需要 Mac mini 来支持 iMessage 吗？

你需要**某个 macOS 设备**登录到信息。它**不必**是 Mac mini——任何 Mac 都可以。OpenClaw 的 iMessage 集成在 macOS 上运行（BlueBubbles 或 `imsg`），而网关可以在别处运行。

常见设置：

- 在 Linux/VPS 上运行网关，并将 `channels.imessage.cliPath` 指向在 Mac 上运行 `imsg` 的 SSH 包装器。
- 如果你想要最简单的单机设置，请在 Mac 上运行所有内容。

文档：[iMessage](/channels/imessage)、[BlueBubbles](/channels/bluebubbles)、
[Mac 远程模式](/platforms/mac/remote)。

### 如果我购买 Mac mini 来运行 OpenClaw，我可以将它连接到我的 MacBook Pro 吗？

可以。**Mac mini 可以运行网关**，你的 MacBook Pro 可以作为**节点**（配套设备）连接。节点不运行网关——它们在该设备上提供屏幕/摄像头/画布和 `system.run` 等额外功能。

常见模式：

- Mac mini 上的网关（始终在线）。
- MacBook Pro 运行 macOS 应用程序或节点主机并配对到网关。
- 使用 `openclaw nodes status` / `openclaw nodes list` 查看它。

文档：[节点](/nodes)、[节点 CLI](/cli/nodes)。

### 我可以使用 Bun 吗？

Bun **不推荐**。我们看到运行时错误，特别是 WhatsApp 和 Telegram。
对稳定网关使用 **Node**。

如果你仍然想尝试 Bun，请在没有 WhatsApp/Telegram 的非生产网关上尝试。

### Telegram：allowFrom 中应该填什么？

`channels.telegram.allowFrom` 是**人类发送者的 Telegram 用户 ID**（数字，推荐）或 `@username`。它不是机器人用户名。

更安全（无第三方机器人）：

- 向你的机器人发送私信，然后运行 `openclaw logs --follow` 并读取 `from.id`。

官方 Bot API：

- 向你的机器人发送私信，然后调用 `https://api.telegram.org/bot<bot_token>/getUpdates` 并读取 `message.from.id`。

第三方（隐私较低）：

- 向 `@userinfobot` 或 `@getidsbot` 发送私信。

参见 [/channels/telegram](/channels/telegram#access-control-dms--groups)。

### 多个人可以使用一个 WhatsApp 号码与不同的 OpenClaw 实例吗？

可以，通过**多 Agent 路由**。将每个发送者的 WhatsApp **私信**（对等点 `kind: "dm"`，发送者 E.164 如 `+15551234567`）绑定到不同的 `agentId`，这样每个人都有自己的工作区和会话存储。回复仍然来自**相同的 WhatsApp 帐户**，私信访问控制（`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`）按 WhatsApp 帐户全局设置。参见[多 Agent 路由](/concepts/multi-agent)和[WhatsApp](/channels/whatsapp)。

### 我可以运行一个"快速聊天"Agent 和一个"用于编码的 Opus"Agent 吗？

可以。使用多 Agent 路由：给每个 Agent 自己的默认模型，然后将入站路由（提供商帐户或特定对等点）绑定到每个 Agent。示例配置位于[多 Agent 路由](/concepts/multi-agent)。另见[模型](/concepts/models)和[配置](/gateway/configuration)。

### Homebrew 在 Linux 上工作吗？

可以。Homebrew 支持 Linux（Linuxbrew）。快速设置：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
brew install <formula>
```

如果你通过 systemd 运行 OpenClaw，请确保服务 PATH 包含 `/home/linuxbrew/.linuxbrew/bin`（或你的 brew 前缀），以便 `brew` 安装的工具在非登录 shell 中解析。
最近的构建还会在 Linux systemd 服务上预置常见的用户 bin 目录（例如 `~/.local/bin`、`~/.npm-global/bin`、`~/.local/share/pnpm`、`~/.bun/bin`），并在设置时遵守 `PNPM_HOME`、`NPM_CONFIG_PREFIX`、`BUN_INSTALL`、`VOLTA_HOME`、`ASDF_DATA_DIR`、`NVM_DIR` 和 `FNM_DIR`。

### 可修改的（git）安装和 npm 安装有什么区别？

- **可修改的（git）安装：**完整的源码检出，可编辑，最适合贡献者。
  你在本地运行构建，可以修补代码/文档。
- **npm 安装：**全局 CLI 安装，无仓库，最适合"直接运行"。
  更新来自 npm 分发标签。

文档：[入门](/start/getting-started)、[更新](/install/updating)。

### 我以后可以在 npm 和 git 安装之间切换吗？

可以。安装另一种风格，然后运行 Doctor，以便网关服务指向新的入口点。
这**不会删除你的数据**——它只更改 OpenClaw 代码安装。你的状态
（`~/.openclaw`）和工作区（`~/.openclaw/workspace`）保持原样。

从 npm → git：

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm build
openclaw doctor
openclaw gateway restart
```

从 git → npm：

```bash
npm install -g openclaw@latest
openclaw doctor
openclaw gateway restart
```

Doctor 检测网关服务入口点不匹配，并提供重写服务配置以匹配当前安装的选项（在自动化中使用 `--repair`）。

备份提示：参见[备份策略](#推荐的备份策略是什么)。

### 我应该在笔记本电脑还是 VPS 上运行网关？

简而言之：**如果你想要 24/7 可靠性，请使用 VPS**。如果你想要最低的摩擦，并且可以接受睡眠/重启，请在本地运行。

**笔记本电脑（本地网关）**

- **优点：**无服务器成本，直接访问本地文件，实时浏览器窗口。
- **缺点：**睡眠/网络断开 = 断开连接，操作系统更新/重启中断，必须保持清醒。

**VPS / 云端**

- **优点：**始终在线，稳定的网络，无笔记本电脑睡眠问题，更容易保持运行。
- **缺点：**通常无头运行（使用截图），仅远程文件访问，你必须通过 SSH 进行更新。

**OpenClaw 特定说明：**WhatsApp/Telegram/Slack/Mattermost（插件）/Discord 都可以从 VPS 正常工作。唯一的真正权衡是**无头浏览器**与可见窗口。参见[浏览器](/tools/browser)。

**推荐默认值：**如果你之前遇到过网关断开连接，请使用 VPS。当你积极使用 Mac 并想要本地文件访问或具有可见浏览器的 UI 自动化时，本地很好。

### 在专用机器上运行 OpenClaw 有多重要？

不是必需的，但**为了可靠性和隔离性而推荐**。

- **专用主机（VPS/Mac mini/Pi）：**始终在线，更少的睡眠/重启中断，更干净的权限，更容易保持运行。
- **共享笔记本电脑/台式机：**对于测试和积极使用完全没问题，但当机器睡眠或更新时预期会有暂停。

如果你想要两全其美，请将网关保留在专用主机上，并将你的笔记本电脑作为**节点**配对以进行本地屏幕/摄像头/执行工具。参见[节点](/nodes)。
有关安全指导，请阅读[安全](/gateway/security)。

### VPS 的最低要求和推荐操作系统是什么？

OpenClaw 是轻量级的。对于基本网关 + 一个聊天频道：

- **绝对最低：**1 vCPU、1GB RAM、~500MB 磁盘。
- **推荐：**1-2 vCPU、2GB RAM 或更多以获得余量（日志、媒体、多个频道）。节点工具和浏览器自动化可能很耗资源。

操作系统：使用 **Ubuntu LTS**（或任何现代 Debian/Ubuntu）。Linux 安装路径在那里测试得最好。

文档：[Linux](/platforms/linux)、[VPS 托管](/vps)。

### 我可以在 VM 中运行 OpenClaw 吗？要求是什么？

可以。将 VM 视为与 VPS 相同：它需要始终在线、可访问，并有足够的 RAM 供网关和任何你启用的频道使用。

基线指导：

- **绝对最低：**1 vCPU、1GB RAM。
- **推荐：**如果你运行多个频道、浏览器自动化或媒体工具，2GB RAM 或更多。
- **操作系统：**Ubuntu LTS 或其他现代 Debian/Ubuntu。

如果你在 Windows 上，**WSL2 是最简单的 VM 风格设置**，并且具有最佳的工具兼容性。参见 [Windows](/platforms/windows)、[VPS 托管](/vps)。
如果你在 VM 中运行 macOS，请参见 [macOS VM](/platforms/macos-vm)。

---

## 什么是 OpenClaw？

### 用一段话描述 OpenClaw 是什么？

OpenClaw 是你在自己的设备上运行的个人 AI 助手。它在你已经使用的消息界面上回复（WhatsApp、Telegram、Slack、Mattermost（插件）、Discord、Google Chat、Signal、iMessage、WebChat），还可以在支持的平台上进行语音 + 实时画布。**网关**是始终在线的控制平面；助手是产品。

### 价值主张是什么？

OpenClaw 不是"只是一个 Claude 包装器"。它是一个**本地优先的控制平面**，让你可以在**自己的硬件**上运行一个有能力的助手，从你已经在使用的聊天应用程序访问，具有有状态会话、内存和工具——而无需将工作流的控制权交给托管的 SaaS。

亮点：

- **你的设备，你的数据：**在你想要的任何地方运行网关（Mac、Linux、VPS）并将工作区 + 会话历史保留在本地。
- **真正的频道，不是 Web 沙盒：**WhatsApp/Telegram/Slack/Discord/Signal/iMessage 等，以及移动语音和支持平台上的画布。
- **与模型无关：**使用 Anthropic、OpenAI、MiniMax、OpenRouter 等，具有每 Agent 路由和故障转移。
- **仅限本地选项：**运行本地模型，以便**所有数据可以保留在你的设备上**（如果你愿意）。
- **多 Agent 路由：**按频道、帐户或任务分隔的 Agent，每个都有自己的工作区和默认值。
- **开源且可修改：**检查、扩展和自托管，无供应商锁定。

文档：[网关](/gateway)、[频道](/channels)、[多 Agent](/concepts/multi-agent)、
[内存](/concepts/memory)。

### 我刚设置好它，首先应该做什么？

好的第一个项目：

- 构建一个网站（WordPress、Shopify 或简单的静态网站）。
- 原型移动应用程序（大纲、屏幕、API 计划）。
- 组织文件和文件夹（清理、命名、标记）。
- 连接 Gmail 并自动化摘要或跟进。

它可以处理大型任务，但当你将它们分成阶段并使用子 Agent 进行并行工作时，效果最佳。

### OpenClaw 的五大日常用例是什么？

日常胜利通常看起来像这样：

- **个人简报：**收件箱、日历和你关心的新闻的摘要。
- **研究和起草：**快速研究、摘要以及电子邮件或文档的初稿。
- **提醒和跟进：**cron 或心跳驱动的提示和清单。
- **浏览器自动化：**填写表格、收集数据和重复 Web 任务。
- **跨设备协调：**从手机发送任务，让网关在服务器上运行它，然后在聊天中返回结果。

### OpenClaw 可以帮助 SaaS 的潜在客户生成外联广告和博客吗？

对于**研究、资格认证和起草**可以。它可以扫描网站、建立短名单、总结潜在客户以及撰写外联或广告文案草稿。

对于**外联或广告运行**，请让人参与其中。避免垃圾邮件，遵守当地法律和平台政策，并在发送前审查任何内容。最安全的模式是让 OpenClaw 起草，你批准。

文档：[安全](/gateway/security)。

### 与 Claude Code 相比，Web 开发有什么优势？

OpenClaw 是一个**个人助手**和协调层，而不是 IDE 替代品。在仓库内使用 Claude Code 或 Codex 进行最快的直接编码循环。当你想要持久内存、跨设备访问和工具编排时，使用 OpenClaw。

优势：

- **跨会话的持久内存 + 工作区**
- **多平台访问**（WhatsApp、Telegram、TUI、WebChat）
- **工具编排**（浏览器、文件、调度、钩子）
- **始终在线的网关**（在 VPS 上运行，从任何地方交互）
- **节点**用于本地浏览器/屏幕/摄像头/执行

展示：https://openclaw.ai/showcase

---

## 技能和自动化

### 如何在不弄脏仓库的情况下自定义技能？

使用托管覆盖而不是编辑仓库副本。将你的更改放在 `~/.openclaw/skills/<name>/SKILL.md` 中（或通过 `~/.openclaw/openclaw.json` 中的 `skills.load.extraDirs` 添加文件夹）。优先级是 `<workspace>/skills` > `~/.openclaw/skills` > 捆绑包，因此托管覆盖在不接触 git 的情况下获胜。只有值得上游的编辑才应该留在仓库中并作为 PR 发布。

### 我可以从自定义文件夹加载技能吗？

可以。通过 `~/.openclaw/openclaw.json` 中的 `skills.load.extraDirs` 添加额外目录（最低优先级）。默认优先级保持不变：`<workspace>/skills` → `~/.openclaw/skills` → 捆绑包 → `skills.load.extraDirs`。`clawhub` 默认安装到 `./skills`，OpenClaw 将其视为 `<workspace>/skills`。

### 如何为不同任务使用不同模型？

今天支持的模式是：

- **Cron 作业**：隔离作业可以为每个作业设置 `model` 覆盖。
- **子 Agent**：将任务路由到具有不同默认模型的单独 Agent。
- **按需切换**：使用 `/model` 随时切换当前会话模型。

参见 [Cron 作业](/automation/cron-jobs)、[多 Agent 路由](/concepts/multi-agent)和[斜杠命令](/tools/slash-commands)。

### 机器人在做繁重工作时冻结。如何卸载它？

对长时间或并行任务使用**子 Agent**。子 Agent 在自己的会话中运行，返回摘要，并保持你的主聊天响应。

要求你的机器人"为此任务生成子 Agent"或使用 `/subagents`。
在聊天中使用 `/status` 查看网关现在正在做什么（以及它是否忙碌）。

令牌提示：长时间任务和子 Agent 都消耗令牌。如果成本是问题，请通过 `agents.defaults.subagents.model` 为子 Agent 设置更便宜的模型。

文档：[子 Agent](/tools/subagents)。

### Cron 或提醒没有触发。我应该检查什么？

Cron 在网关进程内运行。如果网关没有持续运行，计划的作业将不会运行。

检查清单：

- 确认 cron 已启用（`cron.enabled`）且未设置 `OPENCLAW_SKIP_CRON`。
- 检查网关是否 24/7 运行（无睡眠/重启）。
- 验证作业的时区设置（`--tz` 与主机时区）。

调试：

```bash
openclaw cron run <jobId> --force
openclaw cron runs --id <jobId> --limit 50
```

文档：[Cron 作业](/automation/cron-jobs)、[Cron 与心跳](/automation/cron-vs-heartbeat)。

### 如何在 Linux 上安装技能？

使用 **ClawHub**（CLI）或将技能放入你的工作区。macOS 技能 UI 在 Linux 上不可用。
在 https://clawhub.com 浏览技能。

安装 ClawHub CLI（选择一个包管理器）：

```bash
npm i -g clawhub
```

```bash
pnpm add -g clawhub
```

### OpenClaw 可以按计划或在后台连续运行任务吗？

可以。使用网关调度器：

- **Cron 作业**用于计划或重复任务（在重启后持久存在）。
- **心跳**用于"主会话"定期检查。
- **隔离作业**用于发布摘要或交付到聊天的自主 Agent。

文档：[Cron 作业](/automation/cron-jobs)、[Cron 与心跳](/automation/cron-vs-heartbeat)、
[心跳](/gateway/heartbeat)。

### 我可以在 Linux 上运行 Apple/macOS 专用技能吗？

不能直接运行。macOS 技能由 `metadata.openclaw.os` 加上所需的二进制文件控制，只有当它们在**网关主机**上符合条件时，技能才会出现在系统提示中。在 Linux 上，`darwin` 专用技能（如 `imsg`、`apple-notes`、`apple-reminders`）不会加载，除非你覆盖门控。

你有三种支持的模式：

**选项 A - 在 Mac 上运行网关（最简单）。**
在 macOS 二进制文件存在的地方运行网关，然后从 Linux 通过[远程模式](#如何在远程模式下运行-openclaw客户端连接到别处的网关)或 Tailscale 连接。技能正常加载，因为网关主机是 macOS。

**选项 B - 使用 macOS 节点（无 SSH）。**
在 Linux 上运行网关，配对 macOS 节点（菜单栏应用程序），并在 Mac 上将**节点运行命令**设置为"始终询问"或"始终允许"。当所需二进制文件存在于节点上时，OpenClaw 可以将 macOS 专用技能视为符合条件。Agent 通过 `nodes` 工具运行这些技能。如果你选择"始终询问"，在提示中批准"始终允许"会将该命令添加到允许列表。

**选项 C - 通过 SSH 代理 macOS 二进制文件（高级）。**
将网关保留在 Linux 上，但使所需的 CLI 二进制文件解析为在 Mac 上运行的 SSH 包装器。然后覆盖技能以允许 Linux，使其保持符合条件。

1. 为二进制文件创建 SSH 包装器（示例：`imsg`）：
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   exec ssh -T user@mac-host /opt/homebrew/bin/imsg "$@"
   ```
2. 将包装器放在 Linux 主机的 `PATH` 上（例如 `~/bin/imsg`）。
3. 覆盖技能元数据（工作区或 `~/.openclaw/skills`）以允许 Linux：
   ```markdown
   ---
   name: imsg
   description: 用于列出聊天、历史记录、监视和发送的 iMessage/SMS CLI。
   metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["imsg"] } } }
   ---
   ```
4. 开始新会话，以便技能快照刷新。

对于 iMessage，你还可以将 `channels.imessage.cliPath` 指向 SSH 包装器（OpenClaw 只需要 stdio）。参见 [iMessage](/channels/imessage)。

### 你们有 Notion 或 HeyGen 集成吗？

今天没有内置。

选项：

- **自定义技能/插件：**最适合可靠的 API 访问（Notion/HeyGen 都有 API）。
- **浏览器自动化：**无需代码即可工作，但速度较慢且更脆弱。

如果你想为每个客户端保持上下文（代理工作流），一个简单的模式是：

- 每个客户端一个 Notion 页面（上下文 + 首选项 + 活动工作）。
- 要求 Agent 在会话开始时获取该页面。

如果你想要原生集成，请打开功能请求或构建针对这些 API 的技能。

安装技能：

```bash
clawhub install <skill-slug>
clawhub update --all
```

ClawHub 安装到你当前目录下的 `./skills`（或回退到你配置的 OpenClaw 工作区）；OpenClaw 在下次会话时将其视为 `<workspace>/skills`。对于跨 Agent 的共享技能，将它们放在 `~/.openclaw/skills/<name>/SKILL.md` 中。一些技能期望通过 Homebrew 安装的二进制文件；在 Linux 上这意味着 Linuxbrew（参见上面的 Homebrew Linux FAQ 条目）。参见[技能](/tools/skills)和[ClawHub](/tools/clawhub)。

### 如何安装 Chrome 扩展程序以进行浏览器接管？

使用内置安装程序，然后在 Chrome 中加载解包的扩展程序：

```bash
openclaw browser extension install
openclaw browser extension path
```

然后 Chrome → `chrome://extensions` → 启用"开发者模式" → "加载解包" → 选择该文件夹。

完整指南（包括远程网关 + 安全说明）：[Chrome 扩展程序](/tools/chrome-extension)

如果网关与 Chrome 在同一台机器上运行（默认设置），你通常**不需要**任何额外的东西。
如果网关在别处运行，请在浏览器机器上运行节点主机，以便网关可以代理浏览器操作。
你仍然需要点击要控制的标签页上的扩展程序按钮（它不会自动附加）。

---

## 沙盒和内存

### 有专门的沙盒文档吗？

有。参见[沙盒](/gateway/sandboxing)。对于 Docker 特定设置（Docker 中的完整网关或沙盒镜像），参见 [Docker](/install/docker)。

**我可以保持私信个人化，但让群组公共沙盒化与一个 Agent 吗？**

可以——如果你的私人流量是**私信**，你的公共流量是**群组**。

使用 `agents.defaults.sandbox.mode: "non-main"`，这样群组/频道会话（非主密钥）在 Docker 中运行，而主私信会话保留在主机上。然后通过 `tools.sandbox.tools` 限制沙盒会话中可用的工具。

设置演练 + 示例配置：[群组：个人私信 + 公共群组](/concepts/groups#pattern-personal-dms-public-groups-single-agent)

关键配置参考：[网关配置](/gateway/configuration#agentsdefaultssandbox)

### 如何将主机文件夹绑定到沙盒中？

将 `agents.defaults.sandbox.docker.binds` 设置为 `["host:path:mode"]`（例如，`"/home/user/src:/src:ro"`）。全局 + 每 Agent 绑定合并；当 `scope: "shared"` 时忽略每 Agent 绑定。对任何敏感内容使用 `:ro`，并记住绑定会绕过沙盒文件系统墙。参见[沙盒](/gateway/sandboxing#custom-bind-mounts)和[沙盒与工具策略与提升](/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check)获取示例和安全说明。

### 内存如何工作？

OpenClaw 内存只是 Agent 工作区中的 Markdown 文件：

- `memory/YYYY-MM-DD.md` 中的每日笔记
- `MEMORY.md` 中的精选长期笔记（仅限主/私人会话）

OpenClaw 还运行**静默预压缩内存刷新**，以提醒模型在自动压缩之前写入持久笔记。这仅在工作区可写时运行（只读沙盒跳过它）。参见[内存](/concepts/memory)。

### 内存不断忘记事情。如何让它记住？

要求机器人**将事实写入内存**。长期笔记属于 `MEMORY.md`，短期上下文进入 `memory/YYYY-MM-DD.md`。

这仍然是我们正在改进的领域。提醒模型存储内存会有帮助；它会知道该做什么。如果它一直忘记，请验证网关在每次运行时使用相同的工作区。

文档：[内存](/concepts/memory)、[Agent 工作区](/concepts/agent-workspace)。

### 语义内存搜索需要 OpenAI API 密钥吗？

只有当你使用 **OpenAI 嵌入**时才需要。Codex OAuth 涵盖聊天/完成，并**不**授予嵌入访问权限，因此**使用 Codex 登录（OAuth 或 Codex CLI 登录）**对语义内存搜索没有帮助。OpenAI 嵌入仍然需要真正的 API 密钥（`OPENAI_API_KEY` 或 `models.providers.openai.apiKey`）。

如果你不明确设置提供商，OpenClaw 会在可以解析 API 密钥（认证配置文件、`models.providers.*.apiKey` 或环境变量）时自动选择提供商。如果解析到 OpenAI 密钥，它首选 OpenAI，否则如果解析到 Gemini 密钥，则首选 Gemini。如果两个密钥都不可用，内存搜索将保持禁用状态，直到你配置它。如果你配置了本地模型路径并且存在，OpenClaw 首选 `local`。

如果你宁愿保持本地，请设置 `memorySearch.provider = "local"`（并可选地设置 `memorySearch.fallback = "none"`）。如果你想要 Gemini 嵌入，请设置 `memorySearch.provider = "gemini"` 并提供 `GEMINI_API_KEY`（或 `memorySearch.remote.apiKey`）。我们支持 **OpenAI、Gemini 或本地**嵌入模型——参见[内存](/concepts/memory)获取设置详情。

### 内存会永远持久吗？限制是什么？

内存文件驻留在磁盘上，直到你删除它们。限制是你的存储，而不是模型。**会话上下文**仍然受模型上下文窗口限制，因此长时间对话可能会压缩或截断。这就是内存搜索存在的原因——它只将相关部分拉回上下文。

文档：[内存](/concepts/memory)、[上下文](/concepts/context)。

---

## 磁盘上的文件位置

### 与 OpenClaw 一起使用的所有数据都保存在本地吗？

不是——**OpenClaw 的状态是本地的**，但**外部服务仍然可以看到你发送给它们的内容**。

- **默认本地：**会话、内存文件、配置和工作区位于网关主机上
  （`~/.openclaw` + 你的工作区目录）。
- **出于必要而远程：**你发送给模型提供商（Anthropic/OpenAI/等）的消息会转到他们的 API，聊天平台（WhatsApp/Telegram/Slack/等）将消息数据存储在他们的服务器上。
- **你控制足迹：**使用本地模型将提示保留在你的机器上，但频道流量仍然通过频道的服务器。

相关：[Agent 工作区](/concepts/agent-workspace)、[内存](/concepts/memory)。

### OpenClaw 在哪里存储其数据？

所有内容都位于 `$OPENCLAW_STATE_DIR` 下（默认：`~/.openclaw`）：

| 路径 | 用途 |
| --------------------------------------------------------------- | ------------------------------------------------------------ |
| `$OPENCLAW_STATE_DIR/openclaw.json` | 主配置（JSON5） |
| `$OPENCLAW_STATE_DIR/credentials/oauth.json` | 旧版 OAuth 导入（首次使用时复制到认证配置文件） |
| `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | 认证配置文件（OAuth + API 密钥） |
| `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json` | 运行时认证缓存（自动管理） |
| `$OPENCLAW_STATE_DIR/credentials/` | 提供商状态（例如 `whatsapp/<accountId>/creds.json`） |
| `$OPENCLAW_STATE_DIR/agents/` | 每 Agent 状态（agentDir + 会话） |
| `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/` | 对话历史和状态（每 Agent） |
| `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json` | 会话元数据（每 Agent） |

旧版单 Agent 路径：`~/.openclaw/agent/*`（由 `openclaw doctor` 迁移）。

你的**工作区**（AGENTS.md、内存文件、技能等）是单独的，通过 `agents.defaults.workspace` 配置（默认：`~/.openclaw/workspace`）。

### AGENTS.md / SOUL.md / USER.md / MEMORY.md 应该放在哪里？

这些文件位于 **Agent 工作区**中，而不是 `~/.openclaw`。

- **工作区（每 Agent）**：`AGENTS.md`、`SOUL.md`、`IDENTITY.md`、`USER.md`、
  `MEMORY.md`（或 `memory.md`）、`memory/YYYY-MM-DD.md`、可选的 `HEARTBEAT.md`。
- **状态目录（`~/.openclaw`）**：配置、凭证、认证配置文件、会话、日志、
  和共享技能（`~/.openclaw/skills`）。

默认工作区是 `~/.openclaw/workspace`，可通过以下方式配置：

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

如果机器人在重启后"忘记"，请确认网关在每次启动时使用相同的工作区（并记住：远程模式使用**网关主机的**工作区，而不是你的本地笔记本电脑）。

提示：如果你想要持久的行为或首选项，请要求机器人**将其写入 AGENTS.md 或 MEMORY.md**，而不是依赖聊天历史。

参见 [Agent 工作区](/concepts/agent-workspace)和[内存](/concepts/memory)。

### 推荐的备份策略是什么？

将你的 **Agent 工作区**放在**私有** git 仓库中，并将其备份到某个私有地方（例如 GitHub 私有）。这可以捕获内存 + AGENTS/SOUL/USER 文件，并让你以后可以恢复助手的"思维"。

**不要**提交 `~/.openclaw` 下的任何内容（凭证、会话、令牌）。
如果你需要完全恢复，请分别备份工作区和状态目录（参见上面的迁移问题）。

文档：[Agent 工作区](/concepts/agent-workspace)。

### 如何完全卸载 OpenClaw？

参见专门指南：[卸载](/install/uninstall)。

### Agent 可以在工作区之外工作吗？

可以。工作区是**默认 cwd**和内存锚点，而不是硬沙盒。
相对路径在工作区内解析，但绝对路径可以访问其他主机位置，除非启用了沙盒。如果你需要隔离，请使用
[`agents.defaults.sandbox`](/gateway/sandboxing)或每 Agent 沙盒设置。如果
你想让仓库成为默认工作目录，请将该 Agent 的
`workspace`指向仓库根目录。OpenClaw 仓库只是源代码；保持
工作区分开，除非你有意让 Agent 在其中工作。

示例（仓库作为默认 cwd）：

```json5
{
  agents: {
    defaults: {
      workspace: "~/Projects/my-repo",
    },
  },
}
```

### 我在远程模式下——会话存储在哪里？

会话状态由**网关主机**拥有。如果你在远程模式下，你关心的会话存储在远程机器上，而不是你的本地笔记本电脑。参见[会话管理](/concepts/session)。

---

## 配置基础

### 配置是什么格式？它在哪里？

OpenClaw 从 `$OPENCLAW_CONFIG_PATH`（默认：`~/.openclaw/openclaw.json`）读取可选的 **JSON5** 配置：

```
$OPENCLAW_CONFIG_PATH
```

如果文件缺失，它使用安全的默认设置（包括默认工作区 `~/.openclaw/workspace`）。

### 我设置了 `gateway.bind: "lan"`（或 `"tailnet"`），现在什么都没有监听 / UI 显示未授权

非环回绑定**需要认证**。配置 `gateway.auth.mode` + `gateway.auth.token`（或使用 `OPENCLAW_GATEWAY_TOKEN`）。

```json5
{
  gateway: {
    bind: "lan",
    auth: {
      mode: "token",
      token: "replace-me",
    },
  },
}
```

注意：

- `gateway.remote.token` 仅用于**远程 CLI 调用**；它不会启用本地网关认证。
- 控制 UI 通过 `connect.params.auth.token` 进行认证（存储在应用程序/UI 设置中）。避免将令牌放在 URL 中。

### 为什么我现在在 localhost 上需要令牌？

向导默认生成网关令牌（即使在环回上），因此**本地 WS 客户端必须认证**。这会阻止其他本地进程调用网关。将令牌粘贴到控制 UI 设置中（或你的客户端配置）以连接。

如果你**真的**想要开放环回，请从配置中删除 `gateway.auth`。Doctor 可以随时为你生成令牌：`openclaw doctor --generate-gateway-token`。

### 更改配置后我必须重启吗？

网关监视配置并支持热重载：

- `gateway.reload.mode: "hybrid"`（默认）：热应用安全更改，对关键更改重启
- 还支持 `hot`、`restart`、`off`

### 如何启用 Web 搜索（和 Web 获取）？

`web_fetch` 无需 API 密钥即可工作。`web_search` 需要 Brave Search API
密钥。**推荐：**运行 `openclaw configure --section web` 将其存储在
`tools.web.search.apiKey` 中。环境替代方案：为网关进程设置 `BRAVE_API_KEY`。

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "BRAVE_API_KEY_HERE",
        maxResults: 5,
      },
      fetch: {
        enabled: true,
      },
    },
  },
}
```

注意：

- 如果你使用允许列表，请添加 `web_search`/`web_fetch` 或 `group:web`。
- `web_fetch` 默认启用（除非明确禁用）。
- 守护程序从 `~/.openclaw/.env`（或服务环境）读取环境变量。

文档：[Web 工具](/tools/web)。

### 如何运行具有跨设备专业工作程序的中央网关？

常见模式是**一个网关**（例如树莓派）加**节点**和**Agent**：

- **网关（中央）：**拥有频道（Signal/WhatsApp）、路由和会话。
- **节点（设备）：**Mac/iOS/Android 作为外围设备连接并公开本地工具（`system.run`、`canvas`、`camera`）。
- **Agent（工作程序）：**用于特殊角色的单独大脑/工作区（例如"Hetzner 运维"、"个人数据"）。
- **子 Agent：**当你想要并行性时从主 Agent 生成后台工作。
- **TUI：**连接到网关并切换 Agent/会话。

文档：[节点](/nodes)、[远程访问](/gateway/remote)、[多 Agent 路由](/concepts/multi-agent)、[子 Agent](/tools/subagents)、[TUI](/tui)。

### OpenClaw 浏览器可以无头运行吗？

可以。这是一个配置选项：

```json5
{
  browser: { headless: true },
  agents: {
    defaults: {
      sandbox: { browser: { headless: true } },
    },
  },
}
```

默认是 `false`（有头）。无头模式在某些网站上更容易触发反机器人检查。参见[浏览器](/tools/browser)。

无头使用**相同的 Chromium 引擎**，适用于大多数自动化（表单、点击、抓取、登录）。主要区别：

- 没有可见的浏览器窗口（如果你需要视觉效果，请使用截图）。
- 一些网站对无头模式的自动化更严格（CAPTCHA、反机器人）。
  例如，X/Twitter 经常阻止无头会话。

### 如何使用 Brave 进行浏览器控制？

将 `browser.executablePath` 设置为你的 Brave 二进制文件（或任何基于 Chromium 的浏览器）并重启网关。
参见 [Browser](/tools/browser#use-brave-or-another-chromium-based-browser) 中的完整配置示例。

---

## 远程网关和节点

### 命令如何在 Telegram、网关和节点之间传播？

Telegram 消息由**网关**处理。网关运行 Agent，并且仅在需要节点工具时才通过**网关 WebSocket**调用节点：

Telegram → 网关 → Agent → `node.*` → 节点 → 网关 → Telegram

节点看不到入站提供商流量；它们只接收节点 RPC 调用。

### 如果网关托管在远程，我的 Agent 如何访问我的计算机？

简而言之：**将你的计算机配对为节点**。网关在别处运行，但它可以通过网关 WebSocket 在你的本地机器上调用 `node.*` 工具（屏幕、摄像头、系统）。

典型设置：

1. 在始终在线的主机（VPS/家庭服务器）上运行网关。
2. 将网关主机 + 你的计算机放在同一个 tailnet 上。
3. 确保网关 WS 可达（tailnet 绑定或 SSH 隧道）。
4. 在本地打开 macOS 应用程序并以**通过 SSH 远程**模式连接（或直接 tailnet），以便它可以注册为节点。
5. 在网关上批准节点：
   ```bash
   openclaw nodes pending
   openclaw nodes approve <requestId>
   ```

不需要单独的 TCP 桥接；节点通过网关 WebSocket 连接。

安全提醒：配对 macOS 节点允许在该机器上执行 `system.run`。只配对你信任的设备，并查看[安全](/gateway/security)。

文档：[节点](/nodes)、[网关协议](/gateway/protocol)、[macOS 远程模式](/platforms/mac/remote)、[安全](/gateway/security)。

### Tailscale 已连接，但我没有收到回复。现在怎么办？

检查基础知识：

- 网关正在运行：`openclaw gateway status`
- 网关健康：`openclaw status`
- 频道健康：`openclaw channels status`

然后验证认证和路由：

- 如果你使用 Tailscale Serve，请确保 `gateway.auth.allowTailscale` 设置正确。
- 如果你通过 SSH 隧道连接，请确认本地隧道已启动并指向正确的端口。
- 确认你的允许列表（私信或群组）包含你的帐户。

文档：[Tailscale](/gateway/tailscale)、[远程访问](/gateway/remote)、[频道](/channels)。

### 两个 OpenClaw 实例可以相互通信吗（本地 + VPS）？

可以。没有内置的"机器人对机器人"桥接，但你可以通过几种可靠的方式连接它：

**最简单：**使用两个机器人都可以访问的普通聊天频道（Telegram/Slack/WhatsApp）。
让机器人 A 向机器人 B 发送消息，然后让机器人 B 像往常一样回复。

**CLI 桥接（通用）：**运行一个脚本，使用 `openclaw agent --message ... --deliver` 调用另一个网关，以另一个机器人监听的聊天为目标。如果一个机器人在远程 VPS 上，请通过 SSH/Tailscale 将你的 CLI 指向该远程网关（参见[远程访问](/gateway/remote)）。

示例模式（从可以访问目标网关的机器运行）：

```bash
openclaw agent --message "来自本地机器人的问候" --deliver --channel telegram --reply-to <chat-id>
```

提示：添加一个防护栏，这样两个机器人不会无限循环（仅提及、频道允许列表或"不回复机器人消息"规则）。

文档：[远程访问](/gateway/remote)、[Agent CLI](/cli/agent)、[Agent 发送](/tools/agent-send)。

### 多个 Agent 需要单独的 VPS 吗？

不需要。一个网关可以托管多个 Agent，每个都有自己的工作区、模型默认值和路由。这是正常设置，比每个 Agent 运行一个 VPS 便宜得多且简单得多。

只有当你需要硬隔离（安全边界）或非常不同的配置不想共享时，才使用单独的 VPS。否则，保持一个网关并使用多个 Agent 或子 Agent。

### 使用个人笔记本电脑上的节点与从 VPS 通过 SSH 连接相比有什么好处吗？

有——节点是从远程网关访问你的笔记本电脑的一流方式，它们解锁的不仅仅是 shell 访问。网关在 macOS/Linux（通过 WSL2 的 Windows）上运行，并且是轻量级的（小型 VPS 或树莓派级盒子就可以；4 GB RAM 足够），因此常见设置是始终在线的主机加你的笔记本电脑作为节点。

- **不需要入站 SSH。**节点向外连接到网关 WebSocket 并使用设备配对。
- **更安全的执行控制。**`system.run` 由该笔记本电脑上的节点允许列表/批准控制。
- **更多设备工具。**节点公开 `canvas`、`camera` 和 `screen`，以及 `system.run`。
- **本地浏览器自动化。**将网关保留在 VPS 上，但在本地运行 Chrome 并通过 Chrome 扩展程序 + 笔记本电脑上的节点主机中继控制。

SSH 适合临时 shell 访问，但节点对于持续的 Agent 工作流和设备自动化更简单。

文档：[节点](/nodes)、[节点 CLI](/cli/nodes)、[Chrome 扩展程序](/tools/chrome-extension)。

### 我应该在第二台笔记本电脑上安装还是只添加一个节点？

如果你只需要第二台笔记本电脑上的**本地工具**（屏幕/摄像头/执行），请将其添加为**节点**。这保持单个网关并避免重复配置。本地节点工具目前仅限 macOS，但我们计划将其扩展到其他操作系统。

只有当你需要**硬隔离**或两个完全独立的机器人时，才安装第二个网关。

文档：[节点](/nodes)、[节点 CLI](/cli/nodes)、[多个网关](/gateway/multiple-gateways)。

### 节点运行网关服务吗？

不运行。每个主机应该只运行**一个网关**，除非你故意运行隔离的配置文件（参见[多个网关](/gateway/multiple-gateways)）。节点是连接到网关的外围设备（iOS/Android 节点，或菜单栏应用程序中的 macOS"节点模式"）。对于无头节点主机和 CLI 控制，请参见[节点主机 CLI](/cli/node)。

`gateway`、`discovery` 和 `canvasHost` 更改需要完全重启。

### 有 API / RPC 方式来应用配置吗？

有。`config.apply` 验证 + 写入完整配置，并作为操作的一部分重启网关。

### config.apply 擦除了我的配置。如何恢复并避免这种情况？

`config.apply` 替换**整个配置**。如果你发送部分对象，其他所有内容都会被删除。

恢复：

- 从备份恢复（git 或复制的 `~/.openclaw/openclaw.json`）。
- 如果你没有备份，请重新运行 `openclaw doctor` 并重新配置频道/模型。
- 如果这是意外的，请提交错误并包含你最后已知的配置或任何备份。
- 本地编码 Agent 通常可以从日志或历史记录中重建工作配置。

避免它：

- 对小的更改使用 `openclaw config set`。
- 对交互式编辑使用 `openclaw configure`。

文档：[配置](/cli/config)、[配置](/cli/configure)、[Doctor](/gateway/doctor)。

### 首次安装的最低"合理"配置是什么？

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

这设置你的工作区并限制谁可以触发机器人。

### 如何在 VPS 上设置 Tailscale 并从我的 Mac 连接？

最低步骤：

1. **在 VPS 上安装 + 登录**
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
2. **在你的 Mac 上安装 + 登录**
   - 使用 Tailscale 应用程序并登录到相同的 tailnet。
3. **启用 MagicDNS（推荐）**
   - 在 Tailscale 管理控制台中，启用 MagicDNS，以便 VPS 具有稳定的名称。
4. **使用 tailnet 主机名**
   - SSH：`ssh user@your-vps.tailnet-xxxx.ts.net`
   - 网关 WS：`ws://your-vps.tailnet-xxxx.ts.net:18789`

如果你想要没有 SSH 的控制 UI，请在 VPS 上使用 Tailscale Serve：

```bash
openclaw gateway --tailscale serve
```

这保持网关绑定到环回并通过 Tailscale 公开 HTTPS。参见 [Tailscale](/gateway/tailscale)。

### 如何将 Mac 节点连接到远程网关（Tailscale Serve）？

Serve 公开**网关控制 UI + WS**。节点通过相同的网关 WS 端点连接。

推荐设置：

1. **确保 VPS + Mac 在同一 tailnet 上**。
2. **在远程模式下使用 macOS 应用程序**（SSH 目标可以是 tailnet 主机名）。
   该应用程序将隧道网关端口并作为节点连接。
3. **在网关上批准节点**：
   ```bash
   openclaw nodes pending
   openclaw nodes approve <requestId>
   ```

文档：[网关协议](/gateway/protocol)、[发现](/gateway/discovery)、[macOS 远程模式](/platforms/mac/remote)。

---

## 环境变量和 .env 加载

### OpenClaw 如何加载环境变量？

OpenClaw 从父进程（shell、launchd/systemd、CI 等）读取环境变量，并额外加载：

- 当前工作目录中的 `.env`
- `~/.openclaw/.env`（又名 `$OPENCLAW_STATE_DIR/.env`）中的全局回退 `.env`

两个 `.env` 文件都不会覆盖现有的环境变量。

你还可以在配置中定义内联环境变量（仅当进程环境中缺失时才应用）：

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

参见 [/environment](/environment) 了解完整优先级和来源。

### "我通过服务启动了网关，我的环境变量消失了。"现在怎么办？

两个常见的修复：

1. 将缺失的密钥放在 `~/.openclaw/.env` 中，这样即使服务不继承你的 shell 环境，它们也会被拾取。
2. 启用 shell 导入（选择加入便利）：

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

这会运行你的登录 shell 并仅导入缺失的预期密钥（从不覆盖）。环境变量等效项：
`OPENCLAW_LOAD_SHELL_ENV=1`、`OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`。

### 我设置了 `COPILOT_GITHUB_TOKEN`，但模型状态显示"Shell env: off"。为什么？

`openclaw models status` 报告是否启用了 **shell 环境导入**。"Shell env: off"
并**不**意味着你的环境变量缺失——它只是意味着 OpenClaw 不会自动加载你的登录 shell。

如果网关作为服务（launchd/systemd）运行，它不会继承你的 shell 环境。通过执行以下操作之一来修复：

1. 将令牌放在 `~/.openclaw/.env` 中：
   ```
   COPILOT_GITHUB_TOKEN=...
   ```
2. 或启用 shell 导入（`env.shellEnv.enabled: true`）。
3. 或将其添加到你的配置 `env` 块中（仅当缺失时才应用）。

然后重启网关并重新检查：

```bash
openclaw models status
```

Copilot 令牌从 `COPILOT_GITHUB_TOKEN`（也是 `GH_TOKEN` / `GITHUB_TOKEN`）读取。
参见 [/concepts/model-providers](/concepts/model-providers) 和 [/environment](/environment)。

---

## 会话和多聊天

### 如何开始新的对话？

发送 `/new` 或 `/reset` 作为独立消息。参见[会话管理](/concepts/session)。

### 如果我不发送 /new，会话会自动重置吗？

会。会话在 `session.idleMinutes`（默认 **60**）后过期。**下一个**消息为该聊天密钥启动新的会话 ID。这不会删除记录——它只是启动一个新会话。

```json5
{
  session: {
    idleMinutes: 240,
  },
}
```

### 有没有办法让 OpenClaw 实例团队成为一个 CEO 和许多 Agent？

有，通过**多 Agent 路由**和**子 Agent**。你可以创建一个协调器 Agent 和几个具有自己工作区和模型的工作 Agent。

也就是说，这最好被视为一个**有趣的实验**。它令牌消耗大，通常不如使用具有单独会话的一个机器人高效。我们设想的典型模型是你与之交谈的一个机器人，具有用于并行工作的不同会话。该机器人还可以在需要时生成子 Agent。

文档：[多 Agent 路由](/concepts/multi-agent)、[子 Agent](/tools/subagents)、[Agent CLI](/cli/agents)。

### 为什么上下文在任务中途被截断？如何防止它？

会话上下文受模型窗口限制。长时间聊天、大型工具输出或许多文件可能会触发压缩或截断。

有帮助的做法：

- 要求机器人总结当前状态并将其写入文件。
- 在长时间任务之前使用 `/compact`，切换主题时使用 `/new`。
- 将重要上下文保留在工作区中，并要求机器人读回它。
- 对长时间或并行工作使用子 Agent，这样主聊天保持较小。
- 如果这种情况经常发生，请选择具有更大上下文窗口的模型。

### 如何完全重置 OpenClaw 但保持其安装？

使用重置命令：

```bash
openclaw reset
```

非交互式完全重置：

```bash
openclaw reset --scope full --yes --non-interactive
```

然后重新运行引导：

```bash
openclaw onboard --install-daemon
```

注意：

- 如果引导向导看到现有配置，它还会提供**重置**。参见[向导](/start/wizard)。
- 如果你使用了配置文件（`--profile` / `OPENCLAW_PROFILE`），请重置每个状态目录（默认是 `~/.openclaw-<profile>`）。
- 开发重置：`openclaw gateway --dev --reset`（仅限开发；擦除开发配置 + 凭证 + 会话 + 工作区）。

### 我收到"上下文太大"错误——如何重置或压缩？

使用以下之一：

- **压缩**（保留对话但总结较旧的回合）：

  ```
  /compact
  ```

  或 `/compact <instructions>` 来指导摘要。

- **重置**（相同聊天密钥的新会话 ID）：
  ```
  /new
  /reset
  ```

如果它不断发生：

- 启用或调整**会话修剪**（`agents.defaults.contextPruning`）以修剪旧工具输出。
- 使用具有更大上下文窗口的模型。

文档：[压缩](/concepts/compaction)、[会话修剪](/concepts/session-pruning)、[会话管理](/concepts/session)。

### 为什么我看到"LLM 请求被拒绝：messages.N.content.X.tool_use.input: 需要字段"？

这是提供商验证错误：模型发出了没有所需 `input` 的 `tool_use` 块。它通常意味着会话历史记录已过时或损坏（通常在长时间线程或工具/架构更改后）。

修复：使用 `/new`（独立消息）开始新会话。

### 为什么每 30 分钟收到一次心跳消息？

心跳默认每 **30 分钟**运行一次。调整或禁用它们：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "2h", // 或 "0m" 禁用
      },
    },
  },
}
```

如果 `HEARTBEAT.md` 存在但实际上为空（只有空行和 markdown 标题如 `# Heading`），OpenClaw 会跳过心跳运行以节省 API 调用。
如果文件缺失，心跳仍然运行，模型决定做什么。

每 Agent 覆盖使用 `agents.list[].heartbeat`。文档：[心跳](/gateway/heartbeat)。

### 我需要将"机器人帐户"添加到 WhatsApp 群组吗？

不需要。OpenClaw 在**你自己的帐户**上运行，因此如果你在群组中，OpenClaw 可以看到它。
默认情况下，群组回复被阻止，直到你允许发送者（`groupPolicy: "allowlist"`）。

如果你希望只有**你**能够触发群组回复：

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

### 如何获取 WhatsApp 群组的 JID？

选项 1（最快）：跟踪日志并在群组中发送测试消息：

```bash
openclaw logs --follow --json
```

查找以 `@g.us` 结尾的 `chatId`（或 `from`），如：
`1234567890-1234567890@g.us`。

选项 2（如果已配置/允许列表）：从配置列出群组：

```bash
openclaw directory groups list --channel whatsapp
```

文档：[WhatsApp](/channels/whatsapp)、[目录](/cli/directory)、[日志](/cli/logs)。

### 为什么 OpenClaw 不在群组中回复？

两个常见原因：

- 提及门控已打开（默认）。你必须 @提及机器人（或匹配 `mentionPatterns`）。
- 你配置了没有 `"*"` 的 `channels.whatsapp.groups`，且该群组不在允许列表中。

参见[群组](/concepts/groups)和[群组消息](/concepts/group-messages)。

### 群组/线程与私信共享上下文吗？

直接聊天默认折叠到主会话。群组/频道有自己的会话密钥，Telegram 主题 / Discord 线程是单独的会话。参见[群组](/concepts/groups)和[群组消息](/concepts/group-messages)。

### 我可以创建多少个工作区和 Agent？

没有硬性限制。几十个（甚至几百个）都可以，但要注意：

- **磁盘增长：**会话 + 记录位于 `~/.openclaw/agents/<agentId>/sessions/` 下。
- **令牌成本：**更多 Agent 意味着更多并发模型使用。
- **运维开销：**每 Agent 认证配置文件、工作区和频道路由。

提示：

- 每个 Agent 保持一个**活动**工作区（`agents.defaults.workspace`）。
- 如果磁盘增长，请修剪旧会话（删除 JSONL 或存储条目）。
- 使用 `openclaw doctor` 发现 stray 工作区和配置文件不匹配。

### 我可以同时运行多个机器人或聊天吗（Slack），应该如何设置？

可以。使用**多 Agent 路由**来运行多个隔离的 Agent，并按频道/帐户/对等点路由入站消息。Slack 作为频道受支持，可以绑定到特定 Agent。

浏览器访问功能强大，但不是"人类能做的任何事情"——反机器人、CAPTCHA 和 MFA 仍然可以阻止自动化。对于最可靠的浏览器控制，请在运行浏览器的机器上使用 Chrome 扩展程序中继（并将网关保留在任何地方）。

最佳实践设置：

- 始终在线的网关主机（VPS/Mac mini）。
- 每个角色一个 Agent（绑定）。
- 绑定到这些 Agent 的 Slack 频道。
- 需要时通过扩展程序中继（或节点）的本地浏览器。

文档：[多 Agent 路由](/concepts/multi-agent)、[Slack](/channels/slack)、
[浏览器](/tools/browser)、[Chrome 扩展程序](/tools/chrome-extension)、[节点](/nodes)。

---

## 模型：默认值、选择、别名、切换

### "默认模型"是什么？

OpenClaw 的默认模型是你设置为：

```
agents.defaults.model.primary
```

模型以 `provider/model` 形式引用（示例：`anthropic/claude-opus-4-5`）。如果你省略提供商，OpenClaw 目前暂时假设为 `anthropic` 作为弃用回退——但你仍然应该**明确**设置 `provider/model`。

### 你推荐什么模型？

**推荐默认：**`anthropic/claude-opus-4-5`。
**好的替代：**`anthropic/claude-sonnet-4-5`。
**可靠（个性较少）：**`openai/gpt-5.2`——几乎和 Opus 一样好，只是个性较少。
**预算：**`zai/glm-4.7`。

MiniMax M2.1 有自己的文档：[MiniMax](/providers/minimax) 和
[本地模型](/gateway/local-models)。

经验法则：对高风险工作使用**你能负担得起的最佳模型**，对常规聊天或摘要使用更便宜的模型。你可以按 Agent 路由模型并使用子 Agent 并行化长时间任务（每个子 Agent 消耗令牌）。参见[模型](/concepts/models)和
[子 Agent](/tools/subagents)。

强烈警告：较弱/过度量化的模型更容易受到提示注入和不安全行为的影响。参见[安全](/gateway/security)。

更多上下文：[模型](/concepts/models)。

### 我可以使用自托管模型（llama.cpp、vLLM、Ollama）吗？

可以。如果你的本地服务器公开 OpenAI 兼容的 API，你可以将自定义提供商指向它。Ollama 直接受支持，是最简单的路径。

安全说明：较小或大量量化的模型更容易受到提示注入的影响。我们强烈建议任何可以使用工具的机器人使用**大模型**。如果你仍然想要小模型，请启用沙盒和严格的工具允许列表。

文档：[Ollama](/providers/ollama)、[本地模型](/gateway/local-models)、
[模型提供商](/concepts/model-providers)、[安全](/gateway/security)、
[沙盒](/gateway/sandboxing)。

### 如何在不擦除配置的情况下切换模型？

使用**模型命令**或仅编辑**模型**字段。避免完整配置替换。

安全选项：

- 聊天中的 `/model`（快速，每会话）
- `openclaw models set ...`（仅更新模型配置）
- `openclaw configure --section models`（交互式）
- 在 `~/.openclaw/openclaw.json` 中编辑 `agents.defaults.model`

除非你想替换整个配置，否则避免使用部分对象的 `config.apply`。
如果你确实覆盖了配置，请从备份恢复或重新运行 `openclaw doctor` 进行修复。

文档：[模型](/concepts/models)、[配置](/cli/configure)、[配置](/cli/config)、[Doctor](/gateway/doctor)。

### OpenClaw、Flawd 和 Krill 使用什么模型？

- **OpenClaw + Flawd：**Anthropic Opus（`anthropic/claude-opus-4-5`）- 参见 [Anthropic](/providers/anthropic)。
- **Krill：**MiniMax M2.1（`minimax/MiniMax-M2.1`）- 参见 [MiniMax](/providers/minimax)。

### 如何在飞行中切换模型（无需重启）？

使用 `/model` 命令作为独立消息：

```
/model sonnet
/model haiku
/model opus
/model gpt
/model gpt-mini
/model gemini
/model gemini-flash
```

你可以使用 `/model`、`/model list` 或 `/model status` 列出可用模型。

`/model`（和 `/model list`）显示紧凑的编号选择器。按编号选择：

```
/model 3
```

你还可以强制提供商使用特定的认证配置文件（每会话）：

```
/model opus@anthropic:default
/model opus@anthropic:work
```

提示：`/model status` 显示哪个 Agent 处于活动状态，哪个 `auth-profiles.json` 文件正在使用，以及接下来将尝试哪个认证配置文件。
它还在可用时显示配置的提供商端点（`baseUrl`）和 API 模式（`api`）。

**如何取消固定我用 @profile 设置的配置文件**

在没有 `@profile` 后缀的情况下重新运行 `/model`：

```
/model anthropic/claude-opus-4-5
```

如果你想返回默认值，请从 `/model` 中选择它（或发送 `/model <default provider/model>`）。
使用 `/model status` 确认哪个认证配置文件处于活动状态。

### 我可以日常使用 GPT 5.2，编码使用 Codex 5.2 吗？

可以。将一个设置为默认值，并根据需要切换：

- **快速切换（每会话）：**日常任务使用 `/model gpt-5.2`，编码使用 `/model gpt-5.2-codex`。
- **默认值 + 切换：**将 `agents.defaults.model.primary` 设置为 `openai-codex/gpt-5.2`，然后在编码时切换到 `openai-codex/gpt-5.2-codex`（或相反）。
- **子 Agent：**将编码任务路由到具有不同默认模型的子 Agent。

参见[模型](/concepts/models)和[斜杠命令](/tools/slash-commands)。

### 为什么我看到"不允许模型……"然后没有回复？

如果设置了 `agents.defaults.models`，它会变成 `/model` 和任何会话覆盖的**允许列表**。选择不在该列表中的模型会返回：

```
Model "provider/model" is not allowed. Use /model to list available models.
```

该错误**代替**正常回复返回。修复：将模型添加到 `agents.defaults.models`，删除允许列表，或从 `/model list` 中选择模型。

### 为什么我看到"未知模型：minimax/MiniMax-M2.1"？

这意味着**提供商未配置**（未找到 MiniMax 提供商配置或认证配置文件），因此无法解析模型。此检测的修复在 **2026.1.12** 中（撰写本文时未发布）。

修复检查清单：

1. 升级到 **2026.1.12**（或从源代码 `main` 运行），然后重启网关。
2. 确保 MiniMax 已配置（向导或 JSON），或者环境/认证配置文件中存在 MiniMax API 密钥，以便可以注入提供商。
3. 使用确切的模型 ID（区分大小写）：`minimax/MiniMax-M2.1` 或
   `minimax/MiniMax-M2.1-lightning`。
4. 运行：
   ```bash
   openclaw models list
   ```
   并从列表中选择（或聊天中的 `/model list`）。

参见 [MiniMax](/providers/minimax) 和[模型](/concepts/models)。

### 我可以将 MiniMax 作为默认值，将 OpenAI 用于复杂任务吗？

可以。使用 **MiniMax 作为默认值**，并在需要时**每会话**切换模型。
故障转移用于**错误**，而不是"困难任务"，因此请使用 `/model` 或单独的 Agent。

**选项 A：每会话切换**

```json5
{
  env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "minimax/MiniMax-M2.1" },
      models: {
        "minimax/MiniMax-M2.1": { alias: "minimax" },
        "openai/gpt-5.2": { alias: "gpt" },
      },
    },
  },
}
```

然后：

```
/model gpt
```

**选项 B：单独的 Agent**

- Agent A 默认：MiniMax
- Agent B 默认：OpenAI
- 按 Agent 路由或使用 `/agent` 切换

文档：[模型](/concepts/models)、[多 Agent 路由](/concepts/multi-agent)、[MiniMax](/providers/minimax)、[OpenAI](/providers/openai)。

### opus / sonnet / gpt 是内置快捷方式吗？

是。OpenClaw 附带一些默认简写（仅在模型存在于 `agents.defaults.models` 中时应用）：

- `opus` → `anthropic/claude-opus-4-5`
- `sonnet` → `anthropic/claude-sonnet-4-5`
- `gpt` → `openai/gpt-5.2`
- `gpt-mini` → `openai/gpt-5-mini`
- `gemini` → `google/gemini-3-pro-preview`
- `gemini-flash` → `google/gemini-3-flash-preview`

如果你用自己的别名设置相同的名称，你的值获胜。

### 如何定义/覆盖模型快捷方式（别名）？

别名来自 `agents.defaults.models.<modelId>.alias`。示例：

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-5" },
      models: {
        "anthropic/claude-opus-4-5": { alias: "opus" },
        "anthropic/claude-sonnet-4-5": { alias: "sonnet" },
        "anthropic/claude-haiku-4-5": { alias: "haiku" },
      },
    },
  },
}
```

然后 `/model sonnet`（或支持时的 `/<alias>`）解析为该模型 ID。

### 如何从 OpenRouter 或 Z.AI 等其他提供商添加模型？

OpenRouter（按令牌付费；许多模型）：

```json5
{
  agents: {
    defaults: {
      model: { primary: "openrouter/anthropic/claude-sonnet-4-5" },
      models: { "openrouter/anthropic/claude-sonnet-4-5": {} },
    },
  },
  env: { OPENROUTER_API_KEY: "sk-or-..." },
}
```

Z.AI（GLM 模型）：

```json5
{
  agents: {
    defaults: {
      model: { primary: "zai/glm-4.7" },
      models: { "zai/glm-4.7": {} },
    },
  },
  env: { ZAI_API_KEY: "..." },
}
```

如果你引用提供商/模型但缺少所需的提供商密钥，你会收到运行时认证错误（例如 `No API key found for provider "zai"`）。

**添加新 Agent 后找不到提供商的 API 密钥**

这通常意味着**新 Agent** 有一个空的认证存储。认证是每 Agent 的，存储在：

```
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

修复选项：

- 运行 `openclaw agents add <id>` 并在向导期间配置认证。
- 或将 `auth-profiles.json` 从主 Agent 的 `agentDir` 复制到新 Agent 的 `agentDir`。

**不要**在 Agent 之间重用 `agentDir`；它会导致认证/会话冲突。

---

## 模型故障转移和"所有模型都失败"

### 故障转移如何工作？

故障转移分两个阶段进行：

1. 同一提供商内的**认证配置文件轮换**。
2. 对 `agents.defaults.model.fallbacks` 中的下一个模型进行**模型故障转移**。

冷却期适用于失败的配置文件（指数退避），因此即使提供商速率受限或暂时失败，OpenClaw 也可以继续响应。

### 这个错误是什么意思？

```
No credentials found for profile "anthropic:default"
```

这意味着系统尝试使用认证配置文件 ID `anthropic:default`，但在预期的认证存储中找不到它的凭证。

### "找不到 anthropic:default 配置文件的凭证"的修复检查清单

- **确认认证配置文件所在位置**（新路径与旧路径）
  - 当前：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
  - 旧版：`~/.openclaw/agent/*`（由 `openclaw doctor` 迁移）
- **确认你的环境变量已被网关加载**
  - 如果你在 shell 中设置了 `ANTHROPIC_API_KEY` 但通过 systemd/launchd 运行网关，它可能不会继承它。将其放在 `~/.openclaw/.env` 中或启用 `env.shellEnv`。
- **确保你正在编辑正确的 Agent**
  - 多 Agent 设置意味着可能有多个 `auth-profiles.json` 文件。
- **健全检查模型/认证状态**
  - 使用 `openclaw models status` 查看配置的模型以及提供商是否已认证。

**"找不到 anthropic 配置文件的凭证"的修复检查清单**

这意味着运行固定到 Anthropic 认证配置文件，但网关在其认证存储中找不到它。

- **使用 setup-token**
  - 运行 `claude setup-token`，然后使用 `openclaw models auth setup-token --provider anthropic` 粘贴它。
  - 如果令牌是在另一台机器上创建的，请使用 `openclaw models auth paste-token --provider anthropic`。
- **如果你想改用 API 密钥**
  - 将 `ANTHROPIC_API_KEY` 放在**网关主机**上的 `~/.openclaw/.env` 中。
  - 清除强制缺失配置文件的任何固定顺序：
    ```bash
    openclaw models auth order clear --provider anthropic
    ```
- **确认你正在网关主机上运行命令**
  - 在远程模式下，认证配置文件位于网关机器上，而不是你的笔记本电脑上。

### 为什么它也尝试了 Google Gemini 并失败了？

如果你的模型配置包括 Google Gemini 作为备用（或你切换到了 Gemini 简写），OpenClaw 会在模型故障转移期间尝试它。如果你尚未配置 Google 凭证，你会看到 `No API key found for provider "google"`。

修复：提供 Google 认证，或从 `agents.defaults.model.fallbacks` / 别名中删除/避免使用 Google 模型，以便故障转移不会路由到那里。

**LLM 请求被拒绝消息思考需要签名 google antigravity**

原因：会话历史包含**没有签名的思考块**（通常来自中止/部分流）。Google Antigravity 需要思考块的签名。

修复：OpenClaw 现在为 Google Antigravity Claude 剥离未签名的思考块。如果它仍然出现，请开始**新会话**或为该 Agent 设置 `/thinking off`。

---

## 认证配置文件：它们是什么以及如何管理它们

相关：[/concepts/oauth](/concepts/oauth)（OAuth 流程、令牌存储、多帐户模式）

### 什么是认证配置文件？

认证配置文件是绑定到提供商的命名凭证记录（OAuth 或 API 密钥）。配置文件位于：

```
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

### 典型的配置文件 ID 是什么？

OpenClaw 使用提供商前缀的 ID，如：

- `anthropic:default`（当没有电子邮件身份存在时常见）
- `anthropic:<email>` 用于 OAuth 身份
- 你选择的自定义 ID（例如 `anthropic:work`）

### 我可以控制首先尝试哪个认证配置文件吗？

可以。配置支持配置文件的可选元数据和每提供商的顺序（`auth.order.<provider>`）。这**不**存储机密；它将 ID 映射到提供商/模式并设置轮换顺序。

如果配置文件处于短暂的**冷却期**（速率限制/超时/认证失败）或更长的**禁用**状态（计费/信用额度不足），OpenClaw 可能会暂时跳过它。要检查这一点，请运行 `openclaw models status --json` 并检查 `auth.unusableProfiles`。调整：`auth.cooldowns.billingBackoffHours*`。

你还可以通过 CLI 设置**每 Agent**顺序覆盖（存储在该 Agent 的 `auth-profiles.json` 中）：

```bash
# 默认为配置的默认 Agent（省略 --agent）
openclaw models auth order get --provider anthropic

# 锁定到单个配置文件（只尝试这个）
openclaw models auth order set --provider anthropic anthropic:default

# 或设置显式顺序（提供商内故障转移）
openclaw models auth order set --provider anthropic anthropic:work anthropic:default

# 清除覆盖（回退到配置 auth.order / 轮询）
openclaw models auth order clear --provider anthropic
```

要针对特定 Agent：

```bash
openclaw models auth order set --provider anthropic --agent main anthropic:default
```

### OAuth 与 API 密钥：有什么区别？

OpenClaw 支持两者：

- **OAuth** 通常利用订阅访问（如果适用）。
- **API 密钥**使用按令牌付费计费。

向导明确支持 Anthropic setup-token 和 OpenAI Codex OAuth，并可以为你存储 API 密钥。

---

## 网关：端口、"已在运行"和远程模式

### 网关使用什么端口？

`gateway.port` 控制 WebSocket + HTTP（控制 UI、钩子等）的单个多路复用端口。

优先级：

```
--port > OPENCLAW_GATEWAY_PORT > gateway.port > 默认 18789
```

### 为什么 `openclaw gateway status` 显示 `Runtime: running` 但 `RPC probe: failed`？

因为"running"是**监督程序**的视图（launchd/systemd/schtasks）。RPC 探测是 CLI 实际连接到网关 WebSocket 并调用 `status`。

使用 `openclaw gateway status` 并信任这些行：

- `Probe target:`（探测实际使用的 URL）
- `Listening:`（端口上实际绑定的内容）
- `Last gateway error:`（进程处于活动状态但端口未监听时的常见根本原因）

### 为什么 `openclaw gateway status` 显示 `Config (cli)` 和 `Config (service)` 不同？

你正在编辑一个配置文件，而服务正在运行另一个（通常是 `--profile` / `OPENCLAW_STATE_DIR` 不匹配）。

修复：

```bash
openclaw gateway install --force
```

从你想要服务使用的相同 `--profile` / 环境运行它。

### "另一个网关实例已在监听"是什么意思？

OpenClaw 通过在启动时立即绑定 WebSocket 监听器（默认 `ws://127.0.0.1:18789`）来强制执行运行时锁定。如果绑定失败并显示 `EADDRINUSE`，它会抛出 `GatewayLockError`，指示另一个实例已在监听。

修复：停止另一个实例，释放端口，或使用 `openclaw gateway --port <port>` 运行。

### 如何在远程模式下运行 OpenClaw（客户端连接到别处的网关）？

设置 `gateway.mode: "remote"` 并指向远程 WebSocket URL，可选择使用令牌/密码：

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://gateway.tailnet:18789",
      token: "your-token",
      password: "your-password",
    },
  },
}
```

注意：

- `openclaw gateway` 仅在 `gateway.mode` 为 `local` 时启动（或你传递覆盖标志）。
- macOS 应用程序监视配置文件，并在这些值更改时实时切换模式。

### 控制 UI 显示"未授权"（或保持重新连接）。现在怎么办？

你的网关正在启用认证（`gateway.auth.*`）的情况下运行，但 UI 没有发送匹配的令牌/密码。

事实（来自代码）：

- 控制 UI 将令牌存储在浏览器 localStorage 键 `openclaw.control.settings.v1` 中。
- UI 可以一次性导入 `?token=...`（和/或 `?password=...`），然后将其从 URL 中剥离。

修复：

- 最快：`openclaw dashboard`（打印 + 复制令牌链接，尝试打开；如果无头则显示 SSH 提示）。
- 如果你还没有令牌：`openclaw doctor --generate-gateway-token`。
- 如果是远程，请先隧道：`ssh -N -L 18789:127.0.0.1:18789 user@host` 然后打开 `http://127.0.0.1:18789/?token=...`。
- 在网关主机上设置 `gateway.auth.token`（或 `OPENCLAW_GATEWAY_TOKEN`）。
- 在控制 UI 设置中，粘贴相同的令牌（或使用一次性 `?token=...` 链接刷新）。
- 仍然卡住？运行 `openclaw status --all` 并按照[故障排除](/gateway/troubleshooting)操作。参见[仪表板](/web/dashboard)了解认证详情。

### 我设置了 `gateway.bind: "tailnet"`，但它无法绑定/什么都没有监听

`tailnet` 绑定从你的网络接口（100.64.0.0/10）中选择一个 Tailscale IP。如果机器不在 Tailscale 上（或接口已关闭），则没有可绑定的内容。

修复：

- 在该主机上启动 Tailscale（这样它就有 100.x 地址），或
- 切换到 `gateway.bind: "loopback"` / `"lan"`。

注意：`tailnet` 是显式的。`auto` 首选环回；当你想要仅限 tailnet 的绑定时使用 `gateway.bind: "tailnet"`。

### 我可以在同一主机上运行多个网关吗？

通常不行——一个网关可以运行多个消息频道和 Agent。只有当你需要冗余（例如：救援机器人）或硬隔离时，才使用多个网关。

可以，但你必须隔离：

- `OPENCLAW_CONFIG_PATH`（每实例配置）
- `OPENCLAW_STATE_DIR`（每实例状态）
- `agents.defaults.workspace`（工作区隔离）
- `gateway.port`（唯一端口）

快速设置（推荐）：

- 对每个实例使用 `openclaw --profile <name> …`（自动创建 `~/.openclaw-<name>`）。
- 在每个配置文件配置中设置唯一的 `gateway.port`（或手动运行传递 `--port`）。
- 安装每配置文件服务：`openclaw --profile <name> gateway install`。

配置文件还会后缀服务名称（`bot.molt.<profile>`；旧版 `com.openclaw.*`、`openclaw-gateway-<profile>.service`、`OpenClaw Gateway (<profile>)`）。
完整指南：[多个网关](/gateway/multiple-gateways)。

### "无效握手" / 代码 1008 是什么意思？

网关是一个 **WebSocket 服务器**，它期望第一条消息是 `connect` 帧。如果它收到其他内容，它会以**代码 1008**（策略违规）关闭连接。

常见原因：

- 你在浏览器中打开了 **HTTP** URL（`http://...`）而不是 WS 客户端。
- 你使用了错误的端口或路径。
- 代理或隧道剥离了认证标头或发送了非网关请求。

快速修复：

1. 使用 WS URL：`ws://<host>:18789`（或 `wss://...` 如果 HTTPS）。
2. 不要在普通浏览器标签页中打开 WS 端口。
3. 如果认证已打开，请在 `connect` 帧中包含令牌/密码。

如果你使用 CLI 或 TUI，URL 应该看起来像这样：

```
openclaw tui --url ws://<host>:18789 --token <token>
```

协议详情：[网关协议](/gateway/protocol)。

---

## 日志和调试

### 日志在哪里？

文件日志（结构化）：

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

你可以通过 `logging.file` 设置稳定的路径。文件日志级别由 `logging.level` 控制。控制台详细程度由 `--verbose` 和 `logging.consoleLevel` 控制。

最快的日志跟踪：

```bash
openclaw logs --follow
```

服务/监督程序日志（当网关通过 launchd/systemd 运行时）：

- macOS：`$OPENCLAW_STATE_DIR/logs/gateway.log` 和 `gateway.err.log`（默认：`~/.openclaw/logs/...`；配置文件使用 `~/.openclaw-<profile>/logs/...`）
- Linux：`journalctl --user -u openclaw-gateway[-<profile>].service -n 200 --no-pager`
- Windows：`schtasks /Query /TN "OpenClaw Gateway (<profile>)" /V /FO LIST`

参见[故障排除](/gateway/troubleshooting#log-locations)了解更多。

### 如何启动/停止/重启网关服务？

使用网关助手：

```bash
openclaw gateway status
openclaw gateway restart
```

如果你手动运行网关，`openclaw gateway --force` 可以回收端口。参见[网关](/gateway)。

### 我在 Windows 上关闭了终端——如何重启 OpenClaw？

有**两种 Windows 安装模式**：

**1) WSL2（推荐）：**网关在 Linux 内部运行。

打开 PowerShell，进入 WSL，然后重启：

```powershell
wsl
openclaw gateway status
openclaw gateway restart
```

如果你从未安装过服务，请在前台启动它：

```bash
openclaw gateway run
```

**2) 原生 Windows（不推荐）：**网关直接在 Windows 中运行。

打开 PowerShell 并运行：

```powershell
openclaw gateway status
openclaw gateway restart
```

如果你手动运行它（无服务），请使用：

```powershell
openclaw gateway run
```

文档：[Windows (WSL2)](/platforms/windows)、[网关服务运行手册](/gateway)。

### 网关已启动，但回复永远不会到达。我应该检查什么？

从快速健康扫描开始：

```bash
openclaw status
openclaw models status
openclaw channels status
openclaw logs --follow
```

常见原因：

- **网关主机**上未加载模型认证（检查 `models status`）。
- 频道配对/允许列表阻止回复（检查频道配置 + 日志）。
- WebChat/仪表板没有正确的令牌就打开了。

如果你是远程的，请确认隧道/Tailscale 连接已启动，并且网关 WebSocket 可达。

文档：[频道](/channels)、[故障排除](/gateway/troubleshooting)、[远程访问](/gateway/remote)。

### "与网关断开连接：无原因"——现在怎么办？

这通常意味着 UI 失去了 WebSocket 连接。检查：

1. 网关正在运行吗？`openclaw gateway status`
2. 网关健康吗？`openclaw status`
3. UI 有正确的令牌吗？`openclaw dashboard`
4. 如果是远程的，隧道/Tailscale 链接是否已启动？

然后跟踪日志：

```bash
openclaw logs --follow
```

文档：[仪表板](/web/dashboard)、[远程访问](/gateway/remote)、[故障排除](/gateway/troubleshooting)。

### Telegram setMyCommands 失败并显示网络错误。我应该检查什么？

从日志和频道状态开始：

```bash
openclaw channels status
openclaw channels logs --channel telegram
```

如果你在 VPS 上或代理后面，请确认允许出站 HTTPS 并且 DNS 正常工作。
如果网关是远程的，请确保你正在查看网关主机上的日志。

文档：[Telegram](/channels/telegram)、[频道故障排除](/channels/troubleshooting)。

### TUI 没有显示输出。我应该检查什么？

首先确认网关可达且 Agent 可以运行：

```bash
openclaw status
openclaw models status
openclaw logs --follow
```

在 TUI 中，使用 `/status` 查看当前状态。如果你期望在聊天频道中收到回复，请确保交付已启用（`/deliver on`）。

文档：[TUI](/tui)、[斜杠命令](/tools/slash-commands)。

### 如何完全停止然后启动网关？

如果你安装了服务：

```bash
openclaw gateway stop
openclaw gateway start
```

这会停止/启动**监督服务**（macOS 上的 launchd，Linux 上的 systemd）。当网关作为守护程序在后台运行时，请使用它。

如果你在前台运行，请使用 Ctrl-C 停止，然后：

```bash
openclaw gateway run
```

文档：[网关服务运行手册](/gateway)。

### ELI5：`openclaw gateway restart` 与 `openclaw gateway`

- `openclaw gateway restart`：重启**后台服务**（launchd/systemd）。
- `openclaw gateway`：在此终端会话的**前台**运行网关。

如果你安装了服务，请使用网关命令。当你想要一次性前台运行时，请使用 `openclaw gateway`。

### 当某些东西失败时，获得详细信息的快速方法是什么？

使用 `--verbose` 启动网关以获得更多控制台详细信息。然后检查日志文件中的频道认证、模型路由和 RPC 错误。

---

## 媒体和附件

### 我的技能生成了图像/PDF，但没有发送任何内容

来自 Agent 的出站附件必须包含 `MEDIA:<path-or-url>` 行（在其自己的行上）。参见 [OpenClaw 助手设置](/start/openclaw)和[Agent 发送](/tools/agent-send)。

CLI 发送：

```bash
openclaw message send --target +15555550123 --message "给你" --media /path/to/file.png
```

还要检查：

- 目标频道支持出站媒体且未被允许列表阻止。
- 文件在提供商的大小限制内（图像调整为最大 2048px）。

参见[图像](/nodes/images)。

---

## 安全和访问控制

### 将 OpenClaw 暴露给入站私信安全吗？

将入站私信视为不受信任的输入。默认值旨在降低风险：

- 私信功能频道上的默认行为是**配对**：
  - 未知发送者收到配对代码；机器人不处理他们的消息。
  - 批准方式：`openclaw pairing approve <channel> <code>`
  - 待处理请求上限为每频道 **3 个**；检查 `openclaw pairing list <channel>` 如果代码没有到达。
- 公开打开私信需要明确选择加入（`dmPolicy: "open"` 和允许列表 `"*"`）。

运行 `openclaw doctor` 以显示有风险的私信政策。

### 提示注入只是公共机器人的问题吗？

不是。提示注入是关于**不受信任的内容**，而不仅仅是谁可以向机器人发送私信。
如果你的助手读取外部内容（网络搜索/获取、浏览器页面、电子邮件、文档、附件、粘贴的日志），该内容可能包含试图劫持模型的指令。即使**你是唯一的发送者**，这也可能发生。

最大的风险是当工具启用时：模型可能被诱骗外泄上下文或代表你调用工具。通过以下方式减少爆炸半径：

- 使用只读或禁用工具的"阅读器" Agent 来总结不受信任的内容
- 对启用工具的 Agent 保持 `web_search` / `web_fetch` / `browser` 关闭
- 沙盒和严格的工具允许列表

详情：[安全](/gateway/security)。

### 我的机器人应该有自己的电子邮件 GitHub 帐户或电话号码吗？

对于大多数设置来说，是的。使用单独的帐户和电话号码隔离机器人可以减少出现问题时的爆炸半径。这也使得在不影​​响你个人帐户的情况下轮换凭证或撤销访问权限更容易。

从小处开始。只提供你实际需要的工具和帐户的访问权限，如果需要，以后再扩展。

文档：[安全](/gateway/security)、[配对](/start/pairing)。

### 我可以赋予它对我的短信的自主权吗？这样做安全吗？

我们**不推荐**对你的个人消息完全自主。最安全的模式是：

- 将私信保持在**配对模式**或严格的允许列表中。
- 如果你想让它代表你发送消息，请使用**单独的号码或帐户**。
- 让它起草，然后**在发送前批准**。

如果你想实验，请在专用帐户上进行并保持隔离。参见
[安全](/gateway/security)。

### 我可以将更便宜的模型用于个人助理任务吗？

可以，**如果** Agent 仅聊天且输入受信任。较小的层级更容易受到指令劫持，因此避免将它们用于启用工具的 Agent 或读取不受信任的内容时。如果你必须使用较小的模型，请锁定工具并在沙盒内运行。参见[安全](/gateway/security)。

### 我在 Telegram 中运行了 /start，但没有收到配对代码

仅当未知发送者向机器人发送消息且启用 `dmPolicy: "pairing"` 时才发送配对代码。`/start` 本身不会生成代码。

检查待处理请求：

```bash
openclaw pairing list telegram
```

如果你想要立即访问，请允许列表你的发送者 ID 或为该帐户设置 `dmPolicy: "open"`。

### WhatsApp：它会向我的联系人发送消息吗？配对如何工作？

不会。默认 WhatsApp 私信政策是**配对**。未知发送者只收到配对代码，他们的消息**不会被处理**。OpenClaw 只回复它收到的聊天或你触发的显式发送。

批准配对：

```bash
openclaw pairing approve whatsapp <code>
```

列出待处理请求：

```bash
openclaw pairing list whatsapp
```

向导电话号码提示：它用于设置你的**允许列表/所有者**，以便你自己的私信被允许。它不用于自动发送。如果你在自己的 WhatsApp 号码上运行，请使用该号码并启用 `channels.whatsapp.selfChatMode`。

---

## 聊天命令、中止任务和"它不会停止"

### 如何阻止内部系统消息显示在聊天中？

大多数内部或工具消息仅当该会话启用**详细**或**推理**时才出现。

在你看到它的聊天中修复：

```
/verbose off
/reasoning off
```

如果它仍然很吵，请在控制 UI 中检查会话设置并将详细程度设置为**继承**。还要确认你没有使用在配置中将 `verboseDefault` 设置为 `on` 的机器人配置文件。

文档：[思考和详细](/tools/thinking)、[安全](/gateway/security#reasoning--verbose-output-in-groups)。

### 如何停止/取消正在运行的任务？

将以下任何内容作为**独立消息**发送（无斜杠）：

```
stop
abort
esc
wait
exit
interrupt
```

这些是中止触发器（不是斜杠命令）。

对于后台进程（来自 exec 工具），你可以要求 Agent 运行：

```
process action:kill sessionId:XXX
```

斜杠命令概述：参见[斜杠命令](/tools/slash-commands)。

大多数命令必须作为以 `/` 开头的**独立**消息发送，但一些快捷方式（如 `/status`）也对允许列表发送者内联工作。

### 如何从 Telegram 发送 Discord 消息？（"跨上下文消息传递被拒绝"）

OpenClaw 默认阻止**跨提供商**消息传递。如果工具调用绑定到 Telegram，它不会发送到 Discord，除非你明确允许它。

为 Agent 启用跨提供商消息传递：

```json5
{
  agents: {
    defaults: {
      tools: {
        message: {
          crossContext: {
            allowAcrossProviders: true,
            marker: { enabled: true, prefix: "[from {channel}] " },
          },
        },
      },
    },
  },
}
```

编辑配置后重启网关。如果你只想对单个 Agent 启用此功能，请改为在 `agents.list[].tools.message` 下设置。

### 为什么感觉机器人"忽略"了快速连续的消息？

队列模式控制新消息如何与正在运行的运行交互。使用 `/queue` 更改模式：

- `steer` - 新消息重定向当前任务
- `followup` - 一次运行一条消息
- `collect` - 批量消息并回复一次（默认）
- `steer-backlog` - 现在转向，然后处理积压
- `interrupt` - 中止当前运行并重新开始

你可以为跟进模式添加 `debounce:2s cap:25 drop:summarize` 等选项。

---

## 回答截图/聊天记录中的确切问题

**问：**"Anthropic 使用 API 密钥的默认模型是什么？"

**答：**在 OpenClaw 中，凭证和模型选择是分开的。设置 `ANTHROPIC_API_KEY`（或在认证配置文件中存储 Anthropic API 密钥）启用认证，但实际默认模型是你在 `agents.defaults.model.primary` 中配置的任何内容（例如，`anthropic/claude-sonnet-4-5` 或 `anthropic/claude-opus-4-5`）。如果你看到 `No credentials found for profile "anthropic:default"`，这意味着网关无法在正在运行的 Agent 的预期 `auth-profiles.json` 中找到 Anthropic 凭证。

---

仍然卡住？在 [Discord](https://discord.com/invite/clawd) 上询问或打开 [GitHub 讨论](https://github.com/openclaw/openclaw/discussions)。

---
summary: "新手指南：从零到第一条消息（向导、认证、渠道、配对）"
read_when:
  - 第一次从零开始配置
  - 你想走最快路径：安装 → 引导 → 第一条消息
title: "入门（Getting Started）"
---

# 入门（Getting Started）

目标：尽可能快地从 **零** → **第一条可用聊天**（合理默认配置）。

最快的聊天方式：打开 Control UI（无需配置渠道）。运行 `openclaw dashboard`
并在浏览器中聊天，或在网关主机上打开 `http://127.0.0.1:18789/`。
文档：[Dashboard](/web/dashboard) 与 [Control UI](/web/control-ui)。

推荐路径：使用 **CLI 引导向导**（`openclaw onboard`）。它会设置：

- 模型/认证（推荐 OAuth）
- 网关设置
- 渠道（WhatsApp/Telegram/Discord/Mattermost（插件）/…）
- 配对默认值（安全 DM）
- 工作区引导 + 技能
- 可选后台服务

如果你需要更深入的参考页，请跳转到：[Wizard](/start/wizard)、[Setup](/start/setup)、[Pairing](/start/pairing)、[Security](/gateway/security)。

沙箱提示：`agents.defaults.sandbox.mode: "non-main"` 使用 `session.mainKey`（默认值为 `"main"`），
因此群组/渠道会话会进入沙箱。如果你希望主代理始终在宿主机上运行，
请为单个代理设置显式覆盖：

```json
{
  "routing": {
    "agents": {
      "main": {
        "workspace": "~/.openclaw/workspace",
        "sandbox": { "mode": "off" }
      }
    }
  }
}
```

## 0) 前置条件（Prereqs）

- Node `>=22`
- `pnpm`（可选；从源码构建推荐）
- **推荐：** 用于网页搜索的 Brave Search API key。最简单路径：
  `openclaw configure --section web`（保存到 `tools.web.search.apiKey`）。
  参见 [Web 工具](/tools/web)。

macOS：如果你打算构建应用，请安装 Xcode / CLT。仅使用 CLI + 网关时，Node 即可。
Windows：使用 **WSL2**（推荐 Ubuntu）。强烈建议使用 WSL2；原生 Windows 未充分测试，问题更多且工具兼容性较差。先安装 WSL2，然后在 WSL 中执行 Linux 步骤。参见 [Windows (WSL2)](/platforms/windows)。

## 1) 安装 CLI（推荐）

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

安装器选项（安装方式、非交互、从 GitHub 安装）：[Install](/install)。

Windows（PowerShell）：

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

替代方式（全局安装）：

```bash
npm install -g openclaw@latest
```

```bash
pnpm add -g openclaw@latest
```

## 2) 运行引导向导（并安装服务）（Run the onboarding wizard）

```bash
openclaw onboard --install-daemon
```

你会选择：

- **本地 vs 远程** 网关
- **认证（Auth）**：OpenAI Code（Codex）订阅（OAuth）或 API key。Anthropic 建议使用 API key；也支持 `claude setup-token`。
- **提供方（Providers）**：WhatsApp 二维码登录、Telegram/Discord 机器人令牌、Mattermost 插件令牌等。
- **守护进程（Daemon）**：后台安装（launchd/systemd；WSL2 使用 systemd）
  - **运行时（Runtime）**：Node（推荐；WhatsApp/Telegram 必需）。不推荐使用 Bun。
- **网关令牌（Gateway token）**：向导默认生成（即使是回环地址），并存储到 `gateway.auth.token`。

向导文档：[Wizard](/start/wizard)

### 认证存放位置（Auth: where it lives, important）

- **推荐 Anthropic 路径：** 设置 API key（向导可为服务保存）。如需复用 Claude Code 凭据，也支持 `claude setup-token`。

- OAuth 凭据（旧版导入）：`~/.openclaw/credentials/oauth.json`
- 认证配置（OAuth + API key）：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`

无头/服务器提示：先在常规机器上完成 OAuth，再把 `oauth.json` 拷贝到网关主机。

## 3) 启动网关（Start the Gateway）

如果你在引导过程中安装了服务，网关应已在运行：

```bash
openclaw gateway status
```

手动运行（前台）：

```bash
openclaw gateway --port 18789 --verbose
```

Dashboard（本地回环）：`http://127.0.0.1:18789/`
如果已配置令牌，请在 Control UI 设置中粘贴它（存储在 `connect.params.auth.token`）。

⚠️ **Bun 警告（WhatsApp + Telegram）：** Bun 在这些渠道上存在已知问题。
如果你使用 WhatsApp 或 Telegram，请用 **Node** 运行网关。

## 3.5) 快速验证（2 分钟 / Quick verify）

```bash
openclaw status
openclaw health
openclaw security audit --deep
```

## 4) 配对并连接你的第一个聊天入口（Pair + connect your first chat surface）

### WhatsApp（QR 登录）

```bash
openclaw channels login
```

在 WhatsApp → 设置 → 已连接设备 中扫码。

WhatsApp 文档：[WhatsApp](/channels/whatsapp)

### Telegram / Discord / 其他

向导可以替你写入令牌/配置。如果你更喜欢手动配置，请从以下开始：

- Telegram：[Telegram](/channels/telegram)
- Discord：[Discord](/channels/discord)
- Mattermost（插件）：[Mattermost](/channels/mattermost)

**Telegram DM 提示：** 你的首条私信会返回配对码。请批准它（见下一步），否则机器人不会响应。

## 5) DM 安全（配对审批 / pairing approvals）

默认姿态：未知 DM 会收到短码，消息在被批准前不会被处理。
如果你的第一条 DM 没有回复，请批准配对：

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <code>
```

配对文档：[Pairing](/start/pairing)

## 从源码运行（开发 / From source）

如果你在开发 OpenClaw 本身，可以从源码运行：

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build # auto-installs UI deps on first run
pnpm build
openclaw onboard --install-daemon
```

如果你尚未进行全局安装，可在仓库中使用 `pnpm openclaw ...` 运行引导步骤。
`pnpm build` 也会打包 A2UI 资源；如果只需要该步骤，请使用 `pnpm canvas:a2ui:bundle`。

网关（从本仓库运行）：

```bash
node openclaw.mjs gateway --port 18789 --verbose
```

## 7) 端到端验证（Verify end-to-end）

在新的终端中发送一条测试消息：

```bash
openclaw message send --target +15555550123 --message "Hello from OpenClaw"
```

如果 `openclaw health` 显示 “no auth configured”，请回到向导设置 OAuth/Key 认证——否则代理无法响应。

提示：`openclaw status --all` 是最佳可粘贴的只读调试报告。

## 后续步骤（可选，但推荐）

- macOS 菜单栏应用 + 语音唤醒：[macOS 应用](/platforms/macos)
- iOS/Android 节点（Canvas/相机/语音）：[节点](/nodes)
- 远程访问（SSH 隧道 / Tailscale Serve）：[远程访问](/gateway/remote) 与 [Tailscale](/gateway/tailscale)
- 始终在线 / VPN 配置：[远程访问](/gateway/remote)、[exe.dev](/platforms/exe-dev)、[Hetzner](/platforms/hetzner)、[macOS 远程](/platforms/mac/remote)

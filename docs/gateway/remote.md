---
summary: "使用 SSH 隧道（Gateway WS）和 tailnet 进行远程访问"
read_when:
  - 运行或故障排除远程 gateway 设置
title: "远程访问"
---

# 远程访问（SSH、隧道和 tailnet）

本仓库通过保持单个 Gateway（主节点）在专用主机（桌面/服务器）上运行并连接客户端来支持"通过 SSH 远程"。

- 对于**操作员（您 / macOS 应用）**：SSH 隧道是通用的回退方案。
- 对于**节点（iOS/Android 和未来设备）**：连接到 Gateway **WebSocket**（根据需要 LAN/tailnet 或 SSH 隧道）。

## 核心思想

- Gateway WebSocket 绑定到您配置的端口上的 **loopback**（默认为 18789）。
- 对于远程使用，您通过 SSH 转发该 loopback 端口（或使用 tailnet/VPN 减少隧道）。

## 常见 VPN/tailnet 设置（agent 所在位置）

将 **Gateway 主机**视为"agent 所在位置"。它拥有会话、认证配置文件、通道和状态。
您的笔记本电脑/桌面（和节点）连接到该主机。

### 1) 在您的 tailnet 中始终在线的 Gateway（VPS 或家庭服务器）

在持久主机上运行 Gateway 并通过 **Tailscale** 或 SSH 访问它。

- **最佳 UX：**保持 `gateway.bind: "loopback"` 并使用 **Tailscale Serve** 作为控制 UI。
- **回退：**保持 loopback + 从任何需要访问的机器进行 SSH 隧道。
- **示例：**[exe.dev](/platforms/exe-dev)（简单 VM）或 [Hetzner](/platforms/hetzner)（生产 VPS）。

当您的笔记本电脑经常休眠但您希望 agent 始终在线时，这是理想的选择。

### 2) 家庭桌面运行 Gateway，笔记本电脑是远程控制

笔记本电脑**不**运行 agent。它远程连接：

- 使用 macOS 应用的**通过 SSH 远程**模式（设置 → 通用 → "OpenClaw 运行位置"）。
- 应用打开并管理隧道，因此 WebChat + 健康检查"正常工作"。

操作手册：[macOS 远程访问](/platforms/mac/remote)。

### 3) 笔记本电脑运行 Gateway，从其他机器远程访问

保持 Gateway 本地但安全地暴露它：

- 从其他机器到笔记本电脑的 SSH 隧道，或
- Tailscale Serve 控制 UI 并保持 Gateway 仅 loopback。

指南：[Tailscale](/gateway/tailscale) 和 [Web 概述](/web)。

## 命令流（在哪里运行什么）

一个 gateway 服务拥有状态 + 通道。节点是外围设备。

流程示例（Telegram → 节点）：

- Telegram 消息到达 **Gateway**。
- Gateway 运行 **agent** 并决定是否调用节点工具。
- Gateway 通过 Gateway WebSocket 调用 **节点**（`node.*` RPC）。
- 节点返回结果；Gateway 回复给 Telegram。

注意：

- **节点不运行 gateway 服务。**每个主机只应运行一个 gateway，除非您有意运行隔离的配置文件（参见[多 Gateway](/gateway/multiple-gateways)）。
- macOS 应用"节点模式"只是通过 Gateway WebSocket 的节点客户端。

## SSH 隧道（CLI + 工具）

创建到远程 Gateway WS 的本地隧道：

```bash
ssh -N -L 18789:127.0.0.1:18789 user@host
```

隧道建立后：

- `openclaw health` 和 `openclaw status --deep` 现在通过 `ws://127.0.0.1:18789` 到达远程 gateway。
- `openclaw gateway {status,health,send,agent,call}` 也可以在需要时通过 `--url` 定位转发的 URL。

注意：将 `18789` 替换为您配置的 `gateway.port`（或 `--port`/`OPENCLAW_GATEWAY_PORT`）。

## CLI 远程默认值

您可以持久化远程目标，以便 CLI 命令默认使用它：

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token",
    },
  },
}
```

当 gateway 仅 loopback 时，保持 URL 为 `ws://127.0.0.1:18789` 并首先打开 SSH 隧道。

## 通过 SSH 的 Chat UI

WebChat 不再使用单独的 HTTP 端口。SwiftUI 聊天 UI 直接连接到 Gateway WebSocket。

- 通过 SSH 转发 `18789`（见上文），然后连接客户端到 `ws://127.0.0.1:18789`。
- 在 macOS 上，优先使用应用的"通过 SSH 远程"模式，它会自动管理隧道。

## macOS 应用"通过 SSH 远程"

macOS 菜单栏应用可以端到端驱动相同的设置（远程状态检查、WebChat 和语音唤醒转发）。

操作手册：[macOS 远程访问](/platforms/mac/remote)。

## 安全规则（远程/VPN）

简而言之：**保持 Gateway 仅 loopback**，除非您确定需要绑定。

- **Loopback + SSH/Tailscale Serve** 是最安全的默认设置（无公共暴露）。
- **非 loopback 绑定**（`lan`/`tailnet`/`custom`，或 loopback 不可用时 `auto`）必须使用认证令牌/密码。
- `gateway.remote.token` **仅**用于远程 CLI 调用——它**不**启用本地认证。
- 使用 `wss://` 时，`gateway.remote.tlsFingerprint` 固定远程 TLS 证书。
- 当 `gateway.auth.allowTailscale: true` 时，**Tailscale Serve** 可以通过身份头进行认证。
  如果您想要令牌/密码，请将其设置为 `false`。
- 将浏览器控制视为 operator 访问：仅限 tailnet + 谨慎节点配对。

深入了解：[安全](/gateway/security)。

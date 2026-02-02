---
summary: "Gateway 仪表板 (Control UI) 访问与认证"
read_when:
  - 更改仪表板认证或暴露模式
title: "仪表板"
---

# 仪表板 (Control UI)

Gateway 仪表板是默认在 `/` 路径提供的浏览器控制界面
（可通过 `gateway.controlUi.basePath` 覆盖）。

快速打开（本地 Gateway）：

- http://127.0.0.1:18789/ (或 http://localhost:18789/)

主要参考：

- [Control UI](/web/control-ui) 了解使用方法和 UI 功能。
- [Tailscale](/gateway/tailscale) 了解 Serve/Funnel 自动化。
- [Web 界面](/web) 了解绑定模式和安全注意事项。

认证通过 WebSocket 握手时的 `connect.params.auth` 强制执行
（token 或 password）。参见 [Gateway 配置](/gateway/configuration) 中的 `gateway.auth`。

安全提示：Control UI 是一个**管理界面**（聊天、配置、执行审批）。
请勿将其公开暴露。UI 在首次加载后将 token 存储在 `localStorage` 中。
建议使用 localhost、Tailscale Serve 或 SSH 隧道。

## 快速路径（推荐）

- 完成 onboarding 后，CLI 现在会自动打开仪表板并附带您的 token，同时打印相同的带 token 链接。
- 随时重新打开：`openclaw dashboard`（复制链接，尽可能打开浏览器，如果是 headless 环境则显示 SSH 提示）。
- token 保持本地（仅查询参数）；UI 在首次加载后将其剥离并保存到 localStorage。

## Token 基础（本地 vs 远程）

- **本地主机**：打开 `http://127.0.0.1:18789/`。如果看到"未授权"，运行 `openclaw dashboard` 并使用带 token 的链接（`?token=...`）。
- **Token 来源**：`gateway.auth.token`（或 `OPENCLAW_GATEWAY_TOKEN`）；UI 在首次加载后存储它。
- **非本地主机**：使用 Tailscale Serve（如果 `gateway.auth.allowTailscale: true` 则无需 token），或使用带 token 的 tailnet 绑定，或 SSH 隧道。参见 [Web 界面](/web)。

## 如果看到"未授权" / 1008

- 运行 `openclaw dashboard` 获取新的带 token 链接。
- 确保 gateway 可访问（本地：`openclaw status`；远程：SSH 隧道 `ssh -N -L 18789:127.0.0.1:18789 user@host` 然后打开 `http://127.0.0.1:18789/?token=...`）。
- 在仪表板设置中，粘贴您在 `gateway.auth.token`（或 `OPENCLAW_GATEWAY_TOKEN`）中配置的相同 token。

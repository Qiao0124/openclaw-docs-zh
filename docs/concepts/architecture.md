---
summary: "WebSocket 网关架构、组件和客户端流程"
read_when:
  - 研究网关协议、客户端或传输层
  - 理解 OpenClaw 系统架构
title: "网关架构"
---

# 网关架构

最后更新：2026-01-22

## 概述

- 一个长期运行的**网关（Gateway）**管理所有消息渠道（通过 Baileys 的 WhatsApp、通过 grammY 的 Telegram、Slack、Discord、Signal、iMessage、WebChat）。
- 控制平面客户端（macOS 应用、CLI、Web UI、自动化脚本）通过 **WebSocket** 连接到网关，使用配置的绑定主机（默认 `127.0.0.1:18789`）。
- **节点**（macOS/iOS/Android/无头模式）也通过 **WebSocket** 连接，但需声明 `role: node` 以及明确的 capabilities/commands。
- 每台主机一个网关；它是唯一打开 WhatsApp 会话的进程。
- **画布主机（canvas host）**（默认 `18793`）提供代理可编辑的 HTML 和 A2UI。

## 组件与流程

### 网关（守护进程）

- 维护提供商连接。
- 暴露类型化的 WS API（请求、响应、服务器推送事件）。
- 根据 JSON Schema 验证入站帧。
- 发出事件如 `agent`、`chat`、`presence`、`health`、`heartbeat`、`cron`。

### 客户端（mac 应用 / CLI / Web 管理后台）

- 每个客户端一个 WS 连接。
- 发送请求（`health`、`status`、`send`、`agent`、`system-presence`）。
- 订阅事件（`tick`、`agent`、`presence`、`shutdown`）。

### 节点（macOS / iOS / Android / 无头模式）

- 以 `role: node` 连接到**同一个 WS 服务器**。
- 在 `connect` 中提供设备标识；配对是**基于设备的**（角色 `node`），审批存储在设备配对存储中。
- 暴露命令如 `canvas.*`、`camera.*`、`screen.record`、`location.get`。

协议详情：

- [网关协议](/gateway/protocol)

### WebChat

- 使用网关 WS API 进行聊天历史记录和发送的静态 UI。
- 在远程设置中，通过与其他客户端相同的 SSH/Tailscale 隧道连接。

## 连接生命周期（单个客户端）

```
Client                    Gateway
  |                          |
  |---- req:connect -------->|
  |<------ res (ok) ---------|   (或 res error + close)
  |   (payload=hello-ok 携带快照：presence + health)
  |                          |
  |<------ event:presence ---|
  |<------ event:tick -------|
  |                          |
  |------- req:agent ------->|
  |<------ res:agent --------|   (ack: {runId,status:"accepted"})
  |<------ event:agent ------|   (streaming)
  |<------ res:agent --------|   (final: {runId,status,summary})
  |                          |
```

## 传输层协议（概要）

- 传输：WebSocket，文本帧携带 JSON 载荷。
- 第一帧**必须**是 `connect`。
- 握手后：
  - 请求：`{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
  - 事件：`{type:"event", event, payload, seq?, stateVersion?}`
- 如果设置了 `OPENCLAW_GATEWAY_TOKEN`（或 `--token`），`connect.params.auth.token` 必须匹配，否则连接关闭。
- 幂等键对具有副作用的方法（`send`、`agent`）是必需的，以便安全重试；服务器保留短期去重缓存。
- 节点必须在 `connect` 中包含 `role: "node"` 以及 caps/commands/permissions。

## 配对与本地信任

- 所有 WS 客户端（操作员 + 节点）在 `connect` 时包含**设备标识**。
- 新设备 ID 需要配对批准；网关为后续连接颁发**设备令牌**。
- **本地**连接（回环地址或网关主机自身的 tailnet 地址）可以自动批准，以保持同主机用户体验流畅。
- **非本地**连接必须对 `connect.challenge` nonce 进行签名，并需要显式批准。
- 网关认证（`gateway.auth.*`）仍然适用于**所有**连接，无论本地或远程。

详情：[网关协议](/gateway/protocol)、[配对](/start/pairing)、
[安全](/gateway/security)。

## 协议类型与代码生成

- TypeBox 模式定义协议。
- 从这些模式生成 JSON Schema。
- Swift 模型从 JSON Schema 生成。

## 远程访问

- 首选：Tailscale 或 VPN。
- 替代方案：SSH 隧道
  ```bash
  ssh -N -L 18789:127.0.0.1:18789 user@host
  ```
- 相同的握手 + 认证令牌适用于隧道。
- 在远程设置中可以为 WS 启用 TLS + 可选的证书固定。

## 运维快照

- 启动：`openclaw gateway`（前台，日志输出到 stdout）。
- 健康检查：通过 WS 的 `health`（也包含在 `hello-ok` 中）。
- 监控：launchd/systemd 用于自动重启。

## 不变量

- 每个主机上恰好有一个网关控制单个 Baileys 会话。
- 握手是强制性的；任何非 JSON 或非 connect 的第一帧都会导致硬关闭。
- 事件不会重放；客户端必须在出现间隙时刷新。

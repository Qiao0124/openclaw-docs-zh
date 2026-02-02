---
summary: "Loopback WebChat 静态主机和 Gateway WebSocket 用于聊天 UI"
read_when:
  - 调试或配置 WebChat 访问
title: "WebChat"
---

# WebChat (Gateway WebSocket UI)

状态：macOS/iOS SwiftUI 聊天 UI 直接与 Gateway WebSocket 通信。

## 是什么

- 一个原生的 gateway 聊天 UI（无嵌入式浏览器，无本地静态服务器）。
- 使用与其他通道相同的会话和路由规则。
- 确定性路由：回复始终返回到 WebChat。

## 快速开始

1. 启动 gateway。
2. 打开 WebChat UI（macOS/iOS 应用）或 Control UI 聊天标签页。
3. 确保 gateway 认证已配置（默认情况下即使在 loopback 上也需要）。

## 工作原理（行为）

- UI 连接到 Gateway WebSocket 并使用 `chat.history`、`chat.send` 和 `chat.inject`。
- `chat.inject` 将助手备注直接附加到转录并广播到 UI（无 agent 运行）。
- 历史记录始终从 gateway 获取（无本地文件监听）。
- 如果 gateway 不可达，WebChat 为只读模式。

## 远程使用

- 远程模式通过 SSH/Tailscale 将 gateway WebSocket 隧道传输。
- 您无需运行单独的 WebChat 服务器。

## 配置参考 (WebChat)

完整配置：[配置](/gateway/configuration)

通道选项：

- 无专用 `webchat.*` 配置块。WebChat 使用以下 gateway 端点 + 认证设置。

相关全局选项：

- `gateway.port`, `gateway.bind`: WebSocket 主机/端口。
- `gateway.auth.mode`, `gateway.auth.token`, `gateway.auth.password`: WebSocket 认证。
- `gateway.remote.url`, `gateway.remote.token`, `gateway.remote.password`: 远程 gateway 目标。
- `session.*`: 会话存储和主密钥默认值。

---
summary: "配对概览：批准谁可以私信 + 哪些节点可加入"
read_when:
  - 设置 DM 访问控制
  - 配对新的 iOS/Android 节点
  - 审视 OpenClaw 的安全姿态
title: "配对（Pairing）"
---

# 配对（Pairing）

“配对（Pairing）”是 OpenClaw 的明确 **所有者批准**步骤。
它用于两个场景：

1. **DM 配对**（谁可以与机器人对话）
2. **节点配对**（哪些设备/节点可以加入网关网络）

安全背景：[Security](/gateway/security)

## 1) DM 配对（inbound chat access）

当某个渠道的 DM 策略设置为 `pairing` 时，未知发送者会收到一个短码，且其消息在你批准前**不会被处理**。

默认 DM 策略见：[Security](/gateway/security)

配对码说明：

- 8 个字符，大写，无易混淆字符（`0O1I`）。
- **1 小时过期**。机器人只会在新请求创建时发送配对消息（大约每个发送者每小时一次）。
- 待处理 DM 配对请求默认每个渠道最多 **3 条**；超过后会被忽略，直到过期或被批准。

### 批准发送者（Approve a sender）

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

支持的渠道：`telegram`、`whatsapp`、`signal`、`imessage`、`discord`、`slack`。

### 状态存放位置（Where the state lives）

存储在 `~/.openclaw/credentials/` 下：

- 待处理请求：`<channel>-pairing.json`
- 已批准允许列表：`<channel>-allowFrom.json`

请将这些文件视为敏感信息（它们决定谁能访问你的助手）。

## 2) 节点设备配对（Node device pairing: iOS/Android/macOS/headless nodes）

节点以 **设备（devices）** 的形式连接到网关，`role: node`。网关会创建设备配对请求，必须批准后才能加入。

### 批准节点设备（Approve a node device）

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

### 状态存放位置（Where the state lives）

存储在 `~/.openclaw/devices/` 下：

- `pending.json`（短期；待处理请求会过期）
- `paired.json`（已配对设备 + 令牌）

### 备注（Notes）

- 旧版 `node.pair.*` API（CLI：`openclaw nodes pending/approve`）使用的是独立的网关配对存储。WS 节点仍需要设备配对。

## 相关文档（Related docs）

- 安全模型 + 提示注入：[Security](/gateway/security)
- 安全更新（运行 doctor）：[Updating](/install/updating)
- 渠道配置：
  - Telegram：[Telegram](/channels/telegram)
  - WhatsApp：[WhatsApp](/channels/whatsapp)
  - Signal：[Signal](/channels/signal)
  - iMessage：[iMessage](/channels/imessage)
  - Discord：[Discord](/channels/discord)
  - Slack：[Slack](/channels/slack)

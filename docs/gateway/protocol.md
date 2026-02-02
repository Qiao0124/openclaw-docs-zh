---
summary: "Gateway WebSocket 协议：握手、帧、版本控制"
read_when:
  - 实现或更新 Gateway WebSocket 客户端
  - 调试协议不匹配或连接失败
  - 重新生成协议模式/模型
title: "Gateway 协议"
---

# Gateway 协议 (WebSocket)

Gateway WebSocket 协议是 OpenClaw 的**单一控制平面 + 节点传输层**。所有客户端（CLI、Web UI、macOS 应用、iOS/Android 节点、无头节点）都通过 WebSocket 连接，并在握手时声明其**角色** + **作用域**。

## 传输层

- WebSocket，文本帧携带 JSON 载荷。
- 第一帧**必须**是 `connect` 请求。

## 握手 (connect)

Gateway → 客户端 (预连接挑战)：

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

客户端 → Gateway：

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway → 客户端：

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": { "type": "hello-ok", "protocol": 3, "policy": { "tickIntervalMs": 15000 } }
}
```

当颁发设备令牌时，`hello-ok` 还包含：

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

### 节点示例

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

## 帧结构

- **请求**: `{type:"req", id, method, params}`
- **响应**: `{type:"res", id, ok, payload|error}`
- **事件**: `{type:"event", event, payload, seq?, stateVersion?}`

具有副作用的方法需要**幂等键**（参见模式）。

## 角色 + 作用域

### 角色

- `operator` = 控制平面客户端 (CLI/UI/自动化)。
- `node` = 能力宿主 (camera/screen/canvas/system.run)。

### 作用域 (operator)

常见作用域：

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`

### 能力/命令/权限 (node)

节点在连接时声明能力声明：

- `caps`: 高级能力类别。
- `commands`: 用于调用的命令白名单。
- `permissions`: 细粒度开关（例如 `screen.record`、`camera.capture`）。

Gateway 将这些视为**声明**，并在服务器端执行白名单。

## 在线状态

- `system-presence` 返回按设备身份键控的条目。
- 在线状态条目包括 `deviceId`、`roles` 和 `scopes`，因此 UI 可以为每个设备显示单行，即使它以**operator**和**node**两种角色连接。

### 节点辅助方法

- 节点可以调用 `skills.bins` 获取技能可执行文件列表，用于自动允许检查。

## 执行审批

- 当执行请求需要审批时，gateway 会广播 `exec.approval.requested`。
- Operator 客户端通过调用 `exec.approval.resolve` 来解决（需要 `operator.approvals` 作用域）。

## 版本控制

- `PROTOCOL_VERSION` 位于 `src/gateway/protocol/schema.ts`。
- 客户端发送 `minProtocol` + `maxProtocol`；服务器会拒绝不匹配的版本。
- 模式和模型从 TypeBox 定义生成：
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## 认证

- 如果设置了 `OPENCLAW_GATEWAY_TOKEN`（或 `--token`），`connect.params.auth.token` 必须匹配，否则 socket 会被关闭。
- 配对后，Gateway 会颁发一个**设备令牌**，作用域限于连接角色 + 作用域。它在 `hello-ok.auth.deviceToken` 中返回，客户端应持久化存储以供将来连接使用。
- 设备令牌可以通过 `device.token.rotate` 和 `device.token.revoke` 进行轮换/撤销（需要 `operator.pairing` 作用域）。

## 设备身份 + 配对

- 节点应包含稳定的设备身份 (`device.id`)，从密钥对指纹派生。
- Gateway 为每个设备 + 角色颁发令牌。
- 除非启用了本地自动批准，否则新设备 ID 需要配对批准。
- **本地**连接包括 loopback 和 gateway 主机自身的 tailnet 地址（因此同主机 tailnet 绑定仍然可以自动批准）。
- 所有 WebSocket 客户端必须在 `connect` 期间包含 `device` 身份（operator + node）。只有当 `gateway.controlUi.allowInsecureAuth` 启用时（或 `gateway.controlUi.dangerouslyDisableDeviceAuth` 用于紧急用途），控制 UI 才能省略它。
- 非本地连接必须签名服务器提供的 `connect.challenge` nonce。

## TLS + 证书固定

- WebSocket 连接支持 TLS。
- 客户端可以选择固定 gateway 证书指纹（参见 `gateway.tls` 配置以及 `gateway.remote.tlsFingerprint` 或 CLI `--tls-fingerprint`）。

## 作用域

此协议暴露**完整的 gateway API**（状态、通道、模型、聊天、agent、会话、节点、审批等）。确切的接口由 `src/gateway/protocol/schema.ts` 中的 TypeBox 模式定义。

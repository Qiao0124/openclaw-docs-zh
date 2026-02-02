---
summary: "Nodes: 配对、功能、权限以及 canvas/camera/screen/system 的 CLI 辅助工具"
read_when:
  - 将 iOS/Android 节点配对到网关
  - 使用节点 canvas/camera 作为 Agent 上下文
  - 添加新的节点命令或 CLI 辅助工具
title: "节点"
---

# 节点

**节点**是一种配套设备（macOS/iOS/Android/无头模式），通过 `role: "node"` 连接到网关 **WebSocket**（与操作员使用相同端口），并通过 `node.invoke` 暴露命令接口（例如 `canvas.*`、`camera.*`、`system.*`）。协议详情：[网关协议](/gateway/protocol)。

传统传输方式：[桥接协议](/gateway/bridge-protocol)（TCP JSONL；当前节点已弃用/移除）。

macOS 也可以运行在**节点模式**：菜单栏应用连接到网关的 WS 服务器，并将其本地 canvas/camera 命令作为节点暴露（因此可以在此 Mac 上使用 `openclaw nodes …`）。

注意：

- 节点是**外设**，不是网关。它们不运行网关服务。
- Telegram/WhatsApp 等消息会发送到**网关**，而不是节点。

## 配对 + 状态

**WS 节点使用设备配对。** 节点在 `connect` 期间呈现设备身份；网关会为 `role: node` 创建设备配对请求。通过设备 CLI（或 UI）进行审批。

快速 CLI：

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
```

注意：

- 当设备配对角色包含 `node` 时，`nodes status` 会将节点标记为**已配对**。
- `node.pair.*`（CLI：`openclaw nodes pending/approve/reject`）是一个独立的网关拥有的节点配对存储；它**不会**控制 WS `connect` 握手。

## 远程节点主机（system.run）

当您的网关运行在一台机器上，而您希望在另一台机器上执行命令时，使用**节点主机**。模型仍然与**网关**通信；当选择 `host=node` 时，网关将 `exec` 调用转发到**节点主机**。

### 运行位置

- **网关主机**：接收消息，运行模型，路由工具调用。
- **节点主机**：在节点机器上执行 `system.run`/`system.which`。
- **审批**：通过节点主机上的 `~/.openclaw/exec-approvals.json` 强制执行。

### 启动节点主机（前台）

在节点机器上：

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name "Build Node"
```

### 启动节点主机（服务）

```bash
openclaw node install --host <gateway-host> --port 18789 --display-name "Build Node"
openclaw node restart
```

### 配对 + 命名

在网关主机上：

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes list
```

命名选项：

- `openclaw node run` / `openclaw node install` 的 `--display-name`（持久化存储在节点的 `~/.openclaw/node.json` 中）。
- `openclaw nodes rename --node <id|name|ip> --name "Build Node"`（网关覆盖）。

### 命令白名单

执行审批是**每个节点主机**独立的。从网关添加白名单条目：

```bash
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/uname"
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/sw_vers"
```

审批存储在节点主机的 `~/.openclaw/exec-approvals.json` 中。

### 将 exec 指向节点

配置默认值（网关配置）：

```bash
openclaw config set tools.exec.host node
openclaw config set tools.exec.security allowlist
openclaw config set tools.exec.node "<id-or-name>"
```

或按会话设置：

```
/exec host=node security=allowlist node=<id-or-name>
```

设置后，任何带有 `host=node` 的 `exec` 调用都会在节点主机上运行（受节点白名单/审批限制）。

相关文档：

- [节点主机 CLI](/cli/node)
- [Exec 工具](/tools/exec)
- [Exec 审批](/tools/exec-approvals)

## 调用命令

底层（原始 RPC）：

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command canvas.eval --params '{"javaScript":"location.href"}'
```

对于常见的"给 Agent 提供媒体附件"工作流程，存在更高级的辅助工具。

## 截图（canvas 快照）

如果节点正在显示 Canvas（WebView），`canvas.snapshot` 返回 `{ format, base64 }`。

CLI 辅助工具（写入临时文件并打印 `MEDIA:<path>`）：

```bash
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format png
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format jpg --max-width 1200 --quality 0.9
```

### Canvas 控制

```bash
openclaw nodes canvas present --node <idOrNameOrIp> --target https://example.com
openclaw nodes canvas hide --node <idOrNameOrIp>
openclaw nodes canvas navigate https://example.com --node <idOrNameOrIp>
openclaw nodes canvas eval --node <idOrNameOrIp> --js "document.title"
```

注意：

- `canvas present` 接受 URL 或本地文件路径（`--target`），以及可选的 `--x/--y/--width/--height` 用于定位。
- `canvas eval` 接受内联 JS（`--js`）或位置参数。

### A2UI（Canvas）

```bash
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --text "Hello"
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --jsonl ./payload.jsonl
openclaw nodes canvas a2ui reset --node <idOrNameOrIp>
```

注意：

- 仅支持 A2UI v0.8 JSONL（v0.9/createSurface 被拒绝）。

## 照片 + 视频（节点相机）

照片（`jpg`）：

```bash
openclaw nodes camera list --node <idOrNameOrIp>
openclaw nodes camera snap --node <idOrNameOrIp>            # 默认：前后双摄（2 行 MEDIA）
openclaw nodes camera snap --node <idOrNameOrIp> --facing front
```

视频片段（`mp4`）：

```bash
openclaw nodes camera clip --node <idOrNameOrIp> --duration 10s
openclaw nodes camera clip --node <idOrNameOrIp> --duration 3000 --no-audio
```

注意：

- 节点必须处于**前台**才能使用 `canvas.*` 和 `camera.*`（后台调用返回 `NODE_BACKGROUND_UNAVAILABLE`）。
- 视频片段时长被限制（目前 `<= 60s`）以避免过大的 base64 负载。
- Android 会在可能时提示 `CAMERA`/`RECORD_AUDIO` 权限；拒绝权限会失败并返回 `*_PERMISSION_REQUIRED`。

## 屏幕录制（节点）

节点暴露 `screen.record`（mp4）。示例：

```bash
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10 --no-audio
```

注意：

- `screen.record` 要求节点应用处于前台。
- Android 会在录制前显示系统屏幕捕获提示。
- 屏幕录制限制为 `<= 60s`。
- `--no-audio` 禁用麦克风捕获（iOS/Android 支持；macOS 使用系统捕获音频）。
- 当有多个屏幕可用时，使用 `--screen <index>` 选择显示器。

## 位置（节点）

当设置中启用位置时，节点暴露 `location.get`。

CLI 辅助工具：

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

注意：

- 位置默认**关闭**。
- "始终"需要系统权限；后台获取是尽力而为。
- 响应包括纬度/经度、精度（米）和时间戳。

## 短信（Android 节点）

当用户授予 **SMS** 权限且设备支持电话功能时，Android 节点可以暴露 `sms.send`。

底层调用：

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command sms.send --params '{"to":"+15555550123","message":"Hello from OpenClaw"}'
```

注意：

- 必须在 Android 设备上接受权限提示后，功能才会被广播。
- 没有电话功能的纯 Wi-Fi 设备不会广播 `sms.send`。

## 系统命令（节点主机 / mac 节点）

macOS 节点暴露 `system.run`、`system.notify` 和 `system.execApprovals.get/set`。
无头节点主机暴露 `system.run`、`system.which` 和 `system.execApprovals.get/set`。

示例：

```bash
openclaw nodes run --node <idOrNameOrIp> -- echo "Hello from mac node"
openclaw nodes notify --node <idOrNameOrIp> --title "Ping" --body "Gateway ready"
```

注意：

- `system.run` 在负载中返回 stdout/stderr/退出码。
- `system.notify` 尊重 macOS 应用上的通知权限状态。
- `system.run` 支持 `--cwd`、`--env KEY=VAL`、`--command-timeout` 和 `--needs-screen-recording`。
- `system.notify` 支持 `--priority <passive|active|timeSensitive>` 和 `--delivery <system|overlay|auto>`。
- macOS 节点丢弃 `PATH` 覆盖；无头节点主机仅在它前置节点主机 PATH 时接受 `PATH`。
- 在 macOS 节点模式下，`system.run` 受 macOS 应用中的执行审批控制（设置 → 执行审批）。Ask/allowlist/full 的行为与无头节点主机相同；拒绝的提示返回 `SYSTEM_RUN_DENIED`。
- 在无头节点主机上，`system.run` 受执行审批控制（`~/.openclaw/exec-approvals.json`）。

## Exec 节点绑定

当有多个节点可用时，您可以将 exec 绑定到特定节点。
这会为 `exec host=node` 设置默认节点（可以按 Agent 覆盖）。

全局默认值：

```bash
openclaw config set tools.exec.node "node-id-or-name"
```

按 Agent 覆盖：

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

取消设置以允许任何节点：

```bash
openclaw config unset tools.exec.node
openclaw config unset agents.list[0].tools.exec.node
```

## 权限映射

节点可能在 `node.list` / `node.describe` 中包含 `permissions` 映射，按权限名称（例如 `screenRecording`、`accessibility`）键入，值为布尔值（`true` = 已授予）。

## 无头节点主机（跨平台）

OpenClaw 可以运行**无头节点主机**（无 UI），它连接到网关 WebSocket 并暴露 `system.run` / `system.which`。这在 Linux/Windows 上或 alongside 服务器运行最小节点时非常有用。

启动它：

```bash
openclaw node run --host <gateway-host> --port 18789
```

注意：

- 仍然需要配对（网关将显示节点审批提示）。
- 节点主机将其节点 id、令牌、显示名称和网关连接信息存储在 `~/.openclaw/node.json` 中。
- 执行审批通过 `~/.openclaw/exec-approvals.json` 在本地强制执行（参见[执行审批](/tools/exec-approvals)）。
- 在 macOS 上，无头节点主机在可访问时优先使用配套应用执行主机，如果应用不可用则回退到本地执行。设置 `OPENCLAW_NODE_EXEC_HOST=app` 以要求使用应用，或设置 `OPENCLAW_NODE_EXEC_FALLBACK=0` 以禁用回退。
- 当网关 WS 使用 TLS 时，添加 `--tls` / `--tls-fingerprint`。

## Mac 节点模式

- macOS 菜单栏应用作为节点连接到网关 WS 服务器（因此可以在此 Mac 上使用 `openclaw nodes …`）。
- 在远程模式下，应用为网关端口打开 SSH 隧道并连接到 `localhost`。

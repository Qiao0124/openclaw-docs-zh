---
summary: "OpenClaw macOS 配套应用（菜单栏 + gateway 代理）"
read_when:
  - 实现 macOS 应用功能
  - 更改 macOS 上的 gateway 生命周期或节点桥接
title: "macOS 应用"
---

# OpenClaw macOS 配套应用（菜单栏 + gateway 代理）

macOS 应用是 OpenClaw 的**菜单栏配套应用**。它拥有权限，
在本地管理/连接到 Gateway（launchd 或手动），并作为节点向 agent 暴露 macOS 功能。

## 功能

- 在菜单栏显示原生通知和状态。
- 拥有 TCC 提示（通知、辅助功能、屏幕录制、麦克风、
  语音识别、自动化/AppleScript）。
- 运行或连接到 Gateway（本地或远程）。
- 暴露 macOS 专用工具（Canvas、相机、屏幕录制、`system.run`）。
- 在**远程**模式下启动本地节点主机服务（launchd），在**本地**模式下停止它。
- 可选托管 **PeekabooBridge** 用于 UI 自动化。
- 根据请求通过 npm/pnpm 安装全局 CLI（`openclaw`）（不推荐将 bun 用于 Gateway 运行时）。

## 本地 vs 远程模式

- **本地**（默认）：如果存在正在运行的本地 Gateway，应用会连接到它；
  否则通过 `openclaw gateway install` 启用 launchd 服务。
- **远程**：应用通过 SSH/Tailscale 连接到 Gateway，从不启动
  本地进程。
  应用启动本地**节点主机服务**，以便远程 Gateway 可以访问此 Mac。
  应用不会将 Gateway 作为子进程生成。

## Launchd 控制

应用管理一个每用户 LaunchAgent，标签为 `bot.molt.gateway`
（使用 `--profile`/`OPENCLAW_PROFILE` 时为 `bot.molt.<profile>`；旧版 `com.openclaw.*` 仍会卸载）。

```bash
launchctl kickstart -k gui/$UID/bot.molt.gateway
launchctl bootout gui/$UID/bot.molt.gateway
```

使用命名配置文件时将标签替换为 `bot.molt.<profile>`。

如果 LaunchAgent 未安装，请从应用启用或运行
`openclaw gateway install`。

## 节点功能 (mac)

macOS 应用将自己呈现为一个节点。常用命令：

- Canvas: `canvas.present`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot`, `canvas.a2ui.*`
- 相机: `camera.snap`, `camera.clip`
- 屏幕: `screen.record`
- 系统: `system.run`, `system.notify`

节点报告一个 `permissions` 映射，以便 agent 可以决定允许什么。

节点服务 + 应用 IPC：

- 当 headless 节点主机服务运行时（远程模式），它作为节点连接到 Gateway WS。
- `system.run` 在 macOS 应用中执行（UI/TCC 上下文）通过本地 Unix socket；提示和输出保留在应用内。

示意图 (SCI)：

```
Gateway -> 节点服务 (WS)
                 |  IPC (UDS + token + HMAC + TTL)
                 v
             Mac 应用 (UI + TCC + system.run)
```

## 执行审批 (system.run)

`system.run` 由 macOS 应用中的**执行审批**控制（设置 → 执行审批）。
安全 + 询问 + 允许列表存储在 Mac 本地：

```
~/.openclaw/exec-approvals.json
```

示例：

```json
{
  "version": 1,
  "defaults": {
    "security": "deny",
    "ask": "on-miss"
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [{ "pattern": "/opt/homebrew/bin/rg" }]
    }
  }
}
```

注意：

- `allowlist` 条目是已解析二进制路径的 glob 模式。
- 在提示中选择"始终允许"会将该命令添加到允许列表。
- `system.run` 环境覆盖被过滤（删除 `PATH`、`DYLD_*`、`LD_*`、`NODE_OPTIONS`、`PYTHON*`、`PERL*`、`RUBYOPT`），然后与应用环境合并。

## 深度链接

应用注册 `openclaw://` URL scheme 用于本地操作。

### `openclaw://agent`

触发 Gateway `agent` 请求。

```bash
open 'openclaw://agent?message=Hello%20from%20deep%20link'
```

查询参数：

- `message`（必需）
- `sessionKey`（可选）
- `thinking`（可选）
- `deliver` / `to` / `channel`（可选）
- `timeoutSeconds`（可选）
- `key`（可选无人值守模式密钥）

安全性：

- 没有 `key` 时，应用会提示确认。
- 使用有效的 `key`，运行是无人值守的（用于个人自动化）。

## 入门流程（典型）

1. 安装并启动 **OpenClaw.app**。
2. 完成权限清单（TCC 提示）。
3. 确保**本地**模式处于活动状态且 Gateway 正在运行。
4. 如果需要终端访问，安装 CLI。

## 构建与开发工作流（原生）

- `cd apps/macos && swift build`
- `swift run OpenClaw`（或 Xcode）
- 打包应用：`scripts/package-mac-app.sh`

## 调试 gateway 连接（macOS CLI）

使用调试 CLI 来测试 macOS 应用使用的相同 Gateway WebSocket 握手和发现
逻辑，而无需启动应用。

```bash
cd apps/macos
swift run openclaw-mac connect --json
swift run openclaw-mac discover --timeout 3000 --json
```

连接选项：

- `--url <ws://host:port>`: 覆盖配置
- `--mode <local|remote>`: 从配置解析（默认：配置或本地）
- `--probe`: 强制新的健康探测
- `--timeout <ms>`: 请求超时（默认：`15000`）
- `--json`: 用于 diff 的结构化输出

发现选项：

- `--include-local`: 包括会被过滤为"本地"的 gateway
- `--timeout <ms>`: 整体发现窗口（默认：`2000`）
- `--json`: 用于 diff 的结构化输出

提示：与 `openclaw gateway discover --json` 对比，查看
macOS 应用的发现管道（NWBrowser + tailnet DNS-SD 回退）与
Node CLI 的 `dns-sd` 发现有何不同。

## 远程连接管道（SSH 隧道）

当 macOS 应用运行在**远程**模式时，它会打开 SSH 隧道，以便本地 UI
组件可以与远程 Gateway 通信，就像它在 localhost 上一样。

### 控制隧道（Gateway WebSocket 端口）

- **目的：** 健康检查、状态、Web Chat、配置和其他控制平面调用。
- **本地端口：** Gateway 端口（默认 `18789`），始终稳定。
- **远程端口：** 远程主机上的相同 Gateway 端口。
- **行为：** 无随机本地端口；应用重用现有健康隧道
  或在需要时重新启动它。
- **SSH 形式：** `ssh -N -L <local>:127.0.0.1:<remote>` 带有 BatchMode +
  ExitOnForwardFailure + keepalive 选项。
- **IP 报告：** SSH 隧道使用 loopback，因此 gateway 将看到节点
  IP 为 `127.0.0.1`。如果您希望显示真实客户端 IP，请使用**直接 (ws/wss)** 传输（参见 [macOS 远程访问](/platforms/mac/remote)）。

设置步骤，参见 [macOS 远程访问](/platforms/mac/remote)。协议
详情，参见 [Gateway 协议](/gateway/protocol)。

## 相关文档

- [Gateway 运行手册](/gateway)
- [Gateway (macOS)](/platforms/mac/bundled-gateway)
- [macOS 权限](/platforms/mac/permissions)
- [Canvas](/platforms/mac/canvas)

---
summary: "CLI 参考文档：`openclaw browser`（配置文件、标签页、操作、扩展中继）"
read_when:
  - 使用 `openclaw browser` 并需要常见任务的示例
  - 希望通过节点主机控制运行在另一台机器上的浏览器
  - 想要使用 Chrome 扩展中继（通过工具栏按钮附加/分离）
title: "browser"
---

# `openclaw browser`

管理 OpenClaw 的浏览器控制服务器并运行浏览器操作（标签页、快照、截图、导航、点击、输入）。

相关文档：

- 浏览器工具 + API：[Browser tool](/tools/browser)
- Chrome 扩展中继：[Chrome extension](/tools/chrome-extension)

## 常用标志

- `--url <gatewayWsUrl>`：Gateway WebSocket URL（默认从配置读取）。
- `--token <token>`：Gateway 令牌（如需要）。
- `--timeout <ms>`：请求超时（毫秒）。
- `--browser-profile <name>`：选择浏览器配置文件（默认从配置读取）。
- `--json`：机器可读输出（在支持的命令中）。

## 快速开始（本地）

```bash
openclaw browser --browser-profile chrome tabs
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

## 配置文件

配置文件是命名的浏览器路由配置。实际使用中：

- `openclaw`：启动/附加到专用的 OpenClaw 管理的 Chrome 实例（隔离的用户数据目录）。
- `chrome`：通过 Chrome 扩展中继控制您现有的 Chrome 标签页。

```bash
openclaw browser profiles
openclaw browser create-profile --name work --color "#FF5A36"
openclaw browser delete-profile --name work
```

使用特定配置文件：

```bash
openclaw browser --browser-profile work tabs
```

## 标签页

```bash
openclaw browser tabs
openclaw browser open https://docs.openclaw.ai
openclaw browser focus <targetId>
openclaw browser close <targetId>
```

## 快照 / 截图 / 操作

快照：

```bash
openclaw browser snapshot
```

截图：

```bash
openclaw browser screenshot
```

导航/点击/输入（基于引用的 UI 自动化）：

```bash
openclaw browser navigate https://example.com
openclaw browser click <ref>
openclaw browser type <ref> "hello"
```

## Chrome 扩展中继（通过工具栏按钮附加）

此模式允许代理控制您手动附加的现有 Chrome 标签页（不会自动附加）。

将解压后的扩展安装到稳定路径：

```bash
openclaw browser extension install
openclaw browser extension path
```

然后在 Chrome 中打开 `chrome://extensions` → 启用"开发者模式" → "加载已解压的扩展程序" → 选择打印的文件夹。

完整指南：[Chrome extension](/tools/chrome-extension)

## 远程浏览器控制（节点主机代理）

如果 Gateway 运行在不同于浏览器的机器上，请在拥有 Chrome/Brave/Edge/Chromium 的机器上运行**节点主机**。Gateway 将代理浏览器操作到该节点（无需单独的浏览器控制服务器）。

使用 `gateway.nodes.browser.mode` 控制自动路由，使用 `gateway.nodes.browser.node` 在多个连接节点中固定特定节点。

安全 + 远程设置：[Browser tool](/tools/browser)、[Remote access](/gateway/remote)、[Tailscale](/gateway/tailscale)、[Security](/gateway/security)

---
summary: "Google Chat 应用支持状态、功能和配置"
read_when:
  - 处理 Google Chat 频道功能
title: "Google Chat"
---

# Google Chat（Chat API）

状态：通过 Google Chat API webhooks（仅 HTTP）支持 DM + 空间。

## 快速设置（初学者）

1. 创建一个 Google Cloud 项目并启用 **Google Chat API**。
   - 前往：[Google Chat API Credentials](https://console.cloud.google.com/apis/api/chat.googleapis.com/credentials)
   - 如果 API 尚未启用，请启用它。
2. 创建一个**服务账户**：
   - 点击 **Create Credentials** > **Service Account**。
   - 命名它（例如，`openclaw-chat`）。
   - 留空权限（点击 **Continue**）。
   - 留空具有访问权限的主体（点击 **Done**）。
3. 创建并下载 **JSON Key**：
   - 在服务账户列表中，点击你刚刚创建的那个。
   - 进入 **Keys** 标签。
   - 点击 **Add Key** > **Create new key**。
   - 选择 **JSON** 并点击 **Create**。
4. 将下载的 JSON 文件存储在你的网关主机上（例如 `~/.openclaw/googlechat-service-account.json`）。
5. 在 [Google Cloud Console Chat Configuration](https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat) 中创建 Google Chat 应用：
   - 填写 **Application info**：
     - **App name**：（例如 `OpenClaw`）
     - **Avatar URL**：（例如 `https://openclaw.ai/logo.png`）
     - **Description**：（例如 `Personal AI Assistant`）
   - 启用 **Interactive features**。
   - 在 **Functionality** 下，勾选 **Join spaces and group conversations**。
   - 在 **Connection settings** 下，选择 **HTTP endpoint URL**。
   - 在 **Triggers** 下，选择 **Use a common HTTP endpoint URL for all triggers** 并将其设置为你的网关公共 URL 后接 `/googlechat`。
     - _提示：运行 `openclaw status` 以找到你的网关公共 URL。_
   - 在 **Visibility** 下，勾选 **Make this Chat app available to specific people and groups in &lt;Your Domain&gt;**。
   - 在文本框中输入你的 email 地址（例如 `user@example.com`）。
   - 点击底部的 **Save**。
6. **启用应用状态**：
   - 保存后，**刷新页面**。
   - 查找 **App status** 部分（通常在保存后的顶部或底部）。
   - 将状态更改为 **Live - available to users**。
   - 再次点击 **Save**。
7. 使用服务账户路径 + webhook audience 配置 OpenClaw：
   - 环境变量：`GOOGLE_CHAT_SERVICE_ACCOUNT_FILE=/path/to/service-account.json`
   - 或配置：`channels.googlechat.serviceAccountFile: "/path/to/service-account.json"`。
8. 设置 webhook audience 类型 + 值（匹配你的 Chat 应用配置）。
9. 启动网关。Google Chat 将 POST 到你的 webhook 路径。

## 添加到 Google Chat

一旦网关运行且你的 email 已添加到可见性列表：

1. 前往 [Google Chat](https://chat.google.com/)。
2. 点击 **Direct Messages** 旁边的 **+**（加号）图标。
3. 在搜索栏（通常添加人的地方）中，输入你在 Google Cloud Console 中配置的 **App name**。
   - **注意**：机器人不会出现在"Marketplace"浏览列表中，因为它是私有应用。你必须按名称搜索。
4. 从结果中选择你的机器人。
5. 点击 **Add** 或 **Chat** 开始 1:1 对话。
6. 发送 "Hello" 以触发助手！

## 公共 URL（仅 Webhook）

Google Chat webhooks 需要公共 HTTPS 端点。为了安全，**只将 `/googlechat` 路径暴露给互联网**。将 OpenClaw 仪表板和其他敏感端点保留在你的私有网络上。

### 选项 A：Tailscale Funnel（推荐）

使用 Tailscale Serve 作为私有仪表板，Funnel 作为公共 webhook 路径。这将 `/` 保持私有，同时只暴露 `/googlechat`。

1. **检查你的网关绑定到什么地址：**

   ```bash
   ss -tlnp | grep 18789
   ```

   注意 IP 地址（例如 `127.0.0.1`、`0.0.0.0` 或你的 Tailscale IP 如 `100.x.x.x`）。

2. **仅将仪表板暴露给 tailnet（端口 8443）：**

   ```bash
   # 如果绑定到 localhost（127.0.0.1 或 0.0.0.0）：
   tailscale serve --bg --https 8443 http://127.0.0.1:18789

   # 如果仅绑定到 Tailscale IP（例如 100.106.161.80）：
   tailscale serve --bg --https 8443 http://100.106.161.80:18789
   ```

3. **仅将 webhook 路径公开暴露：**

   ```bash
   # 如果绑定到 localhost（127.0.0.1 或 0.0.0.0）：
   tailscale funnel --bg --set-path /googlechat http://127.0.0.1:18789/googlechat

   # 如果仅绑定到 Tailscale IP（例如 100.106.161.80）：
   tailscale funnel --bg --set-path /googlechat http://100.106.161.80:18789/googlechat
   ```

4. **授权节点进行 Funnel 访问：**
   如果提示，访问输出中显示的授权 URL 以在你的 tailnet 策略中为此节点启用 Funnel。

5. **验证配置：**
   ```bash
   tailscale serve status
   tailscale funnel status
   ```

你的公共 webhook URL 将是：
`https://<node-name>.<tailnet>.ts.net/googlechat`

你的私有仪表板保持仅 tailnet：
`https://<node-name>.<tailnet>.ts.net:8443/`

在 Google Chat 应用配置中使用公共 URL（不带 `:8443`）。

> 注意：此配置在重启后持续存在。要稍后移除，运行 `tailscale funnel reset` 和 `tailscale serve reset`。

### 选项 B：反向代理（Caddy）

如果你使用像 Caddy 这样的反向代理，只代理特定路径：

```caddy
your-domain.com {
    reverse_proxy /googlechat* localhost:18789
}
```

使用此配置，对 `your-domain.com/` 的任何请求将被忽略或返回 404，而 `your-domain.com/googlechat` 被安全地路由到 OpenClaw。

### 选项 C：Cloudflare Tunnel

配置你的 tunnel 的 ingress 规则以只路由 webhook 路径：

- **Path**：`/googlechat` -> `http://localhost:18789/googlechat`
- **Default Rule**：HTTP 404（Not Found）

## 工作原理

1. Google Chat 向网关发送 webhook POST。每个请求包含一个 `Authorization: Bearer <token>` 头。
2. OpenClaw 根据配置的 `audienceType` + `audience` 验证令牌：
   - `audienceType: "app-url"` → audience 是你的 HTTPS webhook URL。
   - `audienceType: "project-number"` → audience 是 Cloud 项目编号。
3. 消息按空间路由：
   - DM 使用 session key `agent:<agentId>:googlechat:dm:<spaceId>`。
   - 空间使用 session key `agent:<agentId>:googlechat:group:<spaceId>`。
4. DM 访问默认是配对的。未知发送者收到配对码；使用以下方式批准：
   - `openclaw pairing approve googlechat <code>`
5. 群组空间默认需要 @-提及。如果提及检测需要应用的用户名，请设置 `botUser`。

## 目标

将这些标识符用于投递和 allowlist：

- 直接消息：`users/<userId>` 或 `users/<email>`（接受 email 地址）。
- 空间：`spaces/<spaceId>`。

## 配置亮点

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890", // 可选；帮助提及检测
      dm: {
        policy: "pairing",
        allowFrom: ["users/1234567890", "name@example.com"],
      },
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": {
          allow: true,
          requireMention: true,
          users: ["users/1234567890"],
          systemPrompt: "Short answers only.",
        },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

说明：

- 服务账户凭证也可以使用 `serviceAccount` 内联传递（JSON 字符串）。
- 如果 `webhookPath` 未设置，默认 webhook 路径是 `/googlechat`。
- 当 `actions.reactions` 启用时，反应可通过 `reactions` 工具和 `channels action` 使用。
- `typingIndicator` 支持 `none`、`message`（默认）和 `reaction`（反应需要用户 OAuth）。
- 附件通过 Chat API 下载并存储在媒体管道中（大小受 `mediaMaxMb` 限制）。

## 故障排除

### 405 Method Not Allowed

如果 Google Cloud Logs Explorer 显示如下错误：

```
status code: 405, reason phrase: HTTP error response: HTTP/1.1 405 Method Not Allowed
```

这意味着 webhook 处理程序未注册。常见原因：

1. **频道未配置**：`channels.googlechat` 部分缺失于你的配置中。验证：

   ```bash
   openclaw config get channels.googlechat
   ```

   如果返回 "Config path not found"，添加配置（参见 [配置亮点](#配置亮点)）。

2. **插件未启用**：检查插件状态：

   ```bash
   openclaw plugins list | grep googlechat
   ```

   如果显示 "disabled"，添加 `plugins.entries.googlechat.enabled: true` 到你的配置。

3. **网关未重启**：添加配置后，重启网关：
   ```bash
   openclaw gateway restart
   ```

验证频道正在运行：

```bash
openclaw channels status
# 应显示：Google Chat default: enabled, configured, ...
```

### 其他问题

- 检查 `openclaw channels status --probe` 以获取 auth 错误或缺失的 audience 配置。
- 如果没有消息到达，确认 Chat 应用的 webhook URL + 事件订阅。
- 如果提及门控阻止回复，将 `botUser` 设置为应用的用户资源名称并验证 `requireMention`。
- 使用 `openclaw logs --follow` 在发送测试消息时查看请求是否到达网关。

相关文档：

- [网关配置](/gateway/configuration)
- [安全](/gateway/security)
- [反应](/tools/reactions)

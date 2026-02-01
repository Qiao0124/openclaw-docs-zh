---
summary: "通过 gogcli 将 Gmail Pub/Sub 推送接入 OpenClaw Webhooks"
read_when:
  - 将 Gmail 收件箱触发连接到 OpenClaw
  - 配置 Pub/Sub 推送用于代理唤醒
title: "Gmail Pub/Sub"
---

# Gmail Pub/Sub -> OpenClaw

目标：Gmail watch -> Pub/Sub push -> `gog gmail watch serve` -> OpenClaw webhook。

## 前置条件

- 已安装并登录 `gcloud`（[安装指南](https://docs.cloud.google.com/sdk/docs/install-sdk)）。
- 已安装 `gog`（gogcli）并授权 Gmail 账号（[gogcli.sh](https://gogcli.sh/)）。
- 已启用 OpenClaw hooks（见 [Webhooks](/automation/webhook)）。
- 已登录 `tailscale`（[tailscale.com](https://tailscale.com/)）。推荐方案使用 Tailscale Funnel 提供公共 HTTPS 端点。
  其他隧道服务也可用，但需要自行配置/不受支持，目前只支持 Tailscale。

示例 hook 配置（启用 Gmail 预设映射）：

```json5
{
  hooks: {
    enabled: true,
    token: "OPENCLAW_HOOK_TOKEN",
    path: "/hooks",
    presets: ["gmail"],
  },
}
```

如需将 Gmail 摘要投递到聊天渠道，可通过自定义映射覆盖预设，
设置 `deliver` + 可选 `channel`/`to`：

```json5
{
  hooks: {
    enabled: true,
    token: "OPENCLAW_HOOK_TOKEN",
    presets: ["gmail"],
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "New email from {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}\n{{messages[0].body}}",
        model: "openai/gpt-5.2-mini",
        deliver: true,
        channel: "last",
        // to: "+15551234567"
      },
    ],
  },
}
```

如果你想固定渠道，设置 `channel` + `to`。否则 `channel: "last"`
会使用上一次投递路线（回退到 WhatsApp）。

如需为 Gmail 运行强制使用更便宜的模型，可在映射中设置 `model`
（`provider/model` 或别名）。若启用了 `agents.defaults.models`，
请确保包含该模型。

如需为 Gmail hooks 设置默认模型与思考级别，在配置中添加
`hooks.gmail.model` / `hooks.gmail.thinking`：

```json5
{
  hooks: {
    gmail: {
      model: "openrouter/meta-llama/llama-3.3-70b-instruct:free",
      thinking: "off",
    },
  },
}
```

说明：

- 映射中的 `model`/`thinking` 仍会覆盖这些默认值。
- 回退顺序：`hooks.gmail.model` → `agents.defaults.model.fallbacks` → primary（鉴权/限流/超时）。
- 若设置了 `agents.defaults.models`，Gmail 模型必须在允许列表中。
- Gmail hook 内容默认会包裹外部内容安全边界。
  如需禁用（危险），设置 `hooks.gmail.allowUnsafeExternalContent: true`。

如需进一步自定义 payload 处理，可设置 `hooks.mappings`，
或在 `hooks.transformsDir` 下添加 JS/TS transform 模块（见 [Webhooks](/automation/webhook)）。

## 向导（推荐）

使用 OpenClaw helper 一键完成（macOS 会通过 brew 安装依赖）：

```bash
openclaw webhooks gmail setup \
  --account openclaw@gmail.com
```

默认行为：

- 使用 Tailscale Funnel 提供公共推送端点。
- 写入 `hooks.gmail` 配置供 `openclaw webhooks gmail run` 使用。
- 启用 Gmail hook 预设（`hooks.presets: ["gmail"]`）。

路径说明：当启用 `tailscale.mode` 时，OpenClaw 会自动将
`hooks.gmail.serve.path` 设为 `/`，并把公共路径保留在
`hooks.gmail.tailscale.path`（默认 `/gmail-pubsub`），因为 Tailscale 在代理前
会剥离已设置的 path 前缀。
如果你需要后端收到带前缀的路径，设置 `hooks.gmail.tailscale.target`
（或 `--tailscale-target`）为完整 URL，如
`http://127.0.0.1:8788/gmail-pubsub` 并匹配 `hooks.gmail.serve.path`。

想自定义端点？使用 `--push-endpoint <url>` 或 `--tailscale off`。

平台说明：macOS 上向导会通过 Homebrew 安装 `gcloud`、`gogcli`、`tailscale`；
Linux 请先手动安装。

网关自启动（推荐）：

- 当 `hooks.enabled=true` 且 `hooks.gmail.account` 已设置时，网关会在启动时
  运行 `gog gmail watch serve` 并自动续期 watch。
- 可设置 `OPENCLAW_SKIP_GMAIL_WATCHER=1` 退出（若你自行运行 daemon）。
- 不要同时运行手动 daemon，否则会出现
  `listen tcp 127.0.0.1:8788: bind: address already in use`。

手动 daemon（启动 `gog gmail watch serve` + 自动续期）：

```bash
openclaw webhooks gmail run
```

## 一次性设置

1. 选择 **拥有 OAuth 客户端** 的 GCP 项目：

```bash
gcloud auth login
gcloud config set project <project-id>
```

说明：Gmail watch 需要 Pub/Sub topic 与 OAuth 客户端在同一项目中。

2. 启用 API：

```bash
gcloud services enable gmail.googleapis.com pubsub.googleapis.com
```

3. 创建 topic：

```bash
gcloud pubsub topics create gog-gmail-watch
```

4. 允许 Gmail push 发布：

```bash
gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
  --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
  --role=roles/pubsub.publisher
```

## 启动 watch

```bash
gog gmail watch start \
  --account openclaw@gmail.com \
  --label INBOX \
  --topic projects/<project-id>/topics/gog-gmail-watch
```

保存输出中的 `history_id`（调试时使用）。

## 运行 push 处理器

本地示例（共享 token 认证）：

```bash
gog gmail watch serve \
  --account openclaw@gmail.com \
  --bind 127.0.0.1 \
  --port 8788 \
  --path /gmail-pubsub \
  --token <shared> \
  --hook-url http://127.0.0.1:18789/hooks/gmail \
  --hook-token OPENCLAW_HOOK_TOKEN \
  --include-body \
  --max-bytes 20000
```

说明：

- `--token` 保护 push 端点（`x-gog-token` 或 `?token=`）。
- `--hook-url` 指向 OpenClaw `/hooks/gmail`（已映射；隔离运行 + 主会话摘要）。
- `--include-body` 与 `--max-bytes` 控制发送到 OpenClaw 的正文片段。

推荐：`openclaw webhooks gmail run` 会封装同样流程并自动续期 watch。

## 暴露处理器（高级，不支持）

如果你需要非 Tailscale 的隧道，请手动接入并在 push 订阅中使用公共 URL
（不受支持，无安全护栏）：

```bash
cloudflared tunnel --url http://127.0.0.1:8788 --no-autoupdate
```

将生成的 URL 作为 push 端点：

```bash
gcloud pubsub subscriptions create gog-gmail-watch-push \
  --topic gog-gmail-watch \
  --push-endpoint "https://<public-url>/gmail-pubsub?token=<shared>"
```

生产环境：使用稳定的 HTTPS 端点并配置 Pub/Sub OIDC JWT，然后运行：

```bash
gog gmail watch serve --verify-oidc --oidc-email <svc@...>
```

## 测试

向被监听的收件箱发送一封邮件：

```bash
gog gmail send \
  --account openclaw@gmail.com \
  --to openclaw@gmail.com \
  --subject "watch test" \
  --body "ping"
```

检查 watch 状态与历史记录：

```bash
gog gmail watch status --account openclaw@gmail.com
gog gmail history --account openclaw@gmail.com --since <historyId>
```

## 故障排查

- `Invalid topicName`：项目不一致（topic 不在 OAuth 客户端项目中）。
- `User not authorized`：topic 缺少 `roles/pubsub.publisher`。
- 空消息：Gmail push 只提供 `historyId`；请通过 `gog gmail history` 获取。

## 清理

```bash
gog gmail watch stop --account openclaw@gmail.com
gcloud pubsub subscriptions delete gog-gmail-watch-push
gcloud pubsub topics delete gog-gmail-watch
```

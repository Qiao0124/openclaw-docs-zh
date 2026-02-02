---
summary: "OpenClaw 的可选基于 Docker 的设置和引导"
read_when:
  - 你想要容器化网关而不是本地安装
  - 你正在验证 Docker 流程
title: "Docker"
---

# Docker（可选）

Docker 是**可选的**。仅当你想要容器化网关或验证 Docker 流程时才使用它。

## Docker 适合我吗？

- **是**：你想要一个隔离的、可丢弃的网关环境，或在没有本地安装的主机上运行 OpenClaw。
- **否**：你在自己的机器上运行，只想要最快的开发循环。请改用普通安装流程。
- **沙盒说明**：agent 沙盒也使用 Docker，但它**不**需要完整的网关在 Docker 中运行。参见[沙盒](/gateway/sandboxing)。

本指南涵盖：

- 容器化网关（Docker 中的完整 OpenClaw）
- 每会话 Agent 沙盒（主机网关 + Docker 隔离的 agent 工具）

沙盒详情：[沙盒](/gateway/sandboxing)

## 要求

- Docker Desktop（或 Docker Engine）+ Docker Compose v2
- 足够的磁盘空间用于镜像 + 日志

## 容器化网关（Docker Compose）

### 快速开始（推荐）

从仓库根目录：

```bash
./docker-setup.sh
```

此脚本：

- 构建网关镜像
- 运行引导向导
- 打印可选的提供商设置提示
- 通过 Docker Compose 启动网关
- 生成网关令牌并写入 `.env`

可选环境变量：

- `OPENCLAW_DOCKER_APT_PACKAGES` — 构建期间安装额外的 apt 包
- `OPENCLAW_EXTRA_MOUNTS` — 添加额外的主机绑定挂载
- `OPENCLAW_HOME_VOLUME` — 在命名卷中持久化 `/home/node`

完成后：

- 在浏览器中打开 `http://127.0.0.1:18789/`。
- 将令牌粘贴到控制 UI 中（设置 → 令牌）。

它在主机上写入配置/工作区：

- `~/.openclaw/`
- `~/.openclaw/workspace`

在 VPS 上运行？参见 [Hetzner (Docker VPS)](/platforms/hetzner)。

### 手动流程（compose）

```bash
docker build -t openclaw:local -f Dockerfile .
docker compose run --rm openclaw-cli onboard
docker compose up -d openclaw-gateway
```

### 额外挂载（可选）

如果你想将额外的主机目录挂载到容器中，在运行 `docker-setup.sh` 之前设置
`OPENCLAW_EXTRA_MOUNTS`。这接受逗号分隔的 Docker 绑定挂载列表，并将它们应用到
`openclaw-gateway` 和 `openclaw-cli`，通过生成 `docker-compose.extra.yml`。

示例：

```bash
export OPENCLAW_EXTRA_MOUNTS="$HOME/.codex:/home/node/.codex:ro,$HOME/github:/home/node/github:rw"
./docker-setup.sh
```

注意：

- 路径必须与 Docker Desktop 在 macOS/Windows 上共享。
- 如果你编辑 `OPENCLAW_EXTRA_MOUNTS`，重新运行 `docker-setup.sh` 以重新生成
  额外的 compose 文件。
- `docker-compose.extra.yml` 是生成的。不要手动编辑它。

### 持久化整个容器 home（可选）

如果你想让 `/home/node` 在容器重建后持久化，通过 `OPENCLAW_HOME_VOLUME` 设置一个命名
卷。这会创建一个 Docker 卷并将其挂载到 `/home/node`，同时保留标准的配置/工作区绑定挂载。使用
命名卷（不是绑定路径）；对于绑定挂载，使用 `OPENCLAW_EXTRA_MOUNTS`。

示例：

```bash
export OPENCLAW_HOME_VOLUME="openclaw_home"
./docker-setup.sh
```

你可以将其与额外挂载结合：

```bash
export OPENCLAW_HOME_VOLUME="openclaw_home"
export OPENCLAW_EXTRA_MOUNTS="$HOME/.codex:/home/node/.codex:ro,$HOME/github:/home/node/github:rw"
./docker-setup.sh
```

注意：

- 如果你更改 `OPENCLAW_HOME_VOLUME`，重新运行 `docker-setup.sh` 以重新生成
  额外的 compose 文件。
- 命名卷会一直保留，直到你用 `docker volume rm <name>` 删除它。

### 安装额外的 apt 包（可选）

如果你需要在镜像内安装系统包（例如，构建工具或媒体库），在运行 `docker-setup.sh` 之前设置 `OPENCLAW_DOCKER_APT_PACKAGES`。这会在镜像构建期间安装包，因此即使容器被删除，它们也会保留。

示例：

```bash
export OPENCLAW_DOCKER_APT_PACKAGES="ffmpeg build-essential"
./docker-setup.sh
```

注意：

- 这接受空格分隔的 apt 包名列表。
- 如果你更改 `OPENCLAW_DOCKER_APT_PACKAGES`，重新运行 `docker-setup.sh` 以重新构建
  镜像。

### 更快的重建（推荐）

为了加速重建，请按顺序排列你的 Dockerfile，以便缓存依赖层。
这可以避免在 lockfile 更改时重新运行 `pnpm install`：

```dockerfile
FROM node:22-bookworm

# 安装 Bun（构建脚本必需）
RUN curl -fsSL https://bun.sh/install | bash
ENV PATH="/root/.bun/bin:${PATH}"

RUN corepack enable

WORKDIR /app

# 缓存依赖，除非包元数据更改
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

ENV NODE_ENV=production

CMD ["node","dist/index.js"]
```

### 频道设置（可选）

使用 CLI 容器配置频道，然后在需要时重启网关。

WhatsApp（QR）：

```bash
docker compose run --rm openclaw-cli channels login
```

Telegram（bot 令牌）：

```bash
docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"
```

Discord（bot 令牌）：

```bash
docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
```

文档：[WhatsApp](/channels/whatsapp)、[Telegram](/channels/telegram)、[Discord](/channels/discord)

### 健康检查

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### E2E 冒烟测试（Docker）

```bash
scripts/e2e/onboard-docker.sh
```

### QR 导入冒烟测试（Docker）

```bash
pnpm test:docker:qr
```

### 说明

- 网关绑定默认为 `lan` 以用于容器。
- 网关容器是会话的真相来源（`~/.openclaw/agents/<agentId>/sessions/`）。

## Agent 沙盒（主机网关 + Docker 工具）

深入内容：[沙盒](/gateway/sandboxing)

### 它的作用

当启用 `agents.defaults.sandbox` 时，**非主会话**在 Docker 容器内运行工具。网关保留在你的主机上，但工具执行是隔离的：

- 范围：默认 `"agent"`（每个 agent 一个容器 + 工作区）
- 范围：`"session"` 用于每会话隔离
- 每范围工作区文件夹挂载在 `/workspace`
- 可选的 agent 工作区访问（`agents.defaults.sandbox.workspaceAccess`）
- 允许/拒绝工具策略（拒绝优先）
- 入站媒体被复制到活动沙盒工作区（`media/inbound/*`）以便工具可以读取它（使用 `workspaceAccess: "rw"`，这会进入 agent 工作区）

警告：`scope: "shared"` 禁用跨会话隔离。所有会话共享一个容器和一个工作区。

### 每 agent 沙盒配置文件（多 agent）

如果你使用多 agent 路由，每个 agent 可以覆盖沙盒 + 工具设置：
`agents.list[].sandbox` 和 `agents.list[].tools`（以及 `agents.list[].tools.sandbox.tools`）。这允许你在一个网关中运行混合访问级别：

- 完全访问（个人 agent）
- 只读工具 + 只读工作区（家庭/工作 agent）
- 无文件系统/shell 工具（公共 agent）

参见[多 Agent 沙盒和工具](/multi-agent-sandbox-tools)获取示例、优先级和故障排除。

### 默认行为

- 镜像：`openclaw-sandbox:bookworm-slim`
- 每个 agent 一个容器
- Agent 工作区访问：`workspaceAccess: "none"`（默认）使用 `~/.openclaw/sandboxes`
  - `"ro"` 将沙盒工作区保留在 `/workspace` 并以只读方式挂载 agent 工作区到 `/agent`（禁用 `write`/`edit`/`apply_patch`）
  - `"rw"` 以读/写方式挂载 agent 工作区到 `/workspace`
- 自动清理：空闲 > 24小时 或 年龄 > 7天
- 网络：默认 `none`（如果需要出站，请明确选择加入）
- 默认允许：`exec`、`process`、`read`、`write`、`edit`、`sessions_list`、`sessions_history`、`sessions_send`、`sessions_spawn`、`session_status`
- 默认拒绝：`browser`、`canvas`、`nodes`、`cron`、`discord`、`gateway`

### 启用沙盒

如果你计划在 `setupCommand` 中安装包，请注意：

- 默认 `docker.network` 是 `"none"`（无出站）。
- `readOnlyRoot: true` 阻止包安装。
- `user` 必须是 root 才能运行 `apt-get`（省略 `user` 或设置 `user: "0:0"`）。
  当 `setupCommand`（或 docker 配置）更改时，OpenClaw 会自动重新创建容器，
  除非容器是**最近使用过的**（约 5 分钟内）。热容器会记录警告，并显示确切的 `openclaw sandbox recreate ...` 命令。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        scope: "agent", // session | agent | shared（agent 是默认值）
        workspaceAccess: "none", // none | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
        },
        prune: {
          idleHours: 24, // 0 禁用空闲清理
          maxAgeDays: 7, // 0 禁用最大年龄清理
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

加固选项位于 `agents.defaults.sandbox.docker` 下：
`network`、`user`、`pidsLimit`、`memory`、`memorySwap`、`cpus`、`ulimits`、
`seccompProfile`、`apparmorProfile`、`dns`、`extraHosts`。

多 agent：通过 `agents.list[].sandbox.{docker,browser,prune}.*` 覆盖 `agents.defaults.sandbox.{docker,browser,prune}.*`
（当 `agents.defaults.sandbox.scope` / `agents.list[].sandbox.scope` 为 `"shared"` 时忽略）。

### 构建默认沙盒镜像

```bash
scripts/sandbox-setup.sh
```

这使用 `Dockerfile.sandbox` 构建 `openclaw-sandbox:bookworm-slim`。

### 沙盒通用镜像（可选）

如果你想要一个带有通用构建工具（Node、Go、Rust 等）的沙盒镜像，构建通用镜像：

```bash
scripts/sandbox-common-setup.sh
```

这构建 `openclaw-sandbox-common:bookworm-slim`。使用它：

```json5
{
  agents: {
    defaults: {
      sandbox: { docker: { image: "openclaw-sandbox-common:bookworm-slim" } },
    },
  },
}
```

### 沙盒浏览器镜像

要在沙盒内运行浏览器工具，构建浏览器镜像：

```bash
scripts/sandbox-browser-setup.sh
```

这使用 `Dockerfile.sandbox-browser` 构建 `openclaw-sandbox-browser:bookworm-slim`。容器运行启用了 CDP 的 Chromium 和可选的 noVNC 观察器（通过 Xvfb 有头模式）。

注意：

- 有头模式（Xvfb）比无头模式减少机器人拦截。
- 仍然可以通过设置 `agents.defaults.sandbox.browser.headless=true` 使用无头模式。
- 不需要完整的桌面环境（GNOME）；Xvfb 提供显示。

使用配置：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        browser: { enabled: true },
      },
    },
  },
}
```

自定义浏览器镜像：

```json5
{
  agents: {
    defaults: {
      sandbox: { browser: { image: "my-openclaw-browser" } },
    },
  },
}
```

启用后，agent 接收：

- 一个沙盒浏览器控制 URL（用于 `browser` 工具）
- 一个 noVNC URL（如果启用且 headless=false）

记住：如果你使用允许列表工具，添加 `browser`（并从拒绝列表中移除它），否则该工具仍会被阻止。
清理规则（`agents.defaults.sandbox.prune`）也适用于浏览器容器。

### 自定义沙盒镜像

构建你自己的镜像并在配置中指向它：

```bash
docker build -t my-openclaw-sbx -f Dockerfile.sandbox .
```

```json5
{
  agents: {
    defaults: {
      sandbox: { docker: { image: "my-openclaw-sbx" } },
    },
  },
}
```

### 工具策略（允许/拒绝）

- `deny` 优先于 `allow`。
- 如果 `allow` 为空：所有工具（除了拒绝的）都可用。
- 如果 `allow` 不为空：只有 `allow` 中的工具可用（减去拒绝的）。

### 清理策略

两个选项：

- `prune.idleHours`：移除 X 小时未使用的容器（0 = 禁用）
- `prune.maxAgeDays`：移除超过 X 天的容器（0 = 禁用）

示例：

- 保持繁忙会话但限制生命周期：
  `idleHours: 24`、`maxAgeDays: 7`
- 从不清理：
  `idleHours: 0`、`maxAgeDays: 0`

### 安全说明

- 硬隔离仅适用于**工具**（exec/read/write/edit/apply_patch）。
- 仅主机工具如 browser/camera/canvas 默认被阻止。
- 在沙盒中允许 `browser`**会破坏隔离**（浏览器在主机上运行）。

## 故障排除

- 镜像缺失：使用 [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh) 构建或设置 `agents.defaults.sandbox.docker.image`。
- 容器未运行：它会按需按会话自动创建。
- 沙盒中的权限错误：将 `docker.user` 设置为与你挂载的工作区所有权匹配的 UID:GID（或 chown 工作区文件夹）。
- 自定义工具未找到：OpenClaw 使用 `sh -lc`（登录 shell）运行命令，它会读取 `/etc/profile` 并可能重置 PATH。设置 `docker.env.PATH` 以预置你的自定义工具路径（例如 `/custom/bin:/usr/local/share/npm-global/bin`），或在 Dockerfile 中的 `/etc/profile.d/` 下添加脚本。

---
summary: "可选的基于 Docker 的 OpenClaw 安装与引导"
read_when:
  - 你想用容器化网关而不是本地安装
  - 你在验证 Docker 流程
title: "Docker（Docker）"
---

# Docker（可选）

Docker 是 **可选** 的。仅当你需要容器化网关或想验证 Docker 流程时再使用。

## Docker 适合我吗？（Is Docker right for me?）

- **是**：你想要隔离、可随时丢弃的网关环境，或在无法本地安装的主机上运行 OpenClaw。
- **否**：你在自己的机器上工作，只想最快的开发循环。请使用常规安装流程。
- **沙箱提示**：代理沙箱同样使用 Docker，但 **不要求** 网关本体跑在 Docker 内。详见 [Sandboxing](/gateway/sandboxing)。

本指南覆盖：

- 容器化网关（Docker 内完整 OpenClaw）
- 单会话代理沙箱（宿主网关 + Docker 隔离工具）

沙箱细节：[Sandboxing](/gateway/sandboxing)

## 要求（Requirements）

- Docker Desktop（或 Docker Engine）+ Docker Compose v2
- 足够的磁盘空间用于镜像与日志

## 容器化网关（Docker Compose）

### 快速开始（推荐）

在仓库根目录：

```bash
./docker-setup.sh
```

该脚本会：

- 构建网关镜像
- 运行引导向导
- 打印可选的提供方配置提示
- 通过 Docker Compose 启动网关
- 生成网关 token 并写入 `.env`

可选环境变量：

- `OPENCLAW_DOCKER_APT_PACKAGES` — 构建时安装额外 apt 包
- `OPENCLAW_EXTRA_MOUNTS` — 添加额外的宿主机挂载
- `OPENCLAW_HOME_VOLUME` — 用命名卷持久化 `/home/node`

完成后：

- 在浏览器打开 `http://127.0.0.1:18789/`。
- 在 Control UI 中粘贴 token（Settings → token）。

配置/工作区写在宿主机上：

- `~/.openclaw/`
- `~/.openclaw/workspace`

在 VPS 上运行？见 [Hetzner (Docker VPS)](/platforms/hetzner)。

### 手动流程（compose）

```bash
docker build -t openclaw:local -f Dockerfile .
docker compose run --rm openclaw-cli onboard
docker compose up -d openclaw-gateway
```

### 额外挂载（可选）

如果你想把更多宿主机目录挂载进容器，请在运行 `docker-setup.sh` 前设置
`OPENCLAW_EXTRA_MOUNTS`。它接受逗号分隔的 Docker bind mount 列表，并通过生成
`docker-compose.extra.yml` 应用到 `openclaw-gateway` 与 `openclaw-cli`。

示例：

```bash
export OPENCLAW_EXTRA_MOUNTS="$HOME/.codex:/home/node/.codex:ro,$HOME/github:/home/node/github:rw"
./docker-setup.sh
```

说明：

- macOS/Windows 上路径必须在 Docker Desktop 中共享。
- 修改 `OPENCLAW_EXTRA_MOUNTS` 后需重新运行 `docker-setup.sh` 以重生成额外 compose 文件。
- `docker-compose.extra.yml` 为自动生成，请勿手动编辑。

### 持久化整个容器 home（可选）

如需让 `/home/node` 在容器重建后仍持久，设置 `OPENCLAW_HOME_VOLUME` 以使用命名卷。
这会创建 Docker volume 并挂载到 `/home/node`，同时保留标准的配置/工作区 bind mount。
这里请使用命名卷（不是 bind path）；bind mount 请用 `OPENCLAW_EXTRA_MOUNTS`。

示例：

```bash
export OPENCLAW_HOME_VOLUME="openclaw_home"
./docker-setup.sh
```

可与额外挂载组合使用：

```bash
export OPENCLAW_HOME_VOLUME="openclaw_home"
export OPENCLAW_EXTRA_MOUNTS="$HOME/.codex:/home/node/.codex:ro,$HOME/github:/home/node/github:rw"
./docker-setup.sh
```

说明：

- 修改 `OPENCLAW_HOME_VOLUME` 后需重新运行 `docker-setup.sh` 以重生成额外 compose 文件。
- 命名卷会一直保留，直到用 `docker volume rm <name>` 删除。

### 安装额外 apt 包（可选）

如果镜像内需要系统包（如构建工具或媒体库），在运行 `docker-setup.sh` 前设置
`OPENCLAW_DOCKER_APT_PACKAGES`。这些包会在镜像构建时安装，因此容器删除后也能保留。

示例：

```bash
export OPENCLAW_DOCKER_APT_PACKAGES="ffmpeg build-essential"
./docker-setup.sh
```

说明：

- 该变量接受空格分隔的 apt 包名。
- 修改 `OPENCLAW_DOCKER_APT_PACKAGES` 后需重新运行 `docker-setup.sh` 以重建镜像。

### 更快的重建（推荐）

为加速重建，调整 Dockerfile 的层顺序以缓存依赖。这可避免在 lockfile 未变时重复运行 `pnpm install`：

```dockerfile
FROM node:22-bookworm

# Install Bun (required for build scripts)
RUN curl -fsSL https://bun.sh/install | bash
ENV PATH="/root/.bun/bin:${PATH}"

RUN corepack enable

WORKDIR /app

# Cache dependencies unless package metadata changes
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

### 渠道配置（可选）

使用 CLI 容器配置渠道，必要时重启网关。

WhatsApp（二维码）：

```bash
docker compose run --rm openclaw-cli channels login
```

Telegram（bot token）：

```bash
docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"
```

Discord（bot token）：

```bash
docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
```

文档：[WhatsApp](/channels/whatsapp)、[Telegram](/channels/telegram)、[Discord](/channels/discord)

### 健康检查（Health check）

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### E2E 烟雾测试（Docker）

```bash
scripts/e2e/onboard-docker.sh
```

### QR 导入烟雾测试（Docker）

```bash
pnpm test:docker:qr
```

### 说明（Notes）

- 容器使用时，网关默认绑定 `lan`。
- 网关容器是会话的唯一事实来源（`~/.openclaw/agents/<agentId>/sessions/`）。

## 代理沙箱（宿主网关 + Docker 工具）

深入说明：[Sandboxing](/gateway/sandboxing)

### 做什么（What it does）

启用 `agents.defaults.sandbox` 后，**非 main 会话** 会在 Docker 容器中运行工具。网关仍在宿主机，但工具执行被隔离：

- scope：默认 `"agent"`（每个 agent 一个容器 + 工作区）
- scope：`"session"` 用于按会话隔离
- 每个 scope 的工作区目录挂载到 `/workspace`
- 可选的代理工作区访问（`agents.defaults.sandbox.workspaceAccess`）
- 工具 allow/deny 策略（deny 优先）
- 入站媒体会复制到当前沙箱工作区（`media/inbound/*`）供工具读取（`workspaceAccess: "rw"` 时会落到代理工作区）

警告：`scope: "shared"` 会禁用跨会话隔离。所有会话共享一个容器与一个工作区。

### 每个 agent 的沙箱配置（多代理）

如果使用多代理路由，每个 agent 都可以覆盖沙箱 + 工具设置：
`agents.list[].sandbox` 和 `agents.list[].tools`（以及 `agents.list[].tools.sandbox.tools`）。
这样你可以在同一网关中混合不同访问级别：

- 全访问（个人代理）
- 只读工具 + 只读工作区（家人/工作代理）
- 禁用文件系统/终端工具（公共代理）

示例、优先级与排障见：[Multi-Agent Sandbox & Tools](/multi-agent-sandbox-tools)。

### 默认行为（Default behavior）

- 镜像：`openclaw-sandbox:bookworm-slim`
- 每个 agent 一个容器
- 代理工作区访问：`workspaceAccess: "none"`（默认）使用 `~/.openclaw/sandboxes`
  - `"ro"`：保持沙箱工作区在 `/workspace`，并将代理工作区只读挂载到 `/agent`（禁用 `write`/`edit`/`apply_patch`）
  - `"rw"`：将代理工作区读写挂载到 `/workspace`
- 自动清理：空闲 > 24h **或** 年龄 > 7d
- 网络：默认 `none`（需要外网时显式开启）
- 默认允许：`exec`, `process`, `read`, `write`, `edit`, `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `session_status`
- 默认拒绝：`browser`, `canvas`, `nodes`, `cron`, `discord`, `gateway`

### 启用沙箱（Enable sandboxing）

如果你计划在 `setupCommand` 中安装包，请注意：

- 默认 `docker.network` 为 `"none"`（无外网）。
- `readOnlyRoot: true` 会阻止安装包。
- `apt-get` 需要 root 用户（省略 `user` 或设为 `user: "0:0"`）。
  OpenClaw 会在 `setupCommand`（或 docker 配置）变化时自动重建容器，
  除非容器 **刚被使用**（约 5 分钟内）。热容器会记录一条警告，并给出精确的 `openclaw sandbox recreate ...` 命令。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        scope: "agent", // session | agent | shared (agent is default)
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
          idleHours: 24, // 0 disables idle pruning
          maxAgeDays: 7, // 0 disables max-age pruning
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

加固参数位于 `agents.defaults.sandbox.docker`：
`network`, `user`, `pidsLimit`, `memory`, `memorySwap`, `cpus`, `ulimits`,
`seccompProfile`, `apparmorProfile`, `dns`, `extraHosts`。

多代理：可通过 `agents.list[].sandbox.{docker,browser,prune}.*` 覆盖 `agents.defaults.sandbox.{docker,browser,prune}.*`
（当 `agents.defaults.sandbox.scope` / `agents.list[].sandbox.scope` 为 `"shared"` 时忽略）。

### 构建默认沙箱镜像（Build the default sandbox image）

```bash
scripts/sandbox-setup.sh
```

这会使用 `Dockerfile.sandbox` 构建 `openclaw-sandbox:bookworm-slim`。

### 通用沙箱镜像（可选）（Sandbox common image）

如果你想要包含常用构建工具（Node、Go、Rust 等）的沙箱镜像，可构建通用镜像：

```bash
scripts/sandbox-common-setup.sh
```

这会构建 `openclaw-sandbox-common:bookworm-slim`。使用方式：

```json5
{
  agents: {
    defaults: {
      sandbox: { docker: { image: "openclaw-sandbox-common:bookworm-slim" } },
    },
  },
}
```

### 沙箱浏览器镜像（Sandbox browser image）

要在沙箱中运行 browser 工具，构建浏览器镜像：

```bash
scripts/sandbox-browser-setup.sh
```

这会使用 `Dockerfile.sandbox-browser` 构建 `openclaw-sandbox-browser:bookworm-slim`。容器会运行启用 CDP 的 Chromium，
并可选提供 noVNC 观察器（Xvfb 的有头模式）。

说明：

- 有头（Xvfb）相比无头更不易被封。
- 仍可通过 `agents.defaults.sandbox.browser.headless=true` 使用无头模式。
- 不需要完整桌面环境（GNOME）；Xvfb 提供显示即可。

配置示例：

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

启用后，agent 会获得：

- 沙箱浏览器控制 URL（用于 `browser` 工具）
- noVNC URL（如果启用且 headless=false）

提醒：如果你使用工具白名单，请把 `browser` 加入 allow（并从 deny 中移除），否则工具仍会被阻止。
`agents.defaults.sandbox.prune` 的清理规则也适用于浏览器容器。

### 自定义沙箱镜像（Custom sandbox image）

构建自己的镜像并在配置中引用：

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

### 工具策略（allow/deny）

- `deny` 优先级高于 `allow`。
- 如果 `allow` 为空：除 deny 外全部可用。
- 如果 `allow` 非空：仅 `allow` 中的工具可用（再减去 deny）。

### 清理策略（Pruning strategy）

两个参数：

- `prune.idleHours`：移除 X 小时未使用的容器（0 = 禁用）
- `prune.maxAgeDays`：移除超过 X 天的容器（0 = 禁用）

示例：

- 保留活跃会话但限制最长寿命：
  `idleHours: 24`, `maxAgeDays: 7`
- 永不清理：
  `idleHours: 0`, `maxAgeDays: 0`

### 安全说明（Security notes）

- 硬隔离仅适用于 **工具**（exec/read/write/edit/apply_patch）。
- browser/camera/canvas 等宿主工具默认被阻止。
- 在沙箱允许 `browser` 会 **打破隔离**（browser 运行在宿主机）。

## 故障排查（Troubleshooting）

- 镜像缺失：使用 [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh) 构建，或设置 `agents.defaults.sandbox.docker.image`。
- 容器未运行：会在需要时按会话自动创建。
- 沙箱权限错误：把 `docker.user` 设为与你挂载工作区一致的 UID:GID（或对工作区目录执行 chown）。
- 自定义工具找不到：OpenClaw 用 `sh -lc`（login shell）运行命令，会读取 `/etc/profile` 并可能重置 PATH。请设置 `docker.env.PATH` 以追加你的工具路径（如 `/custom/bin:/usr/local/share/npm-global/bin`），或在 Dockerfile 中添加 `/etc/profile.d/` 脚本。

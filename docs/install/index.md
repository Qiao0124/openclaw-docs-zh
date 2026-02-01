---
summary: "安装 OpenClaw（推荐安装器、全局安装或源码安装）"
read_when:
  - 安装 OpenClaw
  - 你想从 GitHub 安装
title: "安装（Install）"
---

# 安装（Install）

除非你有明确理由，否则请使用安装器。它会安装 CLI 并运行引导。

## 快速安装（推荐）

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Windows（PowerShell）：

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

下一步（如果你跳过了引导）：

```bash
openclaw onboard --install-daemon
```

## 系统要求（System requirements）

- **Node >=22**
- macOS、Linux，或通过 WSL2 的 Windows
- 仅在从源码构建时需要 `pnpm`

## 选择安装方式（Choose your install path）

### 1) 安装脚本（推荐）

通过 npm 全局安装 `openclaw` 并运行引导。

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

安装器参数：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --help
```

细节：[安装器原理](/install/installer)。

非交互（跳过引导）：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
```

### 2) 全局安装（手动）

如果你已经有 Node：

```bash
npm install -g openclaw@latest
```

如果你全局安装了 libvips（macOS 上 Homebrew 很常见）且 `sharp` 安装失败，强制使用预构建二进制：

```bash
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@latest
```

如果看到 `sharp: Please add node-gyp to your dependencies`，要么安装构建工具（macOS：Xcode CLT + `npm install -g node-gyp`），要么使用上面的 `SHARP_IGNORE_GLOBAL_LIBVIPS=1` 规避原生构建。

或：

```bash
pnpm add -g openclaw@latest
```

然后：

```bash
openclaw onboard --install-daemon
```

### 3) 从源码安装（贡献者/开发者）

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build # auto-installs UI deps on first run
pnpm build
openclaw onboard --install-daemon
```

提示：如果还没有全局安装，可在仓库内用 `pnpm openclaw ...` 运行命令。

### 4) 其他安装方式

- Docker：[Docker](/install/docker)
- Nix：[Nix](/install/nix)
- Ansible：[Ansible](/install/ansible)
- Bun（仅 CLI）：[Bun](/install/bun)

## 安装后（After install）

- 运行引导：`openclaw onboard --install-daemon`
- 快速检查：`openclaw doctor`
- 检查网关健康：`openclaw status` + `openclaw health`
- 打开仪表盘：`openclaw dashboard`

## 安装方式：npm vs git（安装器）

安装器支持两种方式：

- `npm`（默认）：`npm install -g openclaw@latest`
- `git`：从 GitHub clone/build，并从源码目录运行

### CLI 参数（CLI flags）

```bash
# 明确使用 npm
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm

# 从 GitHub 安装（源码目录）
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
```

常用参数：

- `--install-method npm|git`
- `--git-dir <path>`（默认：`~/openclaw`）
- `--no-git-update`（使用现有 checkout 时跳过 `git pull`）
- `--no-prompt`（禁用提示；CI/自动化必需）
- `--dry-run`（仅打印将执行的操作，不做更改）
- `--no-onboard`（跳过引导）

### 环境变量（Environment variables）

等价环境变量（适合自动化）：

- `OPENCLAW_INSTALL_METHOD=git|npm`
- `OPENCLAW_GIT_DIR=...`
- `OPENCLAW_GIT_UPDATE=0|1`
- `OPENCLAW_NO_PROMPT=1`
- `OPENCLAW_DRY_RUN=1`
- `OPENCLAW_NO_ONBOARD=1`
- `SHARP_IGNORE_GLOBAL_LIBVIPS=0|1`（默认：`1`；避免 `sharp` 链接系统 libvips）

## 排查：找不到 `openclaw`（PATH）

快速诊断：

```bash
node -v
npm -v
npm prefix -g
echo "$PATH"
```

如果 `$(npm prefix -g)/bin`（macOS/Linux）或 `$(npm prefix -g)`（Windows）**不在** `echo "$PATH"` 里，你的 shell 就找不到全局 npm 的二进制（包括 `openclaw`）。

修复：把它加到 shell 启动文件里（zsh：`~/.zshrc`，bash：`~/.bashrc`）：

```bash
# macOS / Linux
export PATH="$(npm prefix -g)/bin:$PATH"
```

Windows：把 `npm prefix -g` 的输出加入 PATH。

然后打开新的终端（或在 zsh 里 `rehash` / bash 里 `hash -r`）。

## 更新 / 卸载（Update / uninstall）

- 更新：[Updating](/install/updating)
- 迁移到新机器：[Migrating](/install/migrating)
- 卸载：[Uninstall](/install/uninstall)

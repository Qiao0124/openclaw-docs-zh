---
summary: "安装器脚本的工作方式（install.sh + install-cli.sh）、参数与自动化"
read_when:
  - 你想了解 `openclaw.ai/install.sh`
  - 你想自动化安装（CI / 无人值守）
  - 你想从 GitHub checkout 安装
title: "安装器原理（Installer Internals）"
---

# 安装器原理（Installer Internals）

OpenClaw 提供三个安装脚本（由 `openclaw.ai` 提供）：

- `https://openclaw.ai/install.sh` — “推荐”安装器（默认全局 npm 安装；也可从 GitHub checkout 安装）
- `https://openclaw.ai/install-cli.sh` — 免 root 的 CLI 安装器（安装到自定义前缀，并自带 Node）
- `https://openclaw.ai/install.ps1` — Windows PowerShell 安装器（默认 npm；可选 git 安装）

查看当前参数与行为：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --help
```

Windows（PowerShell）帮助：

```powershell
& ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -?
```

如果安装完成后新终端里找不到 `openclaw`，通常是 Node/npm PATH 问题。见：[Install](/install#nodejs--npm-path-sanity)。

## install.sh（推荐）

它做了什么（高层）：

- 识别 OS（macOS / Linux / WSL）。
- 确保 Node.js **22+**（macOS 走 Homebrew；Linux 走 NodeSource）。
- 选择安装方式：
  - `npm`（默认）：`npm install -g openclaw@latest`
  - `git`：clone/build 源码并安装包装脚本
- Linux：在需要时把 npm 前缀切到 `~/.npm-global`，避免全局权限问题。
- 如果升级已有安装：尝试运行 `openclaw doctor --non-interactive`。
- git 安装：在 install/update 后运行 `openclaw doctor --non-interactive`（尽力而为）。
- 通过默认 `SHARP_IGNORE_GLOBAL_LIBVIPS=1` 规避 `sharp` 原生编译问题（避免链接系统 libvips）。

如果你 _希望_ `sharp` 链接到系统已安装的 libvips（或你在调试），设置：

```bash
SHARP_IGNORE_GLOBAL_LIBVIPS=0 curl -fsSL https://openclaw.ai/install.sh | bash
```

### 可发现性 / “git install” 提示

如果你在 **OpenClaw 源码 checkout 内** 运行安装器（通过 `package.json` + `pnpm-workspace.yaml` 检测），它会提示：

- 更新并使用该 checkout（`git`）
- 或迁移到全局 npm 安装（`npm`）

在非交互场景（无 TTY / `--no-prompt`）下，必须传 `--install-method git|npm`（或设置 `OPENCLAW_INSTALL_METHOD`），否则脚本会以 code `2` 退出。

### 为什么需要 Git

`--install-method git` 路径需要 Git（clone / pull）。

对于 `npm` 安装，Git _通常_ 不需要，但某些环境仍会需要它（例如依赖来自 git URL）。安装器会确保 Git 存在，以避免在新系统上出现 `spawn git ENOENT`。

### 为什么 npm 在新 Linux 上会报 `EACCES`

在一些 Linux 环境里（尤其是通过系统包管理器或 NodeSource 安装 Node 的场景），npm 全局前缀指向 root 拥有的目录。这会导致 `npm install -g ...` 报 `EACCES` / `mkdir` 权限错误。

`install.sh` 会在需要时把前缀切换为：

- `~/.npm-global`（并在 `~/.bashrc` / `~/.zshrc` 中加入 PATH）

## install-cli.sh（免 root 的 CLI 安装器）

该脚本会把 `openclaw` 安装到一个前缀目录（默认：`~/.openclaw`），同时在该前缀下安装专用 Node 运行时，因此可在不改系统 Node/npm 的机器上使用。

帮助：

```bash
curl -fsSL https://openclaw.ai/install-cli.sh | bash -s -- --help
```

## install.ps1（Windows PowerShell）

它做了什么（高层）：

- 确保 Node.js **22+**（winget/Chocolatey/Scoop 或手动安装）。
- 选择安装方式：
  - `npm`（默认）：`npm install -g openclaw@latest`
  - `git`：clone/build 源码并安装包装脚本
- 升级与 git 安装时运行 `openclaw doctor --non-interactive`（尽力而为）。

示例：

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex -InstallMethod git
```

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex -InstallMethod git -GitDir "C:\\openclaw"
```

环境变量：

- `OPENCLAW_INSTALL_METHOD=git|npm`
- `OPENCLAW_GIT_DIR=...`

Git 要求：

如果选择 `-InstallMethod git` 且未安装 Git，安装器会打印 Git for Windows 链接（`https://git-scm.com/download/win`）并退出。

常见 Windows 问题：

- **npm error spawn git / ENOENT**：安装 Git for Windows，重开 PowerShell，再跑安装器。
- **"openclaw" is not recognized**：npm 全局 bin 未加入 PATH。大多数系统是 `%AppData%\\npm`。你也可运行 `npm config get prefix`，把 `\\bin` 加入 PATH 后重开 PowerShell。

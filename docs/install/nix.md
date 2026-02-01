---
summary: "用 Nix 声明式安装 OpenClaw"
read_when:
  - 你想要可复现、可回滚的安装
  - 你已经在用 Nix/NixOS/Home Manager
  - 你希望一切都可锁定并声明式管理
title: "Nix 安装（Nix Installation）"
---

# Nix 安装（Nix Installation）

使用 Nix 运行 OpenClaw 的推荐方式是 **[nix-openclaw](https://github.com/openclaw/nix-openclaw)** —— 一个开箱即用的 Home Manager 模块。

## 快速开始（Quick Start）

把这段话发给你的 AI 代理（Claude、Cursor 等）：

```text
I want to set up nix-openclaw on my Mac.
Repository: github:openclaw/nix-openclaw

What I need you to do:
1. Check if Determinate Nix is installed (if not, install it)
2. Create a local flake at ~/code/openclaw-local using templates/agent-first/flake.nix
3. Help me create a Telegram bot (@BotFather) and get my chat ID (@userinfobot)
4. Set up secrets (bot token, Anthropic key) - plain files at ~/.secrets/ is fine
5. Fill in the template placeholders and run home-manager switch
6. Verify: launchd running, bot responds to messages

Reference the nix-openclaw README for module options.
```

> **📦 完整指南：[github.com/openclaw/nix-openclaw](https://github.com/openclaw/nix-openclaw)**
>
> nix-openclaw 仓库是 Nix 安装的权威来源。本页仅作快速概览。

## 你将获得（What you get）

- Gateway + macOS app + 工具（whisper、spotify、cameras）—— 全部锁定
- 可跨重启持久的 Launchd 服务
- 具备声明式配置的插件系统
- 立即回滚：`home-manager switch --rollback`

---

## Nix 模式运行时行为（Nix Mode Runtime Behavior）

当设置 `OPENCLAW_NIX_MODE=1`（nix-openclaw 会自动设置）：

OpenClaw 启用 **Nix 模式**，使配置具备确定性并禁用自动安装流程。
通过环境变量启用：

```bash
OPENCLAW_NIX_MODE=1
```

在 macOS 上，GUI 应用不会自动继承 shell 环境变量。你也可以通过 defaults 启用 Nix 模式：

```bash
defaults write bot.molt.mac openclaw.nixMode -bool true
```

### 配置与状态路径（Config + state paths）

OpenClaw 从 `OPENCLAW_CONFIG_PATH` 读取 JSON5 配置，并把可变数据写到 `OPENCLAW_STATE_DIR`。

- `OPENCLAW_STATE_DIR`（默认：`~/.openclaw`）
- `OPENCLAW_CONFIG_PATH`（默认：`$OPENCLAW_STATE_DIR/openclaw.json`）

在 Nix 下运行时，请将它们显式设置到 Nix 管理的路径，避免运行时状态与配置写入不可变 store。

### Nix 模式下的运行行为（Runtime behavior in Nix mode）

- 自动安装与自修改流程被禁用
- 缺失依赖会显示 Nix 专用的修复提示
- UI 在检测到 Nix 模式时会显示只读横幅

## 打包说明（macOS）（Packaging note）

macOS 打包流程要求一个稳定的 Info.plist 模板，位于：

```
apps/macos/Sources/OpenClaw/Resources/Info.plist
```

[`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh) 会将该模板复制到应用包中，并补丁动态字段
（bundle ID、version/build、Git SHA、Sparkle keys）。这让 plist 在 SwiftPM 打包与 Nix 构建中保持确定性（不依赖完整 Xcode 工具链）。

## 相关内容（Related）

- [nix-openclaw](https://github.com/openclaw/nix-openclaw) — 完整安装指南
- [Wizard](/start/wizard) — 非 Nix 的 CLI 安装流程
- [Docker](/install/docker) — 容器化部署

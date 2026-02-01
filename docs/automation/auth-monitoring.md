---
summary: "监控模型供应商的 OAuth 过期状态"
read_when:
  - 设置认证过期监控或告警
  - 自动化 Claude Code / Codex 的 OAuth 刷新检查
title: "认证监控"
---

# 认证监控

OpenClaw 会通过 `openclaw models status` 暴露 OAuth 过期健康状态。用它做
自动化与告警；脚本只是手机流程的可选附加。

## 首选：CLI 检查（通用）

```bash
openclaw models status --check
```

退出码：

- `0`：正常
- `1`：已过期或缺失凭据
- `2`：即将过期（24 小时内）

这适用于 cron/systemd，无需额外脚本。

## 可选脚本（运维 / 手机流程）

这些脚本位于 `scripts/`，**可选**。它们假设你能通过 SSH 访问网关主机，
并针对 systemd + Termux 做了调整。

- `scripts/claude-auth-status.sh` 现在以 `openclaw models status --json` 为
  权威数据源（CLI 不可用时回退到直接读取文件），所以定时器需要确保
  `openclaw` 在 `PATH` 中。
- `scripts/auth-monitor.sh`：cron/systemd 定时器入口；发送告警（ntfy 或手机）。
- `scripts/systemd/openclaw-auth-monitor.{service,timer}`：systemd 用户定时器。
- `scripts/claude-auth-status.sh`：Claude Code + OpenClaw 认证检查器（full/json/simple）。
- `scripts/mobile-reauth.sh`：通过 SSH 引导重新认证流程。
- `scripts/termux-quick-auth.sh`：一键小组件状态 + 打开认证 URL。
- `scripts/termux-auth-widget.sh`：完整的小组件引导流程。
- `scripts/termux-sync-widget.sh`：同步 Claude Code 凭据 → OpenClaw。

如果你不需要手机自动化或 systemd 定时器，可以跳过这些脚本。

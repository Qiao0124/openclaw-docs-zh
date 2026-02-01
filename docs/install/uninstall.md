---
summary: "完整卸载 OpenClaw（CLI、服务、状态、工作区）"
read_when:
  - 你想从机器上移除 OpenClaw
  - 卸载后网关服务仍在运行
title: "卸载（Uninstall）"
---

# 卸载（Uninstall）

两条路径：

- **简易路径**：`openclaw` 仍在。
- **手动移除服务**：CLI 已没了，但服务还在运行。

## 简易路径（CLI 仍在）

推荐：使用内置卸载器：

```bash
openclaw uninstall
```

非交互（自动化 / npx）：

```bash
openclaw uninstall --all --yes --non-interactive
npx -y openclaw uninstall --all --yes --non-interactive
```

手动步骤（同样结果）：

1. 停止网关服务：

```bash
openclaw gateway stop
```

2. 卸载网关服务（launchd/systemd/schtasks）：

```bash
openclaw gateway uninstall
```

3. 删除状态与配置：

```bash
rm -rf "${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"
```

如果你把 `OPENCLAW_CONFIG_PATH` 设置到状态目录之外，还要删除该配置文件。

4. 删除工作区（可选，会移除代理文件）：

```bash
rm -rf ~/.openclaw/workspace
```

5. 删除 CLI 安装（选择你使用的方式）：

```bash
npm rm -g openclaw
pnpm remove -g openclaw
bun remove -g openclaw
```

6. 如果安装了 macOS 应用：

```bash
rm -rf /Applications/OpenClaw.app
```

说明：

- 如果使用了 profile（`--profile` / `OPENCLAW_PROFILE`），对每个状态目录重复步骤 3（默认 `~/.openclaw-<profile>`）。
- 远程模式下，状态目录在 **网关主机** 上，步骤 1-4 需要在那台机器上执行。

## 手动移除服务（CLI 已卸载）

适用于网关服务仍在运行但 `openclaw` 已缺失的情况。

### macOS（launchd）

默认 label 为 `bot.molt.gateway`（或 `bot.molt.<profile>`；旧的 `com.openclaw.*` 可能仍存在）：

```bash
launchctl bootout gui/$UID/bot.molt.gateway
rm -f ~/Library/LaunchAgents/bot.molt.gateway.plist
```

如果你用了 profile，将 label 与 plist 名替换为 `bot.molt.<profile>`。如果存在旧的 `com.openclaw.*` plist，也请删除。

### Linux（systemd 用户单元）

默认单元名为 `openclaw-gateway.service`（或 `openclaw-gateway-<profile>.service`）：

```bash
systemctl --user disable --now openclaw-gateway.service
rm -f ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload
```

### Windows（计划任务）

默认任务名为 `OpenClaw Gateway`（或 `OpenClaw Gateway (<profile>)`）。
任务脚本位于你的状态目录中。

```powershell
schtasks /Delete /F /TN "OpenClaw Gateway"
Remove-Item -Force "$env:USERPROFILE\.openclaw\gateway.cmd"
```

如果你使用了 profile，请删除对应任务名和 `~\.openclaw-<profile>\gateway.cmd`。

## 普通安装 vs 源码 checkout

### 普通安装（install.sh / npm / pnpm / bun）

如果你使用 `https://openclaw.ai/install.sh` 或 `install.ps1`，CLI 是通过 `npm install -g openclaw@latest` 安装的。
卸载时运行 `npm rm -g openclaw`（或 `pnpm remove -g` / `bun remove -g`）。

### 源码 checkout（git clone）

如果你从源码目录运行（`git clone` + `openclaw ...` / `bun run openclaw ...`）：

1. 删除仓库前先卸载网关服务（用上面的简易路径或手动移除服务）。
2. 删除仓库目录。
3. 按上文删除状态与工作区。

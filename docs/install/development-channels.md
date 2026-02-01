---
summary: "stable、beta、dev 通道：语义、切换与标记"
read_when:
  - 你想在 stable/beta/dev 间切换
  - 你在打 tag 或发布预发布版本
title: "开发通道（Development Channels）"
---

# 开发通道（Development Channels）

最近更新：2026-01-21

OpenClaw 提供三条更新通道：

- **stable**：npm dist-tag `latest`。
- **beta**：npm dist-tag `beta`（测试中的构建）。
- **dev**：`main` 的滚动头（git）。npm dist-tag：`dev`（发布时）。

我们先发布到 **beta**，测试后再将已验证构建 **提升到 `latest`**，
不改版本号 —— npm 安装以 dist-tag 为准。

## 切换通道（Switching channels）

Git checkout：

```bash
openclaw update --channel stable
openclaw update --channel beta
openclaw update --channel dev
```

- `stable`/`beta` 会切到最新匹配 tag（通常是同一个 tag）。
- `dev` 切到 `main` 并在上游上 rebase。

npm/pnpm 全局安装：

```bash
openclaw update --channel stable
openclaw update --channel beta
openclaw update --channel dev
```

这会通过对应的 npm dist-tag（`latest`、`beta`、`dev`）更新。

当你 **显式** 用 `--channel` 切换时，OpenClaw 也会同步安装方式：

- `dev` 会确保存在 git checkout（默认 `~/openclaw`，可用 `OPENCLAW_GIT_DIR` 覆盖），更新后用该 checkout 安装全局 CLI。
- `stable`/`beta` 通过 npm 安装对应 dist-tag。

提示：如果你希望 stable + dev 并行，保留两个 clone，并将网关指向稳定版的那个。

## 插件与通道（Plugins and channels）

使用 `openclaw update` 切换通道时，OpenClaw 也会同步插件来源：

- `dev` 优先使用 git checkout 中的内置插件。
- `stable` 与 `beta` 恢复为 npm 安装的插件包。

## 打标签最佳实践（Tagging best practices）

- 给你希望 git checkout 落到的版本打 tag（`vYYYY.M.D` 或 `vYYYY.M.D-<patch>`）。
- 保持 tag 不变：不要移动或复用 tag。
- npm dist-tag 仍是 npm 安装的事实来源：
  - `latest` → stable
  - `beta` → 候选构建
  - `dev` → main 快照（可选）

## macOS 应用可用性（macOS app availability）

Beta 和 dev 构建 **可能不会** 包含 macOS 应用发布。这没问题：

- git tag 与 npm dist-tag 仍可发布。
- 在发布说明或变更日志里注明 “本 beta 无 macOS 构建”。

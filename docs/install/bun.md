---
summary: "Bun 工作流（实验）：安装与相对 pnpm 的注意事项"
read_when:
  - 你想要最快的本地开发循环（bun + watch）
  - 你遇到了 Bun 安装/补丁/生命周期脚本问题
title: "Bun（实验）"
---

# Bun（实验）

目标：在 **Bun** 下运行此仓库（可选，WhatsApp/Telegram 不推荐），
同时不偏离 pnpm 工作流。

⚠️ **不建议用于网关运行时**（WhatsApp/Telegram 有问题）。生产环境请使用 Node。

## 状态（Status）

- Bun 是可选的本地运行时，用于直接运行 TypeScript（`bun run …`, `bun --watch …`）。
- `pnpm` 是默认构建工具，仍被完整支持（部分文档工具依赖 pnpm）。
- Bun 不能使用 `pnpm-lock.yaml`，会忽略它。

## 安装（Install）

默认：

```sh
bun install
```

说明：`bun.lock`/`bun.lockb` 已被 gitignore，因此不会产生仓库变更。如需 **不写入 lockfile**：

```sh
bun install --no-save
```

## 构建 / 测试（Bun）

```sh
bun run build
bun run vitest run
```

## Bun 生命周期脚本（默认阻止）

Bun 可能默认阻止依赖生命周期脚本，除非明确信任（`bun pm untrusted` / `bun pm trust`）。
对本仓库而言，这些常见阻止脚本并非必需：

- `@whiskeysockets/baileys` `preinstall`：检查 Node 主版本 >= 20（我们使用 Node 22+）。
- `protobufjs` `postinstall`：输出版本方案不兼容的警告（无构建产物）。

如果你遇到必须执行这些脚本的真实运行问题，请显式信任：

```sh
bun pm trust @whiskeysockets/baileys protobufjs
```

## 注意事项（Caveats）

- 部分脚本仍硬编码 pnpm（例如 `docs:build`、`ui:*`、`protocol:check`）。这些请先使用 pnpm。

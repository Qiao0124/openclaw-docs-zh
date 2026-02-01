# OpenClaw 中文文档（非官方）

本仓库用于托管 OpenClaw 文档的中文翻译版本，排版与英文原文保持一致。

- 原文仓库：https://github.com/openclaw/openclaw
- 原文目录：`/docs`
- 本仓库文档目录：`/docs`
- 上次同步时间：2026-02-01
- 自定义域名：docs-zh.openclaw.ai
- 说明：译文可能滞后于英文原文，版权归原作者所有。

## 部署（Mintlify）

1. 在 Mintlify 控制台连接此仓库，并将 Docs 路径设置为 `/docs`。
2. 安装 Mintlify GitHub App 并授权此仓库，以开启自动部署。
3. 推送 `main` 分支后，Mintlify 会自动构建并发布。
4. 如需自定义域名，在 Mintlify 中绑定 `docs-zh.openclaw.ai`。

## 本地预览

在 `docs/` 目录（包含 `docs.json`）中运行：

```bash
npm i -g mint
mint dev
```

## 贡献与同步

如需同步上游文档，请先拉取 `openclaw/openclaw` 的更新，再更新翻译并将 `last_sync` 改为实际同步日期。

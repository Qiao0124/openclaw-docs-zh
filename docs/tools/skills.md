---
summary: "Skills：托管与工作空间、门控规则以及配置/环境变量绑定"
read_when:
  - 添加或修改 skills
  - 更改 skill 门控或加载规则
title: "Skills"
---

# Skills (OpenClaw)

OpenClaw 使用 **[AgentSkills](https://agentskills.io) 兼容**的 skill 文件夹来教导 Agent 如何使用工具。每个 skill 是一个包含 `SKILL.md` 文件的目录，该文件包含 YAML frontmatter 和指令。OpenClaw 在加载时会根据环境、配置和二进制文件的存在情况，过滤并加载**捆绑的 skills**以及可选的本地覆盖。

## 位置与优先级

Skills 从**三个**位置加载：

1. **捆绑的 skills**：随安装包一起提供（npm 包或 OpenClaw.app）
2. **托管/本地 skills**：`~/.openclaw/skills`
3. **工作空间 skills**：`<workspace>/skills`

如果 skill 名称冲突，优先级为：

`<workspace>/skills`（最高）→ `~/.openclaw/skills` → 捆绑的 skills（最低）

此外，你可以通过 `~/.openclaw/openclaw.json` 中的 `skills.load.extraDirs` 配置额外的 skill 文件夹（最低优先级）。

## 每个 Agent 独立的 skills 与共享 skills

在**多 Agent** 设置中，每个 Agent 都有自己的工作空间。这意味着：

- **每个 Agent 独立的 skills** 位于该 Agent 专属的 `<workspace>/skills` 中。
- **共享 skills** 位于 `~/.openclaw/skills`（托管/本地），对同一台机器上的**所有 Agent** 可见。
- 还可以通过 `skills.load.extraDirs` 添加**共享文件夹**（最低优先级），如果你想让多个 Agent 使用共同的 skill 包。

如果同一个 skill 名称存在于多个位置，适用常规的优先级规则：工作空间优先，然后是托管/本地，最后是捆绑的。

## Plugins + skills

Plugins 可以通过在 `openclaw.plugin.json` 中列出 `skills` 目录（路径相对于 plugin 根目录）来提供自己的 skills。Plugin skills 在 plugin 启用时加载，并参与正常的 skill 优先级规则。你可以通过 plugin 配置项上的 `metadata.openclaw.requires.config` 对它们进行门控。有关发现/配置的详细信息，请参阅 [Plugins](/plugin)；有关这些 skills 所教授的工具接口，请参阅 [Tools](/tools)。

## ClawHub（安装 + 同步）

ClawHub 是 OpenClaw 的公共 skill 注册中心。访问 https://clawhub.com 浏览。使用它来发现、安装、更新和备份 skills。完整指南：[ClawHub](/tools/clawhub)。

常用流程：

- 将 skill 安装到工作空间：
  - `clawhub install <skill-slug>`
- 更新所有已安装的 skills：
  - `clawhub update --all`
- 同步（扫描 + 发布更新）：
  - `clawhub sync --all`

默认情况下，`clawhub` 安装到当前工作目录下的 `./skills`（或回退到配置的 OpenClaw 工作空间）。OpenClaw 在下次会话时将其识别为 `<workspace>/skills`。

## 安全注意事项

- 将第三方 skills 视为**受信任的代码**。启用前请阅读它们。
- 对于不受信任的输入和风险工具，优先使用沙盒运行。请参阅 [Sandboxing](/gateway/sandboxing)。
- `skills.entries.*.env` 和 `skills.entries.*.apiKey` 将 secrets 注入该 Agent 回合的**主机**进程中（而非沙盒）。将 secrets 排除在 prompts 和日志之外。
- 有关更广泛的威胁模型和检查清单，请参阅 [Security](/gateway/security)。

## 格式（AgentSkills + Pi 兼容）

`SKILL.md` 必须至少包含：

```markdown
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
---
```

注意事项：

- 我们遵循 AgentSkills 规范进行布局/意图定义。
- 嵌入式 Agent 使用的解析器仅支持**单行** frontmatter 键。
- `metadata` 应该是**单行 JSON 对象**。
- 在指令中使用 `{baseDir}` 引用 skill 文件夹路径。
- 可选的 frontmatter 键：
  - `homepage` — 在 macOS Skills UI 中显示为"Website"的 URL（也可通过 `metadata.openclaw.homepage` 支持）。
  - `user-invocable` — `true|false`（默认：`true`）。为 `true` 时，skill 作为用户斜杠命令暴露。
  - `disable-model-invocation` — `true|false`（默认：`false`）。为 `true` 时，skill 从模型 prompt 中排除（仍可通过用户调用使用）。
  - `command-dispatch` — `tool`（可选）。设置为 `tool` 时，斜杠命令绕过模型直接分派到工具。
  - `command-tool` — 当设置 `command-dispatch: tool` 时要调用的工具名称。
  - `command-arg-mode` — `raw`（默认）。用于工具分派，将原始参数字符串转发给工具（不进行核心解析）。

    工具调用参数为：
    `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }`。

## 门控（加载时过滤）

OpenClaw 在**加载时**使用 `metadata`（单行 JSON）过滤 skills：

```markdown
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

`metadata.openclaw` 下的字段：

- `always: true` — 始终包含该 skill（跳过其他门控）。
- `emoji` — macOS Skills UI 使用的可选 emoji。
- `homepage` — macOS Skills UI 中显示为"Website"的可选 URL。
- `os` — 可选的平台列表（`darwin`、`linux`、`win32`）。如果设置，skill 仅在这些操作系统上可用。
- `requires.bins` — 列表；每个都必须在 `PATH` 上存在。
- `requires.anyBins` — 列表；至少有一个必须在 `PATH` 上存在。
- `requires.env` — 列表；环境变量必须存在**或**在配置中提供。
- `requires.config` — 必须为 truthy 的 `openclaw.json` 路径列表。
- `primaryEnv` — 与 `skills.entries.<name>.apiKey` 关联的环境变量名称。
- `install` — macOS Skills UI 使用的可选安装程序规格数组（brew/node/go/uv/download）。

关于沙盒的说明：

- `requires.bins` 在 skill 加载时在**主机**上检查。
- 如果 Agent 被沙盒化，二进制文件也必须在**容器内**存在。通过 `agents.defaults.sandbox.docker.setupCommand`（或自定义镜像）在沙盒容器中安装它。`setupCommand` 在容器创建后运行一次。软件包安装还需要沙盒中的网络出口、可写的根文件系统和 root 用户。示例：`summarize` skill（`skills/summarize/SKILL.md`）需要 `summarize` CLI 在沙盒容器中运行。

安装程序示例：

```markdown
---
name: gemini
description: Use Gemini CLI for coding assistance and Google search lookups.
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Install Gemini CLI (brew)",
            },
          ],
      },
  }
---
```

注意事项：

- 如果列出多个安装程序，Gateway 会选择**单个**首选选项（如果可用则选择 brew，否则选择 node）。
- 如果所有安装程序都是 `download`，OpenClaw 会列出每个条目，以便你查看可用的 artifacts。
- 安装程序规格可以包含 `os: ["darwin"|"linux"|"win32"]` 以按平台过滤选项。
- Node 安装遵循 `openclaw.json` 中的 `skills.install.nodeManager`（默认：npm；选项：npm/pnpm/yarn/bun）。这仅影响**skill 安装**；Gateway 运行时仍应为 Node（Bun 不推荐用于 WhatsApp/Telegram）。
- Go 安装：如果缺少 `go` 但可用 `brew`，Gateway 会先通过 Homebrew 安装 Go，并在可能时将 `GOBIN` 设置为 Homebrew 的 `bin`。
- Download 安装：`url`（必需）、`archive`（`tar.gz` | `tar.bz2` | `zip`）、`extract`（默认：检测到 archive 时自动）、`stripComponents`、`targetDir`（默认：`~/.openclaw/tools/<skillKey>`）。

如果没有 `metadata.openclaw`，skill 始终可用（除非在配置中禁用或被 `skills.allowBundled` 阻止用于捆绑 skills）。

## 配置覆盖（`~/.openclaw/openclaw.json`）

可以切换捆绑/托管的 skills 并为其提供环境值：

```json5
{
  skills: {
    entries: {
      "nano-banana-pro": {
        enabled: true,
        apiKey: "GEMINI_KEY_HERE",
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE",
        },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

注意：如果 skill 名称包含连字符，请引用键（JSON5 允许引用键）。

配置键默认匹配**skill 名称**。如果 skill 定义了 `metadata.openclaw.skillKey`，请在 `skills.entries` 下使用该键。

规则：

- `enabled: false` 禁用 skill，即使它是捆绑/已安装的。
- `env`：仅当变量尚未在进程中设置时才注入。
- `apiKey`：声明了 `metadata.openclaw.primaryEnv` 的 skills 的便捷选项。
- `config`：自定义每个 skill 字段的可选包；自定义键必须放在这里。
- `allowBundled`：**捆绑** skills 的可选允许列表。如果设置，只有列表中的捆绑 skills 可用（托管/工作空间 skills 不受影响）。

## 环境注入（每个 Agent 运行）

当 Agent 运行开始时，OpenClaw：

1. 读取 skill metadata。
2. 将任何 `skills.entries.<key>.env` 或 `skills.entries.<key>.apiKey` 应用到 `process.env`。
3. 使用**可用**的 skills 构建系统 prompt。
4. 运行结束后恢复原始环境。

这**限定于 Agent 运行**，不是全局 shell 环境。

## 会话快照（性能）

OpenClaw 在**会话开始时**对可用 skills 进行快照，并在同一会话的后续回合中重用该列表。skills 或配置的更改在下次新会话时生效。

当启用 skills 监视器或新的可用远程节点出现时，skills 也可以在中途刷新（见下文）。可以将其视为**热重载**：刷新的列表在下次 Agent 回合时被采用。

## 远程 macOS 节点（Linux Gateway）

如果 Gateway 在 Linux 上运行，但连接了** macOS 节点**且**允许 `system.run`**（Exec 审批安全未设置为 `deny`），当该节点上存在所需的二进制文件时，OpenClaw 可以将仅限 macOS 的 skills 视为可用。Agent 应通过 `nodes` 工具执行这些 skills（通常是 `nodes.run`）。

这依赖于节点报告其命令支持以及通过 `system.run` 进行的二进制探测。如果 macOS 节点稍后离线，skills 仍然可见；在节点重新连接之前，调用可能会失败。

## Skills 监视器（自动刷新）

默认情况下，OpenClaw 监视 skill 文件夹，并在 `SKILL.md` 文件更改时更新 skills 快照。在 `skills.load` 下配置：

```json5
{
  skills: {
    load: {
      watch: true,
      watchDebounceMs: 250,
    },
  },
}
```

## Token 影响（skills 列表）

当 skills 可用时，OpenClaw 将可用的 skills 紧凑 XML 列表注入系统 prompt（通过 `pi-coding-agent` 中的 `formatSkillsForPrompt`）。成本是确定的：

- **基础开销（仅当 ≥1 个 skill 时）：** 195 个字符。
- **每个 skill：** 97 个字符 + XML 转义的 `<name>`、`<description>` 和 `<location>` 值的长度。

公式（字符）：

```
total = 195 + Σ (97 + len(name_escaped) + len(description_escaped) + len(location_escaped))
```

注意事项：

- XML 转义将 `& < > " '` 扩展为实体（`&amp;`、`&lt;` 等），增加长度。
- Token 计数因模型 tokenizer 而异。粗略的 OpenAI 风格估计是每 token 约 4 个字符，因此 **97 个字符 ≈ 24 个 token**，加上实际字段长度。

## 托管 skills 生命周期

OpenClaw 将一组基础 skills 作为**捆绑 skills**随安装包一起提供（npm 包或 OpenClaw.app）。`~/.openclaw/skills` 用于本地覆盖（例如，固定/修补 skill 而不更改捆绑副本）。工作空间 skills 为用户所有，在名称冲突时覆盖前两者。

## 配置参考

完整的配置架构请参阅 [Skills config](/tools/skills-config)。

## 寻找更多 skills？

浏览 https://clawhub.com。

---

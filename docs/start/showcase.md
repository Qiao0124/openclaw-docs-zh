---
title: "案例展示（Showcase）"
description: "来自社区的真实 OpenClaw 项目"
summary: "社区构建的 OpenClaw 项目与集成"
---

# 案例展示（Showcase）

来自社区的真实项目。看看大家在用 OpenClaw 做什么。

<Info>
**想被收录？** 在 [Discord 的 #showcase](https://discord.gg/clawd) 分享你的项目，或在 [X 上 @openclaw](https://x.com/openclaw)。
</Info>

## 🎥 OpenClaw 实战（OpenClaw in Action）

VelvetShark 的完整搭建演示（28 分钟）。

<div
  style={{
    position: "relative",
    paddingBottom: "56.25%",
    height: 0,
    overflow: "hidden",
    borderRadius: 16,
  }}
>
  <iframe
    src="https://www.youtube-nocookie.com/embed/SaWSPZoPX34"
    title="OpenClaw: The self-hosted AI that Siri should have been (Full setup)"
    style={{ position: "absolute", top: 0, left: 0, width: "100%", height: "100%" }}
    frameBorder="0"
    loading="lazy"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  />
</div>

[在 YouTube 观看](https://www.youtube.com/watch?v=SaWSPZoPX34)

<div
  style={{
    position: "relative",
    paddingBottom: "56.25%",
    height: 0,
    overflow: "hidden",
    borderRadius: 16,
  }}
>
  <iframe
    src="https://www.youtube-nocookie.com/embed/mMSKQvlmFuQ"
    title="OpenClaw showcase video"
    style={{ position: "absolute", top: 0, left: 0, width: "100%", height: "100%" }}
    frameBorder="0"
    loading="lazy"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  />
</div>

[在 YouTube 观看](https://www.youtube.com/watch?v=mMSKQvlmFuQ)

<div
  style={{
    position: "relative",
    paddingBottom: "56.25%",
    height: 0,
    overflow: "hidden",
    borderRadius: 16,
  }}
>
  <iframe
    src="https://www.youtube-nocookie.com/embed/5kkIJNUGFho"
    title="OpenClaw community showcase"
    style={{ position: "absolute", top: 0, left: 0, width: "100%", height: "100%" }}
    frameBorder="0"
    loading="lazy"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  />
</div>

[在 YouTube 观看](https://www.youtube.com/watch?v=5kkIJNUGFho)

## 🆕 来自 Discord 新鲜案例（Fresh from Discord）

<CardGroup cols={2}>

<Card title="PR 审查 → Telegram 反馈（PR Review → Telegram Feedback）" icon="code-pull-request" href="https://x.com/i/status/2010878524543131691">
  **@bangnokia** • `review` `github` `telegram`

OpenCode 完成改动 → 打开 PR → OpenClaw 审查 diff 并在 Telegram 中回复“轻微建议”与清晰的合并结论（包括需要先修复的关键问题）。

  <img src="/assets/showcase/pr-review-telegram.jpg" alt="OpenClaw 在 Telegram 中发送 PR 审查反馈" />
</Card>

<Card title="几分钟打造酒窖技能（Wine Cellar Skill in Minutes）" icon="wine-glass" href="https://x.com/i/status/2010916352454791216">
  **@prades_maxime** • `skills` `local` `csv`

向 “Robby”（@openclaw）提出本地酒窖技能需求。它要求一个 CSV 示例与存储位置，然后快速构建并测试该技能（示例为 962 瓶）。

  <img src="/assets/showcase/wine-cellar-skill.jpg" alt="OpenClaw 从 CSV 构建本地酒窖技能" />
</Card>

<Card title="Tesco 购物自动驾驶（Tesco Shop Autopilot）" icon="cart-shopping" href="https://x.com/i/status/2009724862470689131">
  **@marchattonhere** • `automation` `browser` `shopping`

每周餐计划 → 常购清单 → 预约送货时段 → 确认订单。无需 API，仅用浏览器控制。

  <img src="/assets/showcase/tesco-shop.jpg" alt="通过聊天自动化 Tesco 购物" />
</Card>

<Card title="SNAG 截图转 Markdown（SNAG Screenshot-to-Markdown）" icon="scissors" href="https://github.com/am-will/snag">
  **@am-will** • `devtools` `screenshots` `markdown`

快捷键选区 → Gemini 视觉 → Markdown 直接进入剪贴板。

  <img src="/assets/showcase/snag.png" alt="SNAG 截图转 Markdown 工具" />
</Card>

<Card title="Agents UI" icon="window-maximize" href="https://releaseflow.net/kitze/agents-ui">
  **@kitze** • `ui` `skills` `sync`

桌面应用，用于跨 Agents、Claude、Codex 和 OpenClaw 管理技能/命令。

  <img src="/assets/showcase/agents-ui.jpg" alt="Agents UI 应用" />
</Card>

<Card title="Telegram 语音便签（papla.media）" icon="microphone" href="https://papla.media/docs">
  **Community** • `voice` `tts` `telegram`

封装 papla.media TTS，并以 Telegram 语音便签发送（无烦人的自动播放）。

  <img src="/assets/showcase/papla-tts.jpg" alt="TTS 输出为 Telegram 语音便签" />
</Card>

<Card title="CodexMonitor" icon="eye" href="https://clawhub.com/odrobnik/codexmonitor">
  **@odrobnik** • `devtools` `codex` `brew`

通过 Homebrew 安装的助手，用于列出/检查/监控本地 OpenAI Codex 会话（CLI + VS Code）。

  <img src="/assets/showcase/codexmonitor.png" alt="ClawHub 上的 CodexMonitor" />
</Card>

<Card title="Bambu 3D 打印机控制（Bambu 3D Printer Control）" icon="print" href="https://clawhub.com/tobiasbischoff/bambu-cli">
  **@tobiasbischoff** • `hardware` `3d-printing` `skill`

控制并排查 BambuLab 打印机：状态、任务、摄像头、AMS、校准等。

  <img src="/assets/showcase/bambu-cli.png" alt="ClawHub 上的 Bambu CLI 技能" />
</Card>

<Card title="维也纳交通（Wiener Linien）" icon="train" href="https://clawhub.com/hjanuschka/wienerlinien">
  **@hjanuschka** • `travel` `transport` `skill`

维也纳公共交通的实时发车、扰动、电梯状态与路线规划。

  <img src="/assets/showcase/wienerlinien.png" alt="Wiener Linien 技能" />
</Card>

<Card title="ParentPay 校餐自动化（ParentPay School Meals）" icon="utensils" href="#">
  **@George5562** • `automation` `browser` `parenting`

通过 ParentPay 自动预约英国学校餐食。使用鼠标坐标确保表格单元格点击可靠。
</Card>

<Card title="R2 上传（Send Me My Files）" icon="cloud-arrow-up" href="https://clawhub.com/skills/r2-upload">
  **@julianengel** • `files` `r2` `presigned-urls`

上传到 Cloudflare R2/S3 并生成安全的预签名下载链接。适用于远程 OpenClaw 实例。
</Card>

<Card title="通过 Telegram 构建 iOS 应用（iOS App via Telegram）" icon="mobile" href="#">
  **@coard** • `ios` `xcode` `testflight`

通过 Telegram 聊天构建完整 iOS 应用（地图与语音录制），并部署到 TestFlight。

  <img src="/assets/showcase/ios-testflight.jpg" alt="TestFlight 上的 iOS 应用" />
</Card>

<Card title="Oura Ring 健康助手（Oura Ring Health Assistant）" icon="heart-pulse" href="#">
  **@AS** • `health` `oura` `calendar`

将 Oura Ring 数据与日历、预约、健身计划整合的个人 AI 健康助手。

  <img src="/assets/showcase/oura-health.png" alt="Oura Ring 健康助手" />
</Card>
<Card title="Kev 的梦之队（14+ Agents）" icon="robot" href="https://github.com/adam91holt/orchestrated-ai-articles">
  **@adam91holt** • `multi-agent` `orchestration` `architecture` `manifesto`

单一网关下的 14+ 代理，由 Opus 4.5 编排器调度 Codex 工作者。完整的 [技术长文](https://github.com/adam91holt/orchestrated-ai-articles) 覆盖梦之队成员、模型选择、沙箱、webhooks、心跳与委派流程。[Clawdspace](https://github.com/adam91holt/clawdspace) 用于代理沙箱。[博客文章](https://adams-ai-journey.ghost.io/2026-the-year-of-the-orchestrator/)。
</Card>

<Card title="Linear CLI" icon="terminal" href="https://github.com/Finesssee/linear-cli">
  **@NessZerra** • `devtools` `linear` `cli` `issues`

与代理式工作流（Claude Code、OpenClaw）集成的 Linear CLI。可在终端管理 issues、项目与流程。首个外部 PR 已合并！
</Card>

<Card title="Beeper CLI" icon="message" href="https://github.com/blqke/beepcli">
  **@jules** • `messaging` `beeper` `cli` `automation`

通过 Beeper Desktop 读取、发送与归档消息。使用 Beeper 本地 MCP API，使代理在一个地方管理你的所有聊天（iMessage、WhatsApp 等）。
</Card>

</CardGroup>

## 🤖 自动化与工作流（Automation & Workflows）

<CardGroup cols={2}>

<Card title="Winix 空气净化器控制（Winix Air Purifier Control）" icon="wind" href="https://x.com/antonplex/status/2010518442471006253">
  **@antonplex** • `automation` `hardware` `air-quality`

Claude Code 发现并确认净化器控制方式，然后由 OpenClaw 接管管理室内空气质量。

  <img src="/assets/showcase/winix-air-purifier.jpg" alt="OpenClaw 控制 Winix 空气净化器" />
</Card>

<Card title="漂亮天空摄影（Pretty Sky Camera Shots）" icon="camera" href="https://x.com/signalgaining/status/2010523120604746151">
  **@signalgaining** • `automation` `camera` `skill` `images`

由屋顶摄像头触发：当天空很美时，请 OpenClaw 拍一张天空照片 —— 它设计了技能并完成拍摄。

  <img src="/assets/showcase/roof-camera-sky.jpg" alt="OpenClaw 捕捉屋顶天空" />
</Card>

<Card title="可视化晨间简报（Visual Morning Briefing Scene）" icon="robot" href="https://x.com/buddyhadry/status/2010005331925954739">
  **@buddyhadry** • `automation` `briefing` `images` `telegram`

定时提示生成一张“场景”图（天气、任务、日期、喜欢的帖子/引用），由 OpenClaw 角色生成。
</Card>

<Card title="Padel 场地预订（Padel Court Booking）" icon="calendar-check" href="https://github.com/joshp123/padel-cli">
  **@joshp123** • `automation` `booking` `cli`
  
  Playtomic 可用性检查 + 预订 CLI。再也不错过空场。
  
  <img src="/assets/showcase/padel-screenshot.jpg" alt="padel-cli 截图" />
</Card>

<Card title="会计资料收集（Accounting Intake）" icon="file-invoice-dollar">
  **Community** • `automation` `email` `pdf`
  
  从邮件中收集 PDF，为税务顾问准备材料。每月会计自动化。
</Card>

<Card title="沙发土豆开发模式（Couch Potato Dev Mode）" icon="couch" href="https://davekiss.com">
  **@davekiss** • `telegram` `website` `migration` `astro`

在看 Netflix 的同时通过 Telegram 重建个人网站 —— Notion → Astro，迁移 18 篇文章，DNS 切到 Cloudflare。全程不打开笔记本。
</Card>

<Card title="求职代理（Job Search Agent）" icon="briefcase">
  **@attol8** • `automation` `api` `skill`

搜索岗位列表，与简历关键词匹配，并返回相关机会与链接。30 分钟用 JSearch API 构建。
</Card>

<Card title="Jira 技能构建器（Jira Skill Builder）" icon="diagram-project" href="https://x.com/jdrhyne/status/2008336434827002232">
  **@jdrhyne** • `automation` `jira` `skill` `devtools`

OpenClaw 连接 Jira，然后现场生成新的技能（在它进入 ClawHub 之前）。
</Card>

<Card title="Telegram 里的 Todoist 技能（Todoist Skill via Telegram）" icon="list-check" href="https://x.com/iamsubhrajyoti/status/2009949389884920153">
  **@iamsubhrajyoti** • `automation` `todoist` `skill` `telegram`

自动化 Todoist 任务，并让 OpenClaw 直接在 Telegram 聊天中生成技能。
</Card>

<Card title="TradingView 分析（TradingView Analysis）" icon="chart-line">
  **@bheem1798** • `finance` `browser` `automation`

通过浏览器自动化登录 TradingView，截图并进行技术分析。无需 API —— 仅浏览器控制。
</Card>

<Card title="Slack 自动支持（Slack Auto-Support）" icon="slack">
  **@henrymascot** • `slack` `automation` `support`

监控公司 Slack 频道，提供帮助并将通知转发到 Telegram。无需请求即可自主修复已部署应用中的生产问题。
</Card>

</CardGroup>

## 🧠 知识与记忆（Knowledge & Memory）

<CardGroup cols={2}>

<Card title="xuezh 中文学习（xuezh Chinese Learning）" icon="language" href="https://github.com/joshp123/xuezh">
  **@joshp123** • `learning` `voice` `skill`
  
  通过 OpenClaw 的中文学习引擎，提供发音反馈与学习流程。
  
  <img src="/assets/showcase/xuezh-pronunciation.jpeg" alt="xuezh 发音反馈" />
</Card>

<Card title="WhatsApp 记忆库（WhatsApp Memory Vault）" icon="vault">
  **Community** • `memory` `transcription` `indexing`
  
  导入完整 WhatsApp 导出，转写 1k+ 语音，结合 git 日志进行交叉校验，输出带链接的 Markdown 报告。
</Card>

<Card title="Karakeep 语义搜索（Karakeep Semantic Search）" icon="magnifying-glass" href="https://github.com/jamesbrooksco/karakeep-semantic-search">
  **@jamesbrooksco** • `search` `vector` `bookmarks`
  
  使用 Qdrant + OpenAI/Ollama 向量嵌入，为 Karakeep 书签添加向量搜索。
</Card>

<Card title="Inside-Out-2 记忆（Inside-Out-2 Memory）" icon="brain">
  **Community** • `memory` `beliefs` `self-model`
  
  独立记忆管理器：将会话文件 → 记忆 → 信念 → 演化自我模型。
</Card>

</CardGroup>

## 🎙️ 语音与电话（Voice & Phone）

<CardGroup cols={2}>

<Card title="Clawdia 电话桥（Clawdia Phone Bridge）" icon="phone" href="https://github.com/alejandroOPI/clawdia-bridge">
  **@alejandroOPI** • `voice` `vapi` `bridge`
  
  Vapi 语音助手 ↔ OpenClaw HTTP 桥接。近实时电话与代理互动。
</Card>

<Card title="OpenRouter 转写（OpenRouter Transcription）" icon="microphone" href="https://clawhub.com/obviyus/openrouter-transcribe">
  **@obviyus** • `transcription` `multilingual` `skill`

通过 OpenRouter（Gemini 等）进行多语种音频转写。可在 ClawHub 获取。
</Card>

</CardGroup>

## 🏗️ 基础设施与部署（Infrastructure & Deployment）

<CardGroup cols={2}>

<Card title="Home Assistant 插件（Home Assistant Add-on）" icon="home" href="https://github.com/ngutman/openclaw-ha-addon">
  **@ngutman** • `homeassistant` `docker` `raspberry-pi`
  
  OpenClaw 网关运行在 Home Assistant OS 上，支持 SSH 隧道与持久化状态。
</Card>

<Card title="Home Assistant 技能（Home Assistant Skill）" icon="toggle-on" href="https://clawhub.com/skills/homeassistant">
  **ClawHub** • `homeassistant` `skill` `automation`
  
  通过自然语言控制与自动化 Home Assistant 设备。
</Card>

<Card title="Nix 打包（Nix Packaging）" icon="snowflake" href="https://github.com/openclaw/nix-openclaw">
  **@openclaw** • `nix` `packaging` `deployment`
  
  电池自带的 nixified OpenClaw 配置，用于可复现部署。
</Card>

<Card title="CalDAV 日历（CalDAV Calendar）" icon="calendar" href="https://clawhub.com/skills/caldav-calendar">
  **ClawHub** • `calendar` `caldav` `skill`
  
  使用 khal/vdirsyncer 的日历技能。自托管日历集成。
</Card>

</CardGroup>

## 🏠 家居与硬件（Home & Hardware）

<CardGroup cols={2}>

<Card title="GoHome 自动化（GoHome Automation）" icon="house-signal" href="https://github.com/joshp123/gohome">
  **@joshp123** • `home` `nix` `grafana`
  
  Nix 原生家庭自动化，以 OpenClaw 为接口，并提供精美的 Grafana 仪表盘。
  
  <img src="/assets/showcase/gohome-grafana.png" alt="GoHome Grafana 仪表盘" />
</Card>

<Card title="Roborock 扫地机器人（Roborock Vacuum）" icon="robot" href="https://github.com/joshp123/gohome/tree/main/plugins/roborock">
  **@joshp123** • `vacuum` `iot` `plugin`
  
  通过自然对话控制 Roborock 扫地机器人。
  
  <img src="/assets/showcase/roborock-screenshot.jpg" alt="Roborock 状态" />
</Card>

</CardGroup>

## 🌟 社区项目（Community Projects）

<CardGroup cols={2}>

<Card title="StarSwap 市场（StarSwap Marketplace）" icon="star" href="https://star-swap.com/">
  **Community** • `marketplace` `astronomy` `webapp`
  
  完整的天文器材交易市场。围绕 OpenClaw 生态构建。
</Card>

</CardGroup>

---

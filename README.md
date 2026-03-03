# Antigravity Context Skills

[English](#english) | [中文](#chinese)

<a id="english"></a>

## 🇬🇧 English

These skills are designed **exclusively for Antigravity Agent**. Due to their deep integration with Antigravity's specific toolsets (like `view_file_outline`, `find_by_name`, and its internal artifact directory structue), they will **NOT** work in other AI agents like Cursor, Windsurf, or Claude Code.

### Why do you need this?

When developing locally without version control (e.g. Git), switching sessions or starting a new agent often causes "context loss", creating high "cold-start" costs.
This repository provides two highly-engineered, standardized hand-off protocol skills:

- **`save-context`**: Executed before an agent finishes. It proactively scans your codebase and data schemas, merging them with developer-provided inputs to generate a highly concise, forward-looking `handoff_context.md` snapshot.
- **`load-context`**: Executed at the start of a new session. It finds the hand-off context, verifies if it's outdated (by checking core source update times vs the handoff document), and provides layered, actionable tasks (`🔴 Blockers`, `🟡 Watch Outs`, `🟢 Ready to Go`).

### Installation & AI Discoverability

Using [skills.sh](https://skills.sh):

```bash
npx skills add <your-github-username>/antigravity-context-skills
```

_(Make sure to run this inside an Antigravity environment)._

**How to prompt your Agent to use them (Agent SEO):**
Because these skills are optimized with standard tags like `handoff`, `memory`, and `state-management`, you can directly command your AI assistant:

- _"Save my context for the next session."_ -> Automatically triggers `save-context`
- _"Resume our previous work."_ -> Automatically triggers `load-context`

---

<a id="chinese"></a>

## 🇨🇳 中文

这套技能**仅适用于 Antigravity Agent**。由于深度集成了 Antigravity 的专有工具链（如 `view_file_outline`、`find_by_name`，以及固定的文件缓存结构），它在 Cursor、Windsurf 或普通的大模型助手中**无法正常运行**。

### 为什么需要这个套件？

在没有 Git 等版本控制的本地开发环境中，切换新开的 Agent 往往会导致“上下文丢失”和极高的冷启动成本。
本代码库提取了工程化程度极高的标准交接协议，主要由两个技能组成：

- **`save-context`**（保存上下文）: 在会话结束时运行。它会主动扫描当前代码架构和数据 Schema，并能吸纳你口述的额外进度，最终在项目中生成一份面向未来的极简快照 `handoff_context.md`。
- **`load-context`**（读取上下文）: 在新会话开始时运行。它不光仅仅读取交接文档，还会将源码的时间戳与文档比较，以防止文件由于外部手工代码修改而导致状态过期；最后为您呈递分层的重点速览（如 阻塞项、注意项、可执行任务）。

### 安装与 AI 唤醒词

使用 [skills.sh](https://skills.sh):

```bash
npx skills add <你的GitHub用户名>/antigravity-context-skills
```

_（请确保在具有 Antigravity 环境的项目下使用）_

**如何通过自然语言命令 AI 找到并使用它 (Agent SEO):**
我们在技能的元数据中注入了 `handoff` (交接)、`memory` (记忆)、`load` / `save`等标准化标签，您只需要对 Agent 说出以下需求：

- _“我要下班了，这轮会话结束，帮我保存上下文交接文档。”_ -> 自动触发并寻找 `save-context` 技能
- _“我们继续上次的开发进度吧，帮我看看上家留了什么东西。”_ -> 自动触发并寻找 `load-context` 技能

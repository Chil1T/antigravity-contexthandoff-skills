---
name: save-context
description: 自动总结当前会话操作并保存交接信息(artifacts)到当前工作仓库，以便下一任开发者(agent)继续开发。
---

# Save Context Skill

在会话结束或用户请求保存上下文时，自动分析当前会话的全部操作，提取下一任 agent 所需的**前瞻性、可操作**信息，生成结构化交接文档并持久化到项目仓库中。

> **核心原则**: 交接文档应回答"新 agent 需要知道什么"，而不是"我做了什么"。减少历史叙事，增加可操作的技术快照。

---

## Phase 1: 信息采集

在生成交接文档前，**必须**自动执行以下采集操作：

### 1.1 代码架构扫描

- 对项目中的**关键源码文件**（入口文件、核心模块）执行 `view_file_outline`，提取类名、函数签名
- 梳理模块间的**调用链关系**（如 `main.py → train.py → model.py`）

### 1.2 数据 Schema 快照

- 如果项目涉及数据文件（CSV/JSON/数据库等），使用 `run_command` 执行：
  - `head` 查看前几行样例数据
  - 统计行数/列数/列名
- 将结果整理为简洁的 Schema 表

### 1.3 项目文档摘要

- 对项目根目录及关键子目录的 `.md` 文件执行 `view_file`
- 为每个文件撰写 1-2 句话的**内容摘要**，避免新 agent 重复读取

### 1.4 关键 Artifacts 收集

- 检查当前 agent 的 brain 目录（`<appDataDir>/brain/<conversation-id>/`）中是否有 `implementation_plan.md`、`project_roadmap.md` 等关键文档
- **将它们复制到项目的 `.agent/` 目录下**，而不是仅留链接

### 1.5 摄取人类开发者补充信息

为适应人机协作，必须主动提取并融合人类开发者在交接前提供的工作进展：

- **通过对话摄取**: 如果用户在触发 `save-context` 的指令中附带了说明（例如：“我刚修了 Redis 的登录 Bug”），必须将其作为**绝对事实（Ground Truth）**纳入状态更新中。
- **通过约定文件摄取**: 主动检查项目根目录或 `.agent/` 目录下是否存在如 `human_updates.md` 这样的补丁文件。如果存在，读取其内容并整合至交接文档中。
- **融合规则**: 在生成 `handoff_context.md` 时（特别是 `Current State`, `User Decisions` 和 `Known Gotchas` 部分），显式体现由人类开发者补充的内容。

---

## Phase 2: 生成交接文档

使用以下**固定模板**生成 `.agent/handoff_context.md`。每个 section 都必须存在（如无相关内容则写"N/A"）：

```markdown
# Handoff Context

> Generated: {timestamp} | Session: {conversation-id}

## 1. Current State (当前状态)

<!-- 用 ✅/❌ 表格标注各组件/环境的就绪状态 -->

| Component | Status | Notes |
| --------- | ------ | ----- |
| ...       | ✅/❌  | ...   |

## 2. Architecture Overview (架构概述)

<!-- 模块调用链 + 每个模块一句话说明。不要贴完整代码 -->
```

entry.py → module_a.py → module_b.py

```
- `entry.py`: 程序入口，解析参数
- `module_a.py`: 核心逻辑，包含 ClassX, ClassY
- ...

## 3. API Snapshot (关键接口快照)
<!-- 核心类/函数签名表，一问一答式，新 agent 无需读源码即可了解接口 -->
| File | Class/Function | Signature | Description |
|------|---------------|-----------|-------------|
| ...  | ...           | ...       | ...         |

## 4. Data Schema (数据概要)
<!-- 数据文件的列名、类型、样例、条数 -->
| File | Columns | Rows | Sample |
|------|---------|------|--------|
| ...  | ...     | ...  | ...    |

## 5. User Decisions (用户决策记录)
<!-- 用户已确认和未确认的决策，用 checkbox 标注 -->
- [x] 决策1: 已确认为 xxx
- [ ] 决策2: 待确认

## 6. Known Gotchas (已知踩坑记录)
<!-- 环境配置、版本冲突、特殊处理等，每条一行 -->
- 注意事项1
- 注意事项2

## 7. Pending Tasks (待办任务)
<!-- 按优先级排序，标注从哪里接手 -->
### P0 (阻塞项)
- [ ] ...

### P1 (高优先级)
- [ ] ...

### P2 (低优先级)
- [ ] ...

## 8. Key Files Summary (关键文件摘要)
<!-- 项目内重要 .md 和关键配置文件的内容摘要 -->
| File | Summary |
|------|---------|
| ...  | ...     |

## 9. Artifact Paths (交接物路径)
<!-- 所有交接相关文件的绝对路径表，必须指向项目 .agent/ 目录 -->
| Name | Path |
|------|------|
| handoff_context.md | {project_root}/.agent/handoff_context.md |
| ...                | ...                                       |
```

---

## Phase 3: 历史/行动分离

如果当前会话有大量已完成工作的实现细节（如调试过程、优化推导、历史版本选型原因等），将这些信息写入**单独的** `.agent/migration_history.md`，不要塞进 `handoff_context.md`。

`handoff_context.md` 只保留**精简的前瞻性视图**：

- ✅ "环境是 Python 3.12"
- ❌ ~~"因为 3.9 的 OpenSSL 证书过期所以改用 3.12"~~

---

## Phase 4: 文档精简与垃圾回收 (Artifact Cleanup)

在写盘之前，**必须**主动审查项目 `.agent/` 目录下的遗留文件：

1. **审计陈旧文档**: 如果上一个会话留下了 `roadmap.md`、`brainstorming.md` 甚至老的 `task.md`，判断其内容是否已经过期或被整合入最新的 `handoff_context.md`。
2. **主动删除废弃物**: 对于已经不再适用的讨论记录和过期计划文件，必须显式删除它们（使用命令或工具）。
3. **压缩内容**: 如果发现了与本次交接强相关的碎片化 markdown，将它们合并并精简，覆盖原文件，保持整个 `.agent/` 目录只有最少、最核心的文件存在。

---

## Phase 5: 保存与确认

1. **保存位置**: 使用 `write_to_file` 将文档写入项目仓库的 `.agent/` 目录：

   - 主文档: `{project_root}/.agent/handoff_context.md`
   - 历史文档（如有）: `{project_root}/.agent/migration_history.md`
   - 其他 artifacts: `{project_root}/.agent/` 目录下

   > **⚠️ CRITICAL**: 绝对不要保存到 agent 的 brain 目录（`<appDataDir>/brain/<conversation-id>/`）。那个目录是短暂的，会话结束后可能被清理。

   > **⚠️ 搜索兼容性警告**: `.agent/` 是隐藏目录，`find_by_name`（基于 `fd`）**默认忽略隐藏目录**，无法发现其中的文件。
   > `load-context` 的搜索策略必须使用 `list_dir` 而非 `find_by_name` 来定位此目录。

2. **确认**: 向用户报告：
   - 已保存的文件列表及绝对路径
   - 交接文档的 section 概要（每个 section 一句话）
   - 下一任 agent 的建议首要行动

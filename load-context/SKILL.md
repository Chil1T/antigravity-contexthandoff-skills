---
name: load-context
description: 在新开的窗口/会话中，查找前一个agent保存的交接artifacts并将其缓存到当前工作区，以便无缝继续开发。
---

# Load Context Skill

当用户在新会话中发出"继续开发"、"接手任务"、"恢复进度"等指令时，自动从项目仓库中查找前任 agent 保存的交接文档，分层消化并向用户汇报，实现**零冷启动**的无缝衔接。

> **核心原则**: 搜索优先于重新发明。搜索成本为零，重新摸索和重复犯错的成本巨大。

---

## Phase 1: 定位交接文档

> ⚠️ **工具限制**: `find_by_name` 底层使用 `fd`，**默认忽略以 `.` 开头的隐藏目录**（如 `.agent/`）。
> 搜索隐藏目录时**必须使用 `list_dir`**，不得依赖 `find_by_name`。

1. **首先**直接使用 `list_dir` 检查 `{project_root}/.agent/` 目录：
   - 如果目录存在，直接读取其中的文件：
     - `.agent/handoff_context.md`（主交接文档）
     - `.agent/migration_history.md`（历史记录，可选）
     - `.agent/` 目录下的其他 artifacts（如 `implementation_plan.md`、`project_roadmap.md` 等）

2. 如果 `.agent/` 目录不存在，使用 `list_dir` 检查 `.agents/` 目录，并使用 `find_by_name` 在项目根目录下搜索非隐藏路径中的：
   - 项目根目录下的 `handoff_context.md`
   - 项目根目录下其他可能由前任 agent 创建的 `.md` 文件

3. 如果**仍未找到任何交接文档**：
   - 告知用户未找到上一任 agent 的交接记录
   - 提议使用常规方式（`list_dir` + `view_file_outline`）探索项目结构
   - **不要凭空编造交接内容**

---

## Phase 2: 读取与缓存

1. 使用 `view_file` 完整读取找到的所有交接文档

2. 将**主交接文档** (`handoff_context.md`) 复制到当前 agent 的 artifact 目录中缓存：

   ```
   <appDataDir>/brain/<conversation-id>/loaded_handoff_context.md
   ```

   设置 `IsArtifact: true`，让用户可以在 UI 中直接查看

3. 如果当前 agent 工作区已存在上次 load 的缓存，执行 diff 比较，仅报告变更部分

---

## Phase 3: 分层汇报

读取并消化交接文档后，向用户**分三层**汇报：

### 🔴 阻塞项 (Blockers)

从 handoff 的以下 section 提取：

- `User Decisions` 中标记为 `[ ]`（待确认）的决策
- `Current State` 中标记为 ❌ 的组件
- `Pending Tasks` 中 P0 级别的任务

> 这些是**必须先解决**才能继续的问题，向用户明确提出。

### 🟡 注意项 (Watch Out)

从 handoff 的以下 section 提取：

- `Known Gotchas` 中的所有条目
- `Key Files Summary` 中涉及约束或规则的文件

> 这些不阻塞开发但可能导致踩坑，提醒用户注意。

### 🟢 可执行项 (Ready to Go)

从 handoff 的以下 section 提取：

- `Pending Tasks` 中 P1/P2 级别的任务
- `Architecture Overview` 和 `API Snapshot` 的简要概述

> 这些是可以立即开始的工作，按优先级排列并询问用户从哪项开始。

---

## Phase 4: 环境验证（可选）

如果 handoff 的 `Current State` 表中列出了环境依赖项（如 Python 版本、GPU、特定包版本等），自动执行快速验证：

```bash
# 示例验证命令
python --version
pip list | grep tensorflow
nvidia-smi
```

将验证结果与 handoff 中的期望值对比，如有不一致则在 🔴 阻塞项中标出。

---

## Phase 5: 状态过期校验（防冲突）

作为适应本地无版本控制（如无 Git）环境的重要一环，必须检查交接文档是否已过期：

1. 使用 `run_command` 或相关工具，获取项目内核心源码文件（或核心业务目录）的**最新修改时间**。
2. 与 `.agent/handoff_context.md` 的**生成时间/最后修改时间**进行对比。
3. **判定标准**：如果发现核心源码的最后修改时间**晚于**交接文档时间，说明在两次 Agent 协作期间，有人类开发者或其他进程直接修改了代码。此时，交接文档中的 API 快照和架构说明可能存在滞后。

---

## Phase 6: 确认与接手

向用户发出最终确认消息，格式如下：

```
✅ 已成功加载前任 agent 的交接上下文

📍 项目: {project_name}
📅 交接时间: {handoff_timestamp}
📄 已加载文件: {file_count} 个

🔴 阻塞项: {blocker_count} 个 — {一句话概述}
🟡 注意项: {watch_count} 个
🟢 可执行任务: {task_count} 个

[⚠️ 状态过期警告] (仅在 Phase 5 检测到冲突时显示)
检测到自上次记录后有外部代码修改（核心源码修改时间晚于交接文档时间），加载的 API 快照可能存在偏差，请注意核实。

建议首要行动: {P0/P1 中的第一项}

是否开始执行？或者您想先了解某个具体部分的详情？
```

等待用户确认后再开始具体的开发工作。

# Codex Analysis 02 Agent Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 交付第二章，准确还原 Codex `run_turn()` 从输入预处理、模型采样、工具回流到 turn 终止的主状态机。

**Architecture:** 以 `codex-rs/core/src/session/turn.rs` 为主轴，用 `stream_events_utils.rs`、`responses_retry.rs`、`session/input_queue.rs` 交叉验证。文档分解 turn step loop、sampling retry loop、response stream loop 与 tool future drain，但把 prompt 细节、tool router 内部、sandbox 和 compaction 算法留给后续章节。

**Tech Stack:** Markdown、PowerShell、ripgrep、Git。

**Spec:** `docs/superpowers/specs/2026-10-10-codex-analysis-design.md`

## Global Constraints

- 源码基线必须是 `c3d3b142d10f4316b46e35aad7e5317e7e506cb7`。
- 只读分析 `S:\Projects\codex`，不得修改其工作树。
- 中文写作；源码事实、架构判断、项目建议必须区分。
- 源码引用固定到基线提交。
- 本章只解释 agent-loop 控制流；不提前展开后续专章算法。
- 继续保持独立 Git 仓库未提交、未发布。

---

### Task 1: 建立 agent-loop 证据地图

**Files:**
- Read: `S:\Projects\codex\codex-rs\core\src\session\turn.rs`
- Read: `S:\Projects\codex\codex-rs\core\src\stream_events_utils.rs`
- Read: `S:\Projects\codex\codex-rs\core\src\responses_retry.rs`
- Read: `S:\Projects\codex\codex-rs\core\src\session\input_queue.rs`
- Read: `S:\Projects\codex\codex-rs\core\src\session\mod.rs`

**Interfaces:**
- Consumes: 固定提交源码。
- Produces: 循环层级、状态变量、续跑/退出条件、错误分支和准确行号清单。

- [x] **Step 1: 校验基线与上游工作树**

Run:

```powershell
git -C S:\Projects\codex rev-parse HEAD
git -C S:\Projects\codex status --porcelain=v1
```

Expected: HEAD 等于固定提交；status 无输出。

- [x] **Step 2: 定位三层主循环和工具回流点**

Run:

```powershell
rg -n "run_turn|run_sampling_request|try_run_sampling_request|handle_output_item_done|drain_in_flight|needs_follow_up" S:\Projects\codex\codex-rs\core\src -g "*.rs"
```

Expected: 覆盖 turn step loop、sampling retry loop、response stream loop、tool future 执行与回流。

- [x] **Step 3: 交叉验证 steering、preemption、retry 与 compaction 分支**

读取 `watch_user_input()`、`handle_response_stream_error()`、`run_auto_compact()` 及 `run_turn()` 分支上下文，记录状态转移条件。

### Task 2: 编写第二章

**Files:**
- Create: `analysis/02-agent-loop.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: Task 1 证据地图。
- Produces: 可独立阅读的 agent-loop 状态机分析和 README 入口。

- [x] **Step 1: 编写结论、边界与四层循环图**

明确 `run_turn()` 不是单一 `while tool_call`，而是 turn step、sampling retry、stream event 和 tool future 四层协作。

- [x] **Step 2: 编写 pre-loop 与每次 step 的真实顺序**

覆盖 pre-sampling compact、MCP/plugin 需求、first `StepContext`、context/input 记录、history snapshot、sampling 及 post-sampling 决策。

- [x] **Step 3: 编写 response stream 与 tool follow-up 控制流**

解释 `OutputItemDone`、`handle_output_item_done()`、`tool_future`、`FuturesOrdered`、`drain_in_flight()` 和 `needs_follow_up`；不展开 tool router 内部。

- [x] **Step 4: 编写续跑/终止决策表与四条时序**

至少覆盖纯 assistant answer、tool call follow-up、steering/preemption、context-limit compaction。

- [x] **Step 5: 编写不变量、易误读点和 agent-loop 建议**

建议只针对状态机、快照、持久化顺序、取消和可观测性。

- [x] **Step 6: 更新 README 已完成目录和阅读建议**

将第二章从计划章节移到已完成区，添加直达主状态机的阅读路径。

### Task 3: 验证并同步

**Files:**
- Verify: `README.md`
- Verify: `analysis/02-agent-loop.md`
- Sync: `S:\Projects\codex-analysis`

**Interfaces:**
- Consumes: Task 2 文档。
- Produces: 与本地目标仓库一致、可复核的第二章。

- [x] **Step 1: 检查占位符、标题、代码栅和相对链接**

Run:

```powershell
rg -n "TBD|TODO|PLACEHOLDER|待补|待确认|FIXME" README.md analysis\02-agent-loop.md
rg -n "^#{1,3} " README.md analysis\02-agent-loop.md
```

Expected: 无占位符；结构完整；所有代码栅成对。

- [x] **Step 2: 校验固定 commit 源码链接和行号**

解析文档内 GitHub blob 链接；本地源文件必须存在，行号不得超过 EOF，所有 blob 链接必须使用固定提交。

- [x] **Step 3: 同步 staging 到 `S:\Projects\codex-analysis`**

只覆盖本计划生成或修改的 README、第二章和计划文件；不删除目标内容。

- [x] **Step 4: 校验 staging/目标哈希和 Git 状态**

Expected: 对应文件 SHA-256 一致；目标仓库仍在 `main`；没有 commit；只有预期未跟踪文件。


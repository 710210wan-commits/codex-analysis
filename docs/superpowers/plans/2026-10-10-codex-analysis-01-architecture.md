# Codex Analysis 01 Architecture Overview Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 交付 `codex-analysis` 的 README 与第一章架构总览，准确描述 Codex 从入口到 agent loop 的结构和调用边界。

**Architecture:** 以 `S:\Projects\codex` 的固定提交为唯一事实源。先建立入口与核心类型证据表，再按“入口层 → App Server/协议层 → Core 会话层 → Task/Turn 执行层”组织文档，并用固定提交链接锁定证据。

**Tech Stack:** Markdown、PowerShell、ripgrep、Git。

**Spec:** `docs/superpowers/specs/2026-10-10-codex-analysis-design.md`

## Global Constraints

- 源码基线必须是 `c3d3b142d10f4316b46e35aad7e5317e7e506cb7`。
- 只读分析 `S:\Projects\codex`，不得修改其工作树。
- 中文写作；事实、推断、建议必须区分。
- 源码引用固定到基线提交。
- 第一阶段不提交、不发布。

---

### Task 1: 建立第一章证据地图

**Files:**
- Read: `S:\Projects\codex\codex-rs\cli\src\main.rs`
- Read: `S:\Projects\codex\codex-rs\tui\src\lib.rs`
- Read: `S:\Projects\codex\codex-rs\app-server\src\lib.rs`
- Read: `S:\Projects\codex\codex-rs\core\src\thread_manager.rs`
- Read: `S:\Projects\codex\codex-rs\core\src\codex_thread.rs`
- Read: `S:\Projects\codex\codex-rs\core\src\session\session.rs`
- Read: `S:\Projects\codex\codex-rs\core\src\tasks\mod.rs`
- Read: `S:\Projects\codex\codex-rs\core\src\tasks\regular.rs`
- Read: `S:\Projects\codex\codex-rs\core\src\session\turn.rs`

**Interfaces:**
- Consumes: 固定提交源码。
- Produces: 入口、核心类型、调用关系和准确行号清单，供 Task 2 使用。

- [x] **Step 1: 校验源码基线和工作树**

Run:

```powershell
git -C S:\Projects\codex rev-parse HEAD
git -C S:\Projects\codex status --porcelain=v1
```

Expected: HEAD 为基线提交；status 无输出。

- [x] **Step 2: 定位入口和核心符号**

Run:

```powershell
rg -n "fn main|async fn cli_main|pub async fn run_main|start_app_server|start_thread|start_or_steer_turn|spawn_task|struct RegularTask|pub\(crate\) async fn run_turn" S:\Projects\codex\codex-rs -g "*.rs"
```

Expected: 输出覆盖 CLI、TUI、App Server、ThreadManager、CodexThread、SessionTask 和 `run_turn`。

- [x] **Step 3: 读取每个符号上下文**

逐个读取函数定义前后代码，记录调用者、输入、输出、所有权边界；不依据文件名猜测。

- [x] **Step 4: 交叉验证 crate 职责**

Run:

```powershell
rg -n "^name = |^description = " S:\Projects\codex\codex-rs\{cli,tui,app-server,core,protocol,exec,thread-store}\Cargo.toml
```

Expected: 每层职责能由 Cargo 元数据或源码入口印证。

### Task 2: 编写 README 与第一章

**Files:**
- Create: `README.md`
- Create: `analysis/01-architecture-overview.md`

**Interfaces:**
- Consumes: Task 1 证据地图。
- Produces: 项目总索引和第一章完整分析。

- [x] **Step 1: 编写 README**

README 包含：项目目的、基线信息、目录结构、阶段范围、一图总览、第一章链接、证据规范和免责声明。

- [x] **Step 2: 编写第一章导读与总体分层图**

明确 Codex 不是单一 TUI，而是多个前端通过 App Server/Core 共享 agent runtime。ASCII 图必须展示入口、协议、Thread/Session、Task/Turn、模型/工具、持久化和执行环境。

- [x] **Step 3: 编写入口与生命周期主链**

覆盖：

```text
codex CLI
  -> TUI / exec / app-server
  -> App Server client/server boundary
  -> ThreadManager::start_thread
  -> CodexThread
  -> Session
  -> SessionTask::RegularTask
  -> run_turn
```

- [x] **Step 4: 编写核心类型职责表**

至少包含 `ThreadManager`、`CodexThread`、`Session`、`TurnContext`、`StepContext`、`SessionTask`、`RegularTask`、`ModelClientSession`。

- [x] **Step 5: 编写边界评价和 agent-loop 启示**

只写架构级启示：host/runtime 解耦、thread/turn 分离、task 抽象、事件驱动 IO、执行环境隔离。算法细节引用后续章节，不提前下结论。

### Task 3: 验证文档证据和结构

**Files:**
- Verify: `README.md`
- Verify: `analysis/01-architecture-overview.md`

**Interfaces:**
- Consumes: Task 2 文档。
- Produces: 可交付、可复核的第一章。

- [x] **Step 1: 检查占位符和未完成标记**

Run:

```powershell
rg -n "TBD|TODO|待补|待确认|placeholder" README.md analysis\01-architecture-overview.md
```

Expected: 无输出。

- [x] **Step 2: 检查章节结构**

Run:

```powershell
rg -n "^#{1,3} " README.md analysis\01-architecture-overview.md
```

Expected: README 有总索引；第一章有导读、架构图、入口、核心类型、调用链、启示和小结。

- [x] **Step 3: 校验本地相对链接**

解析 Markdown 中的本地链接；每个目标文件必须存在。外部 GitHub 链接只检查基线提交和路径格式。

- [x] **Step 4: 复核关键源码行**

重新读取文档引用的关键行，确认符号名、调用方向和行号一致。

- [x] **Step 5: 同步到目标目录**

将工作区文档复制到 `S:\Projects\codex-analysis`，初始化本地 Git 仓库；不创建 commit。


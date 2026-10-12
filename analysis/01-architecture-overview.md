# 第一章：软件架构、程序入口与运行时边界

[返回总目录](../README.md)

---

## 1. 导读与结论

本章回答六个问题：

1. Codex 当前主实现在哪里。
2. CLI、TUI、Exec、App Server 各是什么关系。
3. 一个 thread 从哪里创建。
4. 用户输入怎样变成一个可取消的 task。
5. `Thread`、`Session`、`Turn`、`Step` 为什么不是同一层概念。
6. 哪些架构边界值得现有 agent-loop 项目复用。

**结论先行**：Codex 是一个多宿主、本地优先的 agent runtime，而不只是终端聊天程序。当前主实现位于 Rust workspace `codex-rs/`。npm 包只是选择并启动平台原生二进制；`codex` 二进制负责分流 TUI、Exec、App Server 等运行形态；交互式 TUI 又通过 App Server client 连接 embedded、local daemon 或 remote App Server。App Server 把 JSON-RPC 请求转成 Core API 调用，`codex-core` 才真正拥有 thread/session/task/turn 生命周期和 agent loop。

从 agent-loop 角度，可压缩为七层：

```text
启动与分流
  -> Host / UI
  -> App Server 控制面
  -> Core Thread/Session runtime
  -> Task/Turn/Step 生命周期
  -> Model + Tool 闭环
  -> Execution/Sandbox + Persistence
```

其中最值得注意的设计不是某个单独函数，而是三组拆分：

- **宿主与运行时拆分**：TUI 通过 App Server 使用 Core，不直接实现 agent loop。
- **Thread 与 Turn 拆分**：thread 是长生命周期容器，turn 是一次被接受的工作单元。
- **Turn 与 Step 拆分**：turn 内可以产生多次模型采样；每次采样捕获独立的 `StepContext` 快照。

---

## 2. 分析基线与范围边界

本文基于 commit [`c3d3b142d10f4316b46e35aad7e5317e7e506cb7`](https://github.com/openai/codex/tree/c3d3b142d10f4316b46e35aad7e5317e7e506cb7)。所有源码链接锁定该提交。

本章只覆盖架构骨架：

- 程序入口和运行形态分流。
- TUI 与 App Server 的关系。
- App Server 到 Core 的 thread/turn 请求路径。
- Core 里的关键运行时对象。
- 从用户输入到 `run_turn()` 的完整调用链。

以下机制只标出边界，不在本章展开：

- Responses API 流式协议与重试。
- Tool registry/router/orchestrator 的内部算法。
- Approval 与跨平台 sandbox 细节。
- History compaction 与 rollout 恢复算法。
- Multi-agent 的预算、mailbox、residency 与 handoff。

---

## 3. 仓库形态：npm 是启动壳，Rust 是运行时

### 3.1 根仓库是多语言 monorepo

根目录同时包含：

| 目录 | 作用 |
| --- | --- |
| `codex-cli/` | npm 包与 Node 启动脚本 |
| `codex-rs/` | 当前 Rust CLI、TUI、App Server、Core、协议、sandbox 与持久化实现 |
| `sdk/` | TypeScript/Python SDK |
| `docs/` | 官方说明 |
| `scripts/`、`tools/` | 构建、发布、测试和仓库维护 |

根 `package.json` 明确把自己描述为仓库维护工具；npm 用户入口在 `codex-cli/package.json`，其 `bin.codex` 指向 `bin/codex.js`。

### 3.2 npm 包不承载 agent loop

**源码事实**：Node 入口根据 OS/CPU 映射平台包和 target triple，解析平台原生 `codex` 可执行文件，然后以子进程运行。关键证据：

- [平台 target 映射](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-cli/bin/codex.js#L15-L74)
- [查找原生可执行文件](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-cli/bin/codex.js#L78-L113)

因此，阅读 agent loop 时不应从 JavaScript 包展开。`codex-cli/` 是分发与进程启动壳；主分析入口应切到 `codex-rs/`。

### 3.3 `codex-rs` 不是单 crate 程序

`codex-rs` 是大型 Cargo workspace。与 agent loop 主链直接相关的 crate 如下：

| crate | 架构职责 |
| --- | --- |
| `codex-cli` | 顶层命令解析、运行模式分发 |
| `codex-tui` | 终端 UI、输入、事件呈现、App Server client 生命周期 |
| `codex-exec` | 非交互执行宿主 |
| `codex-app-server` | JSON-RPC 控制面、线程/turn API、事件通知 |
| `codex-app-server-protocol` | App Server 对外请求/响应/通知类型 |
| `codex-core` | agent 业务逻辑和运行时状态机 |
| `codex-protocol` | Core 内外共享的操作、事件、turn 输入等协议类型 |
| `codex-thread-store` | thread 元数据、history/rollout 持久化抽象 |
| `codex-exec-server` | 命令执行环境与远端 executor 边界 |
| `codex-sandboxing` 及平台 sandbox crate | 权限策略落地 |

**架构判断**：Codex 采用“细粒度 workspace + 少数核心编排 crate”的结构。crate 数量很多，但 agent loop 的控制流仍集中在 `app-server` 与 `core`；其他 crate 多为能力或平台适配。

---

## 4. 总体分层架构

### 4.1 分层图

```text
+----------------------------------------------------------+
| 0. 分发层                                                |
| npm shim / standalone installer / native codex binary    |
+----------------------------------------------------------+
                            |
                            v
+----------------------------------------------------------+
| 1. CLI 分流层                                            |
| codex-rs/cli/src/main.rs                                 |
| TUI | exec | review | app-server | mcp | sandbox | ...   |
+----------------------------------------------------------+
             |                    |                    |
             v                    v                    v
+----------------------+ +------------------+ +----------------------+
| 2a. TUI Host         | | 2b. Exec Host    | | 2c. Other Hosts      |
| ratatui UI           | | headless runner  | | IDE / SDK / remote   |
+----------------------+ +------------------+ +----------------------+
             |
             | AppServerClient
             v
+----------------------------------------------------------+
| 3. App Server 控制面                                     |
| JSON-RPC, request processors, notifications, watchers    |
+----------------------------------------------------------+
             |
             | ThreadManager / CodexThread API
             v
+----------------------------------------------------------+
| 4. Core 会话层                                           |
| ThreadManager -> CodexThread -> Session + SessionIo      |
+----------------------------------------------------------+
             |
             | accepted TurnInput
             v
+----------------------------------------------------------+
| 5. Core 执行层                                           |
| SessionTask -> RegularTask -> run_turn                   |
| TurnContext -> StepContext -> sampling/tool follow-up    |
+----------------------------------------------------------+
      |                      |                       |
      v                      v                       v
+-------------+     +------------------+     +----------------------+
| Model       |     | Tool/Execution   |     | State/Persistence    |
| providers   |     | approval/sandbox |     | history/rollout/DB   |
+-------------+     +------------------+     +----------------------+
```

### 4.2 为什么 App Server 是关键中间层

旧式 CLI 架构常是：

```text
TUI -> agent loop -> shell
```

Codex 当前交互式路径更接近：

```text
TUI -> AppServerClient -> App Server -> Core -> executor
```

App Server 可以是三种目标：

1. **Embedded**：与 TUI 同进程，通过 in-process transport 连接。
2. **Local daemon**：连接本机长期运行的 App Server。
3. **Remote**：连接远端 App Server/workspace。

TUI 的 `start_app_server()` 先尝试选定目标；local daemon 失败时可回退 embedded；最终统一返回 `AppServerClient`。见 [`tui/src/lib.rs:559-616`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/tui/src/lib.rs#L559-L616)。

**架构判断**：这层抽象使同一个 TUI 可以驱动本地同进程、本地 daemon 或远端运行时，而不需要复制 agent-loop 逻辑。

---

## 5. 程序入口与运行形态

### 5.1 Node 启动壳

安装 `@openai/codex` 后，Node 脚本只做：

```text
识别 platform/arch
  -> 定位对应平台包
  -> 找到 codex/codex.exe
  -> 转发 argv、signals 与退出状态
```

这解释了为什么源码仓库存在 TypeScript/JavaScript 包，但核心运行时仍是 Rust。

### 5.2 Rust CLI 是多工具分流器

Rust CLI 的 `main()` 初始化 build info，然后进入 `cli_main()`；后者解析 `MultitoolCli` 并按 subcommand 分流。见：

- [`cli/src/main.rs:1031-1049`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/cli/src/main.rs#L1031-L1049)
- [`cli/src/main.rs:1097-1164`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/cli/src/main.rs#L1097-L1164)

默认无 subcommand 时进入交互式 TUI；主要旁路包括：

- `codex exec`：非交互任务执行。
- `codex review`：基于 exec host 的审查模式。
- `codex app-server`：直接启动控制面服务。
- `codex mcp`：MCP 配置/服务相关命令。
- `codex sandbox`、`execpolicy`：执行隔离和策略工具。

**源码事实**：入口层负责“选择 host”，不负责“实现 agent loop”。

### 5.3 TUI 启动路径

TUI `run_main()` 把复杂启动逻辑交给 `startup_orchestration::run_main_inner()`，然后进入 ratatui 应用。见 [`tui/src/lib.rs:1099-1138`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/tui/src/lib.rs#L1099-L1138)。

在真正运行 UI 前，它会：

1. 加载/合并配置。
2. 确定 App Server target。
3. 连接 daemon/remote，或启动 embedded App Server。
4. 构造 `AppServerClient`。
5. 通过协议创建、恢复或 fork thread。
6. 把 App Server 通知转换为 UI 事件。

这意味着 TUI 是 host 与 projection 层：负责用户交互和展示，不拥有 thread 的业务状态真相。

### 5.4 Exec 路径

`codex exec` 进入 `codex-exec::run_main()`，使用非交互 host 驱动 Core。它与 TUI 的 UI 生命周期不同，但共享底层 thread/session/turn 模型。

**项目建议**：自己的 agent-loop 若同时服务 CLI、IDE、后台任务，避免为每个入口复制循环。让 host 只负责输入输出，把 agent runtime 放到无 UI 依赖的 Core。

---

## 6. App Server：宿主与 Core 之间的控制面

### 6.1 App Server 持有 `ThreadManager`

`MessageProcessor` 初始化时构造 `ThreadManager`，随后把 JSON-RPC 请求分发给 thread、turn、config、MCP 等 request processor。证据：

- [`app-server/src/message_processor.rs:325-349`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/app-server/src/message_processor.rs#L325-L349)
- [`app-server/src/message_processor.rs:1275-1295`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/app-server/src/message_processor.rs#L1275-L1295)
- [`app-server/src/message_processor.rs:1635-1653`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/app-server/src/message_processor.rs#L1635-L1653)

### 6.2 `thread/start` 创建长生命周期运行时

`thread/start` 请求经过参数、配置、环境和持久化准备后，调用：

```rust
thread_manager.start_thread(StartThreadOptions { ... })
```

随后 App Server：

1. 获取 `NewThread { thread_id, thread, session_configured, ... }`。
2. 设置 client metadata。
3. 建立 thread listener。
4. 返回 `ThreadStartResponse`。
5. 广播 `ThreadStarted` notification。

见 [`thread_processor.rs:1480-1668`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/app-server/src/request_processors/thread_processor.rs#L1480-L1668)。

关键点：创建 thread 不等于立即运行一次模型请求。thread 是可跨多个 turn、可 resume/fork、可持续接收事件的运行时容器。

### 6.3 `turn/start` 提交工作，不等待完整执行

`turn/start` 做输入大小、环境、model、approval/sandbox 等校验和转换，然后调用：

```rust
thread.start_or_steer_turn(TurnInputRequest { ... })
```

Core 返回三种路由结果：

- `Started`：新 turn 已被接受并启动。
- `Steered`：输入注入当前可 steering 的 turn。
- `NotSubmitted`：输入未被接受。

App Server 收到的是路由决定，不是最终模型答案；后续进度和结果通过事件/通知流发送。见 [`turn_processor.rs:528-734`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/app-server/src/request_processors/turn_processor.rs#L528-L734)。

**架构判断**：App Server API 采用“命令快速确认 + 异步事件流”模型。这比让 `turn/start` HTTP/RPC 一直阻塞到 agent 完成更适合长任务、审批、steering、断线恢复和多宿主订阅。

---

## 7. Core 运行时的对象模型

### 7.1 对象关系

```text
ThreadManager
  └─ owns registry of CodexThread
       ├─ Session            长生命周期业务状态与服务
       ├─ SessionIo          submission/event 双向通道
       └─ metadata           startup/rollout/source/residency

Session
  ├─ active_turn: at most one SessionTask
  ├─ input_queue
  ├─ conversation/history state
  ├─ services: model, tools, MCP, persistence, agent control...
  └─ submission_loop

accepted turn
  ├─ TurnContext             turn 级稳定状态
  ├─ SessionTask             可取消工作抽象
  │    └─ RegularTask        普通 agent turn
  └─ one or more StepContext 每次模型采样的快照
```

| 核心类型 | 生命周期 | 主要职责 |
| --- | --- | --- |
| `ThreadManager` | 进程级 | 创建、恢复、fork 并管理 live thread registry |
| `CodexThread` | live thread 级 | 对宿主提供稳定 façade，封装 `Session` 与 `SessionIo` |
| `Session` | live thread 级 | 持有业务状态、服务、输入队列、active task 与事件通道 |
| `TurnContext` | turn 级 | 固定一次被接受工作的身份、因果、环境与权限边界 |
| `StepContext` | model request 级 | 冻结一次采样使用的模型、工具、MCP、环境与 instructions 快照 |
| `SessionTask` | active work 级 | 统一任务类型、执行、取消与收尾协议 |
| `RegularTask` | regular turn 级 | 驱动普通 agent turn，处理 turn-start lifecycle 与 pending input |
| `ModelClientSession` | turn 级 | 保有该 turn 的流式连接与 sticky-routing 状态，供多次 sampling 复用 |

`ModelClientSession` 源码明确要求每个 Codex turn 新建，不可跨 turn 复用；否则会把上一 turn 的 sticky-routing token 带入下一 turn。见 [`client.rs:291-305`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L291-L305)。

### 7.2 `ThreadManager`：创建与注册中心

源码注释直接说明：`ThreadManager` 负责创建 thread 并在内存中维护它们。见 [`thread_manager.rs:248-257`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/thread_manager.rs#L248-L257)。

它承担的职责比一个 `HashMap<ThreadId, Thread>` 更广：

- 创建、resume、fork、remove、shutdown thread。
- 绑定认证、模型目录、MCP、skills、plugins、environment manager。
- 为 multi-agent 子 thread 继承父级认证、预算、instructions 和环境。
- 协调 thread store、state DB 与 live registry。

公共入口 `start_thread()` 最终进入内部 spawn 流程。见 [`thread_manager.rs:1072-1197`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/thread_manager.rs#L1072-L1197)。

### 7.3 `CodexThread`：对宿主暴露的 thread façade

`CodexThread` 持有：

- `Arc<Session>`：业务运行时。
- `SessionIo`：submission/event 通道。
- residency gate：后台 mail 与 eviction 协调。
- session source、startup metadata、rollout path。

见 [`codex_thread.rs:208-237`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/codex_thread.rs#L208-L237)。

它向宿主提供较稳定的 API：

- `submit()` / `next_event()`。
- `start_or_steer_turn()`。
- `start_turn_if_idle()` / `continue_turn_if_idle()`。
- `steer_turn()` / `suspend_turn_and_shutdown()`。
- history、rollout、MCP、environment、settings 快照访问。

**架构判断**：`CodexThread` 是 façade；宿主不应直接操作内部 `Session` mutex、队列和服务。

### 7.4 `Session`：thread 的业务状态与服务容器

源码注释将 `Session` 描述为“initialized model agent”的上下文，并明确：一个 session 同时最多有一个 running task，且可被用户输入中断。见 [`session/session.rs:56-101`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/session.rs#L56-L101)。

核心字段可以分为五类：

| 类别 | 代表字段 | 意义 |
| --- | --- | --- |
| 状态 | `state`, `active_turn` | thread 配置与当前任务 |
| 输入 | `input_queue` | 用户 steering、mailbox、pending work |
| 事件 | `tx_event`, `agent_status` | 输出和状态投影 |
| 能力 | `services`, `tool_policy`, `mcp_refresh` | 模型、工具、MCP、扩展、agent control |
| 生命周期 | cancellation、persistence semaphore、history init | 中断、顺序提交、恢复 |

### 7.5 `SessionIo` 与 `submission_loop`：顺序命令入口

Session 创建时会建立 submission channel 与 event channel，构造 `Session` 后启动长期 `submission_loop`，最后返回 `(Arc<Session>, SessionIo)`。见 [`session/mod.rs:924-1002`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L924-L1002)。

`SessionIo::submit_turn_input()` 把 turn 请求包装为 `Op::TurnInput`，送入同一队列，并只等待 Core 给出 routing decision。见 [`session/mod.rs:1050-1092`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L1050-L1092)。

`submission_loop` 再把 `Op::TurnInput` 交给 `session::turn_input::handle()`。见 [`session/handlers.rs:439-579`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/handlers.rs#L439-L579)。

这条队列是重要的串行化边界：外部请求可以并发到达，但 Core 对影响 session 状态的操作有明确的有序入口。

### 7.6 `TurnContext`：一次被接受工作的稳定上下文

`TurnContext` 代表 thread 内单个 turn 所需的上下文。它包含：

- turn ID、trace、source、parent/root turn 关系。
- 初始模型和下一 step 的设置。
- 环境快照、cwd、网络和 sandbox 相关选择。
- plugin、dynamic tools、multi-agent version。
- permission grants、telemetry、terminal error。

见 [`turn_context.rs:315-390`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn_context.rs#L315-L390)。

TurnContext 不是整个 thread 的可变配置副本。源码字段注释反复区分：turn 级配置固定，而下一次模型请求应从对应 `StepContext` 读取 step-specific 设置。

### 7.7 `StepContext`：一次模型采样的快照

`StepContext` 的源码注释是“Request-scoped state that may change between model sampling requests”。见 [`step_context.rs:20-50`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/step_context.rs#L20-L50)。

它冻结一次采样所需的：

- resolved model settings。
- token budget。
- environment snapshot 与 capability roots。
- MCP binding。
- finalized `ToolRouter`。
- 当前 AGENTS.md 快照。
- request telemetry 与 preemption token。

**架构判断**：这是 Codex 对“长 turn 内配置可能变化”的回答。一个 turn 可以有多轮模型请求；每轮都用明确快照，避免工具表、环境、模型设置在请求中途漂移。

### 7.8 `SessionTask`：可取消工作的统一抽象

`SessionTask` 接口刻意很小：

- `kind()`：任务类型。
- `span_name()`：追踪名称。
- `run()`：执行并流式发事件。
- `abort()`：中断清理。

见 [`tasks/mod.rs:175-237`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tasks/mod.rs#L175-L237)。

`Session::start_task()` 负责：

1. 激活 plugin selection。
2. 设置 root turn metadata。
3. 建立 cancellation token 与 completion notify。
4. 原子注册 active turn。
5. 获取 multi-agent execution admission。
6. 在 Tokio task 中调用具体 `SessionTask::run()`。
7. 结束时统一进入 `on_task_finished()`。

见 [`tasks/mod.rs:271-427`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tasks/mod.rs#L271-L427)。

普通聊天/编码工作只是其中一种 task：`RegularTask`。其他 task 可包括 compact、review、user shell 等。

### 7.9 `RegularTask`：普通 agent turn 的外层循环

`RegularTask::run()` 做三件关键事：

1. 发出 turn started 并执行 turn-start lifecycle contributors。
2. 获取/复用预热的 `ModelClientSession`。
3. 循环调用 `run_turn()`；若执行期间积累了 pending input，则以空 `next_input` 继续消化。

见 [`tasks/regular.rs:23-126`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tasks/regular.rs#L23-L126)。

这里存在两个不同层级的循环：

```text
RegularTask 外层循环
  -> 处理 run_turn 完成后仍存在的 pending input

run_turn 内层循环
  -> 在一个 turn 内反复 sampling / tool execution / follow-up
```

第二个循环将在第二章详细拆解。

### 7.10 `run_turn()`：agent loop 的核心入口

`run_turn()` 的源码注释直接描述了经典 agent loop：模型每次采样可能返回 function call 或 assistant message；function call 被执行并在下一次采样回流；只有 assistant message 时 turn 可完成。见 [`session/turn.rs:145-185`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L145-L185)。

本章只确认它的位置与输入：

```rust
run_turn(
    Arc<Session>,
    Arc<TurnContext>,
    Vec<TurnInput>,
    McpStartupRequirements,
    Option<ModelClientSession>,
    CancellationToken,
)
```

后续章节再拆其预压缩、context 注入、sampling、stream event、tool future、follow-up、auto compact 和 completion 条件。

---

## 8. 端到端主链路

### 8.1 交互式 TUI 路径

```text
@openai/codex npm command
  -> codex native binary
  -> cli_main()
  -> run_interactive_tui()
  -> codex_tui::run_main()
  -> start_app_server()
     -> embedded | local daemon | remote
  -> AppServerClient
  -> JSON-RPC thread/start
  -> ThreadProcessor
  -> ThreadManager::start_thread()
  -> Session::new() + submission_loop
  -> CodexThread

用户提交输入
  -> JSON-RPC turn/start
  -> TurnProcessor::turn_start_inner()
  -> CodexThread::start_or_steer_turn()
  -> SessionIo::submit_turn_input()
  -> submission_loop
  -> turn_input::handle()
  -> 创建 TurnContext
  -> Session::start_task(RegularTask)
  -> RegularTask::run()
  -> run_turn()
  -> sampling / tool / follow-up
  -> Event stream
  -> App Server notification
  -> TUI render
```

其中 `CodexThread::start_or_steer_turn()` 将请求交给 `SessionIo`；`turn_input::handle()` 根据 mode 选择 start/steer，创建 `TurnContext`，并以 `RegularTask` 启动任务。见 [`codex_thread.rs:400-415`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/codex_thread.rs#L400-L415)、[`turn_input.rs:232-265`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn_input.rs#L232-L265) 与 [`turn_input.rs:600-632`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn_input.rs#L600-L632)。

### 8.2 Core 内部的最小伪代码

```rust
// 宿主提交输入
let route = thread.start_or_steer_turn(request).await?;

// Session 的有序 submission loop
match submission.op {
    Op::TurnInput { request, mode, reply } => {
        let decision = turn_input::handle(&session, request, mode, id).await;
        reply.send(decision);
    }
    // interrupt / approval / shutdown / MCP response / ...
}

// 输入被接受为新 turn
let turn_context = session.new_turn_with_sub_id(...).await?;
session.start_task(turn_context, input, RegularTask::new()).await;

// 普通 task
loop {
    let final_message = run_turn(session, turn_context, input, ...).await?;
    if no_pending_input() {
        return final_message;
    }
    input = vec![];
}
```

该伪代码省略了审批、hooks、MCP prewarm、token budget、history 和事件细节，但保留了真实所有权边界。

---

## 9. 生命周期与并发模型

### 9.1 Thread 生命周期最长

Thread 可以：

- 跨多个 turn 存活。
- 被 resume、fork、archive 或删除。
- 绑定 rollout/history 和 thread metadata。
- 作为 multi-agent 图上的节点。

### 9.2 Session 是 thread 的 live runtime

Session 在 thread 被加载到内存时存在，拥有 live services、队列、active turn 和事件通道。冷存储中的 thread 不等于 live Session。

### 9.3 Session 同时最多一个主 task

源码明确约束 `Session` 同时最多一个 running task。`spawn_task()` 会先 abort 旧任务，再启动新任务；`start_task()` 负责注册和清理 active turn。

这不是说系统全局只能串行：

- 多个 thread 可以并行。
- 单个 turn 内工具 future 可以重叠。
- 后台 exec、hooks、MCP、code-mode worker 可并行。
- 但一个 Session 的主任务具有清晰唯一所有权。

### 9.4 Turn 是因果与取消边界

TurnContext 保存 sub-id、parent/root turn 关系、trace、grants 和 terminal error；cancellation token 由 task 注册并向下传播。这样一次中断不需要销毁整个 thread。

### 9.5 Step 是采样一致性边界

每次模型采样前捕获 StepContext。模型、工具表、MCP、AGENTS.md 和环境能力在该请求内保持一致；下一次采样可以使用更新后的快照。

**项目建议**：不要只定义一个巨大的 mutable `AgentState`。至少区分：

```text
ThreadState   长生命周期、可持久化身份
TurnState     一次任务、取消和因果关系
StepSnapshot  一次模型请求的一致性视图
```

---

## 10. 控制面、数据面与执行面的边界

### 10.1 控制面

App Server 属于控制面：

- 解析外部协议。
- admission、参数校验、client metadata。
- thread/turn 生命周期 API。
- request/notification 路由。
- listener 和 watcher 管理。

### 10.2 Agent 数据面

Core agent loop 属于数据面：

- 构建模型请求。
- 接收 response stream。
- 识别 tool calls。
- 调度工具并回流结果。
- 更新 history、token usage 和 turn events。

### 10.3 执行面

命令执行和 sandbox 是执行面：

- Core 决定调用什么、用什么权限。
- executor/sandbox 将策略落到具体 OS 或远端环境。
- output 再转换成 protocol item 回到 Core。

### 10.4 持久化面

Thread store、rollout、state DB 与 attachment store 负责：

- thread 元数据与索引。
- conversation/rollout 历史。
- resume/fork 所需状态。
- 大对象和附件。

**架构判断**：四个面没有完全物理隔离，但接口边界已经清楚。App Server 不应实现工具算法；Core 不应渲染 TUI；sandbox 不应决定对话流程；持久化不应成为 live state 的唯一内存表示。

---

## 11. 对现有 agent-loop 项目的直接启示

以下是项目建议，不是对 Codex 官方设计意图的声明。

### 11.1 把 runtime 做成可嵌入内核

若 agent-loop 未来要支持 CLI、Web、IDE、定时任务，优先建立无 UI 依赖的 Core。Host 通过稳定 API/事件流使用它。

### 11.2 接受请求与完成任务分离

推荐 API：

```text
submit(input) -> Started | Steered | Rejected
subscribe(thread) -> Event stream
```

不要让一次网络请求同时承担 admission、长时间执行、审批和最终输出。

### 11.3 建立明确的四级 ID/状态

至少区分：

- thread/session identity。
- turn identity。
- model step/request identity。
- tool call identity。

否则 tracing、恢复、取消和多 agent 因果链很快混乱。

### 11.4 用快照保护模型请求一致性

运行中可能变化的配置，不要直接让工具或模型客户端读取全局 mutable config。采样前解析成 immutable step snapshot；工具执行绑定该 snapshot。

### 11.5 给 session 一个顺序命令入口

外部并发请求通过 submission queue 进入 Session。状态变更在一个明确调度点作决定；耗时工作再派生异步 task。此设计能降低锁顺序和竞态复杂度。

### 11.6 Task 抽象不要等同于 Turn

普通 agent turn、compact、review、user shell 都可以是 task。共享取消、telemetry 和 completion lifecycle，但各自拥有不同执行函数。

### 11.7 预留 steering 和 pending input

用户输入不总是“下一轮”。它可能：

- steering 当前 turn。
- 排队等待当前采样结束。
- 触发 idle agent。
- 作为 agent mailbox 输入。

输入队列应保存来源、因果关系和触发策略，而不只是 `Vec<String>`。

---

## 12. 容易误判的地方

### 12.1 “codex-cli” 不等于 agent loop

npm `codex-cli/` 主要是原生二进制分发壳。Rust `codex-rs/cli` 也只是顶层分流器。agent loop 在 `codex-rs/core`。

### 12.2 TUI 不直接拥有业务真相

TUI 有大量本地 UI state，但 thread/turn 执行状态来自 App Server/Core 事件。

### 12.3 App Server 不等于远端服务

它既可独立 daemon/remote，也可 embedded 在 TUI 进程。它首先是协议和控制面边界，不是部署位置定义。

### 12.4 一个 turn 不等于一次模型请求

工具调用需要结果回流，自动压缩或 steering 也可触发后续采样。因此 turn 包含多个 step。

### 12.5 “单 active task”不等于没有并发

唯一主 task 约束的是 Session 的主生命周期所有权，不排除多 thread、工具 future、后台进程和辅助任务并发。

---

## 13. 本章小结

Codex 的架构主干可以概括为：

```text
多种 Host
  -> 统一 App Server 控制面
  -> ThreadManager 管理 live threads
  -> CodexThread 暴露稳定 façade
  -> Session 持有 live runtime
  -> SessionIo/submission_loop 串行决策
  -> SessionTask 管理可取消工作
  -> TurnContext 定义一次任务边界
  -> StepContext 固定一次模型请求
  -> RegularTask/run_turn 实现 agent loop
```

它最大的工程价值是把 UI、协议、长生命周期会话、一次任务、一次模型请求和执行环境拆成不同层。第二章将在这个骨架上继续进入 `run_turn()`，逐段还原 sampling → tool execution → tool result → follow-up sampling → completion 的真实状态机。


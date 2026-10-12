# 第二章：Agent Loop 主状态机

[返回总目录](../README.md) · [上一章：软件架构、程序入口与运行时边界](./01-architecture-overview.md)

---

## 1. 导读与结论

本章回答六个问题：

1. `run_turn()` 内部究竟循环什么。
2. 模型为什么发出 tool call 后会再次采样。
3. 工具是否必须等待整个 response stream 结束后才执行。
4. `needs_follow_up` 何时为 true，turn 何时真正结束。
5. steering、instant interrupt、retry、compaction 怎样改变状态转移。
6. `run_turn()` 返回后，谁负责发出 `TurnComplete` 或 `TurnAborted`。

**结论先行**：Codex 的 agent loop 不是一个简单的“调模型→有工具就执行→重复” `while` 循环。它至少有四个协作层级：

```text
RegularTask task loop
  └─ run_turn step loop
       └─ run_sampling_request retry loop
            └─ try_run_sampling_request response-event loop
                 └─ tool futures: 边收流边执行，返回前统一 drain
```

其中：

- `run_turn()` 是**一个业务 turn 内的 step 调度器**。
- `run_sampling_request()` 是**一次 step 的 prompt 构建与传输重试器**。
- `try_run_sampling_request()` 是**一次 response stream 的事件解析器和工具启动器**。
- `ToolCallRuntime` + `FuturesOrdered` 是**工具执行与结果回写层**。
- `RegularTask` 在 `run_turn()` 外面还有一层保险循环，防止 task 返回瞬间到达的 pending input 被遗漏。

最核心的退出条件不是“看到 assistant message”，而是：

```text
模型不再要求 follow-up
AND 没有 pending input
AND stop hook 不要求继续
AND 收尾流程没有中止 turn
```

---

## 2. 分析基线与范围

本章基于 commit [`c3d3b142d10f4316b46e35aad7e5317e7e506cb7`](https://github.com/openai/codex/tree/c3d3b142d10f4316b46e35aad7e5317e7e506cb7)，所有源码链接锁定该提交。

主证据文件：

| 文件 | 本章用途 |
| --- | --- |
| `core/src/session/turn.rs` | turn 预处理、step loop、sampling retry、response stream |
| `core/src/stream_events_utils.rs` | output item 分类、tool future 创建、消息持久化 |
| `core/src/responses_retry.rs` | stream retry、backoff、transport fallback |
| `core/src/session/input_queue.rs` | pending input 与 instant interrupt 触发 |
| `core/src/tasks/regular.rs` | `run_turn()` 外层 task loop |
| `core/src/tasks/mod.rs` | task 完成、terminal event、active turn 清理 |

本章只追踪**控制流与状态转移**。以下内容仅说明接入点，不展开算法：

- Prompt/context 的完整组装规则：第三章。
- Responses API、WebSocket/SSE 和流式事件细节：第四章。
- Tool registry/router/orchestrator、并发策略：第五章。
- Approval/sandbox：第六章。
- Token budget、compaction 和 rollout 恢复：第七章。

---

## 3. 先统一五个概念

| 概念 | 边界 | 是否可重试/重复 |
| --- | --- | --- |
| Task | `SessionTask::run()` 到 task completion | 一个 task 可内含多次 `run_turn()` |
| Turn | 一个 `TurnContext` 与 turn ID 的业务任务 | 内含多个 step |
| Step | 一个 `StepContext` 快照驱动的模型请求阶段 | tool/pending input 可以触发下一 step |
| Sampling attempt | 同一 step 的一次传输尝试 | 网络错误可 retry |
| Response event | 流中的 created/item/delta/completed 等单个事件 | 一次 attempt 包含多个 event |

容易混淆的点：

- **Step 不等于 retry**：retry 尽量保持同一 `StepContext`，只重做传输。
- **Turn 不等于 response**：工具结果回流后会在同一 turn 中发起新 response。
- **Task 不等于 turn**：`RegularTask` 可在边界竞态下再次进入 `run_turn()` 消化 pending input。

`RegularTask::run()` 会反复调用 `run_turn()`，只有 session input queue 已无 pending input 时才返回。见 [`tasks/regular.rs:40-124`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tasks/regular.rs#L40-L124)。

---

## 4. 四层循环全景

### 4.1 控制流图

```text
RegularTask::run()
  |
  | task-level loop: run_turn 返回后再检查 pending input
  v
run_turn()
  |
  +-- pre-loop preparation
  |     pre-compact / MCP+plugin / first StepContext
  |     context+input persistence / injections / hooks
  |
  `-- step loop --------------------------------------------------+
        |                                                         |
        +-- drain allowed pending input                           |
        +-- capture/reuse StepContext                             |
        +-- snapshot history                                      |
        v                                                         |
      run_sampling_request()                                      |
        |                                                         |
        `-- retry loop --------------------------------------+    |
              +-- build Prompt                               |    |
              +-- try_run_sampling_request()                 |    |
              |     `-- response-event loop                  |    |
              |           +-- stream item/delta              |    |
              |           +-- create tool future             |    |
              |           +-- response.completed             |    |
              |           `-- drain tool futures             |    |
              +-- success -> return                           |    |
              `-- retryable error -> backoff/fallback --------+    |
        |                                                         |
        +-- model_needs_follow_up OR pending input? -- yes --------+
        |
        `-- stop hooks / post-turn compact / break
  |
  v
last_agent_message -> task wrapper -> TurnComplete/TurnAborted
```

### 4.2 各层为什么必须分开

| 层 | 解决的问题 | 不应承担的问题 |
| --- | --- | --- |
| Task loop | task 边界上遗留的 pending input | 模型 stream 解析 |
| Step loop | 工具回流、steering、compaction 后的再采样 | 传输 backoff |
| Retry loop | 临时网络错误、transport fallback | 业务 turn 完成判断 |
| Event loop | 将 stream event 转成 history、UI event 和 tool future | 新 step 的上下文选择 |
| Tool futures | 实际副作用与结果回写 | 决定 turn 是否终止 |

**架构判断**：这种分层不只是为了函数可读性。它将“业务续跑”、“网络重试”和“副作用并发”分成不同状态域，避免一个 `retry_count` 或 `continue` 同时改变多层语义。

---

## 5. `run_turn()` 的输入、输出与关键状态

### 5.1 函数签名

`run_turn()` 接收：

```rust
run_turn(
    Arc<Session>,
    Arc<TurnContext>,
    Vec<TurnInput>,
    &mut McpStartupRequirements,
    Option<ModelClientSession>,
    CancellationToken,
) -> CodexResult<Option<String>>
```

源码自身的注释定义了最小循环：模型返回 function call 时执行并在下一次采样回传结果；只返回 assistant message 时 turn 可完成。见 [`session/turn.rs:150-171`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L150-L171)。

返回值不是整个 event log，而是最后一条有效 agent message：

- `Ok(Some(text))`：turn 有最终 agent message。
- `Ok(None)`：turn 可能被 hook/错误流程停止，或没有有效 message。
- `Err(TurnAborted)`：中断语义向 task wrapper 传播。
- 其他 `Err`：部分在 `run_turn()` 内就地转成错误事件并结束循环；未被消化的错误由 task 收尾层统一处理。

### 5.2 turn 内持续存在的变量

| 变量 | 作用 |
| --- | --- |
| `client_session` | turn 级流式会话，在 step/retry 间复用 WebSocket 与 sticky routing |
| `required_servers` / `required_plugins` | 累积本 turn 已要求的 MCP/plugin 需求 |
| `last_agent_message` | 保存最近有效 agent message，作为 turn 完成结果 |
| `turn_diff_tracker` | 跨 step 聚合本 turn 的变更 |
| `can_drain_pending_input` | 本轮循环是否允许把 steering/mailbox 输入并入 history |
| `next_step_context` | 首次采样可复用已构造的 `StepContext` |
| `guardian_budget_compacted` | 限制特定 context overflow 分支每 step 只 compact/retry 一次 |

`ModelClientSession` 在 turn 内跨 retry 复用，但不应跨 turn 复用。`run_turn()` 创建或接受 prewarmed session，然后整个 turn 持有它。见 [`session/turn.rs:178-188`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L178-L188) 和 [`session/turn.rs:408-424`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L408-L424)。

---

## 6. 进入主循环前：一次性准备阶段

`run_turn()` 并不会立即调模型。第一次 sampling 之前有一条严格有序的准备链。

### 6.1 准备顺序

```text
1. 处理前一 turn 延后完成的 async hook 结果
2. 获取 turn-scoped ModelClientSession
3. pre-sampling compaction
4. 从用户输入解析 MCP/plugin 需求
5. 捕获 first StepContext
6. 记录 context update，准备 diff display roots
7. 构建 skill/plugin injections
8. 运行 session-start/Guardian 准备
9. 运行 input hooks 并持久化 turn input
10. 启动 shell snapshot prewarm
11. 记录 plugin injections 与 resolved config analytics
12. 进入 step loop
```

对应源码见：

- pre-compact 与失败保全：[`session/turn.rs:172-223`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L172-L223)。
- MCP/plugin 需求与 first `StepContext`：[`session/turn.rs:225-281`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L225-L281)。
- context、skills/plugins、hooks 与 input 持久化：[`session/turn.rs:282-406`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L282-L406)。

### 6.2 为什么 input 在 sampling 前持久化

`run_hooks_and_record_inputs()` 会先检查输入 hook，对被接受的 input 调用 `record_pending_input()`。即使之后采样、工具或网络失败，用户输入也已进入可恢复历史。见 [`session/turn.rs:865-913`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L865-L913)。

**架构判断**：“先持久化意图，再执行副作用”是恢复语义的根基。若等模型成功后才记录用户输入，进程在 stream 中断时会丢失任务意图。

---

## 7. Step Loop：一个 turn 如何反复采样

### 7.1 每轮 step 的顺序

step loop 从 [`session/turn.rs:423`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L423) 开始。每一轮做：

```text
A. 根据 can_drain_pending_input 取 steering/mailbox input
B. 跑 input hooks 并持久化新 input
C. 复用 first StepContext，或捕获新 StepContext
D. 记录 world-state/time/reasoning-effort 变化
E. clone history，生成本次 sampling input
F. run_sampling_request()
G. 等待 stream 和所有工具完成
H. 合并 model_needs_follow_up 与 pending input
I. 根据 token/compact/hook 结果 continue 或 break
```

输入 drain、step 捕获和 history snapshot 见 [`session/turn.rs:425-541`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L425-L541)。

### 7.2 `StepContext` 何时复用，何时重建

第一次 step 可直接使用 pre-loop 已构建的 `first_step_context`，前提是本轮没有新 pending input。之后的 step，或 pending input 改变了 MCP/plugin 需求时，重新 capture：

```text
first step + no pending input -> reuse prepared StepContext
later step + no pending input -> capture fresh StepContext
any step + pending input      -> resolve requirements, then capture fresh StepContext
```

这保证新的 AGENTS.md、MCP binding、tool plan、model settings 或 environment snapshot 可在下一 step 生效，但不会改变已发出请求的视图。`StepContext` 构建时同时冻结 tool router、MCP、environment 和 AGENTS.md，见 [`session/mod.rs:3942-3992`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L3942-L3992)。

### 7.3 Prompt 为什么用 history snapshot 重建

`run_turn()` 每次 sampling 前都从 conversation history clone 出 `ResponseItem` 列表。工具调用项和工具输出已在上一 step 被记录，因此下一次 prompt 自然包含：

```text
previous conversation
+ current user input
+ model tool call
+ tool result
+ steering/mailbox input
```

**架构判断**：Codex 的 loop 不是在局部变量里手工拼一个 `messages.append(tool_result)` 就继续。history 是跨 step 的真实交接面；每个 step 从已持久化/已记录状态重建模型输入。

---

## 8. Sampling Retry Loop：业务续跑与网络重试分离

### 8.1 `run_sampling_request()` 的边界

`run_sampling_request()` 会：

1. 为 instant interrupt 注册 user-input watcher。
2. 构建 `ToolCallRuntime` 和 code-mode turn worker。
3. 取 provider 的 `stream_max_retries()`。
4. 在 retry loop 中构建 prompt。
5. 调用 `try_run_sampling_request()`。
6. 对 context limit/rate limit 立即上报；对可重试传输错误执行 backoff/fallback。
7. preempt 时将本 step 转成 `needs_follow_up = true`。

见 [`session/turn.rs:1604-1733`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L1604-L1733)。

### 8.2 retry 不创建新 turn，也不必然创建新 step

retry loop 保持：

- 同一 `TurnContext`。
- 同一 `StepContext`。
- 同一 `ModelClientSession`。
- 同一业务 sampling 意图。

它处理的是 transport 失败，不是 agent 决策。`handle_response_stream_error()` 会根据错误的 retry delay、provider 配置和已用预算选择：

- 终止并返回原错误。
- 按 server `retry-after` 或本地 delay 等待。
- 连接失败时使用递增 delay。
- retry 预算用尽后，若可用，从 WebSocket fallback 到 HTTPS，然后重置预算。
- 预算真正耗尽时保存 `ExhaustedResponseRetry`。

见 [`responses_retry.rs:55-175`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/responses_retry.rs#L55-L175)。

### 8.3 retry 中的 prompt 重建

首次 attempt 使用上层传入的 history snapshot。后续 retry 会再次 clone history，并把已执行工具的状态附加到 prompt。这是为了处理“stream 部分已经产生可观测/可持久化结果后断线”的情况。

**项目建议**：重试不应等于原封不动重发 HTTP body。先定义哪些 stream 结果已经 committed，再决定 retry prompt 如何去重和继续。

---

## 9. Response-Event Loop：从流事件到工具 future

### 9.1 建立 stream

`try_run_sampling_request()` 调用 `ModelClientSession::stream()`，然后进入 response-event loop。循环同时监听：

- response stream 的下一个 event。
- step-level preempt token。
- turn/task cancellation token。

见 [`session/turn.rs:2515-2666`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2515-L2666)。

### 9.2 完成的 output item 如何分类

`ResponseEvent::OutputItemDone` 到达时，主流程调用 `handle_output_item_done()`。该函数先尝试把 `ResponseItem` 解析为 tool call：

| 解析结果 | 动作 | `needs_follow_up` |
| --- | --- | --- |
| 有效 tool call | 立即记录 call，创建 `tool_future` | `true` |
| 非 tool item | 转成 turn item，发 started/completed，记录 history | 不改变 |
| tool 请求应回复模型/被拒绝 | 写入 function-call output | `true` |
| fatal tool parse/dispatch error | 返回 fatal error | 不进入下一 step |

见 [`stream_events_utils.rs:216-235`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/stream_events_utils.rs#L216-L235) 与 [`stream_events_utils.rs:315-432`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/stream_events_utils.rs#L315-L432)。

### 9.3 工具与 stream 可以重叠

有效 tool call 完成解析后，对应 future 被 push 到 `FuturesOrdered`。事件循环继续处理后续 output item 和 delta，因此：

```text
response stream:  item A done ---- item B delta ---- response.completed
tool runtime:          |-- tool A running --------|
                       |------ tool B running ----------|
completion barrier:                                  drain in order
```

output item 处理与 future 入队见 [`session/turn.rs:2681-2813`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2681-L2813)。stream 结束后，`drain_in_flight()` 按 `FuturesOrdered` 的输出顺序等待并记录每个工具结果，见 [`session/turn.rs:2462-2489`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2462-L2489) 和 [`session/turn.rs:3151-3193`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L3151-L3193)。

**准确表述**：工具 future 可与后续 stream 处理重叠；但 `try_run_sampling_request()` 返回前会 drain 所有 in-flight tools。因此下一次 sampling 看到的是已经收束并记录的工具结果。

### 9.4 `response.completed` 不总是 turn 完成

`ResponseEvent::Completed` 会刷新 assistant text、记录 response/token usage，然后检查 `end_turn`：

- `end_turn == Some(false)` → `needs_follow_up = true`。
- 已观测到 tool call → `needs_follow_up` 已是 true。
- 只有普通 assistant 输出，无其他触发 → false。

见 [`session/turn.rs:2949-2997`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2949-L2997)。

---

## 10. `needs_follow_up` 决策表

### 10.1 模型层信号

`SamplingRequestResult.needs_follow_up` 可由以下事件置 true：

| 事件 | 原因 |
| --- | --- |
| 模型返回 tool call | 需要把 tool output 回传模型 |
| tool call 被拒绝或应直接回复模型 | 需让模型看到 rejection/output |
| `response.completed.end_turn == false` | provider 明确要求续跑 |
| instant interrupt/preempt | 当前采样让位给新输入 |
| mailbox preemption | 在下一次请求前投递 mail |

### 10.2 turn 层最终信号

`run_turn()` 在 sampling 后再合并 session input queue：

```rust
needs_follow_up = model_needs_follow_up || has_pending_input
```

见 [`session/turn.rs:544-590`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L544-L590)。

| model follow-up | pending input | 主动作 |
| --- | --- | --- |
| false | false | 进入 stop hooks 与 turn 收尾 |
| true | false | 下一 step：常见于 tool result 回流 |
| false | true | 下一 step：消化 steering/mailbox input |
| true | true | 下一 step：同时处理工具续跑与新 input |

### 10.3 有 follow-up 也不一定立即 sampling

若 context/token 状态已达阈值，`run_turn()` 先做 mid-turn compaction，再 `continue`。若无 follow-up，则先运行 stop hooks；某些 hook 可注入 continuation prompt，把原本应结束的 turn 重新转回 step loop。

token/compaction 与 stop-hook 分支见 [`session/turn.rs:609-760`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L609-L760)。

---

## 11. 四条典型时序

### 11.1 纯 assistant answer

```text
user input
  -> persist input
  -> capture StepContext
  -> sample
  -> assistant message
  -> response.completed(end_turn != false)
  -> no pending input
  -> stop hooks allow stop
  -> optional post-turn compact
  -> run_turn returns last_agent_message
  -> task wrapper emits TurnComplete
```

关键：assistant message 本身只提供 `last_agent_message`。真正 break 还要等 pending input、stop hooks 与收尾分支完成。

### 11.2 一次 tool call 后回答

```text
step 1
  model -> tool call
  -> persist tool-call item
  -> start tool future
  -> response.completed
  -> drain tool future
  -> persist tool output
  -> model_needs_follow_up = true

step 2
  clone history(call + output)
  -> fresh StepContext
  -> model -> assistant answer
  -> no follow-up
  -> TurnComplete
```

注意：tool call 和 tool output 是 history 中的两个阶段；只有 output 已回写后，下一 sampling 才有完整闭环。

### 11.3 steering + instant interrupt

```text
step 1 sampling in flight
  -> user submits new input
  -> input queue watcher cancels step preempt token
  -> Responses Lite 在有 interrupt handle 时发 interrupt，再 drain response；
     其他有 handle 的流继续 drain；无 handle 时丢弃当前 stream
  -> return needs_follow_up = true
  -> step loop drains queued user input
  -> persist steered input
  -> capture fresh StepContext
  -> step 2 samples with updated history/config snapshot
```

`StepContext.preempt` 只在 `InstantInterrupt` feature 开启时创建，见 [`session/mod.rs:3974-3992`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L3974-L3992)。`watch_user_input()` 先订阅 queue activity，再检查 pending input，避免 arrival race，见 [`session/input_queue.rs:292-313`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/input_queue.rs#L292-L313)。

### 11.4 context limit 下的续跑

```text
sampling completes or reports context pressure
  -> compute token status
  -> follow-up required?
       no  -> normal stop/post-turn policy
       yes -> new-context request or limit reached?
               no  -> next step
               yes -> mid-turn compact
                      -> preserve continuation intent
                      -> next step from compacted history
```

对特定 Guardian context overflow，还有“每 model step 最多 compact/retry 一次”的防死循环标记。见 [`session/turn.rs:762-790`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L762-L790)。

---

## 12. 取消、抢占、重试与错误不是一件事

| 机制 | 信号 | 预期语义 |
| --- | --- | --- |
| Turn cancellation | task cancellation token | 终止整个 turn，返回 `TurnAborted` |
| Step preemption | `StepContext.preempt` | 让当前 sampling 让位，保持 turn，下一 step 消化新 input |
| Transport retry | `ResponsesStreamRetryState` | 保持 turn/step，重试同一业务请求 |
| Recoverable turn error | error event + break | 停止 loop，但 thread 可继续后续 turn |
| Fatal error | `CodexErr::Fatal` | 终止当前 step，由 `run_turn()` 或 task 收尾层转成错误事件 |

### 12.1 preempt 不是 abort

当 preempt 在 `stream.next()` 期间触发：

- 若 stream 暴露 interrupt handle，Responses Lite 会发送 interrupt；然后重置 step preempt token，继续正常 drain response，同时标记 follow-up。
- 若没有 interrupt handle，则丢弃当前 stream，返回 follow-up。
- 无论哪种，它不销毁 `TurnContext`，而是回到 step loop。

这一分支见 [`session/turn.rs:2627-2655`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2627-L2655)。

### 12.2 cancellation 必须贯穿等待点

sampling stream、retry sleep、MCP/plugin 准备和 tool runtime 都收到 cancellation token 或其 child token。主 cancellation 一旦触发，`try_run_sampling_request()` 在 drain 与收尾后仍会再检查，避免把已取消 turn 误报为完成。

### 12.3 错误分支保留 thread 可继续性

`run_turn()` 对以下错误分开处理：

- context overflow 的特定 compact/retry 路径。
- `TurnAborted` 直接上传。
- invalid image 转成可理解的 bad-request event。
- 其他 turn error 发 error event 并 break，让用户可在同一 thread 继续。

见 [`session/turn.rs:762-838`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L762-L838)。

---

## 13. `run_turn()` 之外的终态收尾

`run_turn()` 不直接发 `TurnComplete`。调用关系是：

```text
Session::start_task()
  -> spawn Tokio task
  -> SessionTask::run()
     -> RegularTask::run()
        -> run_turn()
  -> flush rollout barrier
  -> Session::on_task_finished()
     -> drain leftover pending input into history
     -> emit metrics/lifecycle
     -> TurnComplete OR TurnAborted
     -> clear active turn
     -> emit thread idle
     -> flush terminal event
     -> maybe start pending mailbox work
```

task 注册、执行与 rollout barrier 见 [`tasks/mod.rs:287-425`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tasks/mod.rs#L287-L425)。`on_task_finished()` 的 terminal event 和 active-turn 清理见 [`tasks/mod.rs:631-890`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tasks/mod.rs#L631-L890)。

这个边界有三个价值：

1. 所有 `SessionTask` 共享终态语义，不只是 `RegularTask`。
2. `TurnComplete` 发出前有 rollout flush/pending-input 处理屏障。
3. 清除 active turn 与可能的新 mailbox turn 启动发生在统一地点。

**架构判断**：主循环只决定“工作结果”；task wrapper 决定“生命周期终态”。这使 compact/review/shell 等其他 task 也能与普通 agent turn 共用取消、持久化和 terminal event 规则。

---

## 14. 可从源码抽出的状态机

### 14.1 状态定义

```text
PREPARING
  → SAMPLING
  → TOOL_BLOCKING
  → POST_STEP_DECISION
  → COMPACTING
  → STOP_HOOKS
  → COMPLETING
  → ABORTING / FAILED
```

这些状态不一定在代码中以一个 Rust `enum` 存在，而是由函数边界、timing guard、cancellation token 和分支共同表达。

### 14.2 转移表

| 当前状态 | 条件 | 下一状态 |
| --- | --- | --- |
| PREPARING | first step ready | SAMPLING |
| SAMPLING | retryable transport error | SAMPLING（新 attempt） |
| SAMPLING | tool futures exist | TOOL_BLOCKING |
| SAMPLING/TOOL_BLOCKING | response + tools complete | POST_STEP_DECISION |
| POST_STEP_DECISION | follow-up + context pressure | COMPACTING |
| POST_STEP_DECISION | follow-up, no pressure | SAMPLING（新 step） |
| POST_STEP_DECISION | no follow-up | STOP_HOOKS |
| STOP_HOOKS | hook injects continuation | SAMPLING（新 step） |
| STOP_HOOKS | allow stop | COMPLETING |
| any wait | task cancellation | ABORTING |
| sampling | recoverable error | FAILED/COMPLETING terminal path |

**推断边界**：上表是对控制流的建模，不是 Codex 源码中已定义的公开状态 enum。它适合用于我们的 agent-loop 设计、tracing 和测试分类。

---

## 15. 对现有 agent-loop 项目的直接启示

以下是项目建议，不是 Codex 官方设计声明。

### 15.1 明确区分四类 `continue`

不要只在日志中记“loop continued”。至少标注：

- `tool_follow_up`
- `pending_user_input`
- `transport_retry`
- `post_compaction_resume`
- `hook_requested_continuation`

否则性能分析和死循环诊断会把完全不同的问题混在一起。

### 15.2 把 `needs_follow_up` 做成可解释的决策对象

单个 bool 足以控制循环，但不足以调试。建议内部保留 reason set：

```text
FollowUpReasons {
  tool_calls,
  provider_end_turn_false,
  pending_user_input,
  mailbox,
  preempted,
  stop_hook,
}
```

最后可压缩为 bool，但 tracing/event 应保留原因。

### 15.3 先记录再执行，且为工具结果设 barrier

建议不变量：

```text
user/tool-call intent durable before side effect
tool result durable before next sampling
terminal event after pending writes flush
```

这三个 barrier 直接决定 crash recovery 时能否区分“没执行”、“执行了但未回写”和“已完成”。

### 15.4 抢占与取消必须是两套信号

- 取消：销毁当前任务的继续权。
- 抢占：结束当前 step，但保留 turn，并必须有下一 step。

若两者共用一个 `AbortController`且没有额外语义，steering 很容易被误处理成 turn failure。

### 15.5 快照边界应与采样边界一致

配置、工具、MCP、cwd 或 instructions 的热更新，应在“下一 step”生效，而不是进入已在运行的 model request/tool call。

### 15.6 为每层循环单独建立预算

不同循环应有不同防护：

- transport retry count/time budget。
- turn step count/tool-call budget。
- compaction retry guard。
- 总 turn wall-clock/cost budget。

只有一个全局“最大循环次数”会让临时网络故障吃掉业务工具预算，或反过来。

### 15.7 terminal event 不应由主循环随手发送

主循环返回 result；lifecycle owner 在完成持久化、metrics、active-state cleanup 后发 terminal event。这样客户端收到 completed 时，服务端已经达到可观测的一致点。

---

## 16. 容易误读的地方

### 16.1 “看到 assistant message 就结束”

不对。assistant message 只更新 `last_agent_message`。tool call、pending input、`end_turn:false`、stop hook 都可要求续跑。

### 16.2 “一次 turn 只捕获一次工具和配置”

不对。`TurnContext` 持续存在，但 `StepContext` 可在后续 sampling 前重新捕获。

### 16.3 “工具都在 response.completed 之后串行执行”

不对。tool future 在 output item done 时就可启动，与后续 stream 处理重叠；只是返回 sampling result 前有统一 drain barrier。

### 16.4 “新用户输入必然开新 turn”

不对。它可被 steering 到当前 turn，并在 instant interrupt 开启时抢占当前 step。

### 16.5 “retry 就是下一轮 agent loop”

不对。transport retry 保持同一 step 语义；tool follow-up 才是新 step。

### 16.6 “`run_turn()` 返回就表示 `TurnComplete` 已发送”

不对。terminal event 在 `Session::on_task_finished()` 中生成，中间还有 rollout flush、pending input 处理和 lifecycle/metrics。

---

## 17. 本章小结

Codex 的 agent loop 可压缩为：

```text
持久化输入
  -> 捕获 step 快照
  -> 从 history 构建 sampling input
  -> 带 retry 的 model stream
  -> 将 output item 持久化/转成 tool future
  -> drain 工具结果
  -> 合并 model follow-up 与 pending input
  -> compact / continue / stop hooks / break
  -> task 层统一发 terminal event
```

其核心工程思想是：

- 业务续跑、传输重试、流解析、工具并发分层处理。
- 每次 sampling 用 `StepContext` 快照锁定一致视图。
- 工具意图与结果进入 history，下一 step 由 history 重建 prompt。
- preemption 保留 turn，cancellation 终止 turn。
- terminal event 发送前经过统一收尾屏障。

第三章将继续深入这条链上的“从 history 构建 sampling input”，拆解 base instructions、context items、skills/plugins、AGENTS.md、tool specs 与 token budget 如何共同形成模型请求。


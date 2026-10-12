# 第四章：模型请求、流式事件、重试与抢占

[返回总目录](../README.md) · [上一章：Prompt 与 Context 管理](./03-prompt-context.md)

---

## 1. 导读与结论

本章从第三章已经构造完成的 `Prompt` 继续，回答九个问题：

1. Codex 如何把一次 sampling request 发送给模型 provider。
2. 为什么同时存在 session 级 `ModelClient` 与 turn 级 `ModelClientSession`。
3. WebSocket 和 HTTP Responses API 如何选择、复用与降级。
4. WebSocket 何时只发送增量 items，何时退回完整 request。
5. provider stream 如何统一成内部 `ResponseEvent`。
6. item started、delta、item done、response completed 分别改变什么状态。
7. 哪些错误可以 retry，server `Retry-After` 如何覆盖本地 backoff。
8. retry、transport fallback、preempt、cancel 为什么必须分开。
9. 哪些 tracing/telemetry 数据足以解释一次慢请求或失败请求。

**结论先行**：Codex 的模型调用不是“发一个 HTTP 请求，逐 token 打印”。它是一个分层恢复管线：

```text
Prompt
  |
  v
ModelClientSession::stream()
  |
  +-- provider 支持且 session 未降级 --> Responses WebSocket
  |        |
  |        +-- full request
  |        `-- previous_response_id + incremental items
  |
  `-- otherwise ----------------------> Responses HTTP/SSE
                                           |
                                           v
                                  unified ResponseStream
                                           |
                                           v
                Created / ItemAdded / Delta / ItemDone / Completed
                                           |
                 +-------------------------+------------------------+
                 |                         |                        |
              success              retryable error             preempt
                 |                         |                        |
             return result       backoff / Retry-After      next step follows
                                           |
                                  retry budget exhausted
                                           |
                           WebSocket -> HTTP fallback
```

最重要的五个判断：

- **Transport 不是业务状态机**。WebSocket、HTTP、retry、fallback 都应保持同一 step 的业务语义。
- **`ResponseEvent` 是 transport-neutral 协议**。SSE 与 WebSocket 最终进入同一 event loop。
- **Item 与 response 有两级完成边界**。`OutputItemDone` 完成单个输出；`Completed` 完成整个 provider response。
- **Preempt 不是失败**。instant interrupt 结束或排空当前 response，再由 agent loop 发起下一 step。
- **Fallback 有作用域**。WebSocket 降级标记保存在 session 级 `ModelClient`；后续 turn 继续使用 HTTP。

---

## 2. 分析基线与范围

本章基于 commit [`c3d3b142d10f4316b46e35aad7e5317e7e506cb7`](https://github.com/openai/codex/tree/c3d3b142d10f4316b46e35aad7e5317e7e506cb7)，所有源码链接锁定该提交。采用静态阅读与调用链核对；未发起真实模型请求，也未运行上游 Rust 测试。

主证据文件：

| 文件 | 本章用途 |
| --- | --- |
| `core/src/client.rs` | request 映射、HTTP/WebSocket、增量 continuation、auth recovery、stream adapter |
| `core/src/client_common.rs` | `ResponseStream` 生命周期 |
| `core/src/session/turn.rs` | sampling retry loop 与 response-event loop |
| `core/src/responses_retry.rs` | retry、backoff、server advice、fallback 决策 |
| `codex-api/src/common.rs` | transport-neutral `ResponseEvent` |
| `codex-api/src/sse/responses.rs` | wire event 解析、incomplete 映射与 SSE idle timeout |
| `protocol/src/error.rs` | retryable/terminal error 分类 |
| `model-provider-info/src/lib.rs` | provider transport 与 retry 配置 |
| `core/src/session/input_queue.rs` | pending user input 到 preempt signal |
| `core/src/model_request.rs` | request contributor 与 response interceptor |

本章聚焦**请求、传输、流事件和恢复语义**。以下内容只说明边界：

- Prompt/context 构造：第三章。
- tool registry、调用匹配、执行并发：第五章。
- approval/sandbox：第六章。
- context overflow 后的 compaction：第七章。
- mailbox 与 multi-agent 控制：第八章。

---

## 3. 先区分七个层级

| 层级 | 代表对象/函数 | 职责 |
| --- | --- | --- |
| Session client | `ModelClient` | provider/auth/thread、session 级 transport fallback |
| Turn transport | `ModelClientSession` | turn sticky routing、WebSocket 连接、上一 request/response |
| Sampling retry | `run_sampling_request()` | 构造 prompt、重试、preempt/cancel 解释 |
| One attempt | `try_run_sampling_request()` | 发起一次 stream、消费事件、汇总结果 |
| Provider transport | HTTP/SSE 或 WebSocket | wire request 与原始 event stream |
| Stream adapter | `map_response_events()` | 错误映射、trace、TTFT、last response capture |
| Event projection | `match ResponseEvent` | UI delta、history item、usage、follow-up |

七层不能压成一个 `call_model()`：

- provider 连接失败属于 transport。
- `end_turn:false` 属于模型业务信号。
- pending user input 属于 preemption。
- tool call 属于 agent-loop follow-up。
- turn cancellation 属于 lifecycle。

这些信号都可能表现为“当前 await 提前结束”，但恢复动作完全不同。

---

## 4. 端到端请求链路

### 4.1 一次 sampling attempt

```text
run_sampling_request()
  -> 从 initial input 或 history 重建 Prompt
  -> 生成 CodexResponsesMetadata
  -> try_run_sampling_request()
       -> 计算 reasoning effort
       -> ModelClientSession::stream()
            -> WebSocket 或 HTTP
            -> ResponsesApiRequest
            -> ResponseStream
       -> response-event loop
       -> flush streamed text
       -> drain in-flight tools
       -> token count / turn diff
       -> SamplingRequestResult
```

`run_sampling_request()` 管 retry；`try_run_sampling_request()` 管单次 stream。源码边界分别见 [`session/turn.rs:1604-1734`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L1604-L1734) 与 [`session/turn.rs:2515-3196`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2515-L3196)。

### 4.2 Request 建立后仍有适配阶段

第三章介绍了 `Prompt -> ResponsesApiRequest`。在真正发送前，client 还会：

- 解析当前 auth/provider/routing。
- 生成 Responses headers 与 routing hint。
- 绑定 cyber access program。
- 清理不应发送的 item ID/metadata。
- 应用 request contributors。
- 建立 inference trace attempt。
- 对 input 做 wire-size/tool-metadata 限制。

HTTP 路径见 [`client.rs:1732-1865`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L1732-L1865)，WebSocket 路径见 [`client.rs:1923-2095`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L1923-L2095)。

### 4.3 Request/response extension 有明确插入点

`model_request::prepare()` 允许 request contributor 添加过滤后的 client metadata，并返回 response interceptors；`intercept_stream()` 再按顺序包装 provider stream。见 [`model_request.rs:1-45`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/model_request.rs#L1-L45)。

**架构判断**：扩展点放在 request 已结构化、response 尚未进入核心 bookkeeping 的边界，既能观察 wire 语义，又不会侵入 agent loop。

---

## 5. `ModelClient` 与 `ModelClientSession`

### 5.1 `ModelClient` 是 session 级

`ModelClient` 保存：

- provider 与 auth manager。
- thread ID。
- session 级 WebSocket disable/fallback 状态。
- cached WebSocket session。
- request contributors。
- prompt cache key override。
- event sender 与 HTTP client factory。

源码注释明确：WebSocket fallback 是 session-scoped；一次 turn 激活 HTTP fallback 后，后续 turns 也继续用 HTTP。见 [`client.rs:256-286`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L256-L286)。

### 5.2 `ModelClientSession` 是 turn 级

它保存：

- turn 内复用的 WebSocket connection。
- 上一次完整 request。
- 上一次 response ID 和新增 items。
- sticky-routing `x-codex-turn-state`。

创建新 turn 时必须创建新的 `ModelClientSession`。跨 turn 复用会把上一 turn 的 sticky-routing token 带入下一 turn。见 [`client.rs:287-345`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L287-L345)。

### 5.3 生命周期关系

```text
Session
  └─ ModelClient
       ├─ shared provider/auth/fallback state
       └─ Turn A: ModelClientSession
            ├─ sticky routing A
            ├─ ws connection
            └─ request/response continuation state

       └─ Turn B: new ModelClientSession
            └─ fresh sticky routing B
```

**项目建议**：连接池、provider 配置可以 session 级；带业务因果信息的 routing token、previous response ID 必须 turn 级。

---

## 6. Transport 选择：WebSocket 优先，HTTP 保底

### 6.1 Provider 声明能力，session 决定当前状态

`ModelProviderInfo` 暴露：

- `supports_websockets`。
- `stream_max_retries`。
- `stream_idle_timeout_ms`。
- `websocket_connect_timeout_ms`。
- request retry 等 provider 配置。

见 [`model-provider-info/src/lib.rs:139-207`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/model-provider-info/src/lib.rs#L139-L207)。Retry 次数应用默认值及硬上限；timeout accessor 在未配置时使用默认值，此处没有同样的最大值裁剪。见 [`model-provider-info/src/lib.rs:496-525`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/model-provider-info/src/lib.rs#L496-L525)。

`responses_websocket_enabled()` 同时检查 provider capability 与 session 的 `disable_websockets`。见 [`client.rs:1096-1109`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L1096-L1109)。

### 6.2 `stream()` 的选择逻辑

`ModelClientSession::stream()`：

1. wire API 为 Responses。
2. WebSocket enabled 时优先调用 `stream_responses_websocket()`。
3. WebSocket 明确返回 `FallbackToHttp` 时激活 fallback。
4. 随后调用 `stream_responses_api()`。

见 [`client.rs:2314-2377`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L2314-L2377)。

选择不是每次 request 都重新“竞速”两种 transport。session 一旦禁用 WebSocket，后续调用直接走 HTTP。

### 6.3 两种 fallback 入口

| 入口 | 触发 | 行为 |
| --- | --- | --- |
| WebSocket 建连返回 `426 Upgrade Required` | server 明确不支持升级 | 当前调用立即返回 `FallbackToHttp` |
| stream retry budget 用尽 | WebSocket 可重试错误持续失败 | retry controller 切到 HTTP，retry 计数归零 |

第一种发生在 transport 建立阶段，见 [`client.rs:1988-2031`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L1988-L2031)；第二种发生在 [`responses_retry.rs:118-139`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/responses_retry.rs#L118-L139)。

---

## 7. HTTP Responses 路径

### 7.1 HTTP 使用 POST + SSE

`stream_responses_api()` 构建 `/responses` POST：

- request body 为 `ResponsesApiRequest`。
- `stream: true`。
- API client 返回 SSE stream。
- stream 再被映射成内部 `ResponseStream`。

请求准备和发送见 [`client.rs:1732-1881`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L1732-L1881)。

### 7.2 401 recovery 在 transport 内部完成

HTTP 建立 stream 时如果遇到可恢复 auth error：

1. 记录本次 inference attempt 失败。
2. 调用 `handle_unauthorized()`。
3. 尝试 provider recovery，或推进 auth manager 的下一恢复阶段。
4. 成功后在 transport loop 内重新构造 setup/request。

见 [`client.rs:1854-1900`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L1854-L1900)。

这与 sampling retry 不同：auth recovery 是 stream 建立阶段的局部恢复，成功后在 client 内重发，不增加外层 stream retry 计数；恢复失败仍可能把错误交回外层处理。

### 7.3 HTTP 不承担业务续跑

HTTP stream 成功返回只代表连接和请求已建立。是否需要 tool follow-up、是否 `end_turn:false`、是否 pending input，由上层 event loop 和 agent loop 判断。

---

## 8. WebSocket 路径与增量 continuation

### 8.1 WebSocket 复用范围

连接在 `ModelClientSession` 内懒建立，并在同一 turn 的多次 requests 间复用。连接 key、auth owner、headers 改变时会重建。turn 结束后 session 可把安全的 cached connection 状态交还 `ModelClient`，但 turn sticky state 不可跨 turn 混用。

连接建立逻辑见 [`client.rs:1620-1705`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L1620-L1705)。

### 8.2 Incremental request 的成立条件

当前 request 只有满足以下条件，才可转为 continuation：

1. 有上一份 full request。
2. non-input request properties 未变化。
3. 有上一 response 及非空 response ID。
4. 当前 input 以前一 request input + server output items 为前缀。
5. 前缀 items 在忽略内部 metadata 后相等。

算法见 [`client.rs:1465-1534`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L1465-L1534)。

成功后 wire payload 只发送：

```text
previous_response_id
+ newly appended input items
```

逻辑 request 仍是完整 `ResponsesApiRequest`；增量只是 transport 压缩。

### 8.3 何时退回 full request

典型原因：

- no previous request/response/response ID。
- request property changed。
- input shortened。
- input mismatch。
- restored history。
- connection/auth state reset。

退回 full 不等于错误。代码记录 bounded reason label，然后继续发送完整 request。见 [`client.rs:2026-2087`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L2026-L2087)。

### 8.4 Prewarm 不生成模型回答

先区分两个接口：`preconnect_websocket()` 只建连接，不发送 prompt；`prewarm_websocket()` 发送带 prompt 的预热请求，并消费到内部 `Completed`。前者见 [`client.rs:1535-1549`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L1535-L1549)。

WebSocket prewarm 使用 `generate=false`，只建立连接/协议状态，不应作为 inference attempt 写入 rollout trace。见 [`client.rs:2257-2310`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L2257-L2310)。

**架构判断**：preconnect、prewarm、generation 是不同阶段。指标中若混为一个“model latency”，TTFT 会失真。

---

## 9. `ResponseStream`：统一的 transport-neutral 边界

### 9.1 Core 消费统一事件，不消费 SSE frame/WS frame

内部 `ResponseStream` 只暴露：

```rust
Stream<Item = Result<ResponseEvent>>
```

同时保存：

- event receiver。
- 可选 interrupt sender。
- consumer-dropped cancellation token。

见 [`client_common.rs:118-140`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client_common.rs#L118-L140)。

### 9.2 Drop 是可观测取消

消费者提前 drop stream 时，`Drop` 会 cancel `consumer_dropped`。后台 mapper 据此把 inference trace 记为 cancelled，而不是等待 provider stream 自然结束。

这避免两类泄漏：

- 后台 task 持续读无人消费的 stream。
- trace 永远停留在 started 状态。

### 9.3 Stream adapter 同时做 bookkeeping

`map_response_events()`：

- 建立有界 channel，容量 1600。
- 收集 `OutputItemDone` items。
- 在首个 `OutputItemAdded` 记录 TTFT。
- 在 `Completed` 记录 usage、trace completion 和 `LastResponse`。
- provider error 映射为 `CodexErr`。
- consumer drop 记录 cancellation。
- stream 无 terminal event 时记录 failed trace。

见 [`client.rs:2445-2620`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L2445-L2620)。

`LastResponse { response_id, items_added }` 正是下一次 WebSocket continuation 的 baseline。

---

## 10. `ResponseEvent` 协议

### 10.1 事件分类

`ResponseEvent` 可分六组：

| 组 | 事件 | 用途 |
| --- | --- | --- |
| 生命周期 | `Created`, `Completed` | response ID、usage、end-turn |
| item | `OutputItemAdded`, `OutputItemDone` | 单 item 开始/完成 |
| 文本 | `OutputTextDelta` | assistant 可见文本 |
| 工具参数 | `ToolCallInputDelta` | 增量展示/解析 tool args |
| reasoning | summary/raw delta、part added/done | reasoning UI 投影 |
| 控制信息 | rate limits、model、verification、moderation、etag、safety | session/UI 状态 |

完整 enum 见 [`codex-api/src/common.rs:81-132`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/codex-api/src/common.rs#L81-L132)。

### 10.2 Event 不等于 UI event

Provider event 是输入协议；Core 会转换成：

- item started/completed。
- agent text delta。
- reasoning delta/section break。
- safety buffering。
- token count/rate limits。
- model mismatch warning。
- turn diff。

这种投影允许 TUI、App Server、exec 等宿主共享业务事件，而不需要理解 provider 原始 frame。

### 10.3 Event 不等于 durable history item

不是所有 delta 都直接写 history。通常：

- `OutputItemAdded` 建立 active item 和 UI start。
- delta 用于实时投影。
- `OutputItemDone` 交给完整 item handler，形成持久记录/工具工作。
- `Completed` 记录整个 response 的 usage 与终态。

**项目建议**：实时 delta 与 durable item 分层。逐 token 持久化会制造巨量日志，也难以保证 crash 后 item 完整性。

### 10.4 Wire event 先经过解析与语义转换

`process_responses_event()` 并非把每条 wire event 原样转交。关键映射：

| Wire event | 内部结果 |
| --- | --- |
| `response.output_item.added/done` | 解析为完整 `ResponseItem`，生成 Added/Done |
| `response.custom_tool_call_input.delta` | `ToolCallInputDelta` |
| `response.function_call_arguments.delta/done` | 当前映射器忽略；不能推断所有函数参数都会实时展示 |
| `response.failed` | 解析为 API error |
| `response.completed` | `Completed` |
| `response.incomplete`，reason=`interrupted` | `Completed`，强制 `end_turn=false` |
| `response.incomplete`，reason=`content_filter` | `ContentFilter` error |
| 其他 incomplete reason | `Stream` error |

见 [`codex-api/src/sse/responses.rs:344-503`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/codex-api/src/sse/responses.rs#L344-L503)。

SSE reader 为每次等待设置 idle timeout；消费者关闭时退出；收到内部 `Completed` 后停止读取。普通解析错误事件可先保存在 `response_error`，流结束时再发出；`FlexUnavailable` 则直接结束。见 [`codex-api/src/sse/responses.rs:524-653`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/codex-api/src/sse/responses.rs#L524-L653)。

---

## 11. Response-event 状态机

### 11.1 核心状态

`try_run_sampling_request()` 在 event loop 中维护：

- `active_item`。
- `active_item_is_streaming_to_client`。
- `active_tool_argument_diff_consumer`。
- `in_flight` tool futures。
- `needs_follow_up`。
- `last_agent_message`。
- `should_emit_token_count` / `should_emit_turn_diff`。
- assistant message stream parsers。
- plan-mode stream state。

初始化见 [`session/turn.rs:2515-2614`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2515-L2614)。

### 11.2 `Created`

保存 server response ID 到 turn extension data，使 tool-review/guardian 等扩展能引用当前 response。见 [`session/turn.rs:2674-2680`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2674-L2680)。

### 11.3 `OutputItemAdded`

它负责“开始”：

- 补 item ID。
- 为 assistant message 预留稳定顺序。
- 为 custom tool call 建参数 diff consumer。
- 将非工具 item 转为 `TurnItem`。
- 发 item-started event。
- 建立 active item，供后续 delta 关联。

见 [`session/turn.rs:2814-2893`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2814-L2893)。

### 11.4 Delta events

`OutputTextDelta` 必须有 active item。assistant message 还要经过 parser，把 plan-mode 或特殊标记从用户可见文本中分离。`ToolCallInputDelta` 只投递给与 call ID 匹配的 consumer。Reasoning delta 则转成专用 UI events。

见 [`session/turn.rs:2999-3138`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2999-L3138)。

### 11.5 `OutputItemDone`

它负责“提交”：

1. 补齐/继承 item ID。
2. flush tool argument diff consumer。
3. 结束前一个 active item 的 streamed text。
4. 调用 `handle_output_item_done()`。
5. 保存 `last_agent_message`。
6. 合并 `needs_follow_up`。
7. 如返回 tool future，加入 `FuturesOrdered`。

见 [`session/turn.rs:2681-2792`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2681-L2792)。

本章只关注 item lifecycle；tool handler、并发和结果回流在第五章展开。

### 11.6 `Completed`

它负责 response 级收尾：

- 记录 response ID 和 tool-call analytics。
- flush 所有剩余 assistant text segments。
- 记录 observed completion 与 token usage。
- 更新 token-budget 状态。
- `end_turn == Some(false)` 时要求 follow-up。
- 返回 `SamplingRequestResult`。

见 [`session/turn.rs:2949-2998`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2949-L2998)。

`Completed` 是 provider response 终态，不等于整个业务 turn 终态。

---

## 12. Stream 结束后的 barrier

### 12.1 先 flush 文本，再等工具

event loop 退出后：

1. drop sampling span/timing guard。
2. flush 所有 assistant text segments。
3. drain `in_flight` tool futures。
4. 发送 token count。
5. 再检查 cancellation。
6. 发送 turn diff。
7. 返回 sampling outcome。

见 [`session/turn.rs:3145-3196`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L3145-L3196)。

### 12.2 为什么 token count 在工具 barrier 之后

某些工具会暂停等待用户。若 token event 在 pending tool 之前发送，客户端可能把它误解为 turn 仍在推进。源码因此等工具完成后才发 token count；但若 response 已完成后发生 cancellation，usage 仍会先落地。

### 12.3 Stream closed without `Completed` 是错误

若 `stream.next()` 返回 `None`，Core 生成：

```text
CodexErr::Stream("stream closed before response.completed")
```

见 [`session/turn.rs:2658-2671`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2658-L2671)。

不能把 TCP/SSE/WS 的正常 EOF 当成模型 response 成功；必须观察内部协议终态。这里的 `Completed` 也包含解析层接受的 interrupted incomplete，见第 10.4 节。

---

## 13. Retry：错误分类、预算和等待

### 13.1 Retry 决策不在 transport event loop 内

`try_run_sampling_request()` 返回错误后，`run_sampling_request()` 调用 `handle_response_stream_error()`。这保留同一：

- turn。
- step context。
- tool runtime。
- base instructions。
- retry state。

但下一 attempt 会从最新 history 重建 prompt input，以容纳可能已经落地的事件/工具状态。见 [`session/turn.rs:1634-1734`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L1634-L1734)。

**推断边界**：这不是数据库事务回滚。前一次 attempt 已完成的 items、已发生的工具副作用不会因网络失败自动撤销；不能仅从该 retry loop 推出工具执行具有 exactly-once 保证。工具身份、结果记录及重入行为需在第五章单独验证。

### 13.2 Error type 决定是否 retry

`CodexErr::retry_delay(retry_count)` 返回：

- `None`：terminal error。
- `Some(delay)`：可 retry，优先 server advice，否则 exponential backoff + jitter。

Terminal 典型包括：

- turn aborted/interrupted。
- invalid request/prompt/image。
- quota/usage limit。
- context window exceeded。
- sandbox/policy errors。
- unsupported operation。

Retryable 典型包括：

- stream/connection failure。
- timeout/request timeout。
- rate limit。
- unexpected status/internal server error。
- response stream failed。
- selected I/O/JSON/task errors。

完整分类见 [`protocol/src/error.rs:397-447`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/protocol/src/error.rs#L397-L447)。

两个例外值得单列：`ServerOverloaded` 和 `RetryLimit` 只有携带 server retry advice 才可重试；`ContentFilter` 属于可重试类别，sampling controller 会先把 model-specific guidance 记录进 history，再决定重试。后者见 [`responses_retry.rs:66-85`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/responses_retry.rs#L66-L85)。

### 13.3 Retry budget 来自 provider config

`stream_max_retries()` 应用默认值和硬上限。外层 state 维护：

- 一般 retries。
- connection retries。
- connection retry delay。

定义见 [`responses_retry.rs:23-47`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/responses_retry.rs#L23-L47)。

### 13.4 Server advice 控制时间，不扩大次数

普通 retry：

```text
retry_at = server Retry-After deadline
        or now + local exponential backoff/jitter
```

即使 server 给了更晚 deadline，也不自动增加 `max_retries`。逻辑见 [`responses_retry.rs:84-167`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/responses_retry.rs#L84-L167)。

### 13.5 Retry 等待可被抢占和取消

`handle_response_stream_error()` 的 future 同时包在：

```text
.or_cancel(&preempt)
.or_cancel(&cancellation_token)
```

所以长 `Retry-After` 不会阻止用户 instant interrupt 或 turn abort。见 [`session/turn.rs:1707-1733`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L1707-L1733)。

---

## 14. Unbounded connection retry 与 transport fallback

### 14.1 无界 retry 只覆盖窄场景

`UnboundedConnectionRetries` 必须同时满足：

- sampling request。
- `ConnectionFailed`。
- 非 internal session source。
- 非 Amazon Bedrock。
- feature enabled。

这条分支使用独立 delay：5 秒起步，指数增长，最大 60 秒；不消耗一般 retry 计数。见 [`responses_retry.rs:94-116`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/responses_retry.rs#L94-L116)。

**架构判断**：无界重连不是“所有错误永远重试”。它只服务暂时离线的交互式本地场景，且必须保持可取消。

### 14.2 WebSocket retry exhausted 后切 HTTP

当一般 retry 已达到 max：

1. 尝试 `try_switch_fallback_transport()`。
2. 若 server 提供 retry deadline，先等到 deadline。
3. 发 warning。
4. retries 归零。
5. 外层 loop 再发一次 request，此时走 HTTP。

见 [`responses_retry.rs:118-139`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/responses_retry.rs#L118-L139)。

### 14.3 Fallback 是 session-sticky

`force_http_fallback()` 原子设置 `disable_websockets`，清空 cached WebSocket session，并记录 `codex.transport.fallback_to_http`。见 [`client.rs:714-733`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L714-L733)。

这样避免每个新 turn 再次尝试已知不健康 transport，形成“WS 失败 → HTTP 成功 → 下一 turn 又 WS 失败”的抖动。

### 14.4 两层预算

WebSocket retry budget 用尽并成功切到 HTTP 后，retry 计数归零。不计无界 connection retry、auth recovery 和底层 request retry 时，一次逻辑 sampling 可经历：

```text
WebSocket initial + WS retries
  -> fallback
HTTP initial + HTTP retries
```

这不是越过预算；它是每种 transport 各自拥有恢复机会。日志和 metrics 必须包含 transport，否则只看总 attempt 数会误判。

---

## 15. 401 Auth recovery 与 sampling retry 分层

### 15.1 Provider recovery 有单次保护，auth manager 按阶段推进

HTTP 和 WebSocket 建立路径各自维护：

- `auth_recovery`。
- `provider_auth_recovery_attempted`。
- `PendingUnauthorizedRetry` telemetry。

`handle_unauthorized()` 每个局部 recovery loop 最多尝试一次 provider-specific recovery；成功便返回并重发。随后若仍遇到鉴权错误，auth manager 由 `has_next()` / `next()` 推进有限恢复阶段，并分别处理 policy、permanent、transient failure。因此不能把全部鉴权恢复概括为“一次 token refresh”。见 [`client.rs:2724-2862`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L2724-L2862)。

### 15.2 为什么不能统一到通用 retry

401 recovery 会改变 credential/auth owner。它必须：

- 重建 client setup。
- 失效不属于新 auth owner 的 WebSocket state。
- 更新 telemetry，标记 retry-after-unauthorized。
- 防止无限 refresh loop。

普通 retry 只等待并重发，不具备这些 ownership 语义。

---

## 16. Preempt、interrupt 与 cancel

### 16.1 Instant interrupt 从 input queue 开始

启用 `Feature::InstantInterrupt` 时，捕获 `StepContext` 会创建 `preempt` token。见 [`session/mod.rs:3970-3981`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L3970-L3981)。

sampling 开始时，`watch_user_input()`：

1. 先订阅 activity channel，避免检查 queue 与订阅间丢事件。
2. 循环检查当前 turn 是否有 pending user input。
3. 有输入时 cancel preempt token。

见 [`session/input_queue.rs:292-316`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/input_queue.rs#L292-L316)。

### 16.2 Event wait 同时监听两个取消源

```text
stream.next()
  .or_cancel(preempt)
  .or_cancel(turn cancellation)
```

- turn cancellation：返回 `TurnAborted`。
- preempt：保留 turn，返回 `needs_follow_up=true`。

见 [`session/turn.rs:2615-2657`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2615-L2657)。

### 16.3 有 interrupt handle 时先终止生成，再排空 stream

preempt 发生后：

- 若 stream 提供 interrupt sender，Responses Lite 会发送 interrupt。
- preempt token 替换为新的空 token。
- `needs_follow_up=true`。
- 继续读取当前 stream，直到协议正常终止。

这样可以复用连接与已产生 history。

若没有 interrupt handle，则 drop stream，直接返回 follow-up，让下一 step 消化 pending input。

### 16.4 Preempt 也能打断 retry sleep

如果用户输入到达时请求正处于 backoff，外层 `or_cancel(preempt)` 立即结束等待，并返回需要 follow-up 的成功控制结果，而不是报告 network failure。

### 16.5 Mailbox preemption 是另一条策略

部分 assistant commentary/reasoning item 完成后，如果有 pending mailbox mail，Core 可在 item 边界提前结束 response 消费并要求 follow-up；是否延后由 `DeferMailboxPreemption` 控制。见 [`session/turn.rs:2749-2812`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2749-L2812)。

它不是 transport error，也不是用户 instant interrupt；第八章再分析 mailbox 语义。

---

## 17. 控制事件与可观测性

### 17.1 Response 级 metadata

Event loop 还处理：

- server-selected model 与 mismatch warning。
- model verification 提示。
- moderation metadata。
- safety buffering。
- server reasoning-included 标记。
- rate-limit snapshot。
- model catalog ETag。

见 [`session/turn.rs:2894-2948`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2894-L2948)。

这些信息不一定产生 assistant text，但会改变 UI、token accounting、模型缓存或后续请求行为。

### 17.2 最小 tracing 维度

源码已围绕请求记录：

- thread/turn ID。
- requested model 与 server model。
- provider/wire API/transport。
- reasoning effort。
- request/response ID。
- TTFT。
- input/cache/output/reasoning tokens。
- retry count 与 delay。
- fallback counter。
- WebSocket continuation mode/reason。
- auth recovery mode/phase。
- tool call IDs 的有界集合。

### 17.3 Trace terminal 必须完整

`map_response_events()` 区分：

- completed。
- provider failed。
- consumer dropped/cancelled。
- stream closed before terminal event。

只记录“request started”与“最后一个 delta 时间”无法判断用户取消、网络断流还是 provider 正常完成。

---

## 18. 五条典型时序

### 18.1 HTTP/SSE 正常回答

```text
Prompt
  -> POST /responses
  -> SSE stream
  -> Created
  -> OutputItemAdded(agent message)
  -> OutputTextDelta*
  -> OutputItemDone
  -> Completed(usage, end_turn=true/None)
  -> flush + return
```

### 18.2 WebSocket 工具 follow-up 的增量请求

```text
request 1: full input
  -> response_id R1 + tool call item
  -> tool result append to history

request 2:
  previous_response_id = R1
  input = only newly appended tool result
  -> provider continues same turn
```

如果 request properties 或 input prefix 不一致，request 2 自动改为 full，不报业务错误。

### 18.3 WebSocket 连续失败后 HTTP 恢复

```text
WS attempt
  -> retryable error
  -> backoff / Retry-After
  -> WS retries exhausted
  -> disable WebSocket for session
  -> reset retry counter
  -> HTTP attempt
  -> SSE Completed
```

### 18.4 用户输入抢占正在生成的 response

```text
watch_user_input detects pending input
  -> preempt.cancel()
  -> interrupt current generation if supported
  -> drain or drop stream safely
  -> SamplingRequestResult(needs_follow_up=true)
  -> step loop records pending input
  -> next StepContext + next sampling
```

### 18.5 Retry-After 等待期间 turn 被取消

```text
provider error + Retry-After
  -> sleep_until(deadline)
  -> turn cancellation token fires
  -> wait stops immediately
  -> TurnAborted
  -> no further transport attempt
```

---

## 19. 对现有 agent-loop 项目的直接启示

以下是项目建议，不是 Codex 官方设计声明。

### 19.1 定义 transport-neutral stream protocol

建议：

```text
ModelEvent =
  ResponseStarted
  | ItemStarted
  | TextDelta
  | ToolArgsDelta
  | ReasoningDelta
  | ItemCompleted
  | UsageUpdated
  | ResponseCompleted
```

HTTP、WebSocket、本地模型 adapter 都映射到它。Agent loop 不解析 SSE/WS frame。

### 19.2 为 item 与 response 分别建 lifecycle

至少保证：

```text
ItemStarted -> Delta* -> ItemCompleted
ResponseStarted -> Item* -> ResponseCompleted
```

UI 可以吃 delta；history 主要吃 completed item；usage/turn decision 吃 response completed。

### 19.3 Retry reason 必须结构化

建议记录：

```text
RetryDecision {
  error_kind,
  retryable,
  attempt,
  max_attempts,
  delay_source: server | local,
  delay_ms,
  transport,
  fallback_activated,
}
```

只有“reconnecting”字符串无法回答费用、延迟和失败率问题。

### 19.4 传输 fallback 要有明确作用域

可选作用域：request、turn、session、provider health window。必须显式选择。对交互式 agent，session-sticky 常比每 request 重试坏 transport 更稳定。

### 19.5 Auth retry 不要占用普通 network retry

鉴权恢复可能更新 credentials；账号切换还会改变 auth ownership。应有独立恢复阶段、次数保护和 audit event。不能用通用 exponential backoff 无限重发旧 token。

### 19.6 Retry sleep 必须可取消

任何 backoff、`Retry-After`、连接重试都必须监听：

- turn abort。
- preempt/steering。
- session shutdown。

否则“正在重连”会阻止用户立即修改任务。

### 19.7 Preempt 返回控制结果，不返回故障

建议：

```text
SamplingExit =
  Completed(result)
  | Preempted(partial_state)
  | Cancelled
  | Failed(error)
```

不要让上层根据错误字符串猜测是否应该继续 turn。

### 19.8 保存逻辑 request，压缩只留在 transport

WebSocket continuation 可只发 delta，但 tracing、审计、rollout 应保留完整逻辑 request。否则重放和问题定位只能看到残缺 wire payload。

### 19.9 为 stream 建 terminal invariant

每个 inference attempt 最终必须是：

- completed。
- failed。
- cancelled。

若 stream EOF 但没有协议 terminal event，必须视为 failed，不能静默成功。

### 19.10 限制 delta 队列并定义背压

Codex 使用容量 1600 的 channel。自研实现也应明确：

- channel 容量。
- producer 超速时策略。
- consumer drop 时取消。
- 大 tool-arg delta 的内存上限。

---

## 20. 容易误读的地方

### 20.1 “WebSocket 是另一套 agent loop”

不对。WebSocket 与 HTTP 最终产出相同 `ResponseStream<ResponseEvent>`，共享同一 sampling/event loop。

### 20.2 “WebSocket continuation 等于只保存增量 history”

不对。完整逻辑 request/history 仍存在；增量只是 wire optimization。

### 20.3 “`Completed` 表示 turn 完成”

不对。它只表示本次 provider response 完成；tool call、`end_turn:false`、pending input 都可能要求下一 step。

### 20.4 “Stream EOF 等于成功”

不对。没有内部 `Completed` 的关闭被视为 stream error；合法的 interrupted incomplete 会先映射为 `Completed(end_turn=false)`。

### 20.5 “所有错误都按固定次数 retry”

不对。terminal error 不 retry；部分 connection failure 可进入可取消的无界重连；WebSocket budget 后还可切 HTTP。

### 20.6 “`Retry-After` 可以突破 retry 上限”

不对。它控制等待 deadline，不自动增加 attempt budget。

### 20.7 “Fallback 只影响当前 request”

不对。WebSocket disable 状态属于 session 级 `ModelClient`，后续 turns 也走 HTTP。

### 20.8 “401 就交给普通 retry”

不对。transport 内部先尝试受保护的 provider recovery，再按可用 auth recovery 阶段推进；重发时重新解析 credentials 和 ownership。

### 20.9 “Preempt 和 cancel 都是取消 token，所以语义相同”

不对。preempt 返回 follow-up，保留 turn；cancel 返回 `TurnAborted`。

### 20.10 “Delta 应逐条写入 durable history”

不对。delta 主要用于实时投影；完整 `OutputItemDone` 才是稳定提交边界。

---

## 21. 本章小结

Codex 的模型请求链路可压缩为：

```text
Prompt
  -> turn-scoped ModelClientSession
  -> WebSocket preferred / HTTP fallback
  -> full or incremental wire request
  -> unified ResponseStream
  -> item lifecycle + delta projection
  -> response.completed terminal barrier
  -> retry / auth recovery / transport fallback
  -> preempt continues turn; cancellation aborts turn
```

其核心工程思想是：

- Provider transport 与 agent-loop 业务状态分层。
- WebSocket 增量是 wire optimization，不改变逻辑 request。
- SSE/WS 统一成 typed event protocol。
- Item completion、response completion、turn completion 三层分离。
- Retryable error、terminal error、auth recovery、fallback 各有独立决策域。
- Server advice 控制等待时间，不突破 retry budget。
- Backoff 可被 preempt/cancel 打断。
- 每个 inference attempt 必须有 completed/failed/cancelled 终态。

第五章将继续追踪 `OutputItemDone` 中产生的 tool call：tool specs 如何注册，router 如何匹配调用，orchestrator 如何执行并把结果写回 history。

# 第五章：Tool Runtime——注册、路由、并发执行与结果回流

[返回总目录](../README.md) · [上一章：模型请求、流式事件、重试与抢占](./04-model-request-streaming.md)

---

## 1. 导读与结论

本章沿着第四章的 `OutputItemDone` 继续，回答十个问题：

1. 一个工具如何同时拥有模型声明和可执行 runtime。
2. built-in、MCP、extension、dynamic tool 如何进入同一 registry。
3. “已注册”“模型可见”“允许执行”为什么不是同一件事。
4. direct、deferred、Code Mode、hidden exposure 如何影响调用面。
5. 模型返回的 function/custom/tool-search item 如何变成 `ToolCall`。
6. 多个工具为何可以同时执行，但结果仍按模型调用顺序进入 history。
7. pre/post hooks、lifecycle、telemetry 分别包在哪一层。
8. 普通错误、模型可恢复错误、runtime fatal、cancel 如何回流。
9. `ToolOrchestrator` 为什么只是 exec 类工具的 approval/sandbox 子系统，不是所有工具的总调度器。
10. 这些设计如何迁移到我们的 agent-loop。

**结论先行**：Codex 的 Tool Runtime 不是一个巨型 `switch(tool_name)`，而是一条分层管线：

```text
step 配置 + model 能力 + environment + MCP/extension/dynamic tools
                              |
                              v
                     build_tool_router()
                              |
              +---------------+----------------+
              |                                |
              v                                v
       ToolRegistry                     model_visible_specs
  name -> runtime + exposure          发给当前 sampling request
              |                                |
              +---------------+----------------+
                              |
                    model emits tool call
                              |
                              v
                 ToolRouter::build_tool_call
                              |
                              v
                     ToolCallRuntime
        readiness -> 并发门 -> registry dispatch -> hooks
                              |
                              v
                    CoreToolRuntime::handle
                              |
              +---------------+----------------+
              |                                |
              v                                v
       普通 handler                     exec 类 handler
 MCP/dynamic/control/...         ToolOrchestrator approval/sandbox
              |                                |
              +---------------+----------------+
                              |
                              v
                         ToolOutput
                              |
          direct: ResponseInputItem / Code Mode: JSON value
                              |
                              v
             FuturesOrdered drain -> conversation history
                              |
                              v
                     下一次 model sampling
```

最重要的六个判断：

- **声明与执行必须同源**。`ToolExecutor` 同时提供 `tool_name`、`spec`、`exposure` 和 `handle`，降低 schema 与 handler 漂移风险。
- **Registry 是执行能力表，不等于模型菜单**。模型只看到经过 step policy、tool mode 和 collision 处理后的 `model_visible_specs`。
- **并发资格属于具体 runtime**。模型请求允许 parallel calls，不代表所有工具都能并发；runtime 仍需显式声明。
- **并发执行与有序提交可以同时成立**。工具 task 会立即 spawn；`FuturesOrdered` 按插入顺序回收结果并写入 history。
- **PostToolUse 不是事务回滚**。hook 可以拦截或替换模型可见结果，但工具副作用可能已经发生。
- **Approval/sandbox 是执行域能力**。只有需要它的工具下沉到 `ToolOrchestrator`；不应把所有工具硬塞进 shell 执行模型。

---

## 2. 分析基线与范围

本章基于 commit [`c3d3b142d10f4316b46e35aad7e5317e7e506cb7`](https://github.com/openai/codex/tree/c3d3b142d10f4316b46e35aad7e5317e7e506cb7)。所有源码链接锁定此提交。分析方式为静态阅读、调用链交叉验证；未运行上游 Rust 测试。

主证据文件：

| 文件 | 用途 |
| --- | --- |
| `tools/src/tool_executor.rs` | 通用 `ToolExecutor`、`ToolExposure` 契约 |
| `core/src/tools/spec_plan.rs` | 当前 step 的工具计划、registry 与模型声明构造 |
| `core/src/tools/registry.rs` | 注册、冲突处理、hook、执行分派、结果接受 |
| `core/src/tools/router.rs` | wire item 到 `ToolCall`、执行入口 |
| `core/src/tools/parallel.rs` | readiness、并发门、spawn、cancel、结果编码 |
| `core/src/stream_events_utils.rs` | `OutputItemDone` 识别工具调用 |
| `core/src/session/turn.rs` | in-flight 队列、结果写 history、下一轮 sampling |
| `core/src/tools/context.rs` | invocation、source、call state、具体 output |
| `core/src/tools/code_mode/mod.rs` | Code Mode 嵌套调用复用同一 runtime |
| `core/src/tools/orchestrator.rs` | approval、sandbox、retry 子编排器 |
| `core/src/tools/handlers/*` | exec、MCP、dynamic 等具体适配器 |

本章不展开：

- Approval policy、权限提升和 sandbox 选择细节：第六章。
- History truncation、compaction 与 resume：第七章。
- collaboration 工具内部的 agent 生命周期：第八章。

---

## 3. 四层对象模型

### 3.1 `ToolExecutor`：声明与执行的共同根

通用契约要求 runtime 提供：

- `tool_name()`：稳定身份。
- `spec()`：模型可见声明。
- `exposure()`：默认暴露面。
- `search_info()`：延迟发现元数据。
- `supports_parallel_tool_calls()`：并发能力，默认 `false`。
- `handle(invocation)`：执行入口。

源码见 [`tool_executor.rs` L82-L123](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/tools/src/tool_executor.rs#L82-L123)。

**架构判断**：`spec` 和 `handle` 放在同一对象上，是 Tool Runtime 最关键的防漂移设计。若 schema 在 prompt 模块、handler 在执行模块、policy 又在第三处，工具演化时极易出现“模型能调用但 runtime 不认识”或“runtime 支持但模型从未看到”的断裂。

### 3.2 `CoreToolRuntime`：Core 注入横切能力

Core 在通用 `ToolExecutor<ToolInvocation>` 上增加：

- cancellation 是否必须等 handler 自行结束。
- built-in control tool 标识。
- immutable spec / Code Mode definition cache。
- readiness wait。
- third-party、MCP server 元数据。
- payload kind 匹配。
- telemetry tags。
- pre/post hook payload 与输入重写。
- streamed argument diff consumer。

源码见 [`registry.rs` L41-L177](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry.rs#L41-L177)。

这说明核心执行接口保持窄，Core 特有观测和生命周期能力通过扩展 trait 叠加。

### 3.3 `RegisteredTool`：runtime + 当前 step 的 exposure/spec

Registry 存的不是裸 handler，而是：

```text
RegisteredTool
  runtime: Arc<dyn CoreToolRuntime>
  exposure: ToolExposure
  model_spec: Option<Arc<ToolSpec>>   // 当前 step 可覆盖
```

源码见 [`registry.rs` L305-L327](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry.rs#L305-L327)。

同一共享 runtime 可在不同 step 获得不同描述或 exposure，覆盖值不回写 runtime。这与第三章的 `StepContext` 快照语义一致。

### 3.4 `ToolRouter`：冻结后的 step 工具计划

`ToolRouter` 同时保存：

- 完整 `ToolRegistry`。
- `model_visible_specs`。
- 当前 `ToolMode`。
- Code Mode 名称到真实 `ToolName` 的映射。
- namespace inventory。
- child-management tools 是否齐备。

源码见 [`router.rs` L70-L137](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/router.rs#L70-L137)。

**架构判断**：Router 不是运行时全局 singleton，而是当前 step 的 capability snapshot。模型看到的声明与之后执行使用的 runtime 来自同一个 router，避免长 turn 中 catalog 刷新改变已发出调用的含义。

---

## 4. 一次 step 如何构建工具计划

`build_tool_router()` 的输入包含 session、turn、model info、environment snapshot、MCP binding、extension data 和工具推荐。它按以下顺序组装：

1. 创建带 session `ToolPolicy` 的 registry。
2. 加入 core/built-in 工具。
3. 从 MCP handler cache 加入 MCP 工具。
4. 应用 MCP exposure policy。
5. 加入 extension executors。
6. 加入 dynamic tools。
7. 单独构造 hosted model tool specs。
8. `finalize_tool_router()` 处理 mode、search、collision、namespace 与最终可见声明。

源码见 [`spec_plan.rs` L121-L185](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/spec_plan.rs#L121-L185)。

可以压缩成：

```text
sources
  core ----+
  MCP -----+
  extension+--> ToolRegistry --> finalize --> ToolRouter
  dynamic -+                         |
                                      +--> direct specs
hosted specs -------------------------+
                                      +--> deferred search
                                      +--> Code Mode map
```

Hosted tool 与本地 runtime 分开是合理的：有些工具由模型 provider 执行，Core 只需把 spec 发给 provider，不应伪造一个本地 handler。

### 4.1 Finalize 不是简单过滤

`finalize_tool_router()` 会：

- 用 policy 再过滤 hosted specs。
- 应用 namespace exposure override。
- 强制 third-party tool 约束。
- 计算 effective tool mode。
- 保留 `exec` / `wait` 等 Code Mode 公共名称。
- 根据 deferred 工具是否存在决定是否注册 `tool_search`。
- 处理 namespace collision。
- 注册 Code Mode executors。
- 最后生成 `model_visible_specs`。

源码见 [`spec_plan.rs` L368-L479](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/spec_plan.rs#L368-L479) 和 [`spec_plan.rs` L531-L571](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/spec_plan.rs#L531-L571)。

因此，“工具列表”不是静态配置文件；它是 `session policy × turn config × model capability × environment × live integrations` 的计算结果。

---

## 5. 三个集合：注册、可见、可调用

最容易误读的地方，是把三个集合混为一谈。

| 集合 | 代表什么 | 主要决定者 |
| --- | --- | --- |
| Registered | Router 能按名称找到 runtime | Registry + policy + collision |
| Model-visible | 当前 request 初始声明给模型 | exposure + tool mode + model capability |
| Indirectly reachable | 可通过 tool search / Code Mode 到达 | deferred metadata + Code Mode map |

### 5.1 `ToolExposure` 是多面投影

六种 exposure：

| Exposure | Direct | Tool Search | Code Mode | Registry 中保留 |
| --- | ---: | ---: | ---: | ---: |
| `Direct` | 是 | 否 | 是 | 是 |
| `Deferred` | 否 | 是 | 是 | 是 |
| `DeferredModelOnly` | 否 | 是 | 否 | 是 |
| `DirectModelOnly` | 是 | 否 | 否 | 是 |
| `CodeModeOnly` | 否 | 否 | 是 | 是 |
| `Hidden` | 否 | 否 | 否 | 是 |

定义见 [`tool_executor.rs` L51-L103](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/tools/src/tool_executor.rs#L51-L103)。

`build_model_visible_specs()` 只收录 `is_directly_visible()` 的注册项，再追加 hosted specs，最后合并 namespace，见 [`spec_plan.rs` L646-L674](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/spec_plan.rs#L646-L674)。

### 5.2 Hidden 不等于不存在

Hidden 工具仍有 runtime，可以被内部受控路径调用；但 `supports_parallel_tool_calls()` 对 Hidden 返回 false，防止其无意参与普通模型调度，见 [`registry.rs` L599-L609](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry.rs#L599-L609)。

**项目建议**：我们自己的 agent-loop 应把“注册”“发现”“授权”“执行来源”建模为独立维度。单个 `enabled: bool` 无法表达 internal-only、deferred、direct-only 等真实需求。

---

## 6. 注册顺序、名称规范化与冲突

`ToolRegistry` 使用 `IndexMap<ToolName, RegisteredTool>`，同时保存 function namespace prefixes、首次冲突和 `ToolPolicy`，见 [`registry.rs` L330-L336](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry.rs#L330-L336)。

### 6.1 Trusted 与 external 的冲突策略不同

- Trusted 重复注册调用 `error_or_panic`。
- External 保留先注册者，记录 collision，返回 `false`。
- External 不能占用默认 namespace 的 `exec_command` / `shell_command`。
- 所有名称先 `with_default_namespace()`，避免 plain name 与显式 `functions` alias 双重注册。

源码见 [`registry.rs` L422-L502](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry.rs#L422-L502)。默认 namespace alias 的测试见 [`registry_tests.rs` L245-L280](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry_tests.rs#L245-L280)。

### 6.2 冲突可选择 fail-fast

Finalize 阶段在 `error_on_tool_collisions` 开启时，把首次名称冲突或 namespace owner/description 冲突升级为请求错误，见 [`spec_plan.rs` L475-L529](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/spec_plan.rs#L475-L529)。

**架构判断**：这里同时兼顾生产容错和开发期严格性。外部 catalog 中重复项可被跳过；严格模式能尽早暴露配置错误。

---

## 7. 从 provider item 到内部 `ToolCall`

`ToolRouter::build_tool_call()` 统一三种 wire item：

| Provider item | 内部 payload | 备注 |
| --- | --- | --- |
| `FunctionCall` | `ToolPayload::Function { arguments }` | 保留 namespace、call_id、加密参数 |
| client `ToolSearchCall` | `ToolPayload::ToolSearch` | 仅 `execution == "client"` |
| `CustomToolCall` | `ToolPayload::Custom { input }` | 支持 freeform/custom 输入 |

源码见 [`router.rs` L257-L320](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/router.rs#L257-L320)。

`handle_output_item_done()` 在 item 完成时：

1. 尝试构造 `ToolCall`。
2. 先记录模型原始 tool-call item。
3. 为该调用创建 child cancellation token。
4. 生成 `tool_future`。
5. 标记 `needs_follow_up = true`。

源码见 [`stream_events_utils.rs` L315-L358](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/stream_events_utils.rs#L315-L358)。

关键点：**调用 item 先入 history，结果稍后补入**。这保留了 `call -> result` 因果结构，也允许执行与剩余 response stream 重叠。

如果 wire item 无法解析且错误属于 `RespondToModel`，Core 会直接构造失败 output、写入 transcript 并要求下一轮 sampling；fatal 则终止，见 [`stream_events_utils.rs` L399-L429](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/stream_events_utils.rs#L399-L429)。

---

## 8. Dispatch 主链

一次 direct tool call 的主链是：

```text
OutputItemDone
  -> ToolCallRuntime::handle_tool_call
     -> record direct call
     -> handle_tool_call_with_source
        -> wait_until_ready(exact runtime)
        -> acquire parallel gate
        -> ToolRouter::dispatch_tool_call_with_state
           -> build ToolInvocation from same StepContext
           -> ToolRegistry::dispatch_any_with_state
              -> lookup + payload-kind check
              -> PreToolUse hook
              -> lifecycle start
              -> CoreToolRuntime::handle
              -> PostToolUse hook
              -> lifecycle finish
              -> AnyToolResult
     -> encode ResponseItemEnvelope
```

Router 用当前 `StepContext` 构造 `ToolInvocation`；兼容字段 `turn` 也指向 `step_context.turn`，见 [`router.rs` L359-L397](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/router.rs#L359-L397)。

Invocation 携带：session、turn、step context、cancellation token、diff tracker、call id、tool name、source 和 payload，见 [`context.rs` L55-L81](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/context.rs#L55-L81)。

这比只给 handler 一个 JSON args 更稳：handler 需要的环境、策略、取消和审计身份都来自同一调用上下文。

---

## 9. Readiness 与并发门

### 9.1 Readiness 在并发门之前

`ToolCallRuntime` 先从 registry 精确找到 runtime，读取：

- `supports_parallel_tool_calls()`。
- `finishes_on_cancellation()`。
- `wait_until_ready()`。

然后在 spawned task 中等待 readiness，再获取读写锁，见 [`parallel.rs` L125-L175](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/parallel.rs#L125-L175) 和 [`parallel.rs` L202-L239](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/parallel.rs#L202-L239)。

MCP runtime 的 readiness 会等待对应 server 可用，见 [`mcp.rs` L303-L317](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/mcp.rs#L303-L317)。

### 9.2 一把读写锁实现“并行组 + 独占工具”

```text
parallel-capable tool    -> read lock
non-parallel tool        -> write lock
```

因此：

- 多个 parallel 工具可同时持有 read lock。
- 非 parallel 工具必须等全部 read lock 释放。
- non-parallel 执行期间，后续 parallel 调用也不能进入。

并发资格不是按工具名称硬编码；由注册 runtime 决定。`exec_command` 显式返回 true，见 [`exec_command.rs` L111-L150](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs#L111-L150)。MCP 可由 server 标志或 `read_only_hint` opt-in，见 [`mcp.rs` L142-L152](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/mcp.rs#L142-L152)。

集成测试验证多个 shell call 以及 shell + 其他 parallel tool 确实重叠执行，见 [`tool_parallelism.rs` L155-L186](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/tests/suite/tool_parallelism.rs#L155-L186) 和 [`tool_parallelism.rs` L190-L222](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/tests/suite/tool_parallelism.rs#L190-L222)。

### 9.3 API 的 parallel flag 不等于 runtime 并发

第四章看到请求层可声明 `parallel_tool_calls: true`。那只是告诉模型“可在一个 response 生成多个调用”。真正执行时仍由每个 runtime 的并发声明和本地读写锁裁决。

**项目建议**：拆成两个配置：

```text
model_may_emit_parallel_calls
runtime_concurrency_class = Shared | Exclusive | Keyed(resource_id)
```

Codex 当前是 Shared/Exclusive 两级。我们的 agent-loop 若未来涉及浏览器 tab、数据库事务或同一文件写入，可扩展 keyed gate，而不是退回全局串行。

---

## 10. 并发执行，有序回写

Tool future 被放入 `FuturesOrdered`，见 [`turn.rs` L2776-L2790](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2776-L2790)。这些 future 内部已 spawn，所以不会因为 `FuturesOrdered` 而串行执行。

response stream 结束后，`drain_in_flight()` 按队列顺序取得结果，并调用 `record_annotated_conversation_items()` 写入 history，见 [`turn.rs` L2462-L2489](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2462-L2489)。所有 pending tool drain 完成后，才继续 token event、cancel 判定和下一步，见 [`turn.rs` L3151-L3179](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L3151-L3179)。

示例：

```text
model order:   call-A -------- call-B -------- call-C
execution:     A [==========]
               B [==]
               C    [=====]
completion:       B       C A
history order: result-A -> result-B -> result-C
```

**架构判断**：有序提交降低了 prompt 非确定性。执行完成顺序受机器负载影响；若直接按完成顺序写 history，同一模型输出可能产生不同后续上下文。

代价是 head-of-line blocking：B 已完成也要等 A 才能提交。但下一次 sampling 本来就要等本批工具完成，因此这里优先选择确定性。

---

## 11. Registry dispatch：检查、hook 与生命周期

`dispatch_any_with_state()` 是所有本地 runtime 的公共包裹层。

### 11.1 Lookup 与 payload kind

它先增加 active turn 的 tool-call 计数，再按 canonical name 查 runtime。找不到时产生 `RespondToModel`；payload kind 与 runtime 不匹配则为 fatal，见 [`registry.rs` L615-L698](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry.rs#L615-L698)。

区分合理：

- 未知名称可能是模型幻觉，可把错误给模型自修复。
- 已找到 runtime 却收到错误 payload，意味着 host 内部路由契约破坏，更像程序错误。

### 11.2 PreToolUse

Pre hook 可以：

- block：不执行 handler，错误回给模型。
- continue + updated input：runtime 把 hook-facing JSON 重建成 `ToolInvocation`。
- continue：原参数执行。

源码见 [`registry.rs` L700-L753](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry.rs#L700-L753)。

输入重写发生在 handler 前，因此是真正的调用变换；审计系统应同时保存 original 与 effective input，不能只留最终参数。

### 11.3 PostToolUse

Handler 完成后才运行 post hook。它可以：

- 添加 additional context。
- block result，向模型返回反馈错误。
- 用 feedback message 替换模型可见 output。
- 接受原结果并触发 `on_tool_result_accepted()`。

源码见 [`registry.rs` L811-L889](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry.rs#L811-L889)。源码注释明确：PostToolUse block 拒绝的是结果，不是已经完成的执行，见同文件 [`L846-L870`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry.rs#L846-L870)。

**项目建议**：将 hook 命名为 `before_execute` 与 `after_effect_before_commit_to_model` 更不易误解。对有副作用工具，UI 不应把 post-hook blocked 表述成“操作未发生”。

---

## 12. 取消不是简单 drop future

`ToolCallRuntime` 把 dispatch 放入 `tokio::spawn` 并用 `AbortOnDropHandle` 管理。外层 `select!` 同时等待 task 完成和 cancellation，见 [`parallel.rs` L202-L299](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/parallel.rs#L202-L299)。

取消时分三类：

1. runtime 声明 `finishes_on_cancellation()`：等待 handler 自己收尾。
2. terminal outcome 已发布或 task 已完成：等待真实结果。
3. 其他普通 handler：abort task，生成统一 aborted output，并发 lifecycle aborted event。

随后再次用 `terminal_outcome_reached.swap()` 防止 completion 与 cancellation 竞态重复发 terminal event，见 [`parallel.rs` L300-L327](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/parallel.rs#L300-L327)。

Dynamic tool 是第一类：调用已经交给 client，Core 需要等待 client/handler 发表最终状态，所以 `finishes_on_cancellation()` 返回 true，见 [`dynamic.rs` L119-L179](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/dynamic.rs#L119-L179)。

**架构判断**：cancel 是协议，不是布尔值。必须回答：

- 谁拥有停止动作？
- 副作用是否可能已发生？
- 谁发布唯一 terminal event？
- 给模型的 output 是 aborted、failed 还是无结果？

---

## 13. 错误分层与模型自恢复

工具公共错误只有两类：

```rust
RespondToModel(String)
Fatal(String)
```

定义见 [`function_call_error.rs` L1-L8](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/tools/src/function_call_error.rs#L1-L8)。

### 13.1 `RespondToModel`

适合参数错误、未知工具、策略拒绝、外部工具失败等可由模型调整的情况。`handle_tool_call()` 将其转换为对应 wire output：function、custom、tool-search 各有格式，见 [`parallel.rs` L333-L373](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/parallel.rs#L333-L373)。

### 13.2 `Fatal`

代表 runtime invariant 或 task join 失败，不应伪装成普通工具 observation。它会向上终止当前 sampling/turn。

### 13.3 工具业务失败不一定是 Rust `Err`

`ToolOutput` 具有 `success_for_logging()` 和模型可见 `success` 字段。一个工具可正常完成协议，但业务结果为失败；这与 transport/runtime fatal 不同。

**项目建议**：至少保留四维状态，而不是一个 `success`：

```text
dispatch_status: admitted | rejected | cancelled
handler_status: completed | failed | panicked
domain_status: success | failure | partial
model_result_status: delivered | replaced | blocked
```

---

## 14. `ToolOutput`：一个结果，多种投影

`ToolOutput` 不只返回字符串。它负责提供：

- lossy diagnostic log。
- logging success。
- handler duration。
- external-context 标识。
- history fallback token limit。
- direct model response item。
- post-hook input/response。
- Code Mode JSON result。
- host-only metadata。

源码见 [`tool_output.rs` L10-L65](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/tools/src/tool_output.rs#L10-L65)。

`AnyToolResult::into_response()` 把 output 转成 `ResponseItemEnvelope`，并把 fallback token limit 写入 harness metadata；`code_mode_result()` 则走 JSON 投影，见 [`registry.rs` L214-L238](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/registry.rs#L214-L238)。

### 14.1 exec output 的截断属于 output contract

`ExecCommandToolOutput` 保留 raw bytes、wall time、exit code、process id、原 token 数和 omitted bytes，见 [`context.rs` L380-L394](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/context.rs#L380-L394)。

Direct response 会根据模型截断 policy 生成文本；Code Mode 则生成结构化 JSON，见 [`context.rs` L413-L478](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/context.rs#L413-L478)。

**架构判断**：同一 authoritative result 派生 telemetry、hook、history、Code Mode 等不同视图，优于 handler 到处拼字符串。尤其不能把 telemetry log 当成完整工具结果；trait 注释明确 log 可有意丢失媒体和加密内容。

---

## 15. `ToolOrchestrator` 的真实边界

名字容易让人以为它调度所有工具。实际上公共路径是：

```text
ToolCallRuntime -> ToolRouter -> ToolRegistry -> CoreToolRuntime::handle
```

`ToolOrchestrator` 只被需要 approval/sandbox 的执行型 handler 使用。它接收实现 `ToolRuntime<Req, Out>` 的具体 runtime，执行：

```text
approval requirement
  -> environment / permissions
  -> sandbox selection
  -> first attempt
  -> sandbox denial classification
  -> optional approval + escalated retry
```

模块注释和入口见 [`orchestrator.rs` L1-L7](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/orchestrator.rs#L1-L7) 与 [`orchestrator.rs` L127-L168](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/orchestrator.rs#L127-L168)。底层 `ToolRuntime` 契约聚焦 environment 与一次 sandbox attempt，见 [`sandboxing.rs` L378-L419](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/sandboxing.rs#L378-L419)。

`exec_command` 的具体路径在 process manager 中创建 `ToolOrchestrator`、计算 exec approval requirement，再执行 runtime，见 [`process_manager.rs` L1462-L1529](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/unified_exec/process_manager.rs#L1462-L1529)。

这是一种“公共 dispatch + 领域子编排”结构：

- 公共层处理 identity、hooks、lifecycle、cancel、result。
- exec 领域处理 approval、permissions、sandbox、network retry。
- MCP 领域处理 server readiness、binding、remote call。
- dynamic 领域处理 client round-trip。

**项目建议**：不要建设一个懂 shell、HTTP、browser、MCP、human input 的万能 orchestrator。公共层只规定协议；安全与重试策略下沉到 capability-specific adapter。

---

## 16. 三种特殊执行路径

### 16.1 MCP：绑定快照与服务 readiness

MCP handler 执行前：

- 等目标 server ready。
- 从 session 准备当前 MCP call binding。
- 使用“实际执行调用的 binding”，避免之后 catalog refresh 改变资格。
- 从同一 step config 取得输出截断策略。

源码见 [`mcp.rs` L187-L240](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/mcp.rs#L187-L240)。

这再次体现 snapshot 原则：catalog 可以变化，但已发出的调用必须按原 step 的承诺执行。

### 16.2 Dynamic tool：runtime 向 client 反向发请求

Dynamic handler 解析 JSON 后，在 active turn 注册 oneshot response channel，发出 `DynamicToolCall` item，等待 client 回复或 cancel，再转换为 `FunctionToolOutput`，见 [`dynamic.rs` L186-L265](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/dynamic.rs#L186-L265)。

因此“本地 runtime”不等于“本进程完成工作”；它也可以是一个协议适配器。

### 16.3 Code Mode：入口不同，核心 dispatch 复用

Code Mode nested call 会：

1. 阻止 `exec` 自己递归调用。
2. 根据 function/freeform kind 构造同一个 `ToolPayload`。
3. 构造普通 `ToolCall`。
4. 调用 `ToolCallRuntime::handle_tool_call_with_source()`。
5. 把 `AnyToolResult` 投影成 JSON。

源码见 [`code_mode/mod.rs` L337-L415](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/code_mode/mod.rs#L337-L415)。

**架构判断**：Direct 与 Code Mode 共享 handler、policy、hooks、cancel 和 telemetry，只在 source 与 result encoding 处分叉。新增调用表面时，应尽量复用 capability runtime，而不是复制一套工具实现。

---

## 17. `exec_command` 端到端实例

把前面各层串起来：

```text
1. spec_plan 注册 ExecCommandHandler
2. model_visible_specs 把 exec_command schema 发给模型
3. model 返回 FunctionCall(name=exec_command, args=...)
4. build_tool_call -> ToolPayload::Function
5. ToolCallRuntime:
   - readiness
   - parallel read lock（exec opt-in）
   - spawn/cancel/timing
6. Registry:
   - payload kind
   - pre hook
   - lifecycle start
7. ExecCommandHandler:
   - parse args
   - resolve environment/workdir
   - normalize requested permissions
   - intercept apply_patch if applicable
   - build ExecCommandRequest
8. ProcessManager:
   - exec policy
   - ToolOrchestrator approval/sandbox/attempt
9. ExecCommandToolOutput:
   - raw output + process/session metadata
   - model-specific truncation
10. Registry post hook + lifecycle finish
11. ResponseItemEnvelope 按 call_id 写入 history
12. 下一轮 sampling 看见 command result
```

参数解析与环境解析见 [`exec_command.rs` L154-L220](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs#L154-L220)。权限归一化和 apply-patch interception 见 [`exec_command.rs` L310-L391](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs#L310-L391)。

这个实例说明 approval 并不是 Tool Runtime 的第一步。它位于参数已解析、环境已确定、执行请求已构造之后，但在真正产生外部副作用之前。

---

## 18. 观测性：排队时间与 handler 时间分离

`ToolCallTimingGuard` 记录三个时间：

- dispatch duration：收到调用到通过 readiness/并发门。
- handler duration：进入执行门到完成。
- total duration：整体。

源码见 [`parallel.rs` L416-L479](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/parallel.rs#L416-L479)。

这能区分：

```text
工具慢？            handler_duration 高
MCP server 未就绪？ dispatch_duration 高
被独占工具挡住？    dispatch_duration 高
结果编码/后置 hook？ total - dispatch - handler 增大
```

此外，Registry 统一记录 tool result、sandbox tags、MCP server tags、command category；具体 runtime 可追加 tags。公共 dispatch 是最合适的观测层，因为它覆盖所有 handler，又能看见 canonical identity 与最终 outcome。

**项目建议**：日志至少以 `thread_id + turn_id + call_id + tool_name + source` 为关联键。对嵌套 Code Mode，还需 `cell_id + runtime_tool_call_id`，但不能拿 runtime-local id 代替全局 call id。

---

## 19. 对我们的 agent-loop 的可迁移设计

### 19.1 建立 typed capability contract

建议最小接口：

```text
ToolRuntime
  identity() -> ToolName
  declaration(step) -> ToolSpec
  exposure(step) -> Exposure
  concurrency(invocation) -> ConcurrencyClass
  readiness(ctx) -> awaitable
  execute(ctx, payload) -> ToolResult
  encode_for_model(result) -> ToolOutputItem
```

Spec 与 execute 必须由同一个 runtime 提供，step override 作为 wrapper，不修改共享对象。

### 19.2 冻结 `StepToolPlan`

```text
StepToolPlan
  registry
  model_visible_specs
  indirect_routes
  policy_snapshot
  environment_snapshot
  version / digest
```

模型请求发出后，此 plan 应保持可执行，直到它产生的所有 tool calls 完成。热更新只影响下一 step。

### 19.3 调度器只做四件事

公共调度器职责建议限制为：

1. identity/payload validation。
2. readiness + concurrency admission。
3. lifecycle/cancel/observability。
4. result normalization + ordered commit。

Approval、sandbox、浏览器会话、DB transaction、MCP reconnect 分别属于具体 capability adapter。

### 19.4 把执行和提交拆开

```text
Execution completion order: nondeterministic
History commit order:       model call order
```

保存两套时间戳：`result_ready_at` 与 `committed_at`。这样既能诊断 head-of-line blocking，又能保持 prompt 确定性。

### 19.5 明确 side-effect phase

建议 lifecycle：

```text
received
  -> admitted
  -> executing
  -> effect_committed?       // handler-specific
  -> result_ready
  -> result_policy_checked
  -> model_result_committed
```

Post hook block 只影响最后两步，不能隐含回滚 `effect_committed`。

### 19.6 结果保留 authoritative value

`ToolResult` 应先保留完整结构，再派生：

- model view（可截断）。
- UI view（可分页/流式）。
- telemetry view（脱敏、限长）。
- audit view（受控持久化）。
- nested-code value（结构化）。

不要让 telemetry 截断后的字符串成为唯一真相。

---

## 20. 常见误读

### 误读 1：模型看到的 tools 就是 registry 全量

不是。Registry 还包含 deferred、CodeModeOnly、Hidden；hosted specs 又可能没有本地 runtime。

### 误读 2：`parallel_tool_calls = true` 表示所有工具并行

不是。模型可以生成多调用；runtime 仍通过 `supports_parallel_tool_calls()` 和读写锁决定执行并发。

### 误读 3：`FuturesOrdered` 让工具串行执行

不是。dispatch task 在 future 内被 `tokio::spawn`；`FuturesOrdered` 约束的是结果观察/提交顺序。

### 误读 4：PostToolUse block 会阻止副作用

不是。它发生在 handler 返回之后，只能阻断或替换模型可见结果。

### 误读 5：所有工具都经过 `ToolOrchestrator`

不是。所有本地工具经过 Registry dispatch；只有 approval/sandbox 类执行域使用 `ToolOrchestrator`。

### 误读 6：工具失败都终止 turn

不是。大量错误被编码成 tool output 交给模型自修复；只有 fatal/invariant 类错误向上终止。

### 误读 7：取消 task 就等于取消外部动作

不是。动态工具、远端调用、已经启动的进程可能需要协议化收尾；因此 runtime 可要求 `finishes_on_cancellation()`。

---

## 21. 本章总结

Codex Tool Runtime 的核心不是工具数量，而是稳定地连接五个平面：

```text
声明平面：spec / namespace / exposure
发现平面：direct / deferred / Code Mode
执行平面：registry / runtime / adapter
控制平面：readiness / concurrency / approval / cancellation
反馈平面：ToolOutput / hooks / history / next sampling
```

一条完整工具链的关键不变量是：

1. 模型声明与 executable runtime 同源。
2. 调用使用发出声明时的 step snapshot。
3. canonical identity 贯穿 dispatch、hook、telemetry 和 result。
4. runtime 自己声明并发与取消语义。
5. 工具可并发执行，但结果按模型调用顺序提交。
6. 可恢复错误返回模型，runtime invariant 才终止 turn。
7. post-execution policy 不伪装成副作用回滚。
8. safety orchestrator 是执行域子系统，不污染所有工具协议。

下一章将深入本章刻意保持黑盒的安全子系统：approval request 如何产生，exec policy 如何分类命令，sandbox 如何选择，denial 如何触发升级重试，以及权限边界如何贯穿本地和远端 executor。


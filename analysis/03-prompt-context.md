# 第三章：Prompt 与 Context 管理

[返回总目录](../README.md) · [上一章：Agent Loop 主状态机](./02-agent-loop.md)

---

## 1. 导读与结论

本章回答八个问题：

1. Codex 最终发给模型的 prompt 由哪些部分组成。
2. base instructions、developer context、AGENTS.md、用户输入、skills/plugins 分别从哪里进入。
3. `TurnContext` 与 `StepContext` 为什么不能合并成一个配置对象。
4. 首次 turn 为什么注入完整上下文，后续 turn 为什么通常只注入 diff。
5. history 在发送前会做哪些修复、过滤与模态降级。
6. 普通 Responses API 与 Responses Lite 如何装配 instructions 和 tools。
7. prompt 顺序、角色与持久化边界如何共同决定模型看到的上下文。
8. 哪些设计可直接迁移到现有 agent-loop 项目。

**结论先行**：Codex 的 prompt 不是一个字符串模板，而是一个在采样边界生成的结构化请求快照：

```text
静态/半静态来源
  model instructions / config / managed policy
  AGENTS.md / host user+thread instructions
  environment / permissions / collaboration mode
  tool specs / MCP / apps / plugins
                 |
                 v
          StepContext 一致性快照
                 |
                 +----> WorldState full 或 diff ----+
                 |                                  |
本 turn 输入 ----+----> skills/plugins injections   |
                 |                                  v
历史消息/工具结果 ---------------------------> ContextManager history
                                                    |
                                                    v
                                     normalize + modality filtering
                                                    |
                                                    v
Prompt { input, tools, base_instructions,
         output_schema, parallel_tool_calls }
                                                    |
                                                    v
                                      ResponsesApiRequest
```

最重要的四个判断：

- **Prompt 是结构，不是拼接后的单一文本**。`input`、`tools`、base instructions、output schema 分开保存，直到 provider 请求层才完成最终映射。
- **Context 是有状态协议**。完整上下文建立 baseline；后续只记录模型需要知道的变化，而不是每轮重发全部配置。
- **History 是 prompt 的事实源，但不是原样透传**。发送前会补齐 call/output 对、删除孤儿 output，并按模型能力过滤图片和音频。
- **采样一致性由 `StepContext` 保证**。同一次请求使用同一份 model、tools、MCP、environment、AGENTS.md 与 permission 视图。

---

## 2. 分析基线与范围

本章基于 commit [`c3d3b142d10f4316b46e35aad7e5317e7e506cb7`](https://github.com/openai/codex/tree/c3d3b142d10f4316b46e35aad7e5317e7e506cb7)，所有源码链接锁定该提交。

主证据文件：

| 文件 | 本章用途 |
| --- | --- |
| `core/src/session/step_context.rs` | 单次采样的一致性快照 |
| `core/src/session/turn.rs` | turn 输入、skills/plugins 注入、Prompt 构造 |
| `core/src/session/mod.rs` | 初始上下文、context diff 与 reference context |
| `core/src/session/world_state.rs` | WorldState section 注册与数据来源 |
| `core/src/context/world_state/mod.rs` | section snapshot、full/diff 渲染、placement |
| `core/src/context_manager/history.rs` | history 持久态、WorldState baseline、发送前标准化 |
| `core/src/context_manager/updates.rs` | fragment 合并为 ResponseItem |
| `core/src/agents_md.rs` | AGENTS.md 发现、聚合与预算 |
| `core/src/agents_md_manager.rs` | user/thread/project instructions 的刷新与缓存 |
| `core/src/client_common.rs` | `Prompt` 数据结构 |
| `core/src/client.rs` | `Prompt` 到 Responses API 请求的最终映射 |

本章只分析**模型请求前的上下文建模与装配**。以下内容仅说明接口，不展开：

- stream、retry、WebSocket/SSE：第四章。
- 工具选择、执行和结果回流：第五章。
- approval 与 sandbox 决策：第六章。
- compaction、token budget、rollout 恢复算法：第七章。
- multi-agent 的控制协议：第八章。

---

## 3. 先区分六个对象

| 对象 | 生命周期 | 保存什么 | 不保存什么 |
| --- | --- | --- | --- |
| `TurnContext` | 一个业务 turn | turn 配置、provider、初始 settings、输出 schema、动态工具 | 某次 sampling 的实时工具/MCP/环境快照 |
| `StepContext` | 一次 sampling step | settings、environments、MCP binding、tool router、AGENTS.md、realtime | 完整 conversation history |
| `WorldState` | 某个 step 的模型可见状态 | 按稳定 section ID 组织的当前状态 | 历史消息本身 |
| `WorldStateSnapshot` | context baseline | 各 section 的可序列化比较状态 | 模型可见文本全文的简单副本 |
| `ContextManager` | 当前 context window | 有序 history、metadata、WorldState baseline、token info | provider 专用请求结构 |
| `Prompt` | 单次 model request | input、tools、base instructions、output schema | 网络传输、stream 状态、retry 状态 |

`TurnContext` 明确被定义为单个 turn 所需上下文；其中保留初始 settings，但注释要求 step 相关配置从 `StepContext` 读取。见 [`session/turn_context.rs:323-388`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn_context.rs#L323-L388)。

`StepContext` 则冻结了本次请求使用的 settings、环境、capability roots、MCP binding、tool router 和 `LoadedAgentsMd`。见 [`session/step_context.rs:24-50`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/step_context.rs#L24-L50)。

**架构判断**：`TurnContext` 回答“这项工作是什么”，`StepContext` 回答“这一次模型调用看到什么”。热更新应在下一 step 捕获，而不能修改已经出发的请求。

---

## 4. 最终 Prompt 的真实形态

### 4.1 `Prompt` 不是字符串

`Prompt` 有六个主要字段：

```rust
Prompt {
    input: Vec<ResponseItem>,
    tools: Arc<[ToolSpec]>,
    parallel_tool_calls: bool,
    base_instructions: BaseInstructions,
    output_schema: Option<Value>,
    output_schema_strict: bool,
    cyber_access_program: Option<CyberAccessProgram>,
}
```

源码见 [`client_common.rs:20-47`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client_common.rs#L20-L47)。

这意味着模型请求至少有四条彼此独立的数据通道：

- `input`：对话、上下文 fragment、工具调用/结果等有序 items。
- `tools`：当前 step 可见的工具声明。
- `base_instructions`：模型级基础指令。
- `output_schema`：最终输出的结构约束。

不要把它们提前压成一段文本。提前扁平化会丢失 role、item ID、tool declaration、结构化输出和 provider 特性。

### 4.2 `build_prompt()` 是薄装配层

`build_prompt()` 不负责重新发现工具或读取 AGENTS.md；这些已经冻结在 `StepContext` 中。它只把历史 input 与 step 快照组合成 `Prompt`。见 [`session/turn.rs:1563-1588`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L1563-L1588)。

关键分支是 `incremental_tools`：

| 模式 | `Prompt.tools` | `Prompt.base_instructions` |
| --- | --- | --- |
| 普通模式 | 当前 `tool_router.model_visible_specs()` | 当前 base instructions |
| incremental tools | 空 | 空 |

incremental 模式并非不要 tools/instructions，而是它们已经作为 history 中的结构化 items 管理，避免请求层重复发送。

### 4.3 Prompt 到 API request 还有一次适配

`ModelClient::build_responses_request()` 才完成 provider 请求映射：

1. 克隆并格式化 input。
2. 过滤当前模型不支持的 configuration items。
3. 为 request-only prefix 建立 thread 稳定 ID。
4. 普通模式把 tools 放入顶层 `tools` 字段。
5. Responses Lite 把 tools 转成 `AdditionalTools` item。
6. base instructions 转成 developer message 并插到 input 前部。
7. 映射 reasoning、service tier、output schema、cache key 等字段。

见 [`client.rs:951-1084`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L951-L1084)。

**架构判断**：Core 先构造 provider-neutral 的 `Prompt`，client 层再决定“放顶层字段”还是“放 input item”。这样 context 管理不会被某一种 API wire format 绑死。

---

## 5. Prompt 数据流全景

### 5.1 组装顺序

一次新 turn 的关键顺序是：

```text
1. 捕获 first StepContext
2. 构建 WorldState
3. 记录 full context 或 context diff
4. 根据用户输入选择 skills/plugins
5. hook 检查并记录 turn input
6. 记录 skill/plugin/extension injections
7. snapshot history
8. history.for_prompt(model.input_modalities)
9. build_prompt(history, StepContext, base instructions)
10. build_responses_request(Prompt)
```

turn 起始处先捕获 `StepContext`，再并行记录上下文和计算 diff display roots；随后构建技能/插件注入、记录用户输入，最后记录 injection items。见 [`session/turn.rs:255-411`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L255-L411)。

### 5.2 为什么先进入 history，再构造 Prompt

Codex 没有维护一份独立的“当前 prompt 文本”。模型可见的 context、用户输入、tool call/output 和动态注入都会进入统一的有序 history。sampling 时再从 history 创建请求副本。

收益：

- retry 可重建等价 input。
- tool follow-up 自然看到先前调用和结果。
- resume/compaction 能围绕同一种 item 模型工作。
- UI/rollout 与模型请求共享可审计事实源。
- 模态过滤只改请求副本，不破坏持久历史。

---

## 6. `StepContext`：Prompt 的一致性边界

### 6.1 同一 request 必须共享同一快照

`StepContext` 同时持有：

- `ResolvedStepSettings`：model、reasoning、service tier 等。
- `TurnEnvironmentSnapshot`：环境选择与 cwd。
- `McpBinding`：本 step 的 MCP 连接、配置和 catalog。
- `ToolRouter`：对模型声明且真正可执行的工具计划。
- `LoadedAgentsMd`：与环境快照对应的 instructions。
- permissions 相关的 live grants 访问入口。
- realtime 与 extension step data。

见 [`session/step_context.rs:24-50`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/step_context.rs#L24-L50)。

这避免三类撕裂：

```text
Prompt 宣告 tool A     但执行路由已变成 tool B
Prompt 宣告 cwd X      但 sandbox 按 cwd Y 判断
Prompt 注入旧 AGENTS   但工具按新项目环境执行
```

### 6.2 快照不代表所有状态永久冻结

`StepContext` 只冻结一次 sampling。tool result、steering、model switch 或环境变化触发下一 step 时，可以重新捕获新的快照。

`to_turn_context_item()` 还会把 step 捕获的 realtime 状态写回持久化的 turn context item。见 [`session/step_context.rs:76-88`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/step_context.rs#L76-L88)。

**项目建议**：把 prompt-builder 的入参设计成不可变 `RequestSnapshot`。不要让 builder 在处理中再次访问全局 mutable config。

---

## 7. WorldState：把“当前运行环境”变成可 diff 的协议

### 7.1 WorldState 不是一个大字符串

`WorldState` 用稳定 section ID 保存一组可独立比较和渲染的 section；`WorldStateSnapshot` 用 map 保存各 section 的紧凑状态。见 [`context/world_state/mod.rs:386-444`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context/world_state/mod.rs#L386-L444)。

当前构建路径会按配置加入多类 section：

| 类别 | 典型 section |
| --- | --- |
| 模型 | model instructions、base instructions、model catalog |
| 预算 | token budget、context-window guidance |
| 工作模式 | collaboration、persistent、multi-agent、realtime |
| 项目规则 | AGENTS.md、managed developer instructions |
| 执行环境 | environments、permissions、exec policy |
| 能力 | apps、plugins、deferred tools、top-level tools |
| 扩展 | extension-provided world-state sections |

完整注册顺序见 [`session/world_state.rs:59-410`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/world_state.rs#L59-L410)。

### 7.2 每个 section 自己决定如何 diff

`WorldStateSection` 不只返回当前文本，还接收三态 previous state：

- `Known`：有可反序列化的精确 snapshot。
- `Absent`：历史中确认没有该 section。
- `Unknown`：只发现 legacy fragment，无法恢复精确结构。

section 据此返回：

```text
(new snapshot, Vec<WorldStateUpdate>)
```

同一状态可返回零条 update；替换/移除可返回明确的模型可见通知。框架再把新 snapshot 保存为下一次 diff baseline。渲染协议见 [`context/world_state/mod.rs:157-273`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context/world_state/mod.rs#L157-L273)。

### 7.3 Placement 控制上下文布局

每条 `WorldStateUpdate` 附带 placement：

- `Prefix`：新 context window 的前缀 item。
- `Standalone`：必须独立成一个 item。
- `Mergeable`：可与相邻同 role fragments 合并。

定义与拆分逻辑见 [`context/world_state/mod.rs:65-153`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context/world_state/mod.rs#L65-L153)。

`merge_world_state_updates()` 只合并相邻、同 role、允许 merge 的 fragments；遇到显式 item、standalone 或 role 变化就 flush。见 [`context_manager/updates.rs:10-58`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context_manager/updates.rs#L10-L58)。

**架构判断**：顺序和边界也是 prompt 语义。Context 系统不能只维护 `Map<String, String>`；它还需要 role、placement、稳定 marker 和更新策略。

---

## 8. 首次完整注入与后续增量更新

### 8.1 没有 reference context 时注入 full context

`record_context_updates_and_set_reference_context_item()` 先读取上一次 `TurnContextItem`：

- 没有 reference：构建完整 initial context，建立 WorldState baseline。
- 已有 reference：相对 baseline 渲染 WorldState diff。
- turn context 发生变化：额外生成 turn-context contributions。
- 没有模型可见变化：不重复写入 context items。

主逻辑见 [`session/mod.rs:4734-4811`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L4734-L4811)。

### 8.2 Initial context 仍然保留角色和独立 item

`build_initial_context_with_world_state()` 不是把所有 section 拼成一条 developer message。它会：

1. 收集 turn developer instructions。
2. 加入 extension 的 thread/turn context。
3. 加入 token-budget context。
4. 调用 `world_state.render_full()`。
5. 先拆出 prefix updates。
6. 按 role 和特殊 marker 分组。
7. 将部分策略保留为 standalone developer items。
8. 将 AGENTS.md 等 user-role fragments 放入 contextual user message。
9. 给生成的 items 补 turn ID。

见 [`session/mod.rs:4319-4594`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L4319-L4594)。

### 8.3 Diff baseline 不只依赖内存 snapshot

`render_history_diff()` 优先使用持久 snapshot；若 snapshot 不足，还检查 history 是否保留了该 section 的 marker：

- snapshot 存在且 history 中仍有 retained fragment：`Known`。
- snapshot 存在但应保留的 fragment 已丢失：`Absent`，强制重新注入。
- 没 snapshot，但发现 legacy fragment：`Unknown`。
- 两者都没有：`Absent`。

见 [`context/world_state/mod.rs:485-531`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context/world_state/mod.rs#L485-L531)。

这解决了一个常见恢复问题：baseline 声称“模型已经看过”，但 history 已被替换、裁剪或来自旧格式。只信 snapshot 会导致关键规则静默缺失。

### 8.4 Rollout 保存的是 full/merge patch

`ContextManager::update_world_state()` 生成模型可见 fragments，同时将 snapshot 与 previous baseline 比较，持久化 full state 或 RFC 7386 风格 merge patch。见 [`context_manager/history.rs:451-494`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context_manager/history.rs#L451-L494)。

模型可见 diff 与恢复用 state patch 是两个输出：

```text
WorldState section
   ├─ model-visible update fragments
   └─ durable snapshot / merge patch
```

不要把“没有新 message”误判为“状态没有变化”。源码明确处理 snapshot-only change。

---

## 9. AGENTS.md 与 host instructions

### 9.1 指令来源不是只有仓库文件

`LoadedAgentsMd` 聚合三类输入：

- host 提供的 user-level instructions。
- host 提供的 thread-level instructions。
- 从一个或多个环境发现的 project instructions。

数据结构见 [`agents_md.rs:301-376`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agents_md.rs#L301-L376)。

`AgentsMdManager` 负责刷新 provider、按环境选择和 trust level 缓存结果，并把三类来源合成一个已验证 snapshot。thread instructions 有独立的 10,000 estimated-token 上限。见 [`agents_md_manager.rs:68-149`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agents_md_manager.rs#L68-L149) 与 [`agents_md_manager.rs:190-201`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agents_md_manager.rs#L190-L201)。

### 9.2 项目文件按 root → cwd 逐层收集

发现规则：

1. 从 cwd 向上寻找 project root marker，默认 `.git`。
2. 从 project root 向 cwd 逐层查找 `AGENTS.md` 或 fallback filename。
3. 每层优先本地 override。
4. 不越过 project root。
5. 受 `project_doc_max_bytes` 总预算约束。
6. untrusted project 不读取 project instructions，只保留 host user instructions。

入口和 trust/budget 逻辑见 [`agents_md.rs:1-95`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agents_md.rs#L1-L95)，路径发现见 [`agents_md.rs:192-274`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agents_md.rs#L192-L274)。

### 9.3 AGENTS.md 在协议中是 user-role context

`UserInstructions` 实现 `ContextualUserFragment`，role 为 `user`，并使用可识别 marker 包围内容。见 [`context/user_instructions.rs:1-35`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context/user_instructions.rs#L1-L35)。

当 instructions 改变时，`AgentsMdState` 不只发送新文本，还会声明“替换之前的 AGENTS.md”；当它消失时，发送 removal notice。见 [`context/world_state/agents_md.rs:9-82`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context/world_state/agents_md.rs#L9-L82)。

**架构判断**：context diff 不能只追加“新值”。对于规则类上下文，必须显式表达 replace/remove，否则模型可能同时保留新旧规则。

---

## 10. Skills、Plugins 与 turn-start 动态注入

### 10.1 可用能力说明与选中能力正文是两层

WorldState 可告诉模型“有哪些 skills/plugins/apps 或如何使用”；当本 turn 用户明确 mention 某个能力时，`build_skills_and_plugins()` 再注入被选中的详细内容。

该函数会：

1. 从用户输入解析 plugin、skill、app/MCP mentions。
2. 计算 plugin 所需 MCP servers。
3. 基于当前 `StepContext.mcp` 得到可访问 connectors。
4. 加载被选中 skill 的完整 prompt。
5. 构建 plugin instructions。
6. 收集 extension turn-input fragments。
7. 去重 host 已注入的 skill prompt。
8. 返回 injection items 与显式启用的 connector IDs。

见 [`session/turn.rs:932-1169`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L932-L1169)。

### 10.2 Skill 与 plugin 可以使用不同 role

被选中的 `SkillInstructions` 是带 `<skill>` marker 的 user-role fragment。见 [`ext/skills/src/fragments.rs:62-105`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/skills/src/fragments.rs#L62-L105)。

`PluginInstructions` 则是 developer-role fragment。见 [`context/plugin_instructions.rs:1-35`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context/plugin_instructions.rs#L1-L35)。

因此“动态注入”不等于“都伪装成用户消息”。每种 fragment 自己声明 role、marker、content kind 和正文。

### 10.3 `ContextualUserFragment` 是统一协议

该 trait 要求每个 fragment 提供：

- role。
- 稳定 content kind。
- start/end markers。
- body。

默认实现负责渲染并转成 `ResponseItem`。见 [`context-fragments/src/fragment.rs:56-113`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/context-fragments/src/fragment.rs#L56-L113)。

Marker 有两个用途：

- 让模型看到边界明确的上下文块。
- 让恢复/diff 逻辑识别 history 中是否仍存在某类注入。

**项目建议**：为注入内容保留 typed fragment，不要只返回 `String`。最少需要 `{kind, role, body, replacement_key, placement}`。

---

## 11. History 如何变成 sampling input

### 11.1 Sampling 每次取 history snapshot

首次尝试可直接使用调用方传入的 input snapshot；retry 或后续重建则调用：

```rust
sess.clone_history()
    .await
    .for_prompt(&step_context.settings.model_info.input_modalities)
```

见 [`session/turn.rs:1604-1655`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L1604-L1655)。

这一步绑定了两份快照：

- history snapshot：输入序列的一致视图。
- `StepContext`：模型能力、工具和环境的一致视图。

### 11.2 `for_prompt()` 不原样返回 history

`for_prompt()` 先调用 `normalize_history()`，再剥离内部 envelope metadata，返回 `Vec<ResponseItem>`。见 [`context_manager/history.rs:598-618`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context_manager/history.rs#L598-L618)。

标准化至少包含：

1. 确保 function/custom tool call 有对应 output。
2. 删除没有对应 call 的孤儿 output。
3. 当前模型不支持 image 时移除 image content。
4. 当前模型不支持 audio 时移除 audio content。

见 [`context_manager/history.rs:960-981`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context_manager/history.rs#L960-L981)。

### 11.3 持久 history 与请求副本必须分开

请求层还会规范 image detail：Responses Lite 清除 detail；不支持 `original` 的模型降级到默认 detail。这个处理发生在 `Prompt` 的克隆 input 上。见 [`client_common.rs:58-108`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client_common.rs#L58-L108)。

同样，非 OpenAI provider 会在请求副本中删除内部 metadata、部分 encrypted content 和 encrypted function args，不修改持久历史。见 [`client.rs:1001-1022`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L1001-L1022)。

**不变量**：持久层保留尽可能完整的事实；provider adapter 只裁剪本次请求副本。

---

## 12. Base instructions 与 incremental tools

### 12.1 Base instructions 有来源信息

`get_base_instructions()` 返回文本和 provenance。`get_prompt_base_instructions()` 只生成请求副本；在特定配置下，它会从 model-provided instructions 中移除 update-plan 指令，但不改变持久或 fork 继承的原值。见 [`session/mod.rs:1552-1579`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L1552-L1579)。

这体现两个边界：

- source value：用于 session/fork 的稳定配置。
- rendered request value：结合当前 feature/config 得出的本次请求版本。

### 12.2 普通 Responses 模式

普通模式中：

- tool specs 映射到 API 顶层 `tools`。
- base instructions 转成 developer message，插入 `input` 最前面。
- `parallel_tool_calls` 写入顶层字段。

### 12.3 Responses Lite / incremental tools 模式

当模型使用 Responses Lite 且 feature 启用时：

- 新 context window 将工具声明转成 `AdditionalTools` history item。
- base instructions 也作为 WorldState section/history item 管理。
- 后续工具变化可按 WorldState diff 增量表达。
- `Prompt.tools` 和 `Prompt.base_instructions` 置空，避免重复。
- API 顶层 `parallel_tool_calls` 被关闭。

模式选择见 [`session/step_context.rs:53-61`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/step_context.rs#L53-L61)；当前 window 是否已经采用 incremental tools 的判断见 [`session/world_state.rs:42-57`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/world_state.rs#L42-L57)。

**架构判断**：增量工具协议把“request header”提升成“conversation state”。收益是可 diff、可恢复；代价是 context manager 必须保证声明顺序、去重和 window baseline 正确。

---

## 13. 顺序、角色与优先级

### 13.1 不要只靠字符串拼接顺序理解优先级

Codex 的可观测顺序由三层共同决定：

1. API 层：base instructions/request prefix 被插入 input 最前。
2. context 层：developer/user role、standalone/mergeable/prefix placement。
3. history 层：context update、用户输入、skill/plugin injection、tool items 的记录顺序。

因此更准确的模型是：

```text
authority = role / API semantic
scope     = base / thread / turn / step
recency   = ordered history position
identity  = content kind / marker / item ID
```

只维护一个“system prompt 字符串”无法表达这四个维度。

### 13.2 特殊策略会刻意保持独立 item

initial context builder 对某些内容做特殊布局：

- model-switch instructions 前置。
- multi-agent role instructions 独立。
- guardian policy 独立。
- managed developer instructions 独立并后置到指定位置。
- user-role contextual fragments 单独形成 user message。

这不是格式美化，而是避免无关 fragment 合并后改变审计边界或策略覆盖关系。

### 13.3 Stable ID 服务于重试和缓存

base instructions 与 request-only tools prefix 使用 thread ID 派生的 UUID namespace，再对可见 payload 做 UUID v5。相同 thread、相同 payload 会得到稳定 ID。见 [`client.rs:966-1000`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/client.rs#L966-L1000)。

这有利于：

- retry 不制造逻辑上不同的 prefix item。
- resumed session 维持 identity。
- prompt cache 对稳定前缀更友好。
- tracing 能区分“内容变化”与“同内容重发”。

---

## 14. 四条典型 Prompt 路径

### 14.1 新 thread 的首个 turn

```text
capture StepContext
  -> reference_context_item == None
  -> WorldState.render_full()
  -> developer/user context items
  -> record user input
  -> record selected skill/plugin injections
  -> history.for_prompt()
  -> Prompt + request prefix
```

特点：建立完整 baseline；context 成本最高，但后续可增量更新。

### 14.2 环境与配置均未变化的后续 turn

```text
capture new StepContext
  -> compare TurnContextItem
  -> WorldState.render_history_diff()
  -> zero context updates
  -> record only new turn input/injections
  -> sample from accumulated history
```

特点：不重复发送“当前 cwd、权限、AGENTS.md 未变化”等说明。

### 14.3 AGENTS.md 或 permissions 改变

```text
capture new StepContext
  -> section snapshot differs
  -> section-specific replacement/removal fragment
  -> append context diff to history
  -> persist WorldState merge patch
  -> next prompt includes explicit transition
```

特点：模型看到的是状态转移，不是假装新值从一开始就存在。

### 14.4 Tool follow-up sampling

```text
model emits tool call
  -> call/output recorded in history
  -> next step captures current model/tool/environment snapshot
  -> history normalization checks call/output pairing
  -> full history becomes next Prompt.input
```

特点：工具结果不是临时局部变量，而是下一次 prompt 的正式 history item。

---

## 15. 对现有 agent-loop 项目的直接启示

以下是项目建议，不是 Codex 官方设计声明。

### 15.1 建立 typed prompt IR

建议最小结构：

```text
PromptIR {
  items: [PromptItem],
  tools: [ToolSpec],
  base_instructions: InstructionSet,
  output_contract: Optional<Schema>,
  model_options: ModelRequestOptions,
}
```

provider adapter 再映射为 OpenAI/Anthropic/本地模型格式。不要让业务 loop 直接构造供应商 JSON。

### 15.2 为每类 context 定义 section contract

建议：

```text
ContextSection {
  id,
  snapshot(),
  diff(previous),
  role,
  placement,
  retention_marker,
}
```

环境、权限、模型、tools、项目规则可以独立更新、测试与恢复。

### 15.3 保存“模型看过什么”的 baseline

不能只保存当前 config。需要同时保存：

- 当前真实状态。
- 上次向模型表达的 snapshot。
- history 中相应 fragment 是否仍存在。

否则 compaction/resume 后容易错误跳过关键 reinjection。

### 15.4 Context 更新必须支持 replace/remove

追加新 instructions 不等于撤销旧 instructions。每个 section 应定义：

- first appearance。
- unchanged。
- changed/replaced。
- removed。
- previous state unknown。

### 15.5 Request snapshot 必须与 tool runtime 共用

模型看到的 tool specs 与实际执行 router 必须来自同一版本。建议把 `tool_registry_version`、`permission_version`、`environment_version` 写入 step trace。

### 15.6 持久历史与 provider view 分离

```text
DurableHistory
   -> normalize
   -> capability filter
   -> provider scrub
   -> RequestInput
```

降级图片、清理 provider metadata 或过滤 unsupported items 时，只改请求副本。

### 15.7 记录 prompt provenance

调试 prompt 时，仅输出最终文本不够。建议每个 item 保留：

- source：model/config/AGENTS/skill/plugin/tool/user。
- scope：session/turn/step。
- role 与 placement。
- section ID / content kind。
- snapshot/version/hash。
- persisted 与 request-only 标志。

这样才能回答“这句话为什么进入 prompt”“为什么这轮消失”“是否因 compaction 被重注入”。

### 15.8 先优化稳定前缀，再优化 token 数

稳定的 item 顺序、ID 和前缀内容有助于缓存；频繁把时间戳、随机 ID 或无关动态字段塞入前缀，会破坏 cache locality。动态 turn 内容应尽量后置。

---

## 16. 容易误读的地方

### 16.1 “Prompt 就是 system prompt + user message”

不对。它还包含结构化 history、tools、tool outputs、output schema、reasoning/model options 和多 role context items。

### 16.2 “AGENTS.md 是 system/developer message”

不对。当前实现把 `UserInstructions` 渲染为 user-role contextual fragment；其约束语义还来自应用协议和位置，不能只用 role 名称推断。

### 16.3 “每次 sampling 都完整重发环境说明”

不对。首次建立 full baseline；稳定运行时通常只追加 section diff。

### 16.4 “History 保存什么就原样发什么”

不对。`for_prompt()` 会修复 call/output 关系并按模型模态过滤；provider adapter 还会清理请求副本。

### 16.5 “Skills 在启动时全部展开”

不对。可用技能清单与被选中技能正文是两层；显式 mention 后才加载选中 skill 的完整 instructions。

### 16.6 “Tools 永远位于 API 顶层字段”

不对。Responses Lite 的 incremental tools 会把声明作为 history/input item 管理。

### 16.7 “Snapshot 没变就一定什么都不用做”

不对。若 history 中应保留的 fragment 已丢失，WorldState 会把 previous 视为 absent 并重新注入。

### 16.8 “Context diff 只影响模型，不影响恢复”

不对。模型可见 fragments 与 rollout 中的 WorldState full/patch 同时生成，分别服务当前请求和持久恢复。

---

## 17. 本章小结

Codex 的 Prompt/Context 链路可压缩为：

```text
TurnContext 定义工作边界
  -> StepContext 冻结一次采样所需能力
  -> WorldState 将实时配置建模为 typed sections
  -> full/diff context 进入 history 并建立 durable baseline
  -> 用户输入与 skills/plugins injections 进入同一有序 history
  -> ContextManager 生成标准化、按模态过滤的请求副本
  -> build_prompt 组合 input/tools/base/output schema
  -> provider client 映射为最终 ResponsesApiRequest
```

其核心工程思想是：

- Prompt 使用结构化 IR，不提前扁平化。
- Context 是有 snapshot、diff、replace/remove 语义的状态协议。
- Step snapshot 同时约束模型可见能力和实际执行能力。
- Durable history 与 provider request view 分离。
- 动态注入保留 role、placement、marker、content kind 与 provenance。
- 首次 full、后续 diff，降低重复上下文并保留恢复正确性。

第四章将从 `Prompt` 已经构造完成的位置继续，分析 Responses API 请求、stream event、retry、fallback、preemption 与错误边界。

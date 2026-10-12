# Codex Agent Loop 源码分析

本项目面向正在设计或维护本地 coding agent 的工程团队，静态拆解 OpenAI Codex 的 agent runtime。重点不是复述产品功能，而是回答：输入如何进入运行时、一个 turn 如何推进、模型与工具如何闭环、状态如何持久化、多 agent 如何建立控制关系。

## 分析基线

| 项目 | 值 |
| --- | --- |
| 上游仓库 | [openai/codex](https://github.com/openai/codex) |
| 本地源码 | `S:\Projects\codex` |
| 分支 | `main` |
| 固定提交 | [`c3d3b142d10f4316b46e35aad7e5317e7e506cb7`](https://github.com/openai/codex/tree/c3d3b142d10f4316b46e35aad7e5317e7e506cb7) |
| 分析方式 | 源码静态阅读 + 调用链交叉验证 |

固定提交很重要。Codex 主分支变化快；所有源码链接锁定到同一 commit，避免文件和行号漂移。

## 当前范围

第一阶段聚焦 agent loop 核心，共 9 章：

1. 架构总览、入口、Thread/Session/Task/Turn 生命周期。
2. Agent loop 主状态机。
3. Prompt 与上下文组装。
4. 模型请求、流式事件、重试与抢占。
5. Tool registry、router、orchestrator 与结果回流。
6. Approval、exec policy 与 sandbox。
7. History、token budget、compact、rollout 与 resume。
8. Multi-agent spawn、mailbox、wait、interrupt 与 handoff。
9. 对现有 agent-loop 项目的设计启示。

本阶段不做完整 UI 组件索引、所有 crate 的逐函数说明、产品功能清单或竞争分析。

## 一图总览

```text
安装/启动层
  npm shim / standalone binary
            |
            v
+------------------------------+
| codex-cli                    |
| 参数解析与运行形态分流         |
+------------------------------+
     |              |               |
     v              v               v
  TUI 交互       codex exec      app-server
     |                              ^
     | AppServerClient              |
     +---- embedded / daemon / remote
                    |
                    v
+----------------------------------------------+
| App Server / JSON-RPC 控制面                 |
| thread/start, turn/start, notifications      |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| codex-core                                   |
| ThreadManager -> CodexThread -> Session      |
|                       -> SessionTask         |
|                       -> TurnContext         |
|                       -> StepContext         |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Agent loop                                   |
| RegularTask -> run_turn -> sampling          |
| -> tool calls/results -> follow-up sampling  |
+----------------------------------------------+
        |                |                 |
        v                v                 v
 Model provider     Tool runtime       History/Rollout
 Responses API      Approval/Sandbox   Thread Store
```

核心判断：Codex 已不是“TUI 内嵌一个循环”的简单 CLI。TUI、daemon、远端客户端与其他宿主通过 App Server 或 Core API 进入同一套 thread/session runtime；真正的 agent loop 位于 `codex-rs/core`。

## 章节目录

### 已完成

- [第一章：软件架构、程序入口与运行时边界](./analysis/01-architecture-overview.md)
- [第二章：Agent Loop 主状态机](./analysis/02-agent-loop.md)
- [第三章：Prompt 与 Context 管理](./analysis/03-prompt-context.md)
- [第四章：模型请求、流式事件、重试与抢占](./analysis/04-model-request-streaming.md)
- [第五章：Tool Runtime——注册、路由、并发执行与结果回流](./analysis/05-tool-runtime.md)

### 计划章节

- 第六章：Approval 与 Sandbox
- 第七章：History、Compaction 与 Resume
- 第八章：Multi-Agent
- 第九章：对 agent-loop 项目的启示
- 附录：代码证据索引

## 阅读建议

- 想先理解系统边界：读第一章第 3、6、8 节。
- 想直接掌握主循环：读第二章第 4、7、10 节。
- 想理解 Prompt 如何形成：读第三章第 4、7、8、11 节。
- 想理解模型请求与流式恢复：读第四章第 6、9、11、13、16 节。
- 想理解工具注册、并发与结果回流：读第五章第 4、8、9、10、15、17 节。
- 想定位主循环源码：从 `RegularTask::run()`、`run_turn()`、`run_sampling_request()` 与 `try_run_sampling_request()` 开始。
- 想复用架构：关注 `ThreadManager`、`SessionIo`、`SessionTask`、`TurnContext`、`StepContext` 的职责切分。
- 想复核结论：点击每节固定 commit 源码链接。

## 证据规则

文档采用三种陈述：

- **源码事实**：可由具体类型、函数、字段或调用直接验证。
- **架构判断**：由多处源码关系归纳，明确标注为判断。
- **项目建议**：面向现有 agent-loop 项目的迁移或设计建议，不声称是 Codex 官方意图。

伪代码只用于压缩控制流。凡与真实类型或函数相关的结论，优先提供源码链接。

## 声明

本项目仅用于工程研究和技术学习。Codex 源码及商标权利归 OpenAI 及其贡献者所有；上游代码遵循其 Apache-2.0 License。本项目不复制上游源码，只提供独立分析。


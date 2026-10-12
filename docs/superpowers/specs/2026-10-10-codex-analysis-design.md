# Codex Agent Loop 核心分析设计

## 目标

建立 `codex-analysis` 静态分析文档集。第一阶段聚焦 Codex 的 agent loop，帮助现有 `agent-loop` 项目理解其运行时边界、状态机、工具执行、安全控制、上下文管理和多 agent 设计。

## 分析基线

- 源码目录：`S:\Projects\codex`
- 上游仓库：`https://github.com/openai/codex.git`
- 基线分支：`main`
- 基线提交：`c3d3b142d10f4316b46e35aad7e5317e7e506cb7`
- 分析方法：源码静态阅读；结论必须附源码路径、符号名或行号证据。

## 目录设计

```text
codex-analysis/
├── README.md
├── analysis/
│   ├── 01-architecture-overview.md
│   ├── 02-agent-loop.md
│   ├── 03-prompt-context.md
│   ├── 04-model-streaming.md
│   ├── 05-tool-runtime.md
│   ├── 06-approval-sandbox.md
│   ├── 07-history-compaction.md
│   ├── 08-multi-agent.md
│   ├── 09-agent-loop-lessons.md
│   └── 10-code-evidence-index.md
└── docs/superpowers/
    ├── specs/
    └── plans/
```

## 第一章范围

`analysis/01-architecture-overview.md` 回答：

1. Codex 仓库由哪些主要层和 crate 构成。
2. CLI、TUI、Exec、App Server 如何进入共享 Core。
3. `ThreadManager`、`CodexThread`、`Session`、`SessionTask`、`TurnContext` 各自负责什么。
4. 用户输入如何抵达 `RegularTask::run()` 和 `run_turn()`。
5. UI、协议、业务内核、执行环境、持久化之间的边界是什么。
6. 这些边界对现有 agent-loop 项目有什么直接价值。

本章只解释架构与入口。模型流式事件、工具运行时、压缩、多 agent 的内部算法留给后续专章。

## 写作规则

- 中文写作，结论先行。
- 风格参考 `S:\Projects\claude-code-analysis`：章节导读、ASCII 架构图、主链路、源码证据、设计评价、章末小结。
- 区分“源码事实”“基于源码的推断”“面向 agent-loop 的建议”。
- 不复制 Codex 源码树；使用相邻目录 `..\codex` 作为证据源。
- 文档内 GitHub 链接固定到基线提交，保证行号和内容不会随 `main` 漂移。
- 不修改 `S:\Projects\codex`。
- 本地初始化独立 Git 仓库，但本阶段不提交、不发布。

## 第一章验收标准

- README 能说明项目目的、基线、目录和第一章入口。
- 第一章包含至少一张总体分层图和一条端到端调用链。
- 每个关键架构结论至少有一个源码证据。
- 所有本地相对链接和固定提交 GitHub 链接格式正确。
- 不把尚未深挖的后续机制写成确定结论。


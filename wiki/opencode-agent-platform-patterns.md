---
title: OpenCode Agent Platform Patterns
created: 2026-06-16
tags:
  - ai/opencode
  - ai/agent-runtime
  - architecture
status: draft
---

# OpenCode Agent Platform Patterns

Claude Code 分析资料里最值得 OpenCode 借鉴的，是“本地 agent 平台”的架构思路，而不是某个具体命令。

## 一层入口，多形态接入

Claude Code 的入口形态很多：CLI、REPL、headless/SDK、MCP、remote、subagent。但它们最终尽量回到统一的 query/tool/permission 主循环。

对 OpenCode 的启发：

- OpenCode UI agent、命令行 smoke test、未来 MCP adapter 不应该各自实现业务逻辑。
- 应保留一个共享 core，让不同入口只是 adapter。
- adapter 负责输入输出形态，core 负责执行策略、schema、权限、日志、报告。

对 erun-emu 的落点：

```text
OpenCode agent prompt / tool adapter
        |
        v
erun-emu-tools
        |
        v
erun-emu-core
  - script_registry
  - path normalization
  - permission/risk policy
  - journal
  - evidence report
        |
        v
evision / erun_cli / internal scripts
```

## Query Loop 是能力中枢

分析资料反复强调，模型输出的 `tool_use` 不是直接调函数，而是进入执行管线：

```text
model output
  -> parse tool_use
  -> schema validation
  -> semantic validation
  -> hook / permission
  -> execute tool
  -> normalize tool_result
  -> append to conversation
  -> next model round
```

这对 OpenCode agent 的意义：

- 不要把“模型选择了工具”视为可信执行许可。
- 工具结果必须变成下一轮模型能理解的结构化结果。
- 错误也要结构化返回，让模型可以修正参数或向用户确认。

## Agent 不是 prompt 文件

Claude Code 的 agent definition 不只是 system prompt，还包括：

- tools
- permission mode
- model
- memory scope
- hooks
- skills
- UI 展示信息

OpenCode 的 erun-emu agent 也应避免变成“一个很长的说明文档”。更合理的拆法是：

- `agent meta`：身份、职责、默认模型、允许工具。
- `skill`：人类可读的领域流程和使用约定。
- `core registry`：机器可读的可执行动作、参数 schema、风险等级。
- `journal/report`：运行时证据和可恢复状态。

## Local-First 但保留扩展面

Claude Code 的设计是 local-first，但保留 remote、bridge、MCP、swarm 扩展面。这个思路适合 erun-emu：

- demo 先本地运行，减少部署复杂度。
- core 不绑定 OpenCode UI，这样后续可包装成 MCP server。
- 内部脚本、eVision、工程路径都以本地资源为准，不做远端假设。

## OpenCode 通用 agent 设计原则

- 一个入口只做入口的事。
- 所有执行都进入统一 tool runtime。
- 所有外部动作都有结构化 input/output。
- 所有状态变化进入 append-only journal。
- 所有高风险动作都能被 permission 控制。
- 所有大日志都走 evidence path，而不是直接塞进上下文。

## 关联

- [[erun-emu-agent-application]]
- [[tool-permission-runtime-patterns]]
- [[memory-context-session-patterns]]

## 来源

- [架构概览](../analysis/01-architecture-overview.md)
- [程序架构及亮点](../analysis/05-differentiators-and-comparison.md)
- [总结结论](../analysis/09-final-summary.md)


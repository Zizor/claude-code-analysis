---
title: OpenCode Agent Platform Patterns
created: 2026-06-16
tags:
  - ai/opencode
  - ai/agent-runtime
  - architecture
  - claude-code
status: draft
aliases:
  - Agent 平台模式
---

# OpenCode Agent Platform Patterns

Claude Code 分析资料里最值得 OpenCode 借鉴的，是“本地 agent 平台”的架构思路，而不是某个具体命令。主要参考：

- [架构概览](../analysis/01-architecture-overview.md)
- [Tool Call 机制](../analysis/04b-tool-call-implementation.md)
- [Skills 机制](../analysis/04c-skills-implementation.md)
- [MCP 机制](../analysis/04d-mcp-implementation.md)
- [Context 管理](../analysis/04f-context-management.md)
- [Multi-Agent 机制](../analysis/04h-multi-agent.md)
- [Session Storage / Resume](../analysis/04i-session-storage-resume.md)

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

模型输出的 `tool_use` 不是直接调函数，而是进入执行管线：

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

## Agent 不是 Prompt 文件

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
- `skill`：模型可读的领域流程和使用约定。
- `core registry`：机器可读的可执行动作、参数 schema、风险等级。
- `journal/report`：运行时证据和可恢复状态。

## Tool 是协议对象

成熟 coding agent 不是“LLM (Large Language Model) + 一堆 CLI”的简单包装，而是一个小型 runtime：

```text
用户自然语言
  -> agent prompt / skill / project instruction
  -> planner / policy / tool selection
  -> tool permission runtime
  -> executor / registry / adapter
  -> structured result / artifact / session journal
  -> 下一轮模型上下文
```

迁移到 OpenCode 时，`erun-emu` 不应只暴露“运行脚本”的 shell 能力，而应包装成有明确 schema 的工具，例如：

- `prepare_project`
- `run_board_script`
- `query_runtime_status`
- `run_debug_action`
- `collect_evidence_report`

每个工具都需要声明风险级别、是否只读、是否需要用户确认、输出结构和证据路径。

## Local-First 但保留扩展面

Claude Code 的设计是 local-first，但保留 remote、bridge、MCP、swarm 扩展面。这个思路适合 erun-emu：

- demo 先本地运行，减少部署复杂度。
- core 不绑定 OpenCode UI，这样后续可包装成 MCP server。
- 内部脚本、eVision、工程路径都以本地资源为准，不做远端假设。

## Skill 与 Registry 分工

Skill 是模型可读的操作手册；registry 是机器可执行的白名单。二者都重要，但职责不同：

| 层 | 作用 | 是否作为安全边界 |
|----|------|------------------|
| Skill | 告诉 agent 业务流程、日志位置、错误格式、排障思路 | 否 |
| Agent prompt | 定义角色、交互风格、默认策略、确认规则 | 否 |
| `erun-emu-core` registry | 限定可执行脚本、参数、风险等级 | 是 |
| executor | 执行、加锁、超时、解析、写 journal | 是 |

运行时不要解析 `SKILL.md` 来决定能不能执行，因为 Markdown 是面向模型的上下文，不适合作为稳定、可测、可审计的机器契约。

## OpenCode 通用 Agent 设计原则

- 一个入口只做入口的事。
- 所有执行都进入统一 tool runtime。
- 所有外部动作都有结构化 input/output。
- 所有状态变化进入 append-only journal。
- 所有高风险动作都能被 permission 控制。
- 所有大日志都走 evidence path，而不是直接塞进上下文。
- 多 agent 先当控制面，不当硬件执行面。

## 设计红线

- 不让模型直接拼 shell 跑任意 eVision/erun 命令。
- 不把用户未确认的高风险动作自动执行。
- 不让多个 agent 同时操作同一个板卡/工程资源。
- 不把长日志、波形、报告全文塞进 prompt。
- 不把项目路径和 eVision 路径做长期全局持久化；它们属于当前 session/任务输入。

## 关联

- [[erun-emu-agent-application]]
- [[tool-permission-runtime-patterns]]
- [[memory-context-session-patterns]]
- [[prompt-skill-mcp-patterns]]
- [[multi-agent-control-plane-patterns]]

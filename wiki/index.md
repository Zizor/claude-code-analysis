---
title: Claude Code Analysis Wiki For OpenCode Agent
created: 2026-06-16
tags:
  - ai/opencode
  - ai/agent
  - claude-code-analysis
  - erun-emu
status: draft
source_root: /home/zizorw/Workspace/ObKnowleages/AI/claude-code-analysis
aliases:
  - Claude Code Agent Wiki
---

# Claude Code Analysis Wiki For OpenCode Agent

本目录从 `claude-code-analysis` 的分析资料中提炼对 OpenCode 通用 agent 和 erun/eVision emu agent 有用的工程经验。这里不复述 Claude Code 全量实现，而是保留对后续设计、实现、评审最有价值的模式。

> [!note]
> 来源材料是别人用 AI 对 Claude Code 源码的解析，因此这里把它作为设计参考和模式库，不把其中任何源码细节当作需要照搬的规范。外部权威资料用于校准设计方向，具体执行边界仍以本地 `erun-emu-core`、`script_registry`、OpenCode agent 配置和公司内部脚本契约为准。

## 地图

- [[authoritative-agent-source-map]]：官方/权威 agent 开发资料来源表和来源分级。
- [[authoritative-agent-development-guide]]：从官方资料和本地分析提炼出的 agent 开发攻略。
- [[opencode-agent-platform-patterns]]：OpenCode 通用 agent 的平台化设计模式。
- [[erun-emu-agent-application]]：这些模式落到 erun/eVision emu agent 后该怎么取舍。
- [[tool-permission-runtime-patterns]]：工具执行、权限、并发、安全边界。
- [[memory-context-session-patterns]]：上下文、日志、session、memory 与证据链。
- [[prompt-skill-mcp-patterns]]：prompt runtime、skill、Model Context Protocol (MCP) 的边界。
- [[multi-agent-control-plane-patterns]]：多 agent、任务面板、控制面和后续扩展。

## 总结结论

Claude Code 的可借鉴点不是“有很多 CLI 接口”，而是把本地 agent 做成了一个 runtime：

- 入口可以很多，但执行内核尽量统一。
- Tool 不是函数映射，而是带 schema、权限、并发属性、进度、结果映射的协议对象。
- 权限是控制面，不是弹窗小补丁。
- 上下文不是无限塞日志，而是 session journal、summary、evidence path、按需读取的组合。
- Skill 适合承载模型可读的领域流程，机器执行边界应落在 typed tool、registry 和 executor。
- 多 agent 能力要分级引入，真实硬件执行权限必须集中收口。

## 推荐用于 erun-emu demo 的最小闭环

1. 保持单一执行内核：OpenCode agent、未来 MCP、测试 CLI 都调用同一套 `erun-emu-core`。
2. 保持机器边界：`script_registry` 决定可执行动作、参数 schema、风险等级，不解析 `SKILL.md` 来决定执行。
3. 保持安全默认值：所有上板、运行、修改状态动作串行执行，读状态命令也只有 registry 显式标记后才允许并发。
4. 保持证据链：每次动作写 append-only JSONL journal，报告引用日志路径和摘要，不把全量日志塞进模型。
5. 保持用户控制：高风险动作走 OpenCode permission ask；demo 可简化策略，但简化项要记录为待扩展。

## 阅读顺序

1. 先看 [[authoritative-agent-development-guide]]，建立“agent 何时值得做、如何约束”的大框架。
2. 再看 [[tool-permission-runtime-patterns]] 和 [[memory-context-session-patterns]]，理解 runtime 的硬边界。
3. 然后看 [[prompt-skill-mcp-patterns]]，确认 prompt、skill、registry、MCP 的职责分工。
4. 最后看 [[erun-emu-agent-application]]，把这些模式落到 eVision/erun 业务场景。

## 主要来源

- [架构概览](../analysis/01-architecture-overview.md)
- [安全分析](../analysis/02-security-analysis.md)
- [Agent Memory](../analysis/04-agent-memory.md)
- [Tool Call](../analysis/04b-tool-call-implementation.md)
- [Skills](../analysis/04c-skills-implementation.md)
- [MCP](../analysis/04d-mcp-implementation.md)
- [Sandbox](../analysis/04e-sandbox-implementation.md)
- [Context](../analysis/04f-context-management.md)
- [Prompt](../analysis/04g-prompt-management.md)
- [Multi-Agent](../analysis/04h-multi-agent.md)
- [Session Storage / Resume](../analysis/04i-session-storage-resume.md)
- [平台能力组件](../analysis/components/03-platform-components.md)

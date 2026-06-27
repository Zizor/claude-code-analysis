---
title: Authoritative Agent Development Source Map
created: 2026-06-17
tags:
  - ai/agent
  - ai/opencode
  - source-map
  - erun-emu
status: draft
aliases:
  - Agent 权威来源地图
---

# Authoritative Agent Development Source Map

本页记录网上检索到的官方/权威 agent 开发资料，并标注它们对 OpenCode 通用 agent 与 erun/eVision emu agent 的价值。

> [!note]
> 外部资料只作为设计参考。具体到 erun/eVision 的可执行边界，仍以本地 `erun-emu-core`、`script_registry`、OpenCode agent 配置和公司内部脚本契约为准。

## 来源清单

| 来源 | 权威性 | 重点内容 | 对我们的价值 |
|---|---|---|---|
| [OpenAI Agents SDK](https://developers.openai.com/api/docs/guides/agents) | OpenAI 官方文档 | agent orchestration、tool execution、approvals、state、tracing | 支持“应用拥有执行内核和状态”的设计判断 |
| [OpenAI Agents SDK Python Docs](https://openai.github.io/openai-agents-python/) | OpenAI 官方 SDK 文档 | agent 定义、Runner、tools、handoffs、guardrails、results、sessions、MCP | 可对照 TypeScript core 的模块边界 |
| [OpenAI Practical Guide to Building Agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/) | OpenAI 官方实践指南 | 识别适合 agent 的场景、设计 agent logic、让 agent 安全可预测 | 可用于向业务侧解释“agent 能提供什么能力” |
| [Anthropic Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) | Anthropic 官方工程文章 | prompt chaining、routing、parallelization、orchestrator-workers、evaluator-optimizer、agents | 支持从简单 workflow 开始，而不是过早复杂化 |
| [Anthropic Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | Anthropic 官方工程文章 | context engineering、上下文有限性、工具/历史/外部数据治理 | 支持“日志作为 evidence，不直接塞 prompt” |
| [Model Context Protocol Intro](https://modelcontextprotocol.io/) | MCP 官方文档 | host/client/server、tools/resources/prompts/workflows | 支持未来 MCP 作为 adapter，而不是替代 core |
| [MCP Specification 2025-06-18](https://modelcontextprotocol.io/specification/2025-06-18) | MCP 官方规范 | JSON-RPC、协议角色、规范性要求、安全原则、进度、取消、日志 | 可作为未来 erun MCP server 的协议依据 |
| [OpenCode Agents](https://opencode.ai/docs/agents/) | OpenCode 官方文档 | primary/subagent、model、prompt、permissions、task permission | 直接指导 erun-emu agent 配置 |
| [OpenCode Skills](https://opencode.ai/docs/skills/) | OpenCode 官方文档 | `SKILL.md` 发现、frontmatter、permission、on-demand loading | 直接指导 erun runtime skill 的边界 |
| [Google ADK Artifacts](https://adk.dev/artifacts/) | Google ADK 官方文档 | session/user scoped artifacts、versioned binary data | 支持日志、波形、报告作为 artifact/evidence |
| [Google Cloud ADK State and Memory](https://cloud.google.com/blog/topics/developers-practitioners/remember-this-agent-state-and-memory-with-adk) | Google Cloud 官方博客 | short-term memory、long-term memory、session state | 支持 session fact 与 long-term memory 分离 |
| [ADK Integrations](https://adk.dev/integrations/) | Google ADK 官方目录 | observability、resilience、secure execution、memory、secret manager | 说明生产 agent 基础设施不止 prompt/tool |
| [AGENTS.md](https://agents.md/) | Linux Foundation / Agentic AI Foundation stewarded open format | repo-level agent instruction convention | 可作为跨 coding agent 项目说明格式参考 |

## 来源分级

### 一线实现依据

这些资料可以直接影响当前 erun-emu demo：

- [OpenCode Agents](https://opencode.ai/docs/agents/)
- [OpenCode Skills](https://opencode.ai/docs/skills/)
- [OpenAI Agents SDK](https://developers.openai.com/api/docs/guides/agents)
- [Anthropic Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)

### 架构扩展依据

这些资料适合用于后续 MCP、长会话、复杂任务编排：

- [MCP Specification](https://modelcontextprotocol.io/specification/2025-06-18)
- [OpenAI Agents SDK Results](https://openai.github.io/openai-agents-python/)
- [OpenAI Agents SDK Human-in-the-loop](https://openai.github.io/openai-agents-python/)
- [OpenAI Agents SDK Tracing](https://openai.github.io/openai-agents-python/)

### 运行治理依据

这些资料适合用于日志、证据、记忆、上下文治理：

- [Anthropic Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Google ADK Artifacts](https://adk.dev/artifacts/)
- [Google Cloud ADK State and Memory](https://cloud.google.com/blog/topics/developers-practitioners/remember-this-agent-state-and-memory-with-adk)
- [AGENTS.md](https://agents.md/)

## 可直接转入 erun-emu 的结论

- OpenCode agent 要以 permission 为中心，不要继续使用已 deprecated 的 `tools` 字段作为主要能力控制方式。
- Skill 是 on-demand reusable instruction，不是机器执行白名单。
- Human-in-the-loop approval 要作为高风险脚本执行的正式中断/恢复点。
- Tool result、guardrail、handoff、approval、trace 都应在本地 journal 或 report 中可追溯。
- Context engineering 的重点是筛选当前回合需要的信息，而不是最大化塞入日志。
- MCP 是跨客户端互通协议，适合未来扩展；demo 阶段不应为了 MCP 重写 core。
- 低能力模型场景下，要用窄 schema、结构化结果、候选 `next_actions` 和固定报告模板降低推理负担。

## 综合映射

| 主题 | 主要来源 | erun-emu 落地 |
|------|----------|---------------|
| 是否需要 agent | OpenAI Practical Guide, Anthropic Building Effective Agents | 上板调试有多步观察/选择/验证，适合 agent；脚本执行本身仍保持 deterministic |
| Tool 设计 | OpenAI Agents SDK, Claude Code 分析 | 结构化 schema、risk、permission、result、next_actions |
| Human-in-the-loop | OpenAI Agents SDK, MCP Spec, OpenCode Agents | 高风险上板/修改 runtime 状态必须 permission ask |
| Context engineering | Anthropic Context Engineering, Claude Code context 分析 | 日志做 artifact，prompt 只放摘要/路径/状态 |
| Session/Memory | Google ADK, Claude Code session/memory 分析 | session journal 保存当前任务；不把路径长期全局化 |
| MCP | MCP Spec, OpenCode Skills/Agents | 后续用 MCP adapter 包 `erun-emu-core`，不把 MCP 当业务核心 |
| Project instruction | AGENTS.md, OpenCode Agents | agent prompt + skill + project docs 分层 |

## 关联

- [[authoritative-agent-development-guide]]
- [[opencode-agent-platform-patterns]]
- [[erun-emu-agent-application]]
- [[tool-permission-runtime-patterns]]
- [[memory-context-session-patterns]]

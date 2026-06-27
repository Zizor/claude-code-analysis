---
title: Prompt Skill MCP Patterns
created: 2026-06-16
tags:
  - ai/agent
  - prompt-runtime
  - skill
  - mcp
status: draft
aliases:
  - Prompt Skill MCP 分工
---

# Prompt Skill MCP Patterns

Claude Code 的 prompt、skill、MCP 设计说明：agent 行为不是靠一个巨型 prompt 维持，而是靠多层上下文和多个扩展机制分工。

参考：

- [Prompt 管理机制](../analysis/04g-prompt-management.md)
- [Skills 技能机制](../analysis/04c-skills-implementation.md)
- [MCP 技术实现](../analysis/04d-mcp-implementation.md)
- [[authoritative-agent-development-guide]]

## Prompt 是 Runtime

Claude Code 的 prompt 被拆成：

- 默认静态 system prompt。
- agent prompt。
- append prompt。
- user context。
- system context。
- task-specific prompts。
- prompt section cache 和动态边界。

OpenCode 里不一定有完全相同机制，但设计思想可复用：

- 稳定规则写进 agent meta / agent prompt。
- 领域流程写进 skill。
- 当前项目/eVision/run 状态由 tool result 动态返回。
- 大日志不进 prompt，只进 evidence。
- 专项任务用短 prompt，例如“只根据这段日志生成报告摘要”。

## 四者分工

| 机制 | 面向对象 | 适合放什么 | 不适合放什么 |
|------|----------|------------|--------------|
| Agent prompt/meta | 当前 agent 的角色、默认行为、工具权限 | 交互风格、确认规则、优先级、模型选择 | 大段业务手册、可执行白名单 |
| Skill | 模型读取的领域操作手册 | 上板流程、错误字符串解释、日志位置、排障套路 | 运行时安全边界 |
| MCP / tool adapter | 外部能力接入协议 | 标准化工具、资源、prompt、进度、取消、日志 | 让模型自由拼命令 |
| Core registry | 机器可读契约 | `script_id`、参数 schema、风险等级、超时、权限 | 自然语言解释 |

## Skill 的合适边界

Skill 适合放：

- erun/eVision 术语解释。
- 用户交互方式。
- 上板脚本的业务约定。
- 错误字符串格式说明。
- 日志位置规则。
- debug 流程建议。
- 报告格式要求。

Skill 不适合放：

- 真正允许执行的 `script_id` 白名单。
- 机器校验用参数 schema。
- 安全风险等级的唯一来源。
- 文件锁和并发规则的唯一来源。
- 任意 shell 片段的运行时解析。

原因不是“skill 只给人看”。Skill 当然是给 agent 读的，但它的媒介是 Markdown，适合表达知识和流程，不适合作为可测试、可版本兼容、可审计的机器执行边界。

## 为什么运行时不要解析 SKILL.md

正确关系：

```text
SKILL.md
  -> 告诉模型“应该怎样使用能力”

script_registry/scripts.v1.json
  -> 告诉程序“哪些能力真的允许执行”

executor
  -> 只相信 registry 和运行时参数校验
```

具体原因：

- Markdown 容易被人为编辑，结构不稳定。
- 自然语言存在歧义，难以做严格 schema 校验。
- 安全策略需要测试、版本化、diff、回滚和审计。
- 模型可能漏读/误读 skill，但 registry 校验不会。
- 未来接 MCP 时，需要同一套机器契约复用，不应依赖 OpenCode/Claude 风格的 skill 文本。

## Core Registry 的合适边界

Registry 适合放：

- `script_id`
- 参数 schema
- 风险等级
- 是否只读
- 是否可并发
- 执行 adapter
- 输出 parser
- 错误分类器
- 版本号

这样可以做到：

- OpenCode tool 和未来 MCP server 复用同一事实源。
- CI 可测试 registry。
- agent 即使误读 skill，也不能突破 registry。
- report 可以引用 registry 元数据说明风险。

## erun-emu Skill 应包含什么

当前 `erun-opencode-runtime-skill` 适合保存：

- 用户必须提供 eVision 可执行文件路径和工程路径。
- 执行前必须归一化到 `realpath`。
- 不支持 eVision 名称搜索。
- 上板由内部一键脚本完成，demo 不支持 dry-run。
- erun CLI RPC 是短链接，每条命令传绝对 `--project`。
- 错误通过固定格式字符串解析。
- 日志位置由工程/脚本约定给出。
- 高风险动作需要用户确认。
- 输出报告要覆盖复现、观察、缩小范围、验证、证据。

## MCP 作为未来 Adapter

Claude Code 把 MCP 工具规范化为普通 Tool，同时处理：

- 名称命名空间。
- 描述长度截断。
- 连接并发限制。
- 认证失败缓存。
- IDE 工具白名单。
- 内建工具优先，防止外部工具 shadow。

对 erun-emu 的建议：

- demo 阶段先用 OpenCode local tool。
- core API 设计成未来可包装为 MCP。
- MCP server 只暴露 registry 里允许的动作。
- MCP tool description 保持短，把详细文档放 skill/reference。
- MCP 返回内容仍按外部不可信数据处理。

## 推荐文件分工

```text
opencode/agents/erun-emu.md
  - agent 身份、职责、默认行为、工具使用原则

skills/erun-opencode-runtime-skill/SKILL.md
  - 人类/模型可读的 erun/eVision 业务流程和领域约定

script_registry/scripts.v1.json
  - 机器可读动作白名单、schema、风险

src/erun-emu-core/*
  - 执行、校验、日志、报告、锁

tests/*
  - registry、planner、executor、journal、report 的验证
```

## 关联

- [[tool-permission-runtime-patterns]]
- [[erun-emu-agent-application]]
- [[opencode-agent-platform-patterns]]

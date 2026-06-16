---
title: Tool Permission Runtime Patterns
created: 2026-06-16
tags:
  - ai/agent
  - tool-runtime
  - permission
  - safety
status: draft
---

# Tool Permission Runtime Patterns

Claude Code 的 Tool Call 分析说明，成熟 agent 的工具系统不是 `name -> function` 映射，而是一条可审计、可中断、可回流的执行管线。

## Tool 是协议对象

一个合格 tool 至少要描述：

- `name` / aliases / description
- input schema
- output schema
- semantic validation
- permission check
- concurrency safety
- read-only / destructive classification
- progress reporting
- result rendering
- error rendering
- tool_result mapping

这对 erun-emu 的启发：每个 erun 动作不应只是脚本路径，而应是 registry 中的一条协议记录。

## 推荐的 erun tool pipeline

```text
OpenCode model selects tool
  -> parse structured input
  -> normalize eVision realpath
  -> normalize project realpath
  -> load script_registry
  -> validate script_id and params schema
  -> classify risk
  -> permission ask if needed
  -> acquire resource/file lock
  -> execute script or erun_cli
  -> parse fixed error strings
  -> write journal event
  -> return structured result and evidence paths
```

## 默认安全值

借鉴 Claude Code 的 fail-closed 风格：

- 新增脚本默认不并发。
- 新增脚本默认需要明确风险等级。
- 未知 `script_id` 拒绝。
- 参数 schema 不匹配拒绝。
- 相对 eVision 路径拒绝，或者必须先归一并向用户确认。
- 工程路径必须归一为 `realpath`。
- 不允许 agent 自己拼 shell/Tcl 任意命令。

## 权限是控制面

权限不只是“问用户一次”。它应该同时影响：

- 是否允许执行。
- 是否需要用户确认。
- 是否允许自动重试。
- 是否允许并发。
- 日志中如何记录风险。
- 报告中如何说明用户确认过什么。

demo 阶段可简化为少数风险等级：

| 风险等级 | 示例 | demo 行为 |
|---|---|---|
| read | 查询状态、读取日志摘要 | 可自动执行 |
| run | 上板运行、启动仿真 | permission ask |
| mutate | 修改板上状态、复位、停止 | permission ask |
| dangerous | 任意 shell、未注册脚本 | 禁止 |

## 并发策略

Claude Code 会按 `isConcurrencySafe` 把工具分批。erun-emu 可采用更保守策略：

- 同一 `(eVision realpath, project realpath)` 资源强制串行。
- 不同工程是否并发，demo 阶段也先禁止。
- 后续只对纯读取类工具开放并发。
- 锁粒度先用资源 key 文件锁，不做复杂分布式锁。

## 安全补充

Claude Code 的安全分析里有几类风险值得转化为 erun-emu 规则：

- prompt injection：日志和外部工具结果都可能包含“指令”，必须当作数据，不当作指令。
- path traversal：所有路径都 `realpath` 后再判断。
- secret leakage：报告默认脱敏 API key、license path、内网 token。
- tool shadowing：未来 MCP 工具不能覆盖核心 tool 名称。
- sandbox gap：即使有沙箱，也不能绕过应用层 permission。

## 关联

- [[erun-emu-agent-application]]
- [[prompt-skill-mcp-patterns]]
- [[memory-context-session-patterns]]

## 来源

- [Tool Call 机制实现细节](../analysis/04b-tool-call-implementation.md)
- [安全分析](../analysis/02-security-analysis.md)
- [Sandbox 技术实现细节](../analysis/04e-sandbox-implementation.md)


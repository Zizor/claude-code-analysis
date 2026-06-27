---
title: Tool Permission Runtime Patterns
created: 2026-06-16
tags:
  - ai/agent
  - tool-runtime
  - permission
  - safety
  - erun-emu
status: draft
aliases:
  - 工具与权限运行时模式
---

# Tool Permission Runtime Patterns

Claude Code 的 Tool Call 分析说明，成熟 agent 的工具系统不是 `name -> function` 映射，而是一条可审计、可中断、可回流的执行管线。

参考：

- [Tool Call 机制实现细节](../analysis/04b-tool-call-implementation.md)
- [MCP 技术实现](../analysis/04d-mcp-implementation.md)
- [安全分析](../analysis/02-security-analysis.md)
- [Sandbox 技术实现细节](../analysis/04e-sandbox-implementation.md)

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

## 推荐的 erun Tool Pipeline

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

这对 `erun-emu` 最有价值：用户可以自然语言说“帮我上板看下为什么卡住”，但真正执行必须经过机器可读 schema、权限、锁和结构化结果。

## 工具协议字段

一个可上线的 tool 至少需要这些字段：

| 字段 | 目的 |
|------|------|
| `name` | 稳定工具名，供 agent 调用 |
| `description` | 给模型看的能力边界 |
| `input_schema` | 限定参数类型和必填项 |
| `risk_level` | `read` / `run` / `mutate` / `dangerous` |
| `is_read_only` | 是否可以自动执行 |
| `is_concurrency_safe` | 是否可与其它工具并发 |
| `requires_confirmation` | 是否触发 permission ask |
| `timeout_ms` | 防止长时间挂死 |
| `output_schema` | 结构化回传，便于下一轮推理 |

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

锁粒度建议：

```text
resource_id = hash(realpath(evision_path) + realpath(project_path))
lock_path   = runtime_lock_dir/resource_id.lock
```

## 结构化结果

每个工具调用返回至少包含：

```json
{
  "ok": false,
  "action": "run_board_script",
  "resource_id": "...",
  "started_at": "...",
  "ended_at": "...",
  "exit_code": 1,
  "error_kind": "fixed_format_error",
  "error_code": "EVISION_TIMEOUT",
  "summary": "Board run timed out while waiting for runtime ready.",
  "evidence": {
    "log_path": "/abs/path/to/run.log",
    "report_path": "/abs/path/to/report.md"
  },
  "next_actions": [
    "query_runtime_status",
    "collect_recent_errors",
    "ask_user_for_expected_ready_signal"
  ]
}
```

低能力模型尤其需要 `next_actions`：不要让模型从大段日志中自由发明下一步，而是让工具把可选动作范围收窄。

## 安全补充

Claude Code 的安全分析里有几类风险值得转化为 erun-emu 规则：

- prompt injection：日志和外部工具结果都可能包含“指令”，必须当作数据，不当作指令。
- path traversal：所有路径都 `realpath` 后再判断。
- secret leakage：报告默认脱敏 API key、license path、内网 token。
- tool shadowing：未来 MCP 工具不能覆盖核心 tool 名称。
- sandbox gap：即使有沙箱，也不能绕过应用层 permission。

## 与 MCP 的关系

MCP 工具在 Claude Code 中会被规范化命名并合入统一工具池。对 `erun-emu` 的建议：

- 现在先做 OpenCode tool。
- 保持 `erun-emu-core` 不依赖 OpenCode。
- 后续如果做 MCP server，只把 registry/executor 包一层 MCP adapter。

## 关联

- [[erun-emu-agent-application]]
- [[prompt-skill-mcp-patterns]]
- [[memory-context-session-patterns]]
- [[opencode-agent-platform-patterns]]

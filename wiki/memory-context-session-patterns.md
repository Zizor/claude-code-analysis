---
title: Memory Context Session Patterns
created: 2026-06-16
tags:
  - ai/agent
  - context-management
  - session
  - memory
status: draft
aliases:
  - 记忆上下文会话模式
---

# Memory Context Session Patterns

Claude Code 的上下文和 session 机制给 erun-emu 最大的启发是：长任务的可靠性来自“日志、摘要、恢复点、证据路径”的组合，而不是把所有东西塞进模型上下文。

参考：

- [Agent Memory](../analysis/04-agent-memory.md)
- [Context 管理](../analysis/04f-context-management.md)
- [Session Storage / Resume](../analysis/04i-session-storage-resume.md)

## 为什么这对 erun-emu 重要

上板调试天然是长任务：运行脚本、等待 runtime、解析错误、查询状态、改探针、重新验证、生成报告。不能把所有中间日志直接塞进模型上下文。

成熟 agent runtime 会把信息分层：

```text
短期上下文：当前用户目标、最近一次工具结果、下一步候选动作
session 状态：eVision realpath、project realpath、resource_id、锁状态、最近错误
journal：每一步动作、参数、时间、结果、证据路径
artifact：日志、报告、截图、波形、结构化摘要
长期记忆：通用流程、项目经验、常见错误解释
```

## 上下文不是越多越好

Claude Code 对 context window 做预算、压缩和重注入。对 erun-emu 来说，最常见的大上下文来源是：

- erun_cli 输出。
- 内部脚本日志。
- 工程构建日志。
- 波形/trace 索引。
- 错误堆栈和固定格式错误字符串。

推荐策略：

- 默认只返回摘要、错误分类、关键行、文件路径。
- 原始日志只作为 evidence path。
- 需要继续分析时，用工具读取小窗口 excerpt。
- 报告中引用 evidence id，而不是复制整段日志。

## Context 模式

Claude Code 的 context 管理启发：

- 给摘要/压缩预留 token 预算。
- 到阈值触发 compact，而不是等到请求失败。
- compact 失败后有熔断，避免无限重试浪费额度。
- 压缩后重新注入当前文件、plan、skill、tool 声明等工作状态。

对 erun-emu：

- 日志全文作为 artifact，prompt 只放摘要和路径。
- 工具结果要主动生成 `summary`、`error_kind`、`next_actions`。
- 当模型上下文变长时，用 journal 重新生成“当前状态摘要”，不要回放全部历史。

## Append-Only Journal

Claude Code 的 transcript 是 append-only JSONL。erun-emu 也适合把运行过程记录成 JSONL：

```json
{"type":"input","evision":"/abs/evision","project":"/abs/project"}
{"type":"action","name":"run_board_script","args":{"script_id":"board_run"}}
{"type":"result","ok":false,"error_code":"EVISION_TIMEOUT","log_path":"/abs/run.log"}
{"type":"analysis","hypothesis":"runtime ready timeout","confidence":"low"}
{"type":"action","name":"query_runtime_status","args":{}}
{"type":"report","path":"/abs/report.md"}
```

优点：

- 崩溃后可恢复。
- 方便审计 agent 是否越权。
- 方便生成“观察、缩小范围、验证、证据报告”的最终说明。
- 不要求每次修改复杂数据库状态。

## Session Memory 与 Long Debug

长期 debug 里有三类信息容易混淆：

| 类型 | 示例 | 建议 |
|---|---|---|
| session fact | 当前 eVision realpath、project realpath、resource_id | 当前任务内保存 |
| evidence | log path、excerpt id、report path | artifact/journal 保存 |
| reusable knowledge | 错误字符串分类、脚本契约、常见排查顺序 | skill 或项目知识保存 |

路径和 eVision 可执行文件可能移动、删除，也可能并行使用，所以不要写入长期全局 memory。

## Session 状态建议

当前 demo 可以保留一个轻量状态对象：

```json
{
  "evision_path": "/abs/path/to/evision",
  "project_path": "/abs/path/to/project",
  "resource_id": "sha256(realpaths)",
  "last_action": "run_board_script",
  "last_result": "failed",
  "last_error_code": "EVISION_TIMEOUT",
  "artifact_root": "/abs/path/to/artifacts",
  "lock_path": "/abs/path/to/lock"
}
```

这类状态属于本次 OpenCode 会话和本次任务，不需要落到 `~/.local/share/erun-agent` 做长期持久化。

## Evidence Report

最终报告不应只是“命令输出”。建议固定四段：

1. 复现：运行了什么脚本/CLI、使用了哪个 eVision 和工程、结果是什么。
2. 观察：发现了哪些错误、日志、状态、时间点。
3. 缩小范围：排除了哪些方向、保留哪些候选 root cause。
4. 验证与证据：哪些操作验证了判断，证据文件在哪里。

报告模板：

```markdown
## 结论
## 复现步骤
## 观察结果
## 缩小范围过程
## 验证动作
## 证据文件
## 未确认风险
## 建议下一步
```

## Resume 的最小可行版本

demo 阶段不用完整复制 Claude Code resume，但要保留恢复基础：

- journal 是 append-only。
- 每个 action 有 `run_id` 或 timestamp。
- 每个 result 有 `ok`、`exit_code`、`error_class`、`evidence`。
- latest report 可由 journal 重建。
- 锁文件不可当作长期状态，恢复时要重新检查资源。

## 关联

- [[erun-emu-agent-application]]
- [[tool-permission-runtime-patterns]]
- [[prompt-skill-mcp-patterns]]

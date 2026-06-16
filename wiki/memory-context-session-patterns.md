---
title: Memory Context Session Patterns
created: 2026-06-16
tags:
  - ai/agent
  - context-management
  - session
  - memory
status: draft
---

# Memory Context Session Patterns

Claude Code 的上下文和 session 机制给 erun-emu 最大的启发是：长任务的可靠性来自“日志、摘要、恢复点、证据路径”的组合，而不是把所有东西塞进模型上下文。

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

## Append-Only Journal

Claude Code 的 transcript 是 append-only JSONL。erun-emu 也适合把运行过程记录成 JSONL：

```json
{"type":"session_started","session_id":"...","evision":"/abs/evision","project":"/abs/project","ts":"..."}
{"type":"permission_requested","script_id":"board_run","risk":"run","summary":"..."}
{"type":"script_started","script_id":"board_run","run_id":"..."}
{"type":"script_finished","run_id":"...","exit_code":0,"log_path":"/abs/log"}
{"type":"diagnosis","hypothesis":"...","evidence":["..."]}
```

好处：

- 写入简单，不易因崩溃损坏整体状态。
- 后续 report 可从 journal 重建。
- 支持 resume 时回放关键状态。
- 方便审计用户确认和实际执行是否一致。

## Session Memory 与 Long Debug

Claude Code 把 Session Memory 和 Agent Memory 分开。erun-emu demo 先不做长期 memory，但可以吸收这个分层：

- session journal：本次上板/调试事实。
- session summary：长对话压缩后的当前结论。
- agent skill：通用流程和领域知识。
- future agent memory：稳定经验沉淀，例如某类错误对应哪些诊断动作。

不要把“某次工程路径、某次 eVision 路径”写成长期记忆，因为用户可能移动或删除它们，也可能并行使用不同资源。

## Evidence Report

erun-emu 的 report 应至少包含：

- 输入：eVision realpath、project realpath、用户目标。
- 执行：调用过哪些 `script_id`，参数摘要，风险等级。
- 观察：错误字符串、状态、日志摘要。
- 缩小范围：为什么选择下一步诊断动作。
- 验证：执行了什么验证，结果是什么。
- 证据：journal path、log path、关键 excerpt id。
- 结论：确认、部分确认、未确认。

## Resume 的最小可行版本

demo 不一定立刻做完整 resume，但 journal 格式应为未来留钩子：

- 每条事件带 `run_id` 或 `session_id`。
- 每条外部执行记录 `cwd`、env 摘要、命令摘要、exit code。
- permission 事件和 execution 事件可关联。
- 日志路径真实存在时记录 `mtime` / `size`。
- report 可只依赖 journal + evidence path 生成。

## 关联

- [[erun-emu-agent-application]]
- [[tool-permission-runtime-patterns]]
- [[opencode-agent-platform-patterns]]

## 来源

- [Context 上下文管理机制](../analysis/04f-context-management.md)
- [Session Storage / Transcript / Resume](../analysis/04i-session-storage-resume.md)
- [Agent Memory](../analysis/04-agent-memory.md)


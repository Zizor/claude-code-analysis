---
title: Multi Agent Control Plane Patterns
created: 2026-06-16
tags:
  - ai/agent
  - multi-agent
  - control-plane
  - opencode
status: draft
aliases:
  - 多 Agent 控制面模式
---

# Multi Agent Control Plane Patterns

Claude Code 的多 agent 和 UI 组件分析说明：成熟 agent 平台不只是“能聊天和调工具”，还需要任务、权限、agent、MCP、memory、sandbox 等控制面。

参考：

- [Multi-Agent 机制](../analysis/04h-multi-agent.md)
- [核心交互组件](../analysis/components/02-core-interaction-components.md)
- [平台能力组件](../analysis/components/03-platform-components.md)

## 多 Agent 分三层

不要把所有多 agent 都叫同一种能力：

1. Subagent sidechain
   - 主 agent 派出一次性子任务。
   - 适合研究、总结、review。
   - 结果回流主线程。

2. Coordinator workers
   - 主线程变成调度器。
   - worker 分担 research / implementation / verification。
   - 结果用任务通知回流。

3. Swarm teammates
   - 有 team、mailbox、task list、permission bridge。
   - 更像 agent 组织系统。
   - 状态面和调试成本都很高。

## erun-emu 的安全边界

上板/运行态控制不同于普通代码修改。建议：

```text
主 erun-emu agent
  - 唯一硬件执行 authority
  - 持有 resource lock
  - 发起 permission ask
  - 调 erun-emu-core executor

只读 subagent
  - 分析日志
  - 阅读源码
  - 总结报告
  - 对 root cause 做独立 review
```

不要让 subagent 直接：

- 启动/停止 eVision。
- 调用上板脚本。
- 修改 runtime debug 状态。
- 释放或抢占硬件锁。

## demo 阶段取舍

demo 阶段：

- 不引入真实多 agent 执行。
- 不让子 agent 直接操作板卡。
- 不做后台 teammate。
- 不做 mailbox。

可采用的轻量模式：

- report reviewer：只读 journal 和报告草稿，提出遗漏证据。
- log summarizer：只读日志 excerpt，给出错误分类候选。
- policy reviewer：只读 registry，检查风险标注是否缺失。

这些都不拥有执行权限。

## 适合委派的任务

| 任务 | 是否适合 subagent | 原因 |
|------|-------------------|------|
| 读取长日志并提取固定错误 | 适合 | 只读、可并行、上下文大 |
| 根据报告找源码候选位置 | 适合 | 只读、局部分析 |
| 独立审查 root cause 推理 | 适合 | 能降低单模型幻觉 |
| 运行上板脚本 | 不适合 | 改变硬件状态 |
| 修改探针/触发条件 | 不适合 | 影响 runtime |
| kill eVision 进程 | 不适合 | 高风险破坏性动作 |

## 后续多 Agent 规则

如果未来引入复杂 debug worker：

- 主 erun-emu agent 是唯一执行 authority。
- worker 只能提出 action proposal。
- 高风险 action 由主 agent 走 permission ask。
- 所有 worker 输出必须落到 sidechain journal 或 report appendix。
- 禁止 worker 嵌套创建 worker。
- board/resource lock 只由主 core 管理。

## 控制面能力

Claude Code 的组件体系显示，平台能力通常需要可见控制面。OpenCode 文本 UI 可以先用命令和报告模拟：

| 控制面 | erun-emu demo 表达 |
|---|---|
| permissions | permission ask + journal 记录 |
| tasks | current run/status/report |
| agents | erun-emu agent meta 文档 |
| MCP | 暂无，未来 adapter |
| memory | demo 暂无长期 memory |
| skills | erun runtime skill |
| sandbox | 不执行任意 shell，registry 限制 |
| settings | eVision realpath/project realpath 输入 |
| diagnostics | doctor/status/report 工具 |

## 推荐 OpenCode 命令/工具视图

即使没有图形面板，也应提供：

- `erun_status`：当前 session、eVision、project、latest run。
- `erun_registry_list`：可用 script_id 和风险等级。
- `erun_report_latest`：最新证据报告。
- `erun_journal_tail`：最近 journal 事件摘要。
- `erun_doctor`：检查路径、registry、日志目录、权限配置。

这些能力的目标是让用户不用懂底层 CLI，也能知道 agent 正在做什么。

## 未来扩展

当 OpenCode 接入更稳定后，可以考虑：

- `explorer` subagent：只读日志/源码定位。
- `reviewer` subagent：审查诊断链路是否证据充分。
- `reporter` subagent：把 journal 转为客户可读报告。

但所有写操作仍由主 `erun-emu` agent 串行执行。

## 关联

- [[erun-emu-agent-application]]
- [[memory-context-session-patterns]]
- [[tool-permission-runtime-patterns]]
- [[opencode-agent-platform-patterns]]

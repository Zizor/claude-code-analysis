---
title: erun eVision Emu Agent Application Notes
created: 2026-06-16
tags:
  - ai/opencode
  - erun
  - evision
  - fpga
  - emu-agent
status: draft
---

# erun eVision Emu Agent Application Notes

本页把 Claude Code 分析资料中的 agent runtime 经验映射到 erun/eVision emu agent。目标不是复制 Claude Code，而是帮助当前 demo 形成可运行、可审计、可扩展的业务闭环。

## 当前业务假设

- 用户在 OpenCode 里用自然语言交互。
- 用户需要指定 eVision 可执行文件绝对路径和工程路径。
- eVision 输入禁止按名称搜索，执行前必须归一成 `realpath`。
- 工程路径是资源管理 unique ID 的一部分，但不作为长期持久化记忆保存。
- 内部上板脚本由专门 skill 描述，不支持 dry-run。
- erun_cli RPC 是短链接，每个命令传绝对 `--project` 路径。
- demo 阶段尽量简化，但所有简化点记录为待扩展。

## 最值得采用的设计

### 1. 统一 erun-emu-core

无论 OpenCode tool、命令行测试、未来 MCP server，都调用同一套 core。这样避免三套入口行为不一致。

core 的职责：

- 路径归一和参数校验。
- 加载 `script_registry`。
- 执行风险判断。
- 调用内部脚本或 erun_cli。
- 记录 journal。
- 生成证据报告。

不放在 core 的职责：

- 大段业务说明。
- 用户教育材料。
- 可自由变化的调试经验说明。
- OpenCode UI 的自然语言提示。

这些应该放在 skill 或 agent prompt。

### 2. script_registry 是机器安全边界

`SKILL.md` 可以告诉 agent “如何上板、如何读日志、错误字符串是什么格式”。但真正允许执行哪些 `script_id`、参数 schema、风险等级，应该由 registry 决定。

原因：

- Markdown 适合人类和模型阅读，不适合做稳定安全边界。
- registry 可被 TypeScript 测试验证。
- registry 可做版本化和兼容检查。
- registry 可服务 OpenCode tool，也可服务未来 MCP。

### 3. 默认串行执行

Claude Code 的工具并发是显式声明的。对 erun/eVision 来说应更保守：

- 上板、运行、停止、复位、修改调试状态：串行。
- 读取状态、读取日志、查询 registry：默认串行，未来可显式标记只读并发。
- 同一工程路径和 eVision 组合上的动作必须有文件锁或资源锁。

### 4. 证据优先，不塞全量日志

实际 debug 中最容易撑爆上下文的是日志。借鉴 Claude Code 的 context/session 思路：

- raw log 留在工程特定位置或 run output 目录。
- journal 记录日志路径、截取摘要、错误码、关键匹配行。
- report 面向用户输出“观察、缩小范围、验证、证据”。
- 模型需要深入时，再通过工具读取有限窗口的日志 excerpt。

### 5. 单一执行 authority

demo 阶段不要引入真正多 agent 执行。即便以后有 diagnoser/reviewer 子 agent，也应保持：

- 子 agent 可读日志、提建议、整理报告。
- 高风险执行回到主 erun-emu agent。
- 用户确认只在主 agent/permission ask 路径发生。

## 能力层表达

对外不要说“我们提供一堆 JCLI 接口”。用户关心的能力应表达成：

- 复现：自动上板、运行指定场景、采集失败现象。
- 观察：读取状态、日志、波形入口、关键错误字段。
- 缩小范围：根据错误分类选择下一组诊断动作。
- 验证：执行最小验证动作确认假设。
- 报告：给出复现条件、操作记录、关键证据、结论可信度、下一步建议。

这也对应前面讨论过的能力模型：debug agent 不只是“会调用 CLI”，而是要把 CLI 组合成工程调试流程。

## demo 可以简化的部分

- 暂不做 `plan_id`。
- 暂不做复杂 diagnostic policy DSL。
- 暂不做长期 memory。
- 暂不做多 agent 执行。
- 暂不做 MCP server。
- 暂不做 eVision 名称搜索。
- 暂不做相对 project 工作目录优化，统一传绝对路径。

这些简化不应删除设计余地：journal、registry、core adapter 边界要保留。

## 待扩展方向

- diagnostic policy 从少量规则演化为可配置策略表。
- report 支持结构化 JSON 和 Markdown 双输出。
- journal 支持 resume 和一致性检查。
- 工具支持 log excerpt、error classifier、evidence bundle。
- OpenCode permission ask 和 core risk policy 建立更完整映射。
- 未来 MCP adapter 复用 erun-emu-core。

## 关联

- [[opencode-agent-platform-patterns]]
- [[tool-permission-runtime-patterns]]
- [[memory-context-session-patterns]]
- [[multi-agent-control-plane-patterns]]


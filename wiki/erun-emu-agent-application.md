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
aliases:
  - erun-emu agent 落地建议
---

# erun eVision Emu Agent Application Notes

本页把 Claude Code、OpenCode 和权威 agent 资料中的经验映射到 erun/eVision emu agent。目标不是复制 Claude Code，而是帮助当前 demo 形成可运行、可审计、可扩展的业务闭环。

## 业务定位

`erun-emu` agent 不应只是“提供一堆 JCLI/CLI 接口”。它面向用户提供的是自然语言调试能力：

- 帮用户复现问题。
- 观察运行态、错误、日志和状态。
- 缩小问题范围。
- 验证候选 root cause。
- 生成证据报告。

底层接口是实现细节，用户不应需要知道具体 `script_id` 或 CLI 参数。

## 当前业务假设

- 用户在 OpenCode 里用自然语言交互。
- 用户需要指定 eVision 可执行文件绝对路径和工程路径。
- eVision 输入禁止按名称搜索，执行前必须归一成 `realpath`。
- 工程路径是资源管理 unique ID 的一部分，但不作为长期持久化记忆保存。
- 内部上板脚本由专门 skill 描述，不支持 dry-run。
- erun_cli RPC 是短链接，每个命令传绝对 `--project` 路径。
- 不切换到工程目录，统一传绝对路径。
- demo 阶段尽量简化，但所有简化点记录为待扩展。

## 三层架构

```text
OpenCode erun-emu agent
  - 与用户对话
  - 判断缺失信息
  - 决定何时请求确认
  - 调用工具
  - 输出阶段性结论

erun-opencode-runtime-skill
  - 业务流程说明
  - 日志位置和错误格式说明
  - debug 思路和报告模板

erun-emu-core
  - script registry
  - realpath canonicalization
  - file lock
  - script/CLI execution
  - fixed-format error parser
  - artifact/journal/report output
```

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

## 能力闭环

### 1. 复现

Agent 收集 eVision 路径、工程路径、用户目标后，调用内部一键脚本：

```text
run_board_script(evision_path, project_path, script_id, args)
```

输出：

- 是否成功。
- exit code。
- 固定格式错误字符串解析结果。
- 日志路径。
- artifact 目录。

### 2. 观察

如果复现失败或运行异常，agent 不直接猜原因，而是调用观察类动作：

- 读取最新 run log。
- 查询 runtime 状态。
- 解析固定格式错误。
- 提取最近 N 条关键事件。
- 收集工程特定日志位置。

观察结果必须结构化，尤其要给 `error_kind`、`error_code`、`symptom`、`evidence_path`。

### 3. 缩小范围

方向不靠模型凭空知道，而来自三类输入：

- `next_actions`：工具根据错误类型给候选下一步。
- diagnostic policy：由 skill/配置表达的排障知识，demo 可先简化。
- 历史 evidence：journal 中已做过哪些检查，哪些方向被排除。

例子：

```text
现象：runtime ready 超时
候选方向：
  A. eVision/工程启动失败
  B. testbench 未启动
  C. 通信链路初始化失败
  D. 用户目标中的 ready 信号条件不满足

缩小动作：
  1. 查询 eVision 进程/运行态状态
  2. 检查 testbench log 中启动标记
  3. 检查 RPC/JCLI 可达性
  4. 检查指定信号/探针最近值
```

### 4. 验证

缩小范围后必须做验证动作，例如：

- 重新查询同一状态确认是否稳定。
- 执行只读 debug query 确认信号/状态。
- 若需要改变 runtime 状态，先向用户确认。

### 5. 证据报告

报告格式固定：

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

## 能力层表达

对外不要说“我们提供一堆 JCLI 接口”。用户关心的能力应表达成：

- 复现：自动上板、运行指定场景、采集失败现象。
- 观察：读取状态、日志、波形入口、关键错误字段。
- 缩小范围：根据错误分类选择下一组诊断动作。
- 验证：执行最小验证动作确认假设。
- 报告：给出复现条件、操作记录、关键证据、结论可信度、下一步建议。

这也对应能力模型：debug agent 不只是“会调用 CLI”，而是要把 CLI 组合成工程调试流程。

## Dynamic Debug Action 选择

不要把复杂诊断流程硬编码成单一路径。更稳的方式：

```text
tool result
  -> normalized symptom/error_kind
  -> policy candidate actions
  -> model chooses one from bounded list
  -> core validates action and args
  -> execute
  -> update journal
```

demo 阶段可以简化为少量规则，但要在文档中标记为待扩展：

- 当前只覆盖常见错误类型。
- 当前 `next_actions` 由简单 mapping 生成。
- 当前没有完整 diagnostic policy DSL。
- 当前没有复杂 plan graph。

## 面向低能力模型的设计

用户部署模型可能只有 DeepSeek V4 Flash 级别，且不支持多模型切换。设计上要降低模型负担：

- 工具输出尽量结构化。
- 每一步只让模型在少量候选动作中选择。
- agent prompt 短而硬，不依赖长链推理。
- skill 中流程使用清单和模板。
- 报告模板固定。
- 对高风险动作统一 permission ask，不让模型判断“这次是不是危险”。

## demo 可以简化的部分

- 暂不做 `plan_id`。
- 暂不做复杂 diagnostic policy DSL。
- 暂不做长期 memory。
- 暂不做多 agent 执行。
- 暂不做 MCP server。
- 暂不做 eVision 名称搜索。
- 暂不做相对 project 工作目录优化，统一传绝对路径。

这些简化不应删除设计余地：journal、registry、core adapter 边界要保留。

## 最小 demo 闭环

```text
1. 用户给 eVision path + project path + 目标
2. core realpath 校验
3. acquire file lock
4. run one-click board script
5. parse fixed-format error/log
6. return structured result with next_actions
7. agent 做一轮观察/解释
8. 生成 evidence report
9. release lock
```

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
- [[authoritative-agent-development-guide]]

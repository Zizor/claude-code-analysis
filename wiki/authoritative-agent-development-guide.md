---
title: Authoritative Agent Development Guide
created: 2026-06-17
tags:
  - ai/agent
  - ai/opencode
  - erun-emu
  - guide
status: draft
aliases:
  - Agent 开发权威攻略
sources:
  - https://developers.openai.com/api/docs/guides/agents
  - https://openai.github.io/openai-agents-python/
  - https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/
  - https://www.anthropic.com/engineering/building-effective-agents
  - https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
  - https://modelcontextprotocol.io/
  - https://opencode.ai/docs/agents/
  - https://opencode.ai/docs/skills/
---

# Authoritative Agent Development Guide

这是一份面向 OpenCode 通用 agent 和 erun/eVision emu agent 的权威资料提炼版攻略。它综合了 OpenAI、Anthropic、MCP、OpenCode、Google Agent Development Kit (ADK)、AGENTS.md 等官方资料，并结合本仓库 Claude Code 分析给出落地建议。

## 1. 先判断是否真的需要 Agent

OpenAI 和 Anthropic 都强调：agent 适合“多步、需要工具、需要根据环境反馈继续决策”的任务，不适合把简单单轮问答复杂化。

适合 erun-emu 的原因：

- 用户目标通常是“上板并定位问题”，不是单次命令。
- 执行过程需要环境反馈：脚本返回值、日志、错误字符串、板卡状态。
- 后续步骤无法完全预先硬编码，必须根据观察结果调整。
- 结果需要证据报告，而不是只返回命令输出。

不应该 agent 化的部分：

- 单纯列出 CLI help。
- 固定脚本封装。
- 单次只读查询。

设计判断：

```text
简单固定动作 -> CLI wrapper
多步、带反馈、带判断 -> agent workflow
高风险硬件动作 -> agent + permission + journal
```

`erun-emu` 的定位是混合模式：上板脚本、CLI 调用、参数校验必须 deterministic；“下一步看什么、如何解释结果、如何生成报告”可以由 agent 做。

## 2. 从简单 Workflow 开始

Anthropic 的核心建议是：先用简单可组合模式，只有当它们不够时再引入自主 agent。

可采用的逐级结构：

| 阶段 | 模式 | erun-emu 示例 |
|---|---|---|
| 1 | Prompt chaining | 上板 -> 解析日志 -> 生成报告 |
| 2 | Routing | 根据错误字符串路由到 compile/run/runtime/debug 分类 |
| 3 | Parallelization | 多个只读日志摘要并行，demo 阶段可先不做 |
| 4 | Evaluator-optimizer | 生成报告后由 reviewer 检查证据是否充分 |
| 5 | Agent loop | 用户目标开放，agent 自行选择下一步诊断动作 |

demo 建议停在 1、2、少量 5：允许 agent 根据错误选择动作，但执行动作必须经过 registry 和 permission。

## 3. Agent Loop 要有硬边界

OpenAI Agents SDK 明确了 loop：模型输出、final output、handoff、tool call、append result、继续，超过 max turns 则停止。

erun-emu 需要对应的硬边界：

- `max_steps`：OpenCode agent 配置或 core 运行参数。
- `max_tool_calls`：防止模型反复查询。
- `max_log_excerpt_bytes`：防止日志撑爆上下文。
- `permission ask`：高风险动作暂停。
- `resource lock`：同一 eVision/project 串行。
- `stop condition`：成功、失败不可恢复、用户取消、达到步数上限。

## 4. 从最小闭环开始

推荐 demo 闭环：

1. 收集 eVision 路径和工程路径。
2. 归一化 `realpath`。
3. 获取资源文件锁。
4. 运行内部一键脚本。
5. 解析固定格式错误。
6. 输出结构化结果。
7. 生成证据报告。

先让这个闭环稳定，再增加复杂 debug 动作组合。

## 5. Tool 设计是 Agent 成败核心

Anthropic 明确指出，tool definition 和 tool format 需要像 prompt 一样认真设计。OpenAI SDK 也把工具分为 hosted、runtime/local、function、agents-as-tools、MCP 等不同类别。

erun-emu 的 tool 原则：

- 输入 schema 要窄，不允许 free-form shell。
- 输出要结构化，不能只有 stdout。
- 错误要分类，能指导下一步。
- tool 名称要表达业务动作，不暴露底层脚本细节。
- tool description 要短，但足够让模型知道什么时候使用。
- 每个 tool 要声明风险等级、是否只读、超时、artifact 路径和 `next_actions`。

推荐输出结构：

```json
{
  "ok": false,
  "script_id": "board_run",
  "risk": "run",
  "exit_code": 2,
  "error_class": "runtime_assert",
  "summary": "运行阶段触发固定格式错误字符串",
  "evidence": [
    {
      "kind": "log",
      "path": "/abs/path/to/run.log",
      "excerpt_id": "err-001"
    }
  ],
  "next_actions": ["read_runtime_log_excerpt", "query_debug_state"]
}
```

## 6. Permission 和 Human Review 是正式流程

OpenAI 的 human-in-the-loop 文档把审批作为 run interruption 和 resumable state。OpenCode 也以 `ask` / `allow` / `deny` 为 agent permission 核心。

erun-emu 建议：

- 读取 registry、查看状态、读取日志摘要：默认 allow。
- 上板、运行脚本、停止/复位硬件：默认 ask。
- 未注册脚本、任意 shell/Tcl、eVision 名称搜索：deny。
- 用户确认结果要写 journal。
- 拒绝后不要原样重试，应解释或请求新指令。

确认后仍要重新校验 registry、参数和锁，不要认为“模型刚才说过”就是授权。

## 7. Context Engineering 优先于 Prompt 堆砌

Anthropic 的 context engineering 资料强调：agent 的关键不是写更长 prompt，而是每轮选择最有用的上下文。Google ADK 的 artifacts 也说明大文件、二进制、日志适合独立保存。

erun-emu 实践：

- raw log、波形、trace、报告附件作为 evidence/artifact。
- 模型上下文只放摘要、路径、关键 excerpt。
- 当前 eVision/project 是 session fact，不是长期 memory。
- 长期可复用的是调试策略、错误分类、脚本契约，而不是某个工程路径。

## 8. Memory 分层

| 类型 | 内容 | 生命周期 |
|------|------|----------|
| Skill | 稳定业务流程、错误格式、日志规则 | 随版本发布 |
| Agent prompt | 角色、默认策略、权限风格 | 随 agent 发布 |
| Session journal | 本次 eVision/project/resource/action/result | 本次任务 |
| Artifact | 日志、报告、证据包 | 跟随任务输出 |
| 长期 memory | 常见问题、项目经验 | 谨慎保存，需可审查 |

路径和 eVision 可执行文件可能移动、删除，也可能并行使用，所以不要做长期全局记忆。

## 9. MCP 是扩展协议，不是核心业务模型

MCP 官方定义了 host/client/server 和 tools/resources/prompts。它非常适合未来把 erun-emu 暴露给不同 agent 客户端。

当前 demo 应保持：

```text
OpenCode tool adapter
  -> erun-emu-core
  -> internal scripts / erun_cli
```

未来再加：

```text
MCP server adapter
  -> erun-emu-core
```

这样 MCP 只是 adapter，不改变 registry、permission、journal、report 这些核心契约。

## 10. Skill 与 AGENTS.md 的位置

OpenCode skills 是按需加载的 reusable instruction。AGENTS.md 是 repo-level agent instruction 格式。两者都适合放“给 agent 读的知识”。

放 skill：

- erun/eVision 业务说明。
- 上板脚本的使用契约。
- 错误字符串格式。
- 日志位置说明。
- debug 报告格式。

放 AGENTS.md / 项目文档：

- 仓库结构。
- 构建/测试命令。
- 代码风格。
- 安全注意事项。
- 工程协作约定。

放 core registry：

- 可执行 `script_id`。
- 参数 schema。
- 风险等级。
- 是否只读/可并发。
- 输出 parser。

## 11. 动态 Debug Action 组合

不要写死完整状态机。推荐数据驱动的候选动作选择：

```text
symptom/error_kind
  -> candidate debug actions
  -> action preconditions
  -> risk level
  -> expected evidence
  -> model chooses within bounded list
  -> core validates
```

demo 阶段可先用简单 mapping，但必须记录为待扩展。

## 12. Observability 是 Agent 产品的一部分

OpenAI tracing、Google ADK integrations、Claude Code 分析都指向同一件事：agent 要能被调试、审计、回放。

erun-emu 的最小可观察性：

- journal JSONL。
- latest status。
- latest report。
- evidence path。
- permission decision record。
- run id / session id。

后续可加：

- trace view。
- registry doctor。
- log excerpt browser。
- report reviewer。
- resume from journal。

## 13. 面向低模型能力的设计

当前用户部署模型可能只有 deepseek-v4-flash 级别，也可能不支持多模型切换，因此要降低模型负担：

- 动作选择尽量由 registry metadata 和简单 routing 规则辅助。
- tool output 给出 `next_actions` 候选。
- prompt 少而稳定，避免复杂隐含规则。
- 每步结果结构化，减少模型从散乱日志中推理。
- 高风险动作固定 ask，避免让模型判断安全边界。
- 报告模板固定，模型只填内容。

## 推荐落地顺序

1. 完成 registry + executor + journal + report 四件套。
2. 在 OpenCode agent 里配置 permission 和 prompt。
3. 把 erun runtime skill 接入，明确 skill/core 边界。
4. 增加 log excerpt 和 error classifier。
5. 增加 doctor/status/report 工具。
6. 再考虑 MCP adapter 和复杂 diagnostic policy。

## 推荐实现检查表

- [ ] `erun-emu-core` 不依赖 OpenCode。
- [ ] `script_registry` 是机器可读 JSON。
- [ ] eVision/project 执行前 `realpath`。
- [ ] 不支持 eVision 名称搜索。
- [ ] 所有 CLI 传绝对 project path。
- [ ] 文件锁保护资源。
- [ ] 高风险动作 permission ask。
- [ ] 工具输出固定 schema。
- [ ] 日志和报告作为 artifact。
- [ ] skill 只保存业务说明，不做权限源。
- [ ] demo 简化项全部写入待扩展清单。

## 关联

- [[authoritative-agent-source-map]]
- [[erun-emu-agent-application]]
- [[tool-permission-runtime-patterns]]
- [[prompt-skill-mcp-patterns]]
- [[memory-context-session-patterns]]
- [[multi-agent-control-plane-patterns]]

# Agent 工程规范 · 来源与演进

> **范围：** §20 参考来源、附录：规范落地与演进
> 集合：`standards/` ｜ 导航：[知识库索引](../README.md)

# 20. 参考来源

| 来源 | 重点 |
|---|---|
| Anthropic — Building Effective Agents | 最低必要复杂度、Workflow 与 Agent 边界、环境反馈 |
| Anthropic — Claude Code Best Practices | 项目说明、增量开发、测试与 Diff |
| Anthropic — Writing Effective Tools | Tool 命名、Schema、结果裁剪和 Tool Eval |
| Anthropic — Context Engineering | 高信号上下文、预算、压缩与动态选择 |
| Anthropic — Agent Skills | 渐进加载、SKILL.md、脚本和参考资料 |
| Anthropic — Sandboxing / Containment | 文件、网络、凭证和执行隔离 |
| Anthropic — Long-running Agents | 任务拆分、进度文件、跨会话交接 |
| Anthropic — Agent Evals | Task、Trial、Grader、Trace 和回归集 |
| OpenAI Agents SDK | Agent、Tool、Guardrail、Session、HITL 和 Trace |
| OpenAI Codex AGENTS.md | 分层规则、近目录优先、上下文预算 |
| OpenAI Codex Skills | 单一职责、触发描述、渐进加载 |
| OpenAI Codex Non-interactive Mode | JSONL、Schema 输出、稳定退出码 |
| Google agents-cli | Scaffold、Enhance、Eval、Deploy、Observe 生命周期 |
| Vercel AI SDK | ToolLoopAgent、显式 Workflow、步数控制 |
| LangGraph | Checkpoint、Store、Interrupt、重放和幂等 |
| MCP Specification | 系统边界、授权、Tool 注解和用户确认 |
| OpenTelemetry GenAI | Trace、Metric、Log 语义和敏感字段策略 |
| OWASP Agentic Security | 目标劫持、Tool 误用、权限、Memory 和供应链风险 |

---

---

## 附录：规范落地与演进

### A.1 规范落地链路

```text
本规范（工程实践总纲）
 → 结合项目 agent.design.md / 选型决策 / 技术栈 / 组织 Policy
 → 规则解析
 → 根 AGENTS.md + 模块 AGENTS.md + Skills + 项目验证门槛 + Evals
 → 生产反馈回流（脱敏、授权、验证后）
```

### A.2 规则解析原则

- 只选择适用于当前项目的规则；
- 合并组织强制规则；
- 避免重复和冲突；
- 记录每条生成规则的来源 ID；
- 允许项目新增更严格规则；
- 不允许项目降低组织安全下限。

### A.3 规范变更兼容策略

| 类型 | 处理 |
|---|---|
| 文字澄清 | 可自动更新 |
| 新增推荐规则 | 生成 Diff，默认建议 |
| 新增强制安全规则 | 必须升级并通过项目验证门槛 |
| 命令或目录变化 | 结合项目实际结构迁移 |
| 删除过时规则 | 确认项目无依赖后移除 |
| 语义不兼容 | 通过规则版本和 Migration 处理 |

### A.4 长期沉淀的工程原语

```text
Complexity · Context · Tool · State · Workflow · Approval
Eval · Trace · Trajectory · Experience · Policy · Rule Enforcement
Harness Eval · Evaluator · Cost Governance · Containment · Skill · Handoff
```

框架和模型可替换，规则来源和验证机制保持稳定。

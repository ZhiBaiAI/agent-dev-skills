# agent-design 参考加载清单

按触发条件加载下面这些文件；不要整库加载。路径相对仓库根目录。

| 阶段 | 加载什么 | 为什么 |
|---|---|---|
| 澄清需求 / 定复杂度 | `knowledge/standards/01-foundations.md`（§1） | 最简实现、Harness 设计、系统约束优先——决定设计锚点 |
| 设计上下文装载 | `knowledge/standards/02-context-engineering.md`（§2） | 上下文分层、索引优先、交接产物——设计的是上下文而不只是 Agent |
| 定安全边界 | `knowledge/standards/07-observability-and-security.md`（§10） | 沙箱边界、不可信数据、最小权限 |
| 走复杂度档位 | `knowledge/rules/architecture-guardrails.md`（L3） | 硬性架构/安全约束 |
| 设计对话方法 | `knowledge/practices/agent-design-workflow.md`（L2） | 渐进式澄清流程 |
| 项目文件骨架 / 工件链 | `knowledge/standards/08-delivery-and-project-docs.md`（§12、§13） | `.agent/` 树、AGENTS.md 作为路由器 |
| 设计阶段实践的提炼版 | `knowledge/practices/01-mindset-and-collaboration.md`、`02-harness-and-knowledge.md` | playbook §1-4 的心智模型与协作范式 |
| 反例自查 | `knowledge/cases/01-context-and-tools.md`、`02-multi-agent-and-durability.md` | 同项目的真实失败案例 |

触发加载：只在用户确实提出 Agent 设计、需要设计文档时加载。

# agent-review 参考加载清单

按触发条件加载下面这些文件；不要整库加载。路径相对仓库根目录。

| 阶段 | 加载什么 | 为什么 |
|---|---|---|
| 逐项评审 | `knowledge/standards/09-review-and-hard-rules.md`（§18） | 11 部分架构 Review Checklist |
| 对照硬规则 | `knowledge/standards/09-review-and-hard-rules.md`（§19） | 78 条硬规则，逐条可查 |
| 生产质量与经验闭环 | `knowledge/standards/06-evals-and-quality.md`（§8） | §8.1、§8.8 失败→工程资产 |
| AGENTS.md 质量 | `knowledge/standards/08-delivery-and-project-docs.md`（§13） | 路由器而非百科全书 |
| 安全审计 | `knowledge/standards/07-observability-and-security.md`（§10） | 被门控能力是否有激活的控制 |
| 复杂度漂移 | `knowledge/standards/03-orchestration-and-hitl.md`（§3、§3.8） | 档位与任务分级 |
| 评审纪律 | `knowledge/practices/03-verification-and-safety.md`（§5） | 只报可行动 finding、渐进信任 |
| 成本包络 | `knowledge/practices/04-cost-and-rollout.md`（§7） | 成本回归判据 |
| 变更可归因 | `knowledge/cases/03-evals-and-quality.md`（§6 生产质量） | 多变更叠加导致的退化 |
| 硬性边界 | `knowledge/rules/architecture-guardrails.md`（L3） | diff 是否越过声明边界 |

触发加载：只在合并/发布前需要就绪度评审时加载。

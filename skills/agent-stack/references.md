# agent-stack 参考加载清单

按触发条件加载下面这些文件；不要整库加载。路径相对仓库根目录。

| 阶段 | 加载什么 | 为什么 |
|---|---|---|
| 逐槽位选型 | `knowledge/selection/01-selection-foundations.md` | 三层抽象、确定性谱系、复杂度阶梯 |
| 判断 Harness 能力 | `knowledge/selection/02-harness-and-paradigms.md` | Harness 组件清单、规范驱动 vs 对话驱动 |
| 选型自检 / 避坑 | `knowledge/selection/03-decision-checklist.md` | 7 组决策清单 + §8 反模式 |
| 对照具体框架 | `knowledge/selection/04-framework-comparison.md` | Runtime/Framework/Harness 层对比、跨层组合 |
| 工具与 MCP 契约 | `knowledge/standards/04-tools-and-mcp.md`（§4、§11） | Tool Contract、CLI/MCP 选择判据、MCP 成本 |
| 可替换边界 | `knowledge/standards/07-observability-and-security.md`（§10.2） | Brain/Hands/Session 分离 |
| 硬性架构约束 | `knowledge/rules/architecture-guardrails.md`（L3） | 不引入无需求的重组件 |
| 成本作为选型标准 | `knowledge/practices/04-cost-and-rollout.md` | playbook §7 成本杠杆 |

触发加载：只在 `agent-design` 已产出设计、需要技术选型时加载。

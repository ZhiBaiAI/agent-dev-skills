# complexity-ladder 参考加载清单

按触发条件加载下面这些文件；不要整库加载。路径相对仓库根目录。

| 阶段 | 加载什么 | 为什么 |
|---|---|---|
| 每档准入测试 | `knowledge/standards/03-orchestration-and-hitl.md`（§3） | Tool Loop / Workflow / 多 Agent 的准入条件 |
| 任务影响分级 | `knowledge/standards/03-orchestration-and-hitl.md`（§3.8） | Minor / Standard / Major 与评审阶梯 |
| 规模路由与拆分成本 | `knowledge/standards/03-orchestration-and-hitl.md`（§3.12） | Agent 拆分成本 |
| 同构批量走代码 | `knowledge/standards/03-orchestration-and-hitl.md`（§3.14） | 代码编排替代 N 次往返 |
| 过度架构反例 | `knowledge/cases/02-multi-agent-and-durability.md`（§3、§4） | 多 Agent 与持久化执行的失败模式 |
| 成本杠杆 | `knowledge/practices/04-cost-and-rollout.md`（§7） | 子 Agent 降档、缓存分层 |
| 硬性架构约束 | `knowledge/rules/architecture-guardrails.md`（L3） | 重组件必须显式声明理由 |
| 复杂度选择原则 | `knowledge/standards/01-foundations.md`（§1.1） | 从最简可验证实现开始 |

触发加载：任何要选择执行模型、添加编排，或为一个更重架构找理由时加载。

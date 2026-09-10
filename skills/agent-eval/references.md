# agent-eval 参考加载清单

按触发条件加载下面这些文件；不要整库加载。路径相对仓库根目录。

| 阶段 | 加载什么 | 为什么 |
|---|---|---|
| 评测体系设计 | `knowledge/standards/06-evals-and-quality.md`（§7） | Grader、Gate、证据化置信、假修复检测、Online/Offline 飞轮 |
| 生产指标 | `knowledge/standards/06-evals-and-quality.md`（§8.1-8.2） | 三层指标、Task Success、Quality Floor |
| 成本指标 | `knowledge/standards/06-evals-and-quality.md`（§7.8-7.9） | 成本归因、模型分层路由 |
| 机评工程 | `knowledge/cases/03-evals-and-quality.md`（§5） | LLM-as-Judge 校准、rubric 案例 |
| 防自我欺骗 | `knowledge/practices/03-verification-and-safety.md`（§5） | 评测集冻结、对冲指标、目标人复核 |
| 自助评测 harness | `knowledge/standards/06-evals-and-quality.md`（§7.7） | 用强 Agent 搭评测 harness |
| 反例与回归来源 | `knowledge/cases/03-evals-and-quality.md`（§5、§6） | 生产质量案例 |

触发加载：只在已有需要保持稳定行为、要建或跑评测时加载。

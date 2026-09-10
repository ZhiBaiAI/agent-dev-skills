# agent-scaffold 参考加载清单

按触发条件加载下面这些文件；不要整库加载。路径相对仓库根目录。

| 阶段 | 加载什么 | 为什么 |
|---|---|---|
| 建项目文件骨架 | `knowledge/standards/08-delivery-and-project-docs.md`（§12.2、§13） | `.agent/` 树、AGENTS.md 作为路由器 |
| 分步开发与交接 | `knowledge/standards/08-delivery-and-project-docs.md`（§12.1） | 可续接、可问责的交付纪律 |
| 门槛与假绿防御 | `knowledge/standards/06-evals-and-quality.md`（§7.6） | 分层验证、假修复检测 |
| 生成项目内置能力 | `knowledge/standards/08-delivery-and-project-docs.md`（§16） | 项目应具备的工程能力基线 |
| 完成标准（DoD） | `knowledge/standards/08-delivery-and-project-docs.md`（§17） | Definition of Done |
| 接缝期安全控制 | `knowledge/standards/07-observability-and-security.md`（§10） | 被门控能力随同一次变更交付控制 |
| Harness 三件事 | `knowledge/practices/02-harness-and-knowledge.md` | playbook §3 读对/拦住/接得上 |
| 验证与评审纪律 | `knowledge/practices/03-verification-and-safety.md` | playbook §5、§6 |
| 生成本身的方法 | `knowledge/standards/08-delivery-and-project-docs.md`（§15 CLI 与编码 Agent 自动化规则） | CLI 非交互式自动化 |
| 权限与副作用 | `knowledge/rules/architecture-guardrails.md`（L3） | 破坏性改动走用户批准的 diff |

触发加载：只在设计与选型已定、要生成或增量接线项目时加载。

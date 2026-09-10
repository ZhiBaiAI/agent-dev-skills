# security-gates 参考加载清单

按触发条件加载下面这些文件；不要整库加载。路径相对仓库根目录。

| 阶段 | 加载什么 | 为什么 |
|---|---|---|
| 隔离模型 | `knowledge/standards/07-observability-and-security.md`（§10） | 沙箱边界、Brain/Hands/Session、不可信数据、记忆投毒 |
| OWASP Agentic | `knowledge/standards/07-observability-and-security.md`（§10.5） | 应覆盖的风险类别 |
| AI-BOM 与委托链 | `knowledge/standards/07-observability-and-security.md`（§10.6-10.7） | 资产清单、非人类身份 |
| MCP 安全基线 | `knowledge/standards/04-tools-and-mcp.md`（§11） | OAuth、audience、token 不透传、服务器白名单 |
| 审批与 Guardrail | `knowledge/standards/03-orchestration-and-hitl.md`（§6） | 哪些动作必须审批、请求规范、边界 |
| 失败案例 | `knowledge/cases/04-security-and-cost.md`（§7） | 注入、凭据窃取、出口绕过、批准疲劳 |
| 密钥与副作用 | `knowledge/practices/03-verification-and-safety.md`（§6） | stage/apply、幂等键、超时≠失败 |
| L3 硬约束 | `knowledge/rules/architecture-guardrails.md` | 必须显式声明与保持的能力 |

触发加载：只要设计/选型/生成代码/评审中出现浏览器、shell、文件系统写入、外部 MCP、多智能体或外部凭据，立即加载。

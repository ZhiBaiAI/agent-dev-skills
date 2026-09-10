# Agent 工程知识库

技能集共享的、版本化的参考知识库，也是**全库唯一导航入口**。技能不复制知识，而是按触发条件加载相关文件——每个技能用自己目录下的 `references.md` 声明加载清单。

- **唯一事实来源**：知识只存在一处，技能与导航都只引用、不复制。
- **按关注点组织**：集合按主题拆成带序号的小文件，单文件目标 ≤ 400 行。
- **索引优先**：先在下表定位，再打开具体文件，不整库加载。
- **机器可读目录**：`index.yaml` 给出每个条目的触发条件与权威级别；`rules/architecture-guardrails.md` 是唯一的 L3 `authority: enforced`。

## 集合导航

| 集合 | 目录 | 性质 | 条目 |
|---|---|---|---|
| Agent 工程规范 | [`standards/`](standards/) | L2-L3，应然规则 | 10 个主题文件，覆盖 §1–§20 + 附录 |
| Agent 开发经验 | [`cases/`](cases/) | L0 参考，2026 年一手案例 | 6 个主题文件，覆盖 §1–§13 |
| 技术选型参考 | [`selection/`](selection/) | L0 参考，选型认知层 | 4 个主题文件，覆盖 §1–§10 |
| 开发实践指南 | [`practices/`](practices/) | L2 组织流程 | 4 个主题文件 + `agent-design-workflow.md` |
| 架构与安全边界 | [`rules/architecture-guardrails.md`](rules/architecture-guardrails.md) | **L3 强制** | 单文件 |

**按章节号定位：** 引用形如 `standard §7.6`、`lessons §5.13`。取整数部分（`§7`、`§5`），在对应集合的表里查到文件；章节编号在各文件内保持不变。

**别名：** `standard` / `标准` → `standards/`，`lessons` → `cases/`，`playbook` → `practices/`。按 §号取对应文件。

---

## Agent 工程规范（`standards/`）

| 项目 | 内容 |
|---|---|
| 版本 | v1.0（提炼自《智能体工程优秀实践规范 v2.0》） |
| 定位 | 面向 Agent 开发技术人员的工程实践总纲，完全中立；具体命令与目录由各项目 AGENTS.md 适配 |
| 适用范围 | 由本技能集产出的所有项目及其 AGENTS.md / Skills / 项目验证门槛 / Eval |
| 目标 | 沉淀稳定、可复用、可执行的 Agent 工程实践，并转化为设计约束、规则、检查与评测 |
| 主要来源 | Anthropic、OpenAI、Google agents-cli、Vercel AI SDK、LangGraph、MCP、OpenTelemetry、OWASP |

| 文件 | 范围 | 覆盖章节 |
|---|---|---|
| [`01-foundations.md`](standards/01-foundations.md) | §1 核心原则：最简实现、Harness 设计、环境反馈、系统约束优先、约束分级、Loop Engineering、Runtime Protocol、规则编译与拦截、规则文件衰减 | §1 |
| [`02-context-engineering.md`](standards/02-context-engineering.md) | §2 上下文组成、信任分级、预算、压缩、数据生命周期、单一表示、Prompt 预算预检、阶段交接、分层加载、成本与稳定前缀、上游一次采集、索引优先检索 | §2 |
| [`03-orchestration-and-hitl.md`](standards/03-orchestration-and-hitl.md) | §3 Agent 与 Workflow 选择（Tool Loop/Workflow/多 Agent 准入、任务分级、协作平面、代码编排）、§6 Human-in-the-loop（审批、Guardrail、结构化澄清、审阅回环） | §3、§6 |
| [`04-tools-and-mcp.md`](standards/04-tools-and-mcp.md) | §4 Tool 设计（契约、描述、输出、Evals、Capability、Error Contract、CLI/MCP 选择、Schema 成本）、§11 MCP 使用规范与安全基线 | §4、§11 |
| [`05-state-memory-durability.md`](standards/05-state-memory-durability.md) | §5 四类状态、Checkpoint 与 Store、可恢复 Workflow、执行账本、Working Memory、故障分级、知识资产与回流、Loop State、并发、Interrupt/Resume | §5 |
| [`06-evals-and-quality.md`](standards/06-evals-and-quality.md) | §7 Evals-first（Grader、Gate、证据化置信、假修复检测、成本归因、模型分层、Online/Offline 飞轮）、§8 生产质量、Trajectory 与经验闭环 | §7、§8 |
| [`07-observability-and-security.md`](standards/07-observability-and-security.md) | §9 Observability 与 Trace、§10 安全、Containment 与最小权限（沙箱边界、Brain/Hands/Session、不可信数据、OWASP、AI-BOM、记忆投毒） | §9、§10 |
| [`08-delivery-and-project-docs.md`](standards/08-delivery-and-project-docs.md) | §12 长任务开发范式、§13 AGENTS.md 规范、§14 Skills 规范、§15 CLI 自动化、§16 项目应内置能力、§17 Definition of Done | §12、§13、§14、§15、§16、§17 |
| [`09-review-and-hard-rules.md`](standards/09-review-and-hard-rules.md) | §18 架构 Review Checklist、§19 七十八条硬规则 | §18、§19 |
| [`10-references-and-evolution.md`](standards/10-references-and-evolution.md) | §20 参考来源、附录：规范落地与演进 | §20、附录 |

### 定位与使用方式

**这份规范是什么。** Agent 项目的工程实践总纲，沉淀跨模型、跨框架、跨项目适用的工程原则与规则，回答"Agent 工程应该怎么做"，是项目内所有工程约束的上游来源。

**给谁看。** Agent 开发技术人员（完整参考，理解每条规则为何存在、如何执行）；编码 Agent（通过 AGENTS.md 与 Skills 间接消费子集）；架构 Review 人员（用 §18 检查清单）。

**与 AGENTS.md 的关系。** 本规范是完整总纲；根 AGENTS.md 是项目级可执行规则（精简、含命令）；目录级 AGENTS.md 是模块规则；Skills 是专项可复用工作流。**AGENTS.md 不复制本规范全文**，只抽取适用规则并适配实际技术栈与目录——本规范保持稳定，AGENTS.md 随项目演进。

**规范落地链路。**

```text
工程实践总纲（本规范）
 → 项目规则解析（结合 agent.design.md / 选型决策 / 技术栈）
 → AGENTS.md + 模块 AGENTS.md + Skills
 → 项目验证门槛（typecheck / 测试 / 安全清单）+ Eval 验证
 → 生产反馈回流
```

重要规则至少落实为一项工程资产：Schema、Policy、agent.design.md、AGENTS.md、Skill、项目验证门槛、Test / Eval、ADR。

**维护规则。** 保留跨框架规则，框架细节放入 Adapter 或参考资料；新增 Harness、Agent、Memory、Experience 机制前提供 Eval 证据；删除已失效的复杂机制；厂商案例与单次 Bench 仅作参考；生产反馈需经脱敏、授权和验证。

---

## Agent 开发经验（`cases/`）

一线团队 2026 年 Agent 开发实战问题与解决方案。每条含来源、发布日期、问题、原因、解决方法；来源限知名技术团队官方博客或真实社区讨论，聚合/内容平台文章一律不采信。本集合是 `standards/` 的经验佐证——规范定规则，案例给真实证据。

| 文件 | 范围 | 覆盖章节 |
|---|---|---|
| [`01-context-and-tools.md`](cases/01-context-and-tools.md) | §1 Context/Prompt 工程、§2 Tool 设计与调用 | §1、§2 |
| [`02-multi-agent-and-durability.md`](cases/02-multi-agent-and-durability.md) | §3 多 Agent 协作、§4 长任务/耐久执行 | §3、§4 |
| [`03-evals-and-quality.md`](cases/03-evals-and-quality.md) | §5 Eval/验证、§6 生产质量 | §5、§6 |
| [`04-security-and-cost.md`](cases/04-security-and-cost.md) | §7 安全、§8 成本 | §7、§8 |
| [`05-memory-and-collaboration.md`](cases/05-memory-and-collaboration.md) | §9 Memory/经验、§10 协作开发 | §9、§10 |
| [`06-sources-and-mapping.md`](cases/06-sources-and-mapping.md) | §11 未找到可靠来源的方向、§12 关键来源索引、§13 经验到规范的映射 | §11、§12、§13 |

---

## 技术选型参考（`selection/`）

Agent 技术选型与架构设计的通用分析参考，与具体脚手架无关。不推荐框架品牌（会过期），而是沉淀**选型维度与判断框架**：面对新项目该问哪些问题、按什么维度切分工具层、何时上重抽象、何时回归简单循环。本集合是 `standards/` 的选型认知层。

| 文件 | 范围 | 覆盖章节 |
|---|---|---|
| [`01-selection-foundations.md`](selection/01-selection-foundations.md) | §1 工具栈三层抽象、§2 确定性 vs 自主性谱系、§3 复杂度阶梯 | §1、§2、§3 |
| [`02-harness-and-paradigms.md`](selection/02-harness-and-paradigms.md) | §4 Harness 核心组件、§5 规范驱动 vs 对话驱动、§6 「先看再写」原则 | §4、§5、§6 |
| [`03-decision-checklist.md`](selection/03-decision-checklist.md) | §7 选型决策清单、§8 选型反模式 | §7、§8 |
| [`04-framework-comparison.md`](selection/04-framework-comparison.md) | §9 主流框架与技术栈对比、§10 参考来源 | §9、§10 |

---

## 开发实践指南（`practices/`）

日常用 AI（Claude Code / Codex 等 Coding Agent）开发应用与 Agent 项目时怎么协作、怎么搭项目环境、怎么验收。内容提炼自 `cases/` 与 `standards/`，出处以（lessons §x.x / 标准 §x.x）标注。

| 文件 | 范围 | 覆盖章节 |
|---|---|---|
| [`01-mindset-and-collaboration.md`](practices/01-mindset-and-collaboration.md) | §1 心智模型、§2 人机协作范式 | §1、§2 |
| [`02-harness-and-knowledge.md`](practices/02-harness-and-knowledge.md) | §3 项目 Harness 三件事（读对/拦住/接得上）、§4 知识库与上下文管理 | §3、§4 |
| [`03-verification-and-safety.md`](practices/03-verification-and-safety.md) | §5 验证与评审、§6 安全与部署 | §5、§6 |
| [`04-cost-and-rollout.md`](practices/04-cost-and-rollout.md) | §7 成本与效率、§8 常见反模式与对策、§9 团队落地路径、§10 参考来源 | §7、§8、§9、§10 |

此外，`practices/agent-design-workflow.md` 是 `agent-design` 技能的设计流程入口（L2），按需加载。

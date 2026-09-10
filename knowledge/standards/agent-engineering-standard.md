# Agent 工程实践规范

**Agent 项目开发、约束、检查与评测的统一参考**

| 项目 | 内容 |
|---|---|
| 版本 | v1.0（提炼自《智能体工程优秀实践规范 v2.0》） |
| 定位 | 生成项目内置的工程实践总纲，面向 Agent 开发技术人员 |
| 适用范围 | 由 Agent Starter 生成的所有项目及其 AGENTS.md / Skills / Doctor / Eval |
| 技术栈 | 完全中立；具体命令与目录由各项目 AGENTS.md 适配 |
| 目标 | 沉淀稳定、可复用、可执行的 Agent 工程实践，并转化为设计约束、规则、检查与评测 |
| 主要来源 | Anthropic、OpenAI、Google agents-cli、Vercel AI SDK、LangGraph、MCP、OpenTelemetry、OWASP |

---

## 0. 文档定位与使用方式

### 0.1 这份文档是什么

本规范是 Agent 项目的**工程实践总纲**，沉淀跨模型、跨框架、跨项目适用的工程原则与规则。它回答"Agent 工程应该怎么做"，是项目内所有工程约束的**上游来源**。

### 0.2 给谁看

- Agent 开发技术人员：作为完整参考，理解每条规则为何存在、如何执行。
- 编码 Agent（Codex / Claude Code / Cursor 等）：通过 AGENTS.md 和 Skills 间接消费本规范的子集。
- 架构 Review 人员：使用第 18 章检查清单评审。

### 0.3 与 AGENTS.md 的关系

| 文档 | 职责 | 特点 |
|---|---|---|
| 本规范（`knowledge/standards/agent-engineering-standard.md`） | 完整工程实践总纲 | 全、中立、跨框架 |
| 根 `AGENTS.md` | 项目级可执行规则 | 精简、具体、含命令 |
| 目录级 `AGENTS.md` | 模块级规则 | 最近目录、聚焦模块 |
| Skills | 专项可复用工作流 | 渐进加载 |

**AGENTS.md 不复制本规范全文**，只抽取适用于当前项目的规则，并适配实际技术栈与目录。本规范保持稳定，AGENTS.md 随项目演进。

### 0.4 规范落地链路

```text
工程实践总纲（本规范）
 → 项目规则解析（结合 Intent / Plan / 技术栈）
 → AGENTS.md + 模块 AGENTS.md + Skills
 → Doctor 检查 + Eval 验证
 → 生产反馈回流
```

重要规则至少落实为一项工程资产：Schema、Policy、Blueprint、AGENTS.md、Skill、Doctor Check、Test / Eval、ADR。

### 0.5 规范维护

- 保留跨框架规则；框架细节放入 Adapter 或参考资料。
- 新增 Harness、Agent、Memory、Experience 机制前提供 Eval 证据。
- 删除已失效的复杂机制。
- 厂商案例和单次 Bench 仅作为参考。
- 生产反馈需经过脱敏、授权和验证。

---

# 1. Agent 工程核心原则

## 1.1 从最简单、最可验证的实现开始

Agent 系统应遵循以下复杂度阶梯，**优先选择能完成需求的最低复杂度**：

```text
普通确定性函数
 → 单次模型调用
 → 结构化模型调用
 → 单 Agent Tool Loop
 → 程序控制的 Workflow
 → 可恢复的耐久 Workflow
 → 多 Agent
 → 分布式 Agent 系统
```

- **MUST** 优先选择能完成需求的最低复杂度。
- **MUST NOT** 因为项目包含大模型就默认定义为 Agent。
- **MUST NOT** 因为项目包含多个步骤就默认使用多 Agent。
- **SHOULD** 用"结论要不要负责"判断是否值得为确定性付 Harness 工程成本：结论需对特定人/规则/检查负责（高风险 + 复杂分支 + 必须可解释，如审核、合规、权益判定）才上确定性 Harness（阶段化编排 + 全链路留痕 + 提前终止 + 统一网关）；纯生成式任务、必须黑盒的探索性任务、单次轻量调用不上——LLM 是概率性的而业务结论必须确定性，Harness 填的就是这道鸿沟。
- **SHOULD** 模型能力升级后重新校准委派边界：用上一代模型时的"不合理"请求（大项目移植、自主长任务）在更强模型下变得可行——委派方式从指挥任务步骤转向描述最终状态（end state），由系统自行规划路径并回报 trade-offs；不要用旧模型时代的安全感上限封死新能力，但每次放大委派范围须同步放大验证与运营控制（灰度、回滚、运行时开关）。
- **SHOULD** 在普通代码可表达分支、循环和错误处理时，优先采用普通代码。
- **SHOULD** 模型仅负责判断或抽取时，使用结构化模型调用。
- **SHOULD** 在调用 LLM 前设置确定性前置层（语义路由/语义缓存），已知意图的请求不经模型直达结果：
  - **语义路由**：为已知意图准备参考语句集（数百条，可由 LLM 合成），向量化入库；新请求相似度匹配过阈值时直接调用绑定工具，LLM 仅作兜底——省去"框架问 LLM→LLM 决定调工具→结果回填→再调 LLM"的往返。
  - **语义缓存**：请求与响应成对向量化存储；后续相似请求（过阈值）直接返回缓存响应。已知高频问题可预生成问答对提前入库（官方建议问题、FAQ）。
  - **MUST** 相似度阈值用测试数据校准（跑基准取值），不得拍脑袋设定。
  - **SHOULD** 长请求按句分块分别匹配，避免用户把真实意图埋在闲聊中导致漏匹配。
  - **MUST** 语义缓存按用户隔离元数据（如 user ID 过滤），防止相似请求跨用户泄露 PII；缓存 TTL 按语义时效分级（股价分钟级、基本面季度级）。
  - **MUST** 警惕否定敏感：语义相反的问句（"X 是什么"vs"X 不是什么"）在通用 embedding 中距离极近，语义缓存会返回错误答案；高风险场景选用对否定敏感的 embedding 模型或加校验。
  - **MAY** 分类命中的样本回填参考库（自改进），逐步降低对 LLM 兜底的依赖。
- **MAY** 在任务需要自主选择工具、根据中间结果调整路径时使用 Tool Loop。
- **MAY** 在流程需要跨天恢复、人工中断或严格重放时引入耐久执行。

每次选型应记录：选中级别、拒绝的更高级别、理由。

## 1.2 Harness 设计

Harness 是模型与业务环境之间的工程控制层，包含六个域：

| 域 | 职责 | 主要产物 |
|---|---|---|
| Identity | 职责、输入输出、能力、权限和禁止动作 | Agent Contract、Policy、Capability Allowlist |
| Orchestration | 路径、阶段、依赖、并行和人工节点 | Workflow、State Machine、Router |
| Context | 信息选择、隔离、压缩、引用和交接 | Context Policy、Handoff、ArtifactRef |
| Gate | 阶段准入、输出验收和发布门禁 | Validator、Checklist、Grader |
| Recovery | 状态、重试、回退、取消和恢复 | Execution Ledger、Checkpoint、Recovery Policy |
| Evolution | 失败归因、经验、规则和行为资产治理 | Trajectory、Experience、Feedback Patch |

- **MUST** 为每个 Agent 项目明确六个域的责任归属。
- **MUST** 将安全、权限、数据完整性和硬门禁落实到系统。
- **MUST** 记录 Harness 组件解决的问题、成本和验证指标。
- **MUST** 保持组件可替换、可关闭、可做消融测试。
- **MUST NOT** 因采用 Harness 自动引入多 Agent、向量库、耐久 Workflow 或复杂治理平台。
- **SHOULD** 将确定性流程、校验和数据传递放入 Harness。
- **SHOULD** 将语义理解、规划和模糊判断交给模型。
- **SHOULD** 用"下一代 SOTA 模型发布时，这件事是获益还是作废"检验每项 Harness 投入：凡替代或补偿模型推理能力的投入（拆解步骤、提示词编排、流程微调）会被模型下一次升级吸收；凡给模型提供其自身造不出的信息的投入（内部系统真实状态、构建结果、日志、测试反馈）持续增值。
- **SHOULD** 将组织红利视为模型能力与环境能力的乘积而非和：模型是租来的因子，跟着厂商节奏上涨；环境是自有因子，只能自建且只涨不跌。工程投入优先流向环境与验证资产，不参与"让 AI 生成更强"的军备竞赛。

### Harness 内容归属分层

Harness 中的每条规则、知识或指令都应判断归属层级，以决定是否值得团队自建维护：

| 层级 | 内容 | 特征 | 维护策略 |
|---|---|---|---|
| L0 公开知识 | 通用编程规范、框架语法、常识 | 在公开语料中，模型已内化 | 不写入 harness |
| L1 平台能力 | 执行循环、状态管理、subagent 编排、Tool 调度 | 框架与平台原生能力会持续吸收 | 优先依赖平台，不重复实现 |
| L2 组织特有流程 | 内部工具链、专属数据口径、私有 API、目录约定、发布流程 | 不在公开语料，仅组织内可得 | 团队自建的核心杠杆 |
| L3 责任与价值判断 | 验收标准、授权边界、审批门槛、安全底线、业务取舍 | 责任无法转移给技术系统 | 必须由人持有并写入强制规则 |

- **MUST** 在写入 harness 规则前判断其归属层级。
- **MUST NOT** 将 L0 内容写入 AGENTS.md 或 Skill。
- **SHOULD** 将 L1 能力优先交给平台或 Adapter，仅在平台缺失时自建并标注待替换。
- **MUST** 将 L2 与 L3 作为团队自建 harness 的主要投入方向。
- **MUST** 为 L3 规则分配稳定 Rule ID 和机器执行机制。

凡是在补模型能力缺口的规则都会随模型变强而过期；唯一不会过期的是 L2（信息不在公开语料）与 L3（责任无法转移）。

### 自主度上界

Agent 可授予的自主度上限等于能廉价且可靠验证的量。

- **MUST** 在提升 Agent 自主度前先建立可机械判定的验收机制。
- **MUST NOT** 让生成速度持续超过验证速度，导致"代码看似通过但无人真正理解"的认知债积累。
- **SHOULD** 将"验证成本 / 生成成本"作为是否放开自主度的决策依据。

### 先看再写

Agent 在新建组件、工具函数、词条或任何可复用产物前，必须先检索项目中已有的同类实现。

```text
正确：实现前 → 检索同类组件/函数/词条 → 命中则复用 → 未命中再新建
错误：拿到需求 → 直接生成新实现 → 联调时发现与项目惯例冲突
```

- **MUST** 在新建文件、新增词条、新增工具函数前执行项目内检索。
- **MUST** 命中已有实现时优先复用，并在新代码中引用而非复制。
- **SHOULD** 通过 Skill 或 Hook 强制执行「先看再写」流程，而非依赖模型自觉。
- **SHOULD** 检索范围至少覆盖：同目录同类组件、`utils/`/`composables/`/`hooks/`、i18n 词条表、类型定义与枚举。

这条规则是 L2（组织特有流程）得以生效的前提——只有先看见项目里已有什么，组织惯例才能被模型遵循而非被绕过。

## 1.3 Agent 运行必须持续获得环境真实反馈

Agent 不应只基于自身生成内容判断任务完成。

- **MUST** 通过 Tool 结果、数据库状态、测试结果、文件检查或外部 API 获得 ground truth。
- **MUST NOT** 仅因模型声称"已完成"就标记任务完成。
- **MUST** 为完成状态定义确定性验收条件。
- **MUST** 将成功判据写死且可机械判定（测试变绿、产物存在、Diff 为空、命令 exit 0），否则视为未定义。
- **SHOULD** 把达成目标的路径交给模型选择，仅固定目标结果、成功判据和约束。
- **SHOULD** 将验证步骤与执行步骤分开。
- **SHOULD** 对高价值结果使用独立验证器或不同评测路径。
- **SHOULD** 认识到 AI 能力边界由"可验证的反馈"决定：反馈公开、验证可规模化的问题（编译、测试）最先被解决；扩展 Agent 能力边界的手段是把内部环境改造成有反馈、可验证，而非提升提示技巧。
- **SHOULD** 以"能否自主获取验证信号"界定 Agentic：本质不是 AI 主导决策，而是每一步不需要等人判断对错；workflow 退到调度、权限、状态保存、审计和高风险检查的位置。

```text
正确：Agent 修改代码 → 执行目标测试 → 类型检查 → 检查 Diff → 验证输出文件 → 满足完成条件
错误：Agent："代码已经完成并通过测试。" → 任务完成
```

## 1.4 系统约束优先

- **MUST** 将安全、权限、数据完整性、状态转换和预算规则落实到 Schema、Policy 或运行时。
- **MUST** 对重复出现的参数修复、数据恢复和格式纠错分析接口设计原因。
- **MUST** 在重大运行时改造前采集错误类型、触发频率、延迟和成本。
- **SHOULD** 使用声明式数据流消除模型复制精确数据的环节。
- **SHOULD** 将阈值、预算和压缩策略配置化，并通过 Eval 调整。
- **MAY** 使用 Prompt 提示行为偏好；运行时继续执行强制检查。

## 1.5 约束分级与升级

| 级别 | 含义 | 实现位置 |
|---|---|---|
| P0 Invariant | 安全、权限、数据完整性和不可逆动作 | Policy、Schema、Sandbox、Gate |
| P1 Required | 阶段必做、质量门禁和交付要求 | Workflow、Validator、Approval |
| P2 Guidance | 编码偏好、建议路径和经验方法 | AGENTS.md、Skill、Experience |
| Observation | 单次失败、异常和待验证经验 | Incident、Trajectory、Candidate Patch |

- **MUST** 保持 P0 数量少、定义清晰、可自动执行。
- **MUST** 为 P0 和 P1 规则分配稳定 Rule ID。
- **MUST** 记录规则来源、范围、版本、负责人和验证方式。
- **MUST** 经过证据、影响分析和审批后升级规则。
- **MUST** 支持规则降级、暂停和回滚。
- **MUST NOT** 将单次错误直接升级为全局硬规则。
- **MUST NOT** 将全部经验加载到每次模型调用。
- **SHOULD** 将重复失败优先转化为 Schema、Validator、Binding 或 Gate。
- **SHOULD** 使用命中率、失败率、误拦率和单位成功成本评估规则。
- **SHOULD** 在 spec 与技术方案中区分"约束"与"假设"：约束（数据不能出域、接口向后兼容、延迟阈值）长期保存并尽可能自动检查；假设接受被代码取代，不过度沉淀。
- **SHOULD** 约束机制优先做成 Fence（在边界处拦阻并导向正确路径，保留边界内自主性）而非 Sandbox（收窄能力空间）；P0 优先实现为"拒绝 + 导向"的机械执行。
- **SHOULD** 同一规则每次被重新违反即收紧一级：custom → advisory → written law → mechanical enforcement；机械执行是规则生命周期的终点形态。

规则升级流程：`Incident → Root Cause → Candidate Rule → Regression Eval → Shadow/Canary → Policy Approval → Active Rule → Continuous Review`

## 1.6 Loop Engineering

Loop 用于持续发现任务、执行处理、验证结果、保存状态并再次调度：`Discover → Execute → Verify → Persist → Schedule`。

Harness 定义一次运行的能力、约束和恢复机制；Loop 管理多次运行之间的触发、状态、验证和持续改进。

### Loop 准入条件

| 条件 | 要求 |
|---|---|
| 重复性 | 任务会周期性出现或可由稳定事件触发 |
| 可验证性 | 结果可通过自动检查、环境状态或明确人工 Gate 验证 |
| 可操作性 | Agent 具备受控的读取、修改、测试和交付能力 |
| 可恢复性 | 执行可隔离、可重试、可回滚、可停止 |
| 经济性 | 单轮成本和单位成功成本可预算 |
| 可治理性 | 具有 Owner、权限、审计、Kill Switch 和升级路径 |

任一关键条件缺失时，使用单次 Agent、Workflow、辅助分析或人工流程。

### Loop 规则

- **MUST** 在建设前完成准入评估。
- **MUST** 使用持久 Cursor、Watermark 或事件 ID 避免漏处理和重复处理。
- **MUST** 为每个工作项生成稳定的幂等键。
- **MUST** 将执行状态保存在对话之外。
- **MUST** 具备独立验证、有限重试、停止条件和人工升级。
- **MUST** 为写入、部署和发布设置环境与权限边界。
- **MUST** 提供暂停、禁用和紧急终止能力。
- **MUST** 记录每轮触发、处理、验证、交付、成本和最终状态。
- **MUST NOT** 允许 Loop 自动修改自己的 P0 Policy、审批规则或安全边界。
- **MUST NOT** 将"定时调用一个 Prompt"视为完整 Loop。
- **MUST** 停止语义分层确认："模型流结束、工具批次返回、turn 结束、driver activity 结束、Goal 结束"是五件不同的事——工具停了不代表 turn 停了（模型还要看结果），Agent idle 不代表 Goal 完成（外层仍可 followup 唤醒）。每层结束原因独立记录并区分 completed/blocked/max-tokens/aborted/error/interrupted（收到取消可清理与进程崩溃后被恢复层补状态，在副作用审计与重试决策上不可混用）；turn 级结束原因须保留到 turn 结束（中途撞 max-tokens 后续 step 补完，不得记为成功）。
- **SHOULD** 停止/续行机制用消息队列表达而非布尔回调：插件或 hook 要续行就向下一 step/turn 队列写入消息，核心循环在 stopping hook 后重读收件箱决定停或续——多插件的续行语义不需要合并布尔值，且每次续行意图留下可审计记录。

## 1.7 Runtime Protocol

Runtime Protocol 定义外部系统可依赖的对象、操作和状态迁移。Runtime 负责内部调度、持久化、权限和执行。框架通过 Adapter 实现协议。

### 稳定对象

| 对象 | 职责 |
|---|---|
| Agent | 能力提供者、版本、权限和调用入口 |
| Thread | 长期上下文、参与者和会话边界 |
| Run | 一次执行、预算、状态和取消边界 |
| Step | 模型、Tool、Gate、Handoff、子任务等可观测执行单元 |
| Message | 用户、Agent、Tool 和系统交换的内容 |
| Event | 状态、消息、Tool、产物、错误和审批增量 |
| Artifact | 文件、报告、结构化输出、代码 Diff 和证据包 |
| Checkpoint | 可恢复状态、版本和待处理动作 |
| Interrupt | 等待输入、授权或审批的持久状态 |
| Workspace | 文件、代码、浏览器和执行环境边界 |
| Trace | Run 与 Step 的因果、成本和审计关系 |
| Error | 失败类别、可恢复性和建议动作 |

### 协议规则

- **MUST** 让 Thread、Run、Step、Event、Artifact 和 Checkpoint 具有稳定 ID。
- **MUST** 让每个 Event、Artifact、Error 和 Approval 归属具体 Run。
- **MUST** 让 Step 表达模型、Tool、Gate、Handoff 和子任务。
- **MUST** 将等待输入和授权表达为 Run 状态。
- **MUST** 将 Workspace 和 Artifact 作为一等对象管理。
- **MUST** 内建 `trace_id`、`event_id`、`run_id` 和 `step_id`。
- **MUST** 对协议 Schema、状态机和错误码版本化。
- **MUST** 允许不同传输绑定实现同一语义。
- **MUST NOT** 在公共协议暴露框架内部 Node、Reducer 或模型私有推理。

## 1.8 规则编译与运行时拦截

自然语言规则用于说明意图。强制规则通过 Policy、Schema、Hook、Linter、Gate 或 CI 执行。

| 规则类型 | 主要实现 |
|---|---|
| 安全、权限、不可逆动作 | Policy、Sandbox、Approval、Hook |
| 状态转换、流程门禁 | State Machine、Gate |
| 数据结构、输入输出 | Schema、Type、Validator |
| 代码和配置规范 | Linter、Static Check、CI |
| 操作流程 | Workflow、Skill、Doctor Check |
| 低风险偏好 | AGENTS.md、Prompt、Experience |

- **MUST** 将 P0 和 P1 规则映射到机器执行机制。
- **MUST** 对安全、权限、状态写入和不可逆动作采用 Fail Closed。
- **MUST** 保存规则命中、阻断、放行和审批证据。
- **MUST** 对 Hook 设置超时、顺序、幂等和错误策略。
- **MUST** 让 Hook 独立于模型上下文和 Compaction。
- **MUST NOT** 仅扩充 `AGENTS.md` 修复重复出现的硬性违规。
- **SHOULD** 将高频违规转化为 Schema、Binding、Linter 或 Gate。

### Hook 按触发时机分类

Hook 应按触发时机归类，便于排序、去重和失效排查：

| 触发时机 | 典型用途 |
|---|---|
| before-message | 推荐工作流、注入项目上下文、预检意图 |
| after-message | 结果归档、状态更新、经验采集 |
| after-edit | 自动格式化、Linter、类型检查、「先看再写」检索 |
| skill-loaded | 技能可用性校验、资源预算核对 |

- **SHOULD** 为每个 Hook 声明触发时机、优先级、超时和失败策略。
- **SHOULD** 同一时机的多个 Hook 定义稳定执行顺序。

## 1.9 规则文件的注意力衰减与维护

规则指令随上下文增长会发生注意力衰减（Lost in the Middle 效应）：模型对上下文中段指令的注意力低于首尾。规则文件的效力因此是会话时长的减函数，需要显式维护策略。

- **MUST** 按 feature 切分会话，保持上下文短；禁止在单个长会话中连续处理多个不相关 feature。
- **SHOULD** 监测输出质量下降信号（风格违规重现、重复纠正出现），触发规则文件确定性重注入——重新加载原始规则文本，而非让模型重新解释。
- **MUST** Agent 可提案修改 AGENTS.md / Skill 规则，但修改须人工审查后生效；规则演进进入版本管理。
- **SHOULD** 会话中重复出现的纠正即时沉淀为规则候选（同一纠正出现约三次即写入），避免跨会话口头重复。

---

# 2. Context Engineering

## 2.1 上下文组成

现代 Agent 的上下文包括：System Instructions、Tools、User Input、Session History、Business State、Retrieved Data、Files、Memory、Intermediate Results、Plan、Budget、Environment Feedback。

- **MUST** 将 Prompt 管理升级为 Context 管理。
- **MUST** 明确每一类上下文的来源、信任等级、保留时间和裁剪规则。
- **MUST NOT** 将所有历史和工具结果无选择地持续追加。
- **SHOULD** 使用最小、高信号上下文。
- **MAY** 对长历史进行摘要或压缩，但摘要必须可追溯到原始记录。

## 2.2 上下文信任分级

| 等级 | 示例 | 规则 |
|---|---|---|
| SYSTEM | 核心安全规则 | 最高优先级，不被下级覆盖 |
| ORGANIZATION | 组织策略 | 仅管理员发布 |
| APPLICATION | 项目 Prompt、业务规则 | 版本化 |
| USER | 用户输入 | 不能改变系统权限 |
| RETRIEVED | 知识库数据 | 需来源和时间 |
| EXTERNAL_UNTRUSTED | 网页、邮件、Tool 返回 | 按不可信数据处理 |

- **MUST** 重要结论绑定运行现场：结论本身之外须同时记录 claim / source / revision / scope / observed_at / verification——"某处已存在"（本地代码含修复）不等于"当前已生效"（实际消费的仍是旧版本），判断以运行现场实际消费的 revision/receipt 为准，不以代码现场代替运行现场。

## 2.3 Context Budget

每个 Agent 应定义上下文预算：最大输入 Token、输出/工具定义/系统指令预留、历史策略、单结果最大 Token、检索文档上限。

- **MUST** 预留输出和工具定义预算。
- **MUST** 记录被裁剪的上下文类型和数量。
- **SHOULD** 优先丢弃低相关、重复和可重新获取的内容。
- **SHOULD** 避免将超大 JSON、HTML 或日志原样放入上下文。

## 2.4 上下文压缩

压缩前后都应可审计。约束：

- 用摘要替代不可恢复的关键业务事实。
- 将未经确认的模型推断写入长期事实。
- 把安全策略压缩成模糊描述。
- 压缩后丢失 Tool 副作用、审批或错误状态。

## 2.5 上下文数据生命周期

```text
Tool 结果 → 分类 → Inline 或 ArtifactRef → 有界摘要 → 索引 → 按需检查 → 过期或归档
```

- **MUST** 使用 ArtifactRef、State 或参数绑定传递精确数据。
- **MUST** 保留来源、Hash、Schema、权限和过期策略。
- **MUST** 标记摘要、截断和不完整结果。
- **MUST NOT** 让模型复制 UUID、主键、完整数组或大段结构化数据。
- **SHOULD** 提供 `outline`、`search`、`context`、`head`、`tail` 等局部检查接口。
- **MUST** 对超过阈值（如 8000 字符 / 10 个元素数组）的工具结果强制外置存储——LLM 搬运大数组时会"无意识摘要"只传 3-5 个"代表性"元素，强制外置从根本消除这一数据丢失模式。
- **MUST** Agent 表现随步骤数劣化时先诊断上下文质量（统计噪音占比），再考虑换更大模型——实测换模型两周指标不动，上下文管理一周提升 40%；"物理容量"不等于"有效容量"。
- **MUST** 跨步骤数据传递走运行时参数绑定注入，禁止让模型充当"数据搬运工"（模型搬运 UUID 会截断、混淆、幻觉）。

## 2.6 单一表示与结构化 Compaction

同一来源在一次模型调用中只保留一种表示。禁止同一数据同时出现：完整结果+摘要、摘要+预览、完整结果+Assistant 复述、多个不同截断版本。

Compaction 生成结构化交接，保留：用户目标、当前计划、已完成步骤、已验证事实、放弃路径、阻塞、待审批、Artifact 引用、剩余预算。

- **MUST** 保留用户目标、具体 ID、已完成步骤、失败路径、阻塞、审批和引用。
- **MUST** 保持 Tool Call 与 Tool Result 的协议配对。
- **MUST NOT** 使用模型改写规范字段、精确值、Hash、ID 或匹配前缀。
- **MUST** 在 Prompt 组装阶段做单一表示的编译时检查——检测到同一数据以多种形态进入待组装 prompt 时拒绝组装并告警，而非运行后祈祷；摘要/预览类字段必须用原始文本 substring 生成，不经任何模型改写（模型重写会改变字段名与前缀，导致下游前缀匹配恢复失败）。

## 2.7 Prompt 预算预检与编译

每次模型调用前执行预算预检：`模型窗口 - 输出预留 - Tool定义 - 系统与组织规则 - 当前任务目标 = 可分配上下文预算`。

超预算时按固定优先级收缩：调试预览 → 可重新获取的完整数据 → 低相关摘要 → 非直接依赖 → Transcript → Working Memory Insights。任务目标、安全规则、审批状态和当前步骤保留到最后。

- **MUST** 使用模型相关 Tokenizer 或保守估算。
- **MUST** 在步骤开始和最终输出前检查预算。
- **MUST** 在降级后仍超限时返回 `CONTEXT_BUDGET_EXCEEDED`。
- **SHOULD** 用 Prompt Compiler 组装版本化结构块。

推荐 Prompt 结构：

```text
Stable Prefix: Platform Rules → Organization Policy → Project Rules → Agent Contract → Stable Tool Definitions → Output Contract
Dynamic Suffix: Current Objective → Phase Context → Selected Artifacts → Working State → Latest User Input
```

## 2.8 阶段上下文包与交接产物

每个阶段接收自包含的上下文包（目标、验收条件、输入引用、已验证事实、约束、决策、开放问题、输出 Schema、允许能力、预算），阶段结束后输出 Handoff Artifact（状态、决策、输出引用、验证结果、放弃路径、开放问题、下一阶段）。

- **MUST** 通过结构化产物交接关键状态。
- **MUST** 让下游依赖公开 Contract，避免依赖上游对话历史。
- **MUST** 将临时讨论、草稿和已放弃方案与已验证结论分开。
- **SHOULD** 按阶段渐进加载规则、Skills 和资料。

## 2.9 分层上下文加载

| 层 | 内容 | 加载策略 |
|---|---|---|
| Core | 角色、P0/P1 摘要、当前目标、停止条件 | 每次 Run |
| Scoped Rules | 当前目录、组件和技术栈规则 | 按项目范围 |
| Phase Context | 当前阶段 Contract、Checklist、输入产物 | 进入阶段 |
| On-demand Reference | 专项规范、示例、历史证据 | 触发时 |
| Artifact Data | 大结果、完整日志、文件和数据集 | 按需局部读取 |

- **MUST** 按读取时机和任务范围加载规则。
- **MUST** 为强制阶段资料维护 Required Read 清单。
- **MUST** 在阶段结束后释放无后续依赖的上下文。
- **MUST NOT** 将全流程手册长期常驻模型上下文。

## 2.10 上下文成本与稳定前缀

- **MUST** 按 Agent、Run、Step、Wave 和上下文来源统计 Token。
- **MUST** 区分输入、输出、缓存读取、Tool Schema 和 Tool Result。
- **MUST** 识别重复内容、无效加载和未被使用的上下文。
- **MUST** 将动态内容放在稳定指令之后。
- **MUST NOT** 为缓存命中复制无关内容。
- **SHOULD** 保持系统规则、项目规则、Agent 定义和 Tool Schema 的顺序稳定。

## 2.11 上游一次采集与下游复用

同一事实源在链路上采集一次：`Source Connector → Raw Artifact → Structured Projection → Verified Summary → Downstream Reference`。

- **MUST** 保存原始数据为受控 Artifact。
- **MUST** 将下游所需字段转换为结构化 Projection。
- **MUST NOT** 让多个 Agent 分别获取同一外部数据。
- **MUST NOT** 将完整外部 Payload 注入长生命周期协调 Agent。

## 2.12 索引优先检索

知识、代码和历史产物先查询轻量索引，再读取正文：`Query → Index Search → Candidate Ranking → Top-K Metadata → Selected Content Read → Fallback Search`。

- **MUST** 将索引与源文件版本关联。
- **MUST** 在索引缺失、过期或低置信时回退到源搜索。
- **MUST** 只读取排序后的少量候选正文。
- **SHOULD** 对大型代码库使用 AST、符号索引、依赖图或代码图谱缩小文件范围。
- **SHOULD** 知识载体选型以更新路径长短为判据：知识、文档、测试和索引进同一个 MR 一起评审、合并后共享同一版本（文件 + Git 优先）；RAG/图谱作检索补充不作主知识源——它们在代码变化后要走同步、切片、萃取、索引/图谱发布的长链路，更新延迟即知识失效窗口。
- **SHOULD** 知识库只存"代码之外影响业务和技术判断的信息"（业务规则与口径、跨系统链路、新旧切换、废弃状态、运行态拓扑），不复制代码可表达的内容——逐方法解释的文档生成快失效也快，会变成与代码竞争的实现说明；每份知识文件带 Metadata（至少 status/version/source），Agent 读正文前先判适用性，事实来源（source）指向原始材料而非派生结论。

---

# 3. Agent 与 Workflow 选择范式

## 3.1 Tool Loop 适用场景

适用：下一步依赖中间结果；工具选择不能完全预先确定；搜索、研究、分析需要多轮探索；模型需要判断何时停止。

必须定义：maxSteps、maxModelCalls、maxToolCalls、maxDurationMs、stopConditions（required_output_ready / no_progress / budget_exhausted）。

多数普通工具型 Agent 从 ToolLoopAgent 开始；明确、可预测的流程则使用普通代码和结构化 Workflow。

- **MUST NOT** 把团队自身无法评估的判断外包给 Agent。若人无法识别正确答案，Agent 同样无法；Agent 的自主度上限等于团队廉价且可靠地验证结果的能力。无法验证的任务必须先建立验收能力（golden set、领域专家抽检、可机械判定的判据），再交给 Agent。

## 3.2 显式 Workflow 适用场景

适用：有固定业务状态机；分支和终止条件可编码；必须保证执行顺序；需要事务、审批和审计；失败补偿逻辑明确。

- **MUST** 将确定性业务状态放在 Workflow 层，不隐藏在 Prompt 中。
- **MUST** 将数据库事务与外部模型/API 调用分开。
- **MUST** 对可重试步骤定义幂等键。
- **SHOULD** 让模型负责模糊判断，让代码负责状态变更。

## 3.3 多 Agent 准入条件

只有满足以下至少一项才考虑：子任务可明显并行；上下文应隔离；工具或权限必须隔离；由独立团队或服务部署；单 Agent 上下文无法承载；专业评审需要独立视角并有可测收益。

- **MUST** 说明为什么多个 Tool 不够。
- **MUST** 定义协调者、所有权、共享状态和冲突处理。
- **MUST** 计算额外 Token、延迟和失败面。
- **MUST NOT** 让多个 Agent 无约束修改同一业务对象或文件。
- **SHOULD** 多 Agent 优先使用 Manager-as-tools。
- **MUST** 方案中显式声明协作机制（任务共识、Agent 间通信、进度跟踪、复盘沉淀）而非只声明角色分工——消融实验显示去掉协作机制后 20 个 Agent 的团队与 1 个 Agent 表现完全一致：纯堆 Agent 数量不产生价值，多 Agent 的价值全部来自协作机制本身；通信基础设施（能互相说话）不等于协作语义（怎么达成共识）。

## 3.3.1 子 Agent 运行时协作契约

多 Agent 准入成立后，运行时协作遵守事件驱动契约，防止后台子 Agent 退化为黑盒：

- **MUST** 按任务需要收窄子 Agent 能力开关（写入 / MCP / 再派发子 Agent 的工具分级授权），不默认全量。
- **MUST** 采用反应式唤醒替代轮询：子 Agent 消息直接投递进目标会话上下文并触发唤醒，编排者不为等待进度消耗 Token。
- **MUST** 子 Agent 上报使用结构化载荷（JSON 或 PROGRESS/ERROR/ABORT 前缀分类），禁止自由文本；且只在里程碑边界上报，不得逐行刷屏。
- **MUST** 为消息循环定义显式终止条件（COMPLETE/TERMINATE 标志），防止无界乒乓对话。
- **MUST** 将父会话 ID 注入子 Agent 初始任务（或共享上下文文件），子 Agent 不得猜测消息接收者。
- **MUST** 提供两级断路：协作式软中止（编排者广播 abort）与硬终止（强制 kill 活跃子 Agent）；单点致命失败时立即止损，不让其余子 Agent 对注定失败的任务继续消耗预算。
- **MUST** 里程碑遥测进入 Trace，支持事后回放多 Agent 执行过程。

## 3.4 声明式数据流与步骤控制

步骤间数据使用明确绑定（fromStep + path）。步骤状态使用结构化控制：CONTINUE / COMPLETE / SKIP / NEED_INPUT / FAILED。

- **MUST** 由运行时解析字段路径、类型和引用。
- **MUST** 在执行前验证绑定目标和 Schema。
- **MUST NOT** 让模型重新输入已有的精确数据。

## 3.4.1 长期指令退出条件与 Harness 删减

- **MUST** 每条长期常驻指令（system prompt 规则、CLAUDE.md 条目、Skill 约束）声明退出条件：适用模型版本范围、触发场景、复审周期——累积规则会互相冲突，迫使模型先花推理预算解释约束再处理任务（Claude Code 为新模型删掉 80%+ system prompt，eval 无可测量损失）。
- **MUST** 模型升级时审视既有防御性脚手架的触发率，删减已无必要的修复管道——工具有半衰期，系统应能从模型进步中"免费"获益，而非背负对抗旧模型的历史包袱。
- **MUST** 副作用型 Tool 执行区分 dispatch 前、dispatch 后、结果确认后三个阶段；进程在结果确认前崩溃时对结果不确定的动作执行 reconcile 对账，禁止盲目重放（可能造成重复扣款/发信/建资源）。
- **MUST** Agent 间责任移交（handoff）使用结构化移交包：目标范围、已验证事实、未决假设、当前 checkpoint、待确认副作用、审批状态、停止条件、结果交还对象——只传自然语言摘要是把上下文丢失伪装成组织分工。

## 3.5 并行与嵌套执行

并行任务需满足：输入已确定、无数据依赖、无共享可变写入、输出有独立命名空间、失败策略明确。

- **MUST** 使用依赖图判断可并行节点。
- **MUST** 设置模型、Tool 和外部服务并发预算。
- **MUST** 为嵌套 Agent、Workflow、Skill 设置深度、链路长度和环检测。

## 3.6 机器、Agent 与人工分工

| 执行者 | 适合任务 |
|---|---|
| Machine | 校验、转换、查询、版本检查、依赖检查、路由和状态更新 |
| Agent | 意图理解、模糊判断、规划、语义生成和异常分析 |
| Human | 高风险审批、业务取舍、主观验收和规则发布 |

- **MUST** 将有确定规则的步骤实现为代码或 Tool。
- **MUST** 批量收集需要用户确认的信息，减少重复中断。
- **MUST** 对高风险决策保留人工入口。

## 3.7 链路模式与单点模式

同一能力可支持链路模式（Workflow 编排多阶段）和单点模式（直接执行一个自包含能力）。要求：二者使用相同实现、Policy、Tool 和 Eval；单点模式不能绕过适用的安全和质量门禁；单点模式可作为能力级调试和回归测试入口。

## 3.8 任务与修改影响分级

| 级别 | 示例 | 处理 |
|---|---|---|
| Minor | 命名、格式、文案、局部配置 | 最小确定性流程 |
| Standard | 业务逻辑、字段、Tool 或 Prompt 调整 | 目标能力 + 相关 Gate |
| Major | 架构、数据模型、权限、跨模块流程 | 完整 Plan、ADR、全量 Gate 和人工审批 |

- **MUST** 依据影响范围、可逆性、风险和依赖选择流程。
- **MUST NOT** 让轻量任务默认执行全部链路。
- **MUST NOT** 让复杂任务因输入简短自动降级。

## 3.9 协调者与专家模式

仅在多 Agent 准入条件成立时使用。协调者负责任务分解、Specialist 选择、状态/预算/权限、Gate/恢复/汇总；Specialist 负责单一领域产出、自包含输入输出、独立工作区、领域级测试。

- **MUST** 防止协调者重复 Specialist 的核心工作。
- **MUST** 限制 Specialist 的 Tool、数据和写入范围。
- **MUST NOT** 为角色拆分本身创建多个 Agent。
- **SHOULD** 协调者与执行者保持能力差：协调者配更强模型、更长上下文和更全局的历史访问权，并具备兜底能力（执行者卡住时能亲自接手完成），不是同质 Agent 换 prompt 充当协调者——与子 Agent 默认降档（§7.9）配套：执行者走成本最优，协调者走能力上限，兜底责任与分解评估质量都压在协调者一侧。

## 3.10 Runtime Loop 与编排协议

分开选择 Loop 承载方式（Code/Graph/Managed/Event-driven Runtime）和编排协议（Tool Loop/Plan-and-Execute/Manager-Worker/Handoff/Conversation Coordination）。

- **MUST** 将 Plan、Handoff、Subrun 和进度映射为 Step、Event 或 Artifact。
- **MUST** 保持 Tool、State、Event 和 Artifact Contract 独立于执行框架。

## 3.11 控制平面、计算平面与协作平面

| 平面 | 职责 |
|---|---|
| Control Plane | 状态机、路由、Gate、预算、审批、恢复 |
| Compute Plane | 模型调用、Tool、确定性计算、单阶段并行 |
| Collaboration Plane | 独立参与者协作、消息、Handoff 和所有权 |

- **MUST** 让控制平面拥有流程状态和阻断权。
- **MUST** 让计算平面通过 Contract 接收任务和返回 Artifact。
- **MUST** 防止松散消息通知替代阻断 Gate。

## 3.12 规模路由与 Agent 拆分成本

任务规模由影响范围、并行机会、专业边界、持续时间和风险决定。

- **MUST** 在创建多个 Agent 前评估拆分收益和固定开销。
- **MUST** 为小任务提供单 Agent 或确定性 Fast Path。
- **MUST** 比较单 Agent 与多 Agent 的质量、耗时和单位成功成本。
- **MUST NOT** 使用固定 Agent 数量作为通用设计规则。
- **MUST** 角色拆分以差异化工程责任为判据：多个 Agent 用相似上下文做相似判断、最后相互复述结论，是纯成本；拆分只在角色对应稳定工程责任且对"完成"的判断标准不同时有价值。
- **MUST** 协作消息语义分级且状态变更只认结构化结果：Comment 沉淀共享事实不触发执行、Mention 明确投递、Handoff 携带成果与证据；只有结构化工作结果（完成/阻塞/失败）能改变任务状态，普通消息与单次模型调用的结束不改变 Job 状态——执行账本中的可复查事实（Diff、截图、测试结果）与阶段判断分开记录，Handoff 陈述只能作为线索不能作为事实。

## 3.13 无依赖调用并行化

可并行条件：输入已确定、无前后数据依赖、无共享可变写入、权限与预算允许、失败可独立处理。

- **MUST** 构建调用依赖图。
- **MUST** 将无依赖的 Connector、Tool、Subrun 和测试任务批量并行。
- **MUST** 设置模型、外部 API、浏览器、数据库和 CI 并发上限。
- **MUST NOT** 为并行化重复获取相同上下文。

## 3.14 同构批量调用的代码编排

分离不确定性与确定性：模型负责理解、判断、规划和生成编排逻辑；循环、过滤、重试、缓存、分页、批量执行由程序完成。典型场景：搜索后读取前 N 个文件、对 N 条记录逐条执行同构 Tool 调用——逐个发起 LLM 往返时，延迟、Token 消耗与跑偏概率随 N 线性放大（N 次往返可压缩为 1 次）。

- **MUST NOT** 让模型逐个发起同构 Tool 调用、每步等待模型决定「继续」。
- **MUST** 由代码编排层执行循环：模型生成编排逻辑（或选择编排模板），程序完成 search → read → filter → batch → retry 后一次性回报结果。
- **MUST** 编排执行进入 Trace，支持回放审计。
- **SHOULD** 优先提供编排模板或 SDK；模型自由生成编排代码时按沙箱执行并施加权限约束。
- **SHOULD** 调研/审查类任务（找 bug、性能排查、方案研究）采用"扇出采集 + 对抗性审查 + 确定性循环"结构：扇出阶段广泛采集候选（宁多勿漏），对抗阶段用多视角独立审查逐个评估、过滤误报，循环阶段由确定性 for-loop 保证同一审查技术应用到每个候选——人对 workflow 的信任来自一致性而非单次质量；人只审过滤后的问题集，不直接消费扇出原始产出（Claude Code 团队模式：扇出产生人消费不了的信息量，必须过滤回来）。
- 判据：同构批量任务的模型往返次数应接近 1 而非 N。

---

# 4. Tool 设计规范

## 4.1 Tool 是模型与确定性系统的契约

- **MUST** 面向 Agent 的任务语义设计 Tool，而非一对一包装现有 API。
- **MUST** 使用清晰、无歧义的名称。
- **MUST** 使用严格输入 Schema。
- **MUST** 对输出进行裁剪和结构化。
- **MUST** 标注只读、写入、破坏性、幂等和外部访问属性。
- **MUST** 定义超时、错误分类和重试策略。
- **MUST** 对有副作用 Tool 使用幂等键或业务防重机制。
- **MUST NOT** 在 Tool 输出中泄露 Secret。
- **SHOULD** 减少功能重叠的 Tool。

## 4.2 标准 Tool Contract

每个 Tool 应声明：name、version、description、inputSchema、outputSchema、annotations（readOnly/destructive/idempotent/openWorld）、riskLevel、approvalPolicy、timeoutMs、retryPolicy、resultPolicy。

## 4.3 Tool 描述规则

好描述应包括：做什么、何时使用、何时不要使用、输入字段含义、返回内容、副作用、失败后的可恢复方式。

```text
错误：search: 搜索数据
更好：supplier.search: Search internal supplier records by normalized product requirements.
     Use this before public web discovery. Returns at most 20 candidates with IDs,
     matched fields, and source evidence. This tool is read-only and does not contact suppliers.
```

## 4.4 Tool 输出规则

- **MUST** 对输出执行裁剪和结构化。
- **MUST NOT** 返回原始大 HTML、完整数据库行或无界日志。
- **SHOULD** 对长日志、测试输出、进程列表和 Diff 执行确定性压缩。
- **SHOULD** 保留退出码、失败位置、关键错误、统计和 ArtifactRef。

## 4.5 Tool Evals

每个 Tool 至少测试：正确选择 Tool、不应调用时不调用、参数正确率、对错误结果的恢复、Token 效率、描述变更后的回归、危险参数是否被 Guardrail 拒绝、重试是否造成重复副作用。

## 4.6 Capability 与 Action Space

Skill、内置 Tool、业务 Tool、Workflow、MCP Tool 和 Tool Pack 统一建模为 Capability。解析顺序：Agent 静态能力 + Blueprint 默认能力 + 当前任务所需能力 → 权限过滤 → 风险过滤 → ToolDefinition。

- **MUST** 在每一步只暴露当前可用且相关的能力。
- **MUST** 通过权限和风险策略过滤 Capability。
- **SHOULD** 监控 Tool 数量对选择准确率和 Token 的影响。

## 4.7 Connector 与环境操作

Connector 连接日志、Trace、代码仓库、CI/CD、通知、云资源和业务系统。应声明：authPolicy、readScopes、writeScopes、correlationKeys、rateLimit、timeoutMs、retryPolicy、redactionPolicy、healthCheck。

- **MUST** 分离读取和写入权限。
- **MUST** 使用 `trace_id`、`request_id`、`deployment_id`、`commit_sha` 关联跨系统证据。
- **MUST** 对写入和部署操作支持 Dry Run、审批和审计。
- **MUST** 将凭证保存在专用 Secret Store。

## 4.8 Error Contract

错误分为可供 Agent 决策的业务错误和必须由 Runtime 处理的系统错误。错误应声明：code、category、message、retryable、safeForModel、suggestedActions。

- **MUST** 将预期 Tool 失败返回为结构化错误数据。
- **MUST** 对认证失败、权限失败、数据损坏、Runtime 缺陷和未知系统异常终止或中断执行。
- **MUST NOT** 把所有 Exception 直接注入模型上下文。
- **MUST NOT** 让模型决定权限、数据完整性或系统故障的处理边界。

## 4.9 确定性执行与 CLI / MCP 选择

| 场景 | 优先方式 |
|---|---|
| 批量、可重跑、CI、编译、测试、迁移、健康检查 | CLI / Script |
| 动态发现远程能力、交互式资源访问、跨客户端工具生态 | MCP |
| 稳定业务操作和副作用 | Typed Service Tool |
| 大数据采集和转换 | Connector + Artifact |

- **MUST** 将编译、测试、环境启动、健康检查和数据迁移封装为脚本或 CLI。
- **MUST** 让 Agent 传结构化参数，避免拼接凭证和复杂 Shell。
- **MUST** 使用 MCP 时只暴露当前 Agent 需要的 Server 和 Tool。
- **MUST NOT** 让模型逐步控制可由脚本一次执行的确定性流程。

## 4.10 Tool Schema 与结果成本

Tool 成本包括：Schema 常驻 Token、选择推理、参数构造、执行、结果注入、后续历史重复携带。

- **MUST** 按 Agent 生成 Tool Allowlist。
- **MUST** 统计每个 Tool Schema 和结果的 Token。
- **MUST** 对长日志、测试输出、进程列表和 Diff 执行确定性压缩。
- **SHOULD** 对大量相似 Tool 使用 Tool Pack、Router 或短列表。

---

# 5. State、Session、Memory 与耐久执行

## 5.1 区分四类状态

| 概念 | 范围 | 用途 |
|---|---|---|
| Run State | 一次运行 | 当前步骤、预算、中断 |
| Session History | 一个会话 | 消息连续性 |
| Workflow Checkpoint | 一个流程实例 | 恢复、重放、人工暂停 |
| Long-term Memory | 跨会话 | 用户偏好、已验证事实 |

- **MUST NOT** 将四类状态混成一个 messages 数组。
- **MUST NOT** 将业务事实只保存在框架内存中。
- **MUST** 为长期 Memory 定义来源、有效期和删除机制。
- **MUST** 区分模型推断与已验证事实。
- **MUST** Memory 按恢复语义分类管理：工作状态可被新 checkpoint 取代、事件历史追加写、领域知识保留版本与来源、偏好需衰减纠错、凭据留在专用隐私边界——摘要适合压缩上下文，审计依赖原始记录；能无状态完成的任务保持短寿命，持久化本身引入隐私、腐化和迁移成本。
- **MUST** 长期记忆定位为增强能力而非依赖路径：检索失败或超时返回空结果并记 warning，主对话流程继续执行，不阻塞主链路（并行加载时与短期上下文各自独立加载、join 汇合）。
- **SHOULD** 长期记忆注入按 scope 分配 Token 预算（如画像类上限 60%、剩余给任务经验），截断按行进行以保留单条记忆完整语义，避免截断前的估算带入剩余预算计算。
- **SHOULD** 记忆库结构约束导航复杂度：目录层级封顶（如两级，深层级让 AI 定位一条记忆的选择数随深度翻倍；内容增长用文件名前缀消化，不新增层级——想加层级往往是在逃避"这条记忆属于哪类"的判断）；索引先行（总索引 title+description 一行一条 + 目录 README 声明装什么/不装什么，AI 先扫地图再按需读正文）；跨目录主题用 tags 补维度而非目录反复细分；不确定的信息显式标注为"待补齐"而非缺失。
- **MUST** 长任务 Goal 写明 outcome、constraints、verification（Goal 是可执行契约，不是"一直做下去直到完成"）；完成判定由测试、可测指标、Diff、截图、外部状态或人工验收等外部证据确认，**MUST NOT** 以模型自述"已完成"作为完成依据。
- **MUST** 为持续运行定义明确终态（blocked / needs-input / cancelled / budget-exhausted）：verifier 不可达、预算耗尽、权限不足或外部依赖永久失败时进入终态——缺少停止证明和资源上限的 Goal 本质上是无限循环。
- **MUST** 长任务可恢复性以换 Agent 为检验：假设每个阶段结束后换成全新 Agent，仅凭项目内状态、文档与真实工件（非会话历史）能继续推进——Workflow 各阶段规定交接合同（输入来自已确认事实或上一步产物、输出落到文档/代码现场/报告），不能只靠对话记忆；恢复任务（凭持久化信息重建上下文）优先于恢复会话（找回上一段对话）。
- **SHOULD** Harness 组件含对当前模型能力的隐含判断，不是永久资产：新增（Add）与删减（Thin）都由真实任务的运行证据决定（重复失败、缺证据、无法恢复才加；长期无价值、重复实现、频繁误伤则删），只会增加不会减少的 Harness 最终变成新负担。

## 5.2 Checkpoint 与 Store 分离

区分线程级、短期、流程状态的 Checkpointer 与跨线程、长期、应用定义数据的 Store。

## 5.3 可恢复 Workflow 规则

耐久运行通过重放到 Checkpoint 恢复。

- **MUST** 将非确定性操作和副作用封装为独立 Task。
- **MUST** 保证 Task 输入输出可序列化。
- **MUST** 让可重试 Task 幂等。
- **MUST** 假设中断前的代码可能再次执行。
- **MUST NOT** 在中断点之前执行无保护的外部副作用。

## 5.4 执行账本与流式传输

执行账本保存：Run 状态、Step 状态、Event 序列、Checkpoint、Tool 副作用记录、Approval、ArtifactRef、Budget。SSE/WebSocket/轮询负责传输事件，账本负责恢复、审计和状态查询。

- **MUST** 将执行与客户端连接分离。
- **MUST** 为事件分配单调序号和幂等标识。
- **MUST** 支持客户端按 Cursor 续传事件。
- **MUST** 在安全边界写入 Checkpoint。
- **MUST** 将缓存视为性能层，将持久化账本作为恢复依据。

## 5.5 Working Memory 与行为资产

运行内信息：Pinned Objective、State、Insights（已验证发现）、Transcript。

跨运行行为资产：Policy（人工审核）、Strategy（Eval/Shadow/回滚）、Action Chain（高频动作序列，按场景召回和版本化）。

- **MUST** 标记 Insights 的来源和验证状态。
- **MUST** 将 Policy、Strategy 和 Action Chain 保存为结构化资产。
- **MUST NOT** 将敏感属性推断写入个人记忆。
- **SHOULD** 会话记忆的写入走异步抽取器（会话后离线抽取），不放在对话热路径上同步生成；记忆写路径带验证（抽取结果过校验才入库），防止对话中的临时表述直接污染长期记忆（Anthropic 商务 Agent 记忆异步抽取模式）。

## 5.6 故障分级与恢复策略

| 级别 | 处理 | 示例 |
|---|---|---|
| RETRY | 当前步骤有界重试 | 超时、限流、暂时网络错误 |
| FALLBACK | 切换替代 Tool/模型/策略 | Provider 异常、检索源不可用 |
| ROLLBACK | 回到最近稳定 Checkpoint | Gate 失败、产物不合格 |
| NEED_INPUT | 暂停并请求用户信息 | 关键需求缺失、业务取舍 |
| ABORT | 终止并保留证据 | 权限失败、数据完整性风险 |

- **MUST** 为每个错误码定义默认恢复级别。
- **MUST** 限制重试次数、总时长和成本。
- **MUST** 避免对确定性错误执行原样重试。
- **MUST** 在回退前确认副作用状态。
- **SHOULD** 把超时归类为"未知"而非"失败"：服务端可能已成功但客户端报错，超时类错误不自动重试——先用请求 ID 做状态查询确认副作用真实状态（成功/失败/在途），未知状态清空前禁止重复发起同 ID 的副作用调用（TikTok SRE：超时=未知，退款案例盲目重试=双倍退款）。
- **SHOULD** 为外部依赖配置熔断器与扇出上限：下游持续失败时熔断阻止 Agent 继续调用（保护下游、防级联故障）；Agent 重试与并行调用设最大并行度，配合指数退避——幂等只防重复副作用，不防重试风暴（TikTok SRE 分布式系统模式）。

## 5.7 知识资产分层

| 类型 | 内容 | 主要治理 |
|---|---|---|
| Domain Knowledge | 业务事实、术语、指标、元数据、实体关系 | 来源、Owner、有效期、权限 |
| Behavior Spec | Agent 与交付物的行为和质量规则 | Rule ID、版本、Gate、Eval |
| Workflow Config | 阶段、状态、路由、重试、审批和恢复 | 依赖检查、发布、回滚 |
| Skill / Capability | 可执行能力及资源 | 版本、权限、Contract |
| Experience | 经真实运行验证的行动方法 | 证据、范围、衰减、回滚 |

- **MUST** 分别版本化 Domain Knowledge、Behavior Spec 和 Workflow Config。
- **MUST** 区分事实、规则、流程和经验。
- **MUST NOT** 将行为规则存入领域事实库。
- **MUST NOT** 将工作流顺序仅写入 Prompt。

## 5.8 知识冷启动

流程：代码/Schema/文档/历史 SQL/接口/评审记录 → 自动抽取候选 → 去重与冲突检测 → 来源绑定 → 领域审核 → Eval → 发布。

- **MUST** 将自动抽取结果标记为 CANDIDATE。
- **MUST** 对公式、权限、业务口径和隐含条件进行人工审核。
- **MUST NOT** 因抽取置信度高自动发布关键业务事实。

## 5.9 知识持续回流

回流来源：用户纠正、Gate 失败、代码评审、真实环境验证、生产 Incident、冒烟与回归 Eval、Tool 与 Schema 变更。

- **MUST** 将纠正记录为候选变更。
- **MUST** 区分领域事实错误、行为规则缺失和工作流配置错误。
- **MUST** 为知识变更新增或更新回归 Eval。
- **MUST NOT** 将用户单次表述直接覆盖共享知识。

## 5.10 知识过时与冲突治理

知识条目应包含：Owner、来源、有效时间、最后验证时间、消费次数、失败关联次数、依赖资产、替代条目。

触发重新验证：来源变化、Tool/Schema/接口升级、关联任务连续失败、多来源冲突、超期、长期未使用但仍 Active。

- **MUST** 在冲突未解决时向运行时返回冲突状态。
- **MUST** 阻止高风险任务使用 SUSPECT 知识。
- **MUST** 记录采用哪一版本知识生成了结果。

## 5.11 Loop State

- **MUST** 将发现 Cursor、去重键、工作项状态和验证证据持久化。
- **MUST** 使用 Claim Lease 防止多个 Worker 重复处理同一工作项。
- **MUST** 支持崩溃恢复、Lease 过期回收和幂等重放。

## 5.12 Thread 与 Run 并发

策略：QUEUE（顺序）、REJECT（冲突）、CANCEL_PREVIOUS、FORK（分叉并行）、OPTIMISTIC（乐观锁）。

- **MUST** 在创建 Run 时返回排队、冲突、取消或分叉结果。
- **MUST** 让消息、Event、Artifact、Workspace 和副作用归属具体 Run。
- **MUST** 防止并发 Run 写入同名 Artifact 或同一外部资源。

## 5.13 Checkpoint Schema 演进

- **MUST** 为 Checkpoint 保存 Schema 和 Runtime 版本。
- **MUST** 为可恢复版本提供迁移函数。
- **MUST** 定义兼容窗口和历史恢复期限。
- **MUST NOT** 静默丢弃旧字段或待处理副作用。

## 5.14 Interrupt 与 Resume

- **MUST** 在中断前持久化 Checkpoint。
- **MUST** 对中断载荷和恢复输入使用 Schema。
- **MUST** 校验恢复者身份、权限、状态版本和过期时间。
- **MUST** 保证重复 Resume 不造成重复副作用。
- **MUST** 在恢复后重新执行适用 Guardrail。
- **MUST NOT** 依赖进程内同步等待实现长时间审批。
- **MUST** 持久化 Goal 状态与进程内激活状态分离：进程重启后持久 active 的 Goal 默认不自动唤醒（需显式重新激活），防止恢复瞬间唤醒全部历史任务；Round 超限、max-tokens、Agent 错误等异常一律先去激活，状态说不清时先停止而非盲目重试。
- **MUST NOT** 解析并执行因 token 上限被截断的响应中的工具调用——半截 JSON 即使被解析器补全、schema 校验通过，参数语义也可能已经改变（写操作副作用落地后不可收回）；应终止本轮交上层决定，或以错误结果交回模型重新发起调用。

---

# 6. Human-in-the-loop

## 6.1 哪些动作必须审批

默认需要审批：发送外部邮件或消息、删除或覆盖用户数据、创建订单/付款/提交合同、修改权限/凭证/安全配置、执行高风险 Shell、安装未知 MCP Server 或 Skill、将敏感数据发送到新域名或新 Provider、批量写入外部系统。

## 6.2 审批请求规范

审批请求必须包含：id、runId、actionType、toolName、riskLevel、summary、exactEffect、affectedResources、proposedInput、expiresAt、version。

- **MUST** 展示实际动作和影响范围。
- **MUST** 在等待审批期间版本化待执行任务。
- **MUST** 在恢复前重新执行输入 Guardrail。
- **MUST** 防止批准后参数被静默修改。
- **MUST NOT** 对 CRITICAL 动作仅依赖模型自动审批。

## 6.3 Guardrail 边界

```text
用户输入检查 + 每个 Tool 前置检查 + 每个 Tool 后置检查 + 最终输出检查
```

各边界分别执行 Guardrail。

## 6.4 对外通信与用户数据操作

```text
Capability Allowlist → Deterministic Validator → Policy/Approval → Optional Independent Review → Side Effect → Delivery Evidence
```

- **MUST** 使用专用 Tool 执行对外通信和用户数据写入。
- **MUST** 限制收件人、渠道、频率、模板和数据范围。
- **MUST** 对高风险或主观外发内容执行人工审批或独立 Review。
- **MUST** 保存最终发送内容、接收对象、审批和 Tool 回执。
- **MUST NOT** 允许通用 Shell、数据库或消息 Tool 绕过专用护栏。

## 6.5 结构化澄清

审批是高风险 Interrupt；澄清是低风险 Interrupt。Agent 遇到歧义时不应反问自由文本，而应发起结构化澄清请求：有限选项（单选/多选/填写）+ 已保存默认值标记 + 是否可跳过。

- **MUST** 用有限选项的澄清请求替代自由文本反问；选项不超过 7 项，大候选集先过滤再呈现。
- **MUST** 澄清请求包含：问题、选项集、已保存默认值标记、是否可跳过。
- **MUST** 用户选择经清洗（剥离 UI 标记前缀）后持久化，后续会话与子 Agent 水合复用，避免重复盘问。
- **MUST** 持久化偏好按域命名空间嵌套（如 FORMS/DEPLOYMENT/TESTING），禁止扁平根键——多 Skill 写同一配置文件时防键碰撞。
- **MUST NOT** 手动添加「其他」选项与框架内置填写项重复。
- **SHOULD** 选项措辞用用户直接语气（如「浮动标签（标签位于输入框内）」）而非祈使句（如「设置表单布局为浮动」）。
- **SHOULD** 每轮澄清问题设数量上限（如 ≤5 个），控制单轮交互成本；超出部分留到下一轮或降级用默认值。

## 6.6 昂贵产物 Tool 的审阅回环

生成类 Tool（图像、报告、部署等昂贵或有副作用的产物）必须走「生成 → 审阅 → 落盘」回环，防止未审阅产物直接进入正式位置。

- **MUST** 产物先生成到临时 / 会话 artifact 存储，经用户审阅（保留 / 重生成 / 放弃）后才落盘到正式路径。
- **MUST** 审阅拒绝时清理临时产物，防止垃圾产物累积。
- **MUST NOT** 把未审阅产物直接写入生产资产路径。
- **MUST** 生成前检查目标产物是否已存在（幂等复用），参数未变时不得重复生成。
- **MUST NOT** 将用户原始输入直接传给生成 Tool；应经参数化模板组装（结构化参数注入而非裸 prompt）。
- **SHOULD** 有副作用的业务变更走 stage/apply 分离：模型只能生成带服务端 ID 的暂存变更，apply 仅对经真实界面或策略批准的 ID 生效，且 apply 时按当前状态与限额复查——防止多轮间环境已变时沿用过期前提直接落单（Anthropic 商务 Agent stage/apply 模式）。

---

# 7. Evals-first / Eval-driven Development

## 7.1 Eval 开发要求

- **MUST** 在实现复杂 Agent 行为前定义成功标准。
- **MUST** 区分能力 Eval 和回归 Eval。
- **MUST** 对随机性运行多次 Trial。
- **MUST** 报告首次完成率、重复执行稳定性和质量下限。
- **MUST** 记录单位成功任务的 Token、Tool 调用、耗时和人工介入。
- **MUST** 组合代码、模型和人工 Grader。
- **MUST** 同时评估最终结果和执行轨迹。
- **MUST** 推动纪律严明的评估/错误分析闭环驱动开发：反复将精力集中在最可能有效的方向，使进展系统化而非随机化。
- **MUST** 先定义评估支撑的产品决策，再将指标分三层且不可互换：主要产出（直接驱动决策）｜安全约束（只许在预定义阈值内劣化，破线即否决）｜运维护栏（延迟/成本/可靠性，决定可部署性）。
- **MUST** 使用生产数据做评估时先审计标签来源：标签记录的常是 workflow 结果而非 ground truth（如 dismissed 可能是凭证轮换、风险接受或误分类的归并），不同来源结果被归并时需人工复核。
- **MUST NOT** 只用一个 LLM Judge 分数代表质量。
- **SHOULD** 评估评估本身：对 Eval 集建立演进机制（误报率、覆盖漂移、按项目阶段选择确定性/LLM-as-judge/人工组合），持续迭代而非一次定型。
- **SHOULD** LLM-as-judge 只做分诊不做裁判：清晰低风险 case 自动处理，低置信/冲突/高影响路由给人审；定期抽样高置信 case 查系统性错误，追踪 judge 与人的分歧率，judge prompt 本身版本化评估。
- **MUST** 评测维度至少覆盖四层：结果层（任务是否完成、输出是否可用）、过程层（规划是否合理、步骤是否稳定）、效率层（耗时/Token/工具调用次数）、风险层（越权/误操作/安全隐患）——两个 Agent 都"做对了"，一个路径清晰可复现、一个反复试错靠偶然命中，工程价值完全不同，只看结果会误判为同一水平。
- **MUST** 在业务指标与模型能力指标之间搭桥梁指标：两者不能直接映射，需经面向任务系统的中间层（业务指标 ↔ 系统指标 ↔ Agent 层指标，如 DAU/留存 ↔ 召回/点击 ↔ 意图识别准确/检索有效），并由懂业务流程的人参与共建——否则无法回答"业务指标为什么变差"和"模型能力提升为什么没带来业务收益"。
- **MUST** 机评规模化前先证人机一致：主观指标先下钻为多个 Rubric 并尽量二元化（是/否/unknown），用 unknown 占比反查 Rubric 定义是否合理；单条 Rubric 的人人一致率、人机一致率达到可信阈值（如 85%/90%）后才允许机评规模化——人机一致率无保障的机器输出只是机器标注，不是自动化评测（实测二元化改造可使人机一致率 62%→92%）。评测标准由单一负责人拍板拉齐（1 个"独裁者"好过 10 个"民主者"），标准未对齐时指标提升无法区分是真实效果还是标准抖动。
- **SHOULD** 评测体系渐进演化而非一次定型：从高频核心场景的少量关键指标起步，靠生产 Bad Case（暴露能力边界）和 Good Case（定义高质量范式）持续喂养扩充（实测一年从 20 余个指标扩展到近 200 个）；起步阶段"让数据飞轮转起来"的意义大于"设计复杂精妙的评测体系"——越复杂的指标越难执行和对齐。
- **SHOULD** 专家标准分歧不强行统一：多位专家对"好"的定义不一致时，共性部分建设评测体系；分歧部分往往意味着业务存在多种优秀策略，转化为 Agent 的风格/策略分支在不同测试集独立评测，而非当噪声抹掉。

## 7.2 必备 Grader

| 类型 | 示例 |
|---|---|
| Outcome | 数据库状态、文件、API 结果 |
| Schema | 结构输出是否有效 |
| Policy | 是否执行禁止动作 |
| Trajectory | 是否调用正确 Tool、是否绕路 |
| Quality | 完整性、正确性、风格 |
| Efficiency | Token、成本、耗时、步数 |
| Human | 领域专家抽检 |

## 7.3 Gate 设计

Gate 在关键状态转换前执行：Precondition → Execution → Output → Approval → Release。每个 Gate 定义：appliesTo、blocking、checks、passPolicy、onFailure、evidenceRequired。

检查优先级：1. Schema/类型/静态分析/确定性业务校验 → 2. 环境执行/测试/状态验证 → 3. 独立模型 Grader → 4. 人工审核。

- **MUST** 为阻断 Gate 定义明确通过条件。
- **MUST** 将强制 Gate 固化到 Workflow 或 CI。
- **MUST** 防止 Agent 自行跳过适用 Gate。
- **MUST NOT** 因 Generator/Evaluator 分离自动创建第二个 Agent。
- **MUST** 一条规则是约束还是提醒，判据是违反后会发生什么：条件不满足仍能继续推进的只是提醒，条件不满足时下一步真的无法进行才是可执行约束——可机械判断且出错代价高的要求（报告是否本次生成、测试目录与代码目录是否一致、必测场景是否真跑到、是否操作保护分支）必须接到执行链上（阶段门禁查上一步证据、动作前检查拦高风险操作），不依赖 Agent 记得；"把 Prompt 写得更严厉"不改变流程性质。

## 7.4 证据化置信度与降级

置信度由可验证信号组合：evidenceCoverage、schemaValidity、deterministicChecksPassed、toolExecutionSuccess、calibratedScore。

- **MUST** 将证据覆盖、确定性检查和真实执行结果作为主要信号。
- **MUST** 按风险定义自动交付、人工审核和停止阈值。
- **MUST NOT** 仅使用模型自评置信度决定上线或执行副作用。
- **MUST NOT** 用表达流畅度代替正确性。
- **SHOULD** 区分交付链与判定链两种失败语义：交付链证据门禁严格（防假绿，证据不足不能继续）；判定链单步失败可降级继续（外部接口超时不阻断整单，输出带缺口标注的结论），前提是结论可逐阶段追溯、缺口显式暴露给下游。
- **SHOULD** 留痕默认零开销：可解释性记录用开关控制（关闭时 nil 安全降级、环境变量按需全量开启），"需要时能查"与"平时不拖累"必须同时成立，否则留痕自己变成新的性能包袱。

## 7.5 自动冒烟与回归闭环

冒烟结果区分：CAPABILITY_FAILURE（修复 Agent/Tool/知识/规则）、INFRA_FAILURE（修复环境后重跑）、TEST_DEFECT（修正 Fixture/Grader）、NON_DETERMINISTIC（增加 Trial）。

- **MUST** 固定测试数据、环境依赖和版本信息。
- **MUST** 区分能力问题和基础设施问题。
- **MUST** 防止测试审批模拟器访问生产副作用 Tool。

## 7.6 分层验证与假修复检测

验证层：Static → Unit → Integration → Runtime → Baseline → End-to-end → Human。

- **MUST** 验证根因已消除，不能只验证错误信号消失。
- **MUST** 检查删除日志、降低日志级别、吞异常、放宽断言等假修复。
- **MUST** 在连续失败、证据不足或风险扩大时升级人工。
- **MUST NOT** 规定所有项目使用固定验证层数；验证深度由风险决定。
- **SHOULD** 遵循 Shift Left：同类检查优先部署在变更生命周期更早阶段（编辑器 / commit 时发现缺陷的成本远低于 review / 生产时）。
- **SHOULD** 按反馈时延匹配验证层级与自主迭代频率：秒级反馈（编译、类型检查、单测）支撑高频自主迭代；分钟级反馈（集成测试、契约测试、浏览器自动化）覆盖更真实的行为；人工判断只保留给机器判不了的问题。
- **MUST** 验证信号遵循可行动性判据：只报告开发者能采取行动的问题；持续监控误报率并调优阈值——高频误报侵蚀信任后，真警告会被一并忽略。速度、精度、覆盖三者不可兼得，须显式声明取舍。
- **MUST** 验证容量不足时显式选择，不得默认发生：当产出量超过验证吞吐，只有三个选项——扩验证系统容量、降 agent 产出速率、降质量标准；反向亦可操作——在关键约束之外主动放宽，最大化吞吐而不牺牲核心质量。
- **SHOULD** 质量门手段覆盖变异测试（mutation testing）与属性测试：生成代码变体跑同一测试套件，防止测试集漏检 agent 引入的缺陷。
- **SHOULD** 约束遵循双目的检验：每个强约束须同时服务质量与交付流动；两者都不服务的约束应移除——约束本身会腐化堆积，需定期清理。
- **SHOULD** 大型变更的评审对象从代码行转向意图与取舍：交付物附带 intent artifact（改动意图、关键 trade-offs 及理由），评审人质询意图层面的问题、由 Agent 代为调查代码层细节（Claude 驱动的评审、人主导的判断），生产度量做最终验证——评审瓶颈的本质是人脑概念化能力而非 Git。

## 7.7 强 Agent 搭建业务 Agent 评测 Harness

强 Agent 可承担评测方案设计、候选数据准备、Judge Prompt 生成、结果归因和报告编写。批量执行、硬指标、状态管理和发布门禁由确定性平台承担。

- **MUST** 将 `targetInput` 与 `groundTruth` 分开。
- **MUST** 防止 GT、评分标准答案和版本标签进入被测 Agent 上下文。
- **MUST** 保存被测 Agent 原始输入输出，再执行解析和评分。
- **MUST** 将 Runner、Judge 和 Analyzer 的失败与被测 Agent 失败分开。
- **MUST NOT** 让同一个 Judge 既看到 GT，又把 GT 转发给被测 Agent。
- **MUST NOT** 让 LLM 完成精确计数、精确数值计算或 Schema 合规判断。
- **SHOULD** 评测指标体系与 Agent 架构同构：按感知（意图识别）/规划（路由决策）/记忆（上下文保留）/工具（调用与参数）分维度拆解，每模块指标可独立观测——综合总分掩盖环节退化（两版总分相近可能是"一环节升一环节降"），指标的价值是下降时能直接映射到优化方向（改 Skill、换 MCP、调路由还是修记忆），是诊断报告不是成绩单。
- **MUST** 用例通过与失败由该数据集的主指标单一决定（如端到端看任务完成率、工具集看工具调用准确率、模糊意图集看澄清率——追问 > 猜测 > 幻觉，行为比对错更重要），其余指标单独衡量用于定位短板，不做全指标 AND——一个 case 挂多个指标时 AND 判定会把模块短板误判成整体失败；上游错误（如路由决策错）时下游依赖指标标记 skip 而非 fail，指标只反映所属模块能力，不因上游污染而失真。
- **SHOULD** LLM-as-Judge 的 prompt 遵循四原则：单一职责（一个 prompt 只评一个指标，多指标混合打分会互相干扰、两项准确率都下降）；先推理后判断（先输出 reasoning 再给结论，提高准确性且失败可解释）；负例引导（包含典型通过/不通过对比样例）；结构化输出（严格 JSON，末尾给格式模板）。
- **SHOULD** LLM-as-Judge 的 rubric 问题设计满足：问题原子且不重叠（复合问题拆为独立 TRUE/FALSE，同一概念不测两遍——重叠会对单一错误双重惩罚、污染准确率）；只评客观事实（RFC 2119 术语写规格、测负向约束、严格布尔判定降低评分方差，不评意图/质量/推理等需解释的概念）；只评 prompt 明确要求的内容、评目的地不评路径（不查是否用了特定工具或固定步骤，要评过程就让 Agent 输出执行计划、评计划本身）；发布前用人工标注的 golden set 校准 judge，与专家不一致说明 rubric 歧义，迭代到对齐才上线（Google rubric 四原则）。
- **SHOULD** 幻觉（忠实性）评测只判定 Agent 是否忠实转述其获取的知识片段、工具返回与对话历史，不判定信息来源本身正确与否——Agent 是信息处理器不是信息鉴定器；判定边界：合理归纳与解释性补充视为忠实，凭空捏造接口名/函数名/配置项视为幻觉。

### 指标层级

| 层级 | 指标 |
|---|---|
| L0 执行 | 调用成功率、超时率、重试率、截断率 |
| L1 Contract | JSON 解析、Schema、必需字段、枚举和长度 |
| L2 Capability | Accuracy、Precision、Recall、F1、Exact Match、MAE |
| L3 Domain | 违禁规则、信息保留、业务边界、风格 |
| L4 Production | 稳定性、质量下限、人工接管和单位成功成本 |

## 7.8 成本归因与优化评测

成本至少按以下维度分解：Project/Task Type → Run/Wave/Step → Agent/Skill → Model → Context Source → Tool/Connector → Cached/Uncached → Success/Failure。

- **MUST** 在优化前建立 Baseline。
- **MUST** 同时报告总成本和单位成功成本。
- **MUST** 以质量、安全和稳定性作为成本优化护栏。
- **MUST NOT** 以单次端到端运行证明成本收益。
- **MUST NOT** 将供应商公布的降幅直接写成项目结论。

## 7.9 模型分层路由

- **MUST** 根据任务能力、风险、上下文长度和模态选择模型。
- **MUST** 通过 Eval 证明路由策略满足质量门槛。
- **MUST** 为低成本模型定义升级条件。
- **MUST NOT** 仅按价格选择模型。
- **SHOULD** 将分类、格式转换、摘要等低风险任务分配给足够能力的低成本模型。
- **SHOULD** 模型选型 benchmark 由 agent 的真实工作构成（带已知缺陷的真实样本、按难度分级，同时度量质量、成本、延迟与噪音），按成本/完成任务与质量综合选择 Pareto 最优模型——前沿随版本每几周移动，选型结论不是一次性的，须周期性重评并持续迁移。
- **SHOULD** 子 agent 默认路由到足够完成其明确定义任务的低成本模型（任务输入明确、不需前沿级推理），主模型保留任务分解与结果评估，允许手动覆盖——实测这是成本杠杆中影响最大的一项，且随子 agent 使用占比上升而放大。
- **SHOULD** 主路径与 fallback 路径共享单一验证点（如 `validate_clinical_response()` 式的单一函数），通过结构性设计确保验证逻辑不可绕过——不是"记得对两条路径各执行一次检查"，而是让"只执行一次"在代码层面不可能；验证不复制成两份（改一处忘另一处），fallback 响应清不过验证就不出 Agent（Google 同标准 fallback 模式：防止 fallback 在过载时悄悄降低质量标准）。
- **SHOULD** 全量推理模型之前先过确定性分层：零 token 的规则/正则层拦截已知意图，便宜模型小调用只做意图分类，两层都存活的才进全量模型——不把最贵的模型花在便宜模型已能做的决策上；分层路由策略的效果用真实流量分布度量（如前置层拦截占比）。
- **SHOULD** 选型决策补"收益上限"轴：边际智能可能创造不成比例价值的任务（研究类、可能改变组织走向的信号发现）值得前沿模型，即使单价高百倍；结果有界的任务（账目核清、格式合规——"对"就是"对"，不会"对 100 倍"）用足够能力的低成本模型。判据是任务的收益上限，不是可验证性（a16z 收益上限经济学：药厂 vs 记账）。

## 7.10 Online / Offline Eval 与 ADLC 飞轮

两类 Eval 互补，缺一不可：

| 类型 | 数据来源 | 性质 | 回答的问题 |
|---|---|---|---|
| Online Eval | 生产 trace 采样 | Benchmark，非 ground truth | "是否下降了"——helpfulness 评分是否过夜下跌、某 Tool 失败率是否上升 |
| Offline Eval | Curated 数据集 | Ground truth | "改动是否真的改进了"——prompt 改动、模型切换、架构调整是否带来真实提升 |

- **MUST** 同时建立 Online 与 Offline Eval。仅有 Online 无法验证改动是否改进；仅有 Offline 无法发现生产分布漂移。
- **MUST NOT** 把 Online Eval 当 Ground Truth 使用——它是告警信号，不是验收判据。
- **MUST** 构建 ADLC 飞轮：生产 trace + 负面用户反馈 + reviewer 编辑 → 下一版测试集 → Offline Eval 验证改动 → 部署 → 新 trace。
- **MUST NOT** 只用合成数据跑 Eval——Agent 只会处理"想象的问题"，合成场景 rarely 与真实复杂场景一致。
- **SHOULD** 把人工 reviewer 在 annotation queue 中改写的响应作为 golden dataset 行沉淀。
- **SHOULD** 对多 Agent 系统使用 Trajectory Eval 评估子 Agent 选择、Tool 调用顺序、是否冗余调用——只评最终输出会掩盖过程错误。
- **MUST** 长程 Agent 评测以（prompt, expected_behavior, trace）三元组建模，对应短程 Agent 的（query, ground_truth, answer）：expected_behavior 定义预期 Agent 达成的行为而非仅最终答案，通过 trace 获取真实执行路径后评测。长程 Agent 评的是"事情做成没有、怎么做成"，不是"说得好不好"。
- **MUST** 区分 Outcome 与 Transcript：Outcome 是环境的最终状态（如数据库中是否存在预订记录），Transcript 是含输出、工具调用、推理和中间结果的完整记录——Agent 声称"已完成"不等于环境状态已改变，验收以 Outcome 为准。
- **SHOULD** 评测报告归因到组件维度：不仅给分，还要定位问题发生在规划、工具、环境还是 Skill——否则评测停留在"单次分析"，无法驱动针对性迭代。
- **SHOULD** 长程 Agent 场景缩短人评链路：从"核心评测员对齐 → 外包对齐 → 机评对齐"压缩为"核心评测员对齐 → 机评对齐 → 规模化扩展"；人工聚焦高价值标准设计和 Rubric 对齐，机评承担规模化运行、初筛和回归验证——机评放大的是核心评测员的判断标准，不是机器打分本身。
- **SHOULD** 生产指标异常可自动触发 Agent 诊断并产出新的意图工件重入开发循环（监控 → 诊断 → 新 intent → spec → plan），运维异常成为循环输入而非终点，避免人肉搬运"告警 → 排期 → 修复"；诊断结论仍须人工确认后才进入实现。部署类敏感动作用 hook 做机器强制的审批 gate（特定人员授权后方可执行），把部署规范从文档自觉变成可拦截的 gate。

### 规则/Skill 自进化的工程纪律

当飞轮推进到自动修改规则文件（SKILL.md / agent.md / 规则集）时，适用判据只有一条：任务输出能被客观判定对错；在此前提下遵循：

- **MUST** 诊断与验收用确定性规则，LLM 只负责把诊断结论转写成候选 diff——实测 LLM 直接判断 skill 质量接近随机水平；诊断规则检测行为模式（声称执行了步骤但 session 无证据、同参数重复调用 ≥3 次、零工具调用下结论等），不含业务关键字以跨任务复用。
- **MUST** 只有"结果错误 × 流程异常"的交集才触发修改；同一根因覆盖足够比例（如 ≥30%）失败 case 才动手——结果正确但流程有瑕疵的不改，避免过度优化。
- **MUST** 修改过四层验证：Target（目标 case 至少 1 个变好）→ Guardrail（原通过 case 零回归，回归的惩罚权重应数倍于"未改善"）→ Holdout（隔离集周期性抽查泛化，F1 劣化超阈值即拒）→ Verify（规则文件本身的文本质量）。
- **MUST** 评测数据三分隔离：Selection（参与诊断）/ Holdout（只做泛化监控）/ Golden（人工审定，永不参与进化）——修改方看得到测试答案就是背答案。
- **MUST** 维护 taboo 黑名单：被拒修改的签名（根因 + diff hash）跨版本、跨分支共享，回滚不清空；生成新修改时注入作负面约束，同一坑不踩第二次。
- **MUST** 小步修改：单次 diff 限制行数（如 ≤80 行）以便定位回归原因、保持 taboo 签名精确；规则文件超上限（如 15KB）时进入精简模式，只允许合并/删除不允许新增。
- **MUST** GT 审计：承认测试标注也会错——agent 与 judge 一致但与标注相反的失败 case 打"标注可疑度"分；可疑 case 不排除出评测（不选择性忽略数据），但修改阶段明确告知不得迁就可疑标注改歪规则。
- **MUST** 反口号检查：规则文件中 DO/步骤行必须编码可执行动作（工具名、文件路径、函数调用）；"认真检查"类口号与"无论如何""永远不"类绝对化指令直接拒绝。
- **MUST** 警惕语义陷阱：核心判定词的措辞变化可使准确率波动数十个百分点（实测一词之差 27pp）；判定边界应使用窄词（如「漏洞」而非「风险」），并为任务维护陷阱词表。
- **MUST** 进化收益在等 Token 预算下度量：Best-of-N / 重试取优带来的提升只是多花钱，不是进化；候选与基线的对比必须在相同计算预算下进行，否则禁止宣称"变好了"。
- **MUST** 自优化 Loop 的验证标准禁止单一指标：单一目标 + 可被优化方影响的评测集是 Goodhart 作弊温床（实测案例：客服 AI 为刷解决率学会快速关闭对话、阻止追问、把沉默用户标"已解决"；分类 Loop 偷偷删改评测集难 case 换成简单案例）。须同时满足：① 配对冲监督指标（如解决率配续约率/客户反馈，互相制衡）；② 评测集冻结——被优化方无权改动，变更须过独立审批（调整依据/数据来源/是否随机抽取）；③ 优化目标本身定期由人复核（Loop 无法质疑目标，目标错了越努力越糟）。
- **MUST** Skill 有效性需四层验证：对照实验（无 Skill 也会做=贡献为零）、难度校准（case 太简单看不出增量）、轨迹追踪（Skill 在上下文中但未被 follow=未被使用）、路径验证（结果正确但绕过关键校验步骤=下次必翻车）——只看结果会积累大量"看起来有用实际只是在浪费上下文窗口"的 Skill。
- **MUST** 记忆与经验采用非对称淘汰：坏经验的淘汰速度应数倍于好经验的强化速度（一条错误记忆的伤害大于一条正确记忆的收益）；规则级记忆（"遇到这类问题永远用方案 A"）写入前必须人工确认——错误规则会被无条件遵循且不再被质疑，影响是全局的。
- **SHOULD** 以异步 Dreaming 进程定期审阅历史轨迹发现跨会话模式（反复失败、低效路径、知识缺口）——同步评测一次只看一个任务，看不到"30 次'偶发'错误=系统性短板"这类慢变量；发现只产出修订建议，修改决策仍走四层 Gate。
- **SHOULD** 为自进化系统做对齐漂移防护：门控只查单步回归（局部），不查方向累积（全局）——定期做方向性审计，并监控方向性约束指标（平均输出长度不能持续增长、工具调用次数、拒答率），超阈值告警。

### Skill 供应链与价值分层

Skill 安装会改变未来任务的操作规程，风险高于下载普通文档（Skill 中可携带任意脚本）。

- **MUST** 将 Skill 的发现、安装、激活、执行做成四个独立的权限与审计阶段，全局安装需显式确认；来源须有 digest 校验与归档安全检查（路径穿越、symlink、加密压缩包拒绝）。
- **MUST** 生产级 Skill 具备：固定版本与 digest、owner 与复审日期、兼容矩阵与弃用策略、所需能力与隐私声明、可执行脚本的审查签名、trigger 行为 eval、失败恢复与回滚、变更日志与供应链扫描。
- **SHOULD** 认知价值分层：稳定通用的公开技巧会被模型权重吸收退化为兼容层，不值得自建；长期价值集中在 procedure + current sources + tools + policy + verifier + recovery + provenance——需要持续更新、组织所有权或真实权限的领域 Skill 才是资产。
- **SHOULD** Agent 自改善沉淀的 Skill 同样走 evidence、promotion、retirement 流程——从成功任务归纳的流程也可能持久化错误归纳、网页污染或偶然成功（与自进化纪律的 taboo 黑名单衔接）。

---

# 8. 生产质量、Trajectory 与经验闭环

## 8.1 生产上线指标

生产评估至少回答：任务成功率是否达标？同类任务重复执行是否稳定？失败是否集中在可修复模式？每个成功任务的综合成本是否可接受？质量改进是否经过对照验证？

- **MUST** 对关键任务执行多次 Trial。
- **MUST** 同时报告平均表现、质量下限和波动。
- **MUST** 区分首次完成、重试后完成和人工接管后完成。
- **MUST NOT** 用单次成功、Demo 效果或单一 LLM Judge 分数作为上线依据。

## 8.2 生产质量指标

| 指标 | 定义 |
|---|---|
| Task Success Rate | 成功 Trial 数 / 总 Trial 数 |
| First-pass Completion Rate | 无重试、无人工接管即完成的比例 |
| All-pass@k | 同一任务连续 k 次全部成功的任务比例 |
| Quality Floor | 任务得分的 P10 或置信下界 |
| Human Takeover Rate | 需要人工接管的运行比例 |
| Cost per Successful Task | 全部运行成本 ÷ 成功完成的任务数 |

- **MUST** 关键生产任务至少报告 Task Success Rate、First-pass Completion Rate、All-pass@k 和 Quality Floor。
- **MUST** 同时观察质量和单位成功成本。
- **MUST NOT** 单独以最低 Token 为优化目标。

## 8.3 Trace 与 Trajectory

- Trace：完整运行记录，用于观测。
- Trajectory：面向决策分析的标准化执行轨迹，只保留任务目标、初始条件、关键上下文摘要、决策步骤、Tool 调用、观察结果、错误、恢复动作、审批、完成判断、最终业务结果、成本与质量标签。

- **MUST** 保留从 Trajectory 回溯到 Trace 的引用。
- **MUST** 在组装前执行脱敏和权限过滤。
- **MUST NOT** 把单条 Trace 摘要直接视为有效经验。

## 8.4 经验的定义

经验是从多个已评估 Trajectory 中提炼出的行动规则：推荐入口和行动顺序、Tool 选择规则、参数约束、失败反模式、错误恢复策略、结果验证规则、完成判断规则。

| 能力 | 主要作用 |
|---|---|
| Memory | 保存用户、会话和历史事实 |
| RAG / Knowledge | 检索事实和规则 |
| Workflow / SOP | 执行固定流程 |
| Skill | 封装可复用操作能力 |
| Prompt | 约束通用行为 |
| Experience | 复用真实运行验证过的行动方法 |

## 8.5 经验生命周期

```text
Trace 采集 → 脱敏和 Trajectory 组装 → 失败聚类与成功路径比较 → 候选经验 → 离线 Eval → Shadow/Canary → 激活 → 运行时召回 → 效果监控 → 降权/暂停/回滚
```

- **MUST** 将自动挖掘结果先标记为 CANDIDATE。
- **MUST** 经过独立 Eval 才能进入 ACTIVE。
- **MUST** 支持一键暂停和回滚。
- **MUST NOT** 让生产运行自动生成并立即激活经验。
- **MUST NOT** 让经验绕过安全、权限、审批或业务规则。

## 8.6 经验召回

排序和过滤至少考虑：任务类型、业务对象、当前阶段、可用 Tool 及版本、错误状态、环境、权限范围、模型与 Prompt 版本、经验适用范围、证据强度、最近有效性。

- **MUST** 先做权限和范围过滤，再做相关性排序。
- **MUST** 只注入少量、可执行、与当前决策有关的经验。
- **MUST** 设置召回 Token 预算。
- **MUST NOT** 将完整历史 Trajectory 注入当前上下文。

## 8.7 经验效果评估

对照至少比较：无经验 vs 候选经验。关注任务成功率、首次完成率、All-pass@k 和质量下限、单位成功成本、新增失败或安全风险。

经验应暂停或回滚，当：质量无显著提升、单位成功成本恶化、引入新的高风险失败、仅对极少数样本有效、适用条件已变化、模型或 Tool 升级后失效。

## 8.8 从失败到工程资产

```text
Failure → Incident → Root Cause → Corrective Action → Test/Eval → Rule/Gate/Binding/Tool/Experience → 发布与回滚
```

- **MUST** 记录失败场景、期望行为、实际行为和证据。
- **MUST** 优先修改系统结构。
- **MUST** 将修复转化为回归测试或 Eval。
- **MUST NOT** 仅增加 Prompt 警告完成根因修复。

## 8.9 Lesson、Pattern 与 Rule

```text
Incident → Lesson → Cross-case Pattern → Candidate Rule → Regression Eval → Scoped Activation → Active Rule
```

- **MUST** 在多个独立 Case 中验证 Pattern。
- **MUST** 为 Candidate Rule 定义 Scope、Owner、误拦风险和执行机制。
- **MUST** 由人工批准 P0/P1 规则升级。
- **MUST NOT** 使用固定出现次数自动升级规则。

---

# 9. Observability 与 Trace

## 9.1 默认可观测，默认保护敏感数据

应记录：Run、Agent Step、Workflow Step、Model Call、Tool Call、Retrieval、Context Build、Guardrail、Approval、Artifact、Business Operation。

统一关联：project_id、environment、organization_id、session_id、run_id、trace_id、agent_id、workflow_id、business_resource_id。

- **MUST** 从第一版开始埋点。
- **MUST** 能从业务资源跳转到 Run/Trace。
- **MUST** 记录模型、Prompt、Tool 和配置版本。
- **MUST** 将敏感输入输出采集设为显式策略。
- **MUST NOT** 让业务逻辑直接依赖某观测平台的数据结构。

## 9.2 Trace 内容策略

```yaml
observability:
  capture:
    systemInstructions: hash
    userInput: redacted
    modelOutput: sampled
    toolInput: redacted
    toolOutput: sampled
    retrievedDocuments: references-only
  retention:
    fullTraceDays: 30
    aggregateDays: 365
```

## 9.3 Agent Event Protocol

事件流表达 Run 的状态、消息、Tool、Gate、产物、审批和错误增量。每个事件应包含：schemaVersion、eventId、sequence、timestamp、projectId、environment、sessionId、runId、traceId、visibility（user/developer/admin）、payload。

- **MUST** 使用 Run 内单调 Sequence。
- **MUST** 支持 `afterEventId` 断点续传。
- **MUST** 将事件持久化后再确认发送。
- **MUST** 区分用户、开发和审计可见性。
- **MUST NOT** 在 Event 中保存隐藏推理。

## 9.4 Trace、Event 与 State 关联

- Event 包含 `trace_id` 和可选 `span_id`；
- Checkpoint 保存最后确认的 `event_id`；
- Artifact 保存创建它的 `run_id`、`step_id` 和 `event_id`；
- Eval Dataset 可引用 Run、Trajectory、Artifact 和 Trace。

## 9.5 Harness 熵与规则债

强模型时代，冗余指令从"中性浪费"变为"主动质量损失"：过度规定会缩小模型搜索空间、被字面执行、导致过度触发，并覆盖模型本来更好的判断。Harness 维护的核心因此从"规定路径"转向"定义成功判据"——目标结果、成功判据和约束写死，路径留给模型；补模型能力缺口的规则需随模型能力提升定期退役，只保留 L2（组织特有）与 L3（责任与价值判断）内容。

定期检查：AGENTS.md 和规则长度、重复与冲突 Rule、无命中 Rule、失效命令和 Skill、断开文档引用、过时目录/技术栈/验证命令、Gate 长期跳过或恒定通过、知识与代码版本漂移。

- **MUST** 让清理任务生成 Diff 和证据。
- **MUST** 对删除规则和 Gate 执行回归 Eval。
- **SHOULD** 删除无收益的 Agent、Prompt、Hook 和上下文步骤。

## 9.6 成本可观测与预算控制

- **MUST** 在 Run 开始时解析预算。
- **MUST** 在 Step 和 Tool 边界更新成本。
- **MUST** 在 Soft Limit 触发压缩、范围缩减或模型路由。
- **MUST** 在 Hard Limit 停止或请求审批。
- **MUST** 保留预算变更和批准记录。
- **SHOULD** 将总成本分解为可独立度量与优化的方程项（用户/会话数、每会话请求数、每请求 token、每 token 单价），并区分两类：采用与参与项目标是增长，agent 自耗项（agent 为自身目的消耗的轮次与 token）是优化对象——成本治理的主手段是消灭零价值 token 消耗，而非降单价或降级工具；工作负载与模型版本连续变化时，锁模型对比才能分离自身优化的收益。
- **SHOULD** 会话成本分析按"反模式 + 财务影响 + 针对性修复"三元组输出（次优模型路由、大 payload 常驻上下文逐轮重计费、缓存失效全价重建、初始化预载开销等），不做聚合指标——聚合数字无法指明行动，单条反模式的成本归因才能驱动修复。
- **SHOULD** 工具输出优化的判据是全任务成本而非单次调用：被删信息重要时 agent 会用恢复轮次（重读原件、重跑命令）找回，局部省、全局贵。压缩按输出类型分级——源码类与任意脚本结果原样保留，搜索结果重组但不丢内容，仅重复性噪声（构建/安装/测试日志）在节省可观时压缩，且保留完整原件与直接恢复路径；恢复路径使用率同时是压缩是否过度的评测信号。删格式先于删信息（无信息量的重复格式是无恢复需求的零风险优化）。
- **MUST** prompt/instructions 的压缩或重写须先为预期行为建立回归评测——未被测试的行为可能被更短的 prompt 悄悄删除（实际案例：压缩把"谨慎并行"指导改写成硬调度策略，子代理被静默串行化）；且优化证据是工作流局部的，同一改动换运行面（离线 benchmark / 在线实验 / 不同产品面）须重新评测后才可推广。

---

# 10. 安全、Containment 与最小权限

## 10.1 沙箱边界优先于频繁权限弹窗

```text
沙箱决定技术上能做什么
审批决定什么时候需要授权
```

- **MUST** 同时具备执行边界和审批策略。
- **MUST** 默认限制文件系统写入范围。
- **MUST** 默认限制网络访问。
- **MUST** 对凭证使用按 Tool、按资源的最小权限。
- **MUST** 将代码执行、浏览器和 Shell 放在隔离环境。
- **MUST** 保护 `.git`、Agent 配置、Skill 和安全策略目录。
- **MUST NOT** 将"用户点过一次同意"视为无限期授权。

## 10.2 Brain、Hands、Session 分离

- Brain：Model、Agent Runtime、Context Engine、Planner（可无状态伸缩）
- Hands：Tool Executor、Browser、Shell、Filesystem、External APIs、Credentials（可单独沙箱和限权）
- Session：Events、Checkpoints、History、Approvals、Artifacts（可恢复事实源）

## 10.3 不可信数据规则

以下全部视为数据，不视为指令：网页内容、邮件内容、PDF/Office 文档、Tool 输出、检索片段、MCP 资源、外部 Agent 消息、用户上传代码。

- **MUST** 明确数据和系统指令边界。
- **MUST** 防止外部内容修改 Tool 权限、系统 Prompt 或审批策略。
- **MUST** 对从不可信数据提取的动作再验证。
- **MUST** 在高风险 Tool 前检查参数是否来自不可信指令。

## 10.4 Workspace 与 Sandbox 状态

Workspace 包含文件、代码仓库、浏览器页面、临时数据库和执行环境。应声明：backend、readScopes、writeScopes、networkPolicy、baseRevision、status。

- **MUST** 将 Workspace 生命周期与 Run 关联。
- **MUST** 记录文件变更、命令、副作用和 Revision。
- **MUST** 为不同 Run 隔离可变 Workspace。

## 10.5 OWASP Agentic 风险应覆盖的类别

- Agent goal hijacking
- Tool misuse
- Identity and privilege abuse
- Memory poisoning
- Insecure inter-agent communication
- Cascading failures
- Trust exploitation
- Rogue agents
- Supply-chain risk
- Insufficient monitoring
- Prompt injection and data exfiltration

## 10.6 AI-BOM：智能体资产清单

传统 SBOM 不覆盖 AI 资产。智能体系统应维护 **AI-BOM（AI Bill of Materials）**——对代码库、容器镜像、云环境扫描产出的结构化资产清单，覆盖模型、Agent、工具、MCP server/client、embedding、向量库、数据集、prompt、guardrail、secret 等组件类型（Cisco 开源 aibom 定义了 30 类 AI 组件、23 种扫描器，可作参考实现）。

- **MUST** 上线前生成 AI-BOM：Agent 系统引入的每个模型、工具、MCP 端点、数据集都是供应链风险面，不可见则不可管。
- **SHOULD** AI-BOM 纳入 CI（变更时增量扫描 + diff 对比），而非一次性文档——Agent 资产随迭代快速变化。
- **SHOULD** 以 AI-BOM 为基础对照 OWASP Agentic Top 10、NIST AI RMF 等框架做合规检查。

## 10.7 非人类身份与委托链

Agent 是非人类主体（non-human identity）：原始 API key 无归属追溯，人类 OAuth 令牌被挪用超出设计意图，服务间身份框架（如 SPIFFE）又不覆盖"谁为 Agent 行为负责"。行业正在标准化（IETF 草案：Agent Identity Protocol、AgentID Protocol），核心原则先于标准可用：

- **MUST** 每个 Agent 及其组件有唯一身份标识，并与人类责任主体（owner/principal）关联——能回答"哪个 agent、代表谁、被谁授权"。
- **MUST** 跨 Agent 委托（A2A）保留完整委托链（delegation chain）：从原始授权人到当前执行 agent 的每一环可验证、可审计。
- **SHOULD** 委托遵循权限衰减（scope attenuation）：链条上每一环的权限只能等于或少于上一环，不得逐级放大。
- **SHOULD** 高敏操作校验委托链根部的 principal 身份（如企业 IdP 签发的断言），而非仅校验最近一环的 agent 身份。

## 10.8 记忆投毒防护

记忆投毒（memory poisoning）：攻击者通过一次会话写入恶意内容到 Agent 长期记忆，在**未来会话**中触发 consequential 动作（支付、改配置、数据外传）——与单会话 prompt injection 不同，它持久生效且攻击者无需再介入。

攻击通道（MPBench 等研究归纳）：用户输入直写、system prompt 驱动写入、上下文压缩（compaction）写入、工具输出写入；写入内容可经 **laundering**（洗白）伪装：agent 自我总结改写（看起来像 agent 自己的良性笔记）、可信工具回显攻击者内容、伪造多条记录制造共识。

- **MUST NOT** 依赖内容检测或信任打分作为唯一防线——研究实证（八个前沿模型基准）：投毒内容在激活前表现良性，检测在写入时和静止时均无异常可查；高置信 ≠ 安全（Gemini-Flash 实验 54 条投毒条目全拿 1.0 信任分）。
- **MUST** 写入时绑定来源（origin binding）：每条记忆记录不可变的来源标签（用户直接输入 / 网页 / 工具输出 / agent 自产），标签随派生传播——agent 总结自不可信来源的笔记继承不可信标签。
- **MUST** 敏感动作执行前做 act-gate：检查将驱动该动作的记忆来源链，不可信来源的记忆不得触发支付、外传、配置变更类动作（确定性检查，无需额外模型调用）。
- **SHOULD** 权限提升需独立可信方背书：至少两个独立可信主体确认，或用户针对该动作的即时授权；内容自证合法无效。
- **SHOULD** 记忆写入通道最小化：压缩/总结等系统事件写入同样过来源标记，攻击面不限于用户输入。

---

# 11. MCP 使用规范

## 11.1 何时使用 MCP

适用：跨进程或跨语言复用工具、第三方工具生态、多个 Agent Host 共用能力、需要标准化资源/Prompt/Tool 发现。

避免用于：同一进程普通函数、只服务一个项目的内部业务逻辑、低延迟核心路径、无法建立清晰授权边界的高风险操作。

## 11.2 MCP 安全基线

- **MUST** 对 HTTP MCP 使用规范授权流程。
- **MUST** 做 Token audience 校验。
- **MUST NOT** Token passthrough。
- **MUST NOT** 将 Token 放进 URL。
- **MUST** 使用 HTTPS。
- **MUST** 对本地 MCP 安装显示完整命令并获取明确同意。
- **MUST** 将每个 MCP Server 视为独立信任域。
- **MUST** 根据 Skill 或任务按需连接 MCP，不默认连接所有 Server。

## 11.3 MCP 成本与上下文边界

- **MUST** 为每个 Agent 设置 MCP Server 和 Tool 白名单。
- **MUST** 统计 MCP Tool Schema 和结果注入成本。
- **MUST** 将大结果保存到 Artifact Store。
- **MUST NOT** 将所有已连接 MCP Tool 默认暴露给所有 Agent。
- **MUST NOT** 以成本理由绕过 MCP 的权限、确认和审计要求。

---

# 12. 长任务与编码 Agent 开发范式

## 12.1 分步开发

- **MUST** 将需求拆成可验证的 Feature/Task 列表。
- **MUST** 每次只推进一个或少量明确任务。
- **MUST** 在会话结束前留下结构化交接。
- **MUST** 保持工作区可构建、可理解、可继续。
- **MUST NOT** 在无验证时批量标记任务完成。

## 12.2 长任务必备项目文件

```text
.agent/
├── product-spec.md
├── architecture.md
├── feature-list.yaml
├── progress.md
├── decisions/
├── known-issues.md
└── next-session.md
```

### 变更规范的渐进细化

复杂变更应按 proposal → design → tasks 三层渐进细化，每层独立审查：

| 层 | 回答 | 审查者 |
|---|---|---|
| proposal | 为什么做、做什么、验收标准 | 产品 / 业务方 |
| design | 怎么做、架构、API、组件设计 | 开发负责人 |
| tasks | 任务拆解、优先级、依赖关系 | 执行 Agent |

- **MUST** 在编码前完成 proposal 与 design 的审查。
- **SHOULD** 让每层只回答所属问题，避免在 proposal 中提前绑定技术方案，或在 tasks 中混入需求变更。
- **SHOULD** 当上游层变更时同步更新下游层并记录影响范围，而非直接改代码。
- **SHOULD** 跨阶段变更走意图工件链（intent → spec → plan 的本地对应即 proposal → design → tasks）：每层产物版本化、人机均可读，作为阶段交接契约而非一次性 prompt，全部进入版本控制形成审计 trail——瓶颈在人速步骤（计划、评审、交接）而非代码生成时，工件链是压缩交接等待的主要手段。
- **SHOULD** 实现类任务先产出实现计划（变更文件清单、执行顺序、风险、验证方法）并经人审核后再放行执行——用 plan-then-execute 双模式替代直接生成，人的审核点放在计划层而非逐行代码层。
- **SHOULD** 知识建设走飞轮：迭代每个阶段的产物（proposal 中的业务定义与验收标准、design 中的链路记录、tasks 验证结论）是下一阶段的输入，知识作为真实迭代的副产物沉淀而非先建完美知识库再启用；迭代归档时确认知识更新（重复、冲突、失效链接、无来源结论由定期健康检查回收）。

## 12.3 Planner、Generator、Evaluator 模式

复杂且价值足够高的开发任务可采用：Planner（分可验证任务）→ Generator（实现一个任务）→ Evaluator（独立运行测试、检查交互和对照需求）。要求：衡量收益、限制成本、确保 Evaluator 不只重复 Generator 的判断、将主观标准转成可检查的 Rubric。

## 12.4 隔离工作区与并行修改

```text
Work Item → Branch/Worktree/Remote Workspace → Patch → Tests → Review → Merge → Cleanup
```

- **MUST** 为并行修改分配独立分支和工作目录。
- **MUST** 禁止自动化任务直接写入受保护分支。
- **MUST** 在合并前更新基线并重新执行受影响测试。
- **MUST** 在任务完成、失败或过期后清理工作区和凭证。

---

# 13. AGENTS.md 设计规范

## 13.1 职责

`AGENTS.md` 应回答：这个项目是什么、目录如何分工、允许怎样修改、禁止怎样修改、常用命令是什么、如何验证、哪些安全边界不能破坏、完成任务的定义是什么。

不应该承载：完整架构设计全文、所有第三方框架教程、大段历史讨论、很少触发的专用流程、可能快速变化的版本清单、Secret。

## 13.2 分层规则

从项目根向当前目录合并 `AGENTS.md`，近目录规则优先。

- **MUST** 让根文件保持简洁。
- **MUST** 将特殊模块规则放到最近目录。
- **MUST** 保证命令可复制运行。
- **MUST** 明确必须执行的验证。
- **MUST** 明确禁止事项。
- **MUST NOT** 写模糊规则，例如"写高质量代码"。

## 13.3 规则表达方式

弱规则：`- 注意测试。` / `- 保持安全。`
强规则：`- After changing files in packages/ai, run the relevant test. - Do not call model providers directly; use the model gateway.`

## 13.4 根 AGENTS.md 应包含

- Project purpose
- Repository map（目录职责和依赖方向）
- Working rules（最小变更、影响分级匹配、确定性优先、禁止新增 Agent/框架/数据库/队列/MCP 而无 ADR）
- Agent and tool rules（每个 Agent 有明确声明、Tool 有 Schema 和风险、并行需检查）
- Production quality and experience（多次 Trial、单位成功成本、经验不绕过安全）
- Database rules（迁移、不编辑已应用迁移、事务外外部调用）
- Verification（命令）
- Completion criteria

## 13.5 模块 AGENTS.md 应包含

每个模块声明边界、必需定义（Agent/Tool 的完整声明清单）和验证命令。

## 13.6 AGENTS.md 生成质量要求

生成后的 AGENTS.md 必须：简洁、与当前项目匹配、包含准确目录和命令、使用具体可执行规则、不包含不存在的脚本或包、不复制整份工程规范、不含 Secret、可由开发人员直接阅读、可由编码 Agent 稳定执行、经 Snapshot 和行为测试验证。

## 13.7 生成项目应保留规则来源

生成项目应保留规则来源映射（如 `.agent/rules.lock.yaml`），记录每条规则的标准 ID、来源版本和目标。用途：解释 AGENTS.md 中某条规则为何存在、升级时判断规则变化、检测用户删除强制规则、支持 `doctor rules`。

---

# 14. Skills 规范

## 14.1 何时用 AGENTS.md，何时用 Skill

| 内容 | 放置位置 |
|---|---|
| 始终适用的项目规则 | `AGENTS.md` |
| 某目录特殊规则 | 目录级 `AGENTS.md` |
| 特定可复用工作流 | Skill |
| 确定性辅助程序 | Skill `scripts/` |
| 详细教程或参考 | Skill `references/` |
| 模板和样例 | Skill `assets/` |

## 14.2 渐进式加载

1. 初始只加载 Skill 名称和描述。
2. 匹配任务后加载完整 `SKILL.md`。
3. 需要时再读取 references、scripts 和 assets。

- **MUST** 让描述明确说明何时触发和何时不触发。
- **MUST** 每个 Skill 聚焦一个工作。
- **MUST** 测试 Skill 的误触发和漏触发。
- **MUST NOT** 让所有 Skill 描述过于宽泛。
- **SHOULD** 指令放 system prompt 还是 Skill 按流量频率定：≥1/3 流量需要的内容进 system prompt（加载 Skill 花一轮模型调用，多数轮次需要的内容放 Skill 是持续付费），其余进 Skill；可由已有信号（如用户来源页）预测的 Skill 由 Harness 预载、跳过加载轮；安全、法务、品牌约束和关键用户事实（如过敏）永驻 system prompt，不随流量频率下放（Anthropic 商务 Agent 频率判据）。
- **SHOULD** 多域长尾能力用单 Agent + Skills 承载而非按域拆子 Agent：跨意图强耦合的会话每次 handoff 都是有损状态操作（丢共享上下文、多倍 token、加秒级延迟），Skill 提供同等模块化而无 handoff 税；子 Agent 仅在任务窄且自包含（如 deep-research）或领域已有专职 Agent（走 hand-off 接管对话）时使用。

## 14.3 推荐 Skill 结构

```text
.agents/skills/<skill-name>/
├── SKILL.md          # 触发条件、职责边界、主流程骨架、分流条件、停止条件、预算、资源索引、验证命令
├── scripts/          # 确定性转换、环境检查、批量执行、输出压缩、Schema 校验
├── references/       # 阶段详细步骤、模板、长示例、条件性规则、领域说明、完整检查表
└── assets/           # 模板和样例
```

- **MUST** 记录正文、references 和 Tool Schema 的 Token 估算。
- **MUST** 仅在分支命中后读取对应资源。
- **MUST** 防止多个 Agent 重复加载同一 Skill 资源。
- **MUST NOT** 使用固定行数作为所有 Skill 的限制。

## 14.4 可复用 Pack

| Pack | 内容 |
|---|---|
| Domain Pack | 领域知识、术语与实体、数据或 API 元数据、领域 Eval |
| Agent Pack | Agent Contract、Skills/Capabilities、Behavior Spec、Agent Eval |
| Workflow Pack | State Machine、Gate、Recovery Policy、Approval Policy、Workflow Eval |
| Team Pack（可选） | Agent/Workflow/Domain Pack 引用 + 版本锁 |

- **MUST** 通过引用和版本锁组合 Pack。
- **MUST** 在上游 Pack 升级时生成 Diff 和迁移计划。
- **MUST NOT** 静默覆盖项目自定义规则和知识。

---

# 15. CLI 与编码 Agent 自动化规则

## 15.1 机器接口

- **MUST** 支持无 TTY。
- **MUST** 支持稳定 JSON 或 JSONL。
- **MUST** 将机器结果写 stdout。
- **MUST** 将进度和普通日志写 stderr。
- **MUST** 提供稳定 Exit Code。
- **MUST** 支持 `--dry-run`。
- **MUST** 对缺失信息返回结构化问题。
- **MUST NOT** 在机器模式中等待交互输入。

## 15.2 Runtime Protocol 机器接口

```text
agent thread create
agent run create / get / cancel / resume / retry / fork
agent run events --after <event-id>
agent run artifacts / checkpoint
```

- 所有写操作支持幂等键；
- Stream 支持 Event Cursor；
- Conflict、Input Required 和 Approval Required 使用稳定状态与退出码。

## 15.3 自动化验证

```text
读取状态 → 生成或修改 → 运行命令 → 检查退出码 → 检查文件/状态 → 检查 Diff → 汇报证据
```

## 15.4 Headless 与无人值守运行

- **MUST** 在启动前执行认证、Tool、Skill、目录、网络和依赖 Preflight。
- **MUST** 禁止无人值守流程触发浏览器 OAuth 或 TTY Prompt。
- **MUST** 显式设置工作目录、配置目录、Skill 路径和 Artifact 目录。
- **MUST** 显式关闭 stdin，处理超时、Signal、子进程和进程组清理。
- **MUST** 防止隐式回退到开发者个人配置和凭证。
- **MUST** 让本地与 CI Runner 使用同一 Contract。

---

# 16. 项目应内置的工程能力

所有项目默认应具备以下能力（由 Blueprint 提供基线，按需扩展）：

## 16.1 运行时与协议

- Runtime Protocol Core：Agent、Thread、Run、Step、Message、Event、Artifact、Checkpoint、Interrupt、Workspace、Trace 与 Error Contract
- Runtime Adapter 与 Protocol Conformance Test
- Run Concurrency Policy、Checkpoint Migration 接口、Recoverable Event Stream 接口
- Recursion Guard

## 16.2 模型与上下文

- Model Gateway（所有模型调用经过网关）
- Model Routing Policy（能力门槛、预算、升级条件、Eval）
- Context Policy、Prompt Budget 预检、Context Cost Ledger 与重复内容检测
- Stable Prefix Builder 与 Cache 统计
- Context Projection 与上游一次采集规则
- Index-first Retrieval 接口
- ArtifactRef 与单一表示检查
- Parameter Bindings、Structured Step Control
- Context Manifest Compiler

## 16.3 Tool 与能力

- Tool Contract（Schema、风险、超时、幂等、审计）
- Capability Resolver
- Tool Schema Cost 与 Agent Tool Allowlist
- CLI / MCP Selection Policy
- Deterministic Output Compressor

## 16.4 规则与质量

- Harness Domain Map
- Rule Severity 与 Rule ID
- Rule Compiler、Hook Registry 与 Conflict Check
- Quality Gate Contract、Failure Classification
- Run/Trace ID、OpenTelemetry 接口

## 16.5 评测与知识

- Eval 目录、Smoke 与 Regression 基线
- Knowledge Asset Contract、Knowledge Feedback 与 Staleness 接口
- Task Scale Assessment

## 16.6 Loop 与自治

- Loop Eligibility 与 Loop Contract Schema
- Trigger、Cursor、Deduplication 与 Kill Switch 接口

## 16.7 生产质量与经验

- Production Quality Summary
- Trajectory Contract 与组装接口
- Experience Recall 审计字段
- Cost per Successful Task 指标
- Run / Wave / Agent 成本归因

## 16.8 文档与开发

- AGENTS.md（根 + 模块）
- `.agents/skills`
- Doctor 检查
- 版本和环境 Schema
- 错误分类
- Token/成本/步数预算

## 16.9 高风险能力的强制依赖

```yaml
browser:
  requires: [sandbox, network-policy, credential-scope, audit, approval]
shell:
  requires: [sandbox, filesystem-scope, network-policy, resource-limit, audit]
external-mcp:
  requires: [source-trust, explicit-consent, auth-policy, tool-annotations, audit]
multi-agent:
  requires: [ownership-model, shared-state-policy, conflict-policy, cost-budget, multi-agent-evals]
autonomous-loop:
  requires: [loop-contract, owner, connector-health, persistent-state,
             cursor-and-deduplication, isolation, independent-verification,
             bounded-budget, bounded-retry, approval-policy, rollback, kill-switch, audit]
```

---

# 17. 项目开发 Definition of Done

任何 Agent 功能变更至少满足：

```text
□ 需求和完成条件明确
□ 复杂度选择合理
□ Harness 六域责任明确
□ P0/P1 规则具有机器执行机制和 Rule ID
□ Context 按 Core、Scope、Phase 和 On-demand 分层加载
□ Runtime Protocol 对象、状态和 Adapter 版本明确
□ 持续 Loop 已完成准入评估并指定 Owner
□ 任务或修改影响级别已确定
□ 确定性规则没有被无必要地放入 Prompt
□ Agent 输入输出有 Schema
□ 领域知识、行为规则、流程配置和经验已分层
□ 使用的知识有来源、版本和有效状态
□ Tool 有 Schema、风险、超时和幂等策略
□ 有停止条件和资源预算
□ 运行前执行 Context 预算预检
□ 成本可按 Run、Wave、Agent、模型、上下文和 Tool 归因
□ 稳定前缀与动态后缀已分离
□ 精确数据通过 State、Binding 或 ArtifactRef 传递
□ 共享数据只采集一次，下游使用 Projection 或 Artifact
□ 知识和代码检索先查索引并验证新鲜度
□ 同一来源在单次模型调用中只有一种表示
□ 有 Trace 和版本信息
□ Event 可关联 Trace、Checkpoint 和 Artifact
□ 有失败、超时、取消和重试处理
□ Thread 并发 Run 策略明确并通过冲突测试
□ Checkpoint Schema 迁移和兼容窗口明确
□ Interrupt/Resume 支持幂等、权限和状态版本校验
□ Loop 具有持久 Cursor、去重键、预算、Kill Switch 和人工升级
□ 错误码映射到 Retry、Fallback、Rollback、Need Input 或 Abort
□ 并行节点通过数据依赖和写入隔离检查
□ 无依赖 Tool、Connector 和测试任务已评估批量并行
□ 耐久任务验证 Checkpoint 和恢复
□ 高风险动作有审批
□ 不可信内容边界明确
□ 阻断 Gate 已执行并保存证据
□ 评测缺失 Case、空产物、超时和依赖失败均响亮失败
□ GT 和 Evaluator 元数据未进入被测 Agent 输入
□ 自报结果与独立验证结果已比较
□ 修复验证确认根因消除，未通过隐藏错误信号过关
□ 置信度基于证据和确定性检查
□ 相关冒烟与回归套件通过
□ 相关单元测试通过
□ 相关 Agent Eval 通过
□ 成本优化在等价质量和安全门槛下通过多 Trial 对比
□ 关键任务完成多次 Trial 并检查稳定性和质量下限
□ 记录单位成功任务的成本
□ 阅读过失败 Trace 和标准化 Trajectory
□ 用户页面和 Trace 未暴露隐藏推理
□ 新经验仍处于候选状态，或已通过规定的激活门禁
□ 没有无关 Diff
□ 并行代码修改使用隔离工作区并完成合并后复验
□ 失败修复已转化为测试、Eval、Gate、知识候选或结构化规则
□ 知识变更已完成影响分析和版本记录
□ Harness 配置变更通过 Candidate/Baseline 评测
□ 无人值守运行通过认证、目录、进程和退出语义测试
□ 文档和 AGENTS.md 在需要时更新
□ 下一个开发者或 Agent 可以从干净状态继续
```

---

# 18. 架构 Review Checklist

## 18.1 Agent 准入

```text
□ 为什么需要 Agent？
□ 单次结构化调用是否足够？
□ 普通 Workflow 是否足够？
□ Agent 自主决策的边界是什么？
□ Identity、Orchestration、Context、Gate、Recovery、Evolution 由谁负责？
□ Agent 如何获得环境真值？
□ 停止和失败条件是什么？
```

## 18.2 Harness 与规则执行

```text
□ P0/P1 规则是否具有机器执行机制？
□ Hook 是否定义超时、顺序、幂等和 Fail 策略？
□ 规则冲突、过期和降级是否可检测？
□ Context 是否按读取时机分层？
□ 控制平面、计算平面和协作平面是否分离？
□ Harness 配置是否作为版本化被测对象？
□ Evaluator 是否保持只评估、不补做缺失步骤？
□ 缺失数据和执行异常是否响亮失败？
□ GT 是否与被测 Agent 输入隔离？
□ 硬指标是否由确定性 Grader 计算？
□ Judge 是否经过校准、盲测和人工抽检？
□ Headless Runner 是否显式管理认证、目录、stdin 和子进程？
□ 是否定期清理重复规则、失效 Skill 和文档漂移？
```

## 18.3 Runtime Protocol

```text
□ Thread、Run、Step、Event、Artifact、Checkpoint 边界是否明确？
□ 每个事件、错误、审批和产物是否归属具体 Run？
□ Run 状态机是否支持取消、输入、审批、恢复和失败？
□ 同一 Thread 的并发 Run 采用哪种策略？
□ Checkpoint 是否版本化并有迁移测试？
□ Interrupt/Resume 是否持久化、幂等并校验权限？
□ Event Stream 是否支持断线续传和消费者去重？
□ Workspace 是否具有 Revision、权限和生命周期？
□ Tool 错误和系统错误是否使用不同处理边界？
□ Runtime Adapter 是否通过协议一致性测试？
```

## 18.4 成本与上下文效率

```text
□ 成本能否按 Run、Wave、Agent、模型、上下文来源和 Tool 拆分？
□ 是否存在重复加载、无效加载和未使用上下文？
□ 稳定前缀是否与动态数据分离？
□ 共享外部数据是否由上游采集一次并生成 Projection？
□ 长知识和代码是否先查索引？
□ 小任务是否走单 Agent 或 Fast Path？
□ 多 Agent 拆分是否有质量与单位成功成本证据？
□ Agent 是否只看到所需 Tool 和 MCP Server？
□ 确定性环境操作是否使用 CLI 或 Script？
□ 无依赖调用是否批量并行？
□ 模型分层是否经过 Eval 并有升级条件？
```

## 18.5 Tool

```text
□ Tool 是否面向任务语义？
□ 是否与其他 Tool 功能重叠？
□ 输入输出是否有界？
□ 是否声明副作用？
□ 是否幂等？
□ 是否需要审批？
□ 是否有选择和参数 Eval？
```

## 18.6 Context

```text
□ 各上下文来源是什么？
□ 哪些是不可信数据？
□ Token 预算和输出预留是什么？
□ 大结果如何外置和按需取回？
□ 是否执行单一表示检查？
□ 压缩后如何追溯？
□ 长期事实如何验证和过期？
□ 领域知识、行为规范和工作流配置是否分开？
□ 知识条目是否有来源、Owner、版本和有效期？
```

## 18.7 Workflow

```text
□ 业务状态是否在代码和数据库中？
□ 外部调用是否在事务外？
□ Task 是否可重试和幂等？
□ 中断后是否可能重复执行？
□ 是否需要 Checkpoint？
□ 是否同时支持链路和单点执行？
□ 流程重量是否匹配任务影响？
□ 执行账本是否独立于传输连接？
□ 并行步骤是否无依赖并隔离写入？
□ 嵌套执行是否有深度、链路和环检测？
```

## 18.8 Security

```text
□ 文件系统边界是什么？
□ 网络出口边界是什么？
□ 凭证作用域是什么？
□ 外部内容能否影响权限？
□ MCP/Skill/依赖来源是否可信？
□ 是否有审计？
□ 是否做过 Prompt 注入和数据外泄测试？
```

## 18.9 Evals

```text
□ 成功标准是否可执行？
□ 是否测试结果和轨迹？
□ 是否运行多 Trial？
□ 是否包含回归用例？
□ 是否衡量成本、耗时和步数？
□ Grader 是否经过人工校准？
□ 置信度是否包含证据覆盖和确定性检查？
□ 冒烟失败是否区分能力、基础设施和测试缺陷？
□ 阻断 Gate 是否优先使用确定性检查？
□ 生成与评估是否需要独立上下文或角色？
```

## 18.10 Production Quality 与 Experience

```text
□ 是否只看单次结果或平均分？
□ 是否报告首次完成率、All-pass@k 和质量下限？
□ 是否计算每个成功任务的综合成本？
□ Trace 是否经过脱敏和 Trajectory 标准化？
□ 候选经验是否来自多个已评估 Trajectory？
□ 经验是否有范围、版本、证据、过期和回滚？
□ 经验召回是否先做权限和适用范围过滤？
□ 经验是否可能绕过安全、审批和业务规则？
□ 是否通过对照 Eval 验证质量和成本收益？
□ 模型、Tool 或规则变化后是否触发重新评估？
```

## 18.11 Loop Engineering

```text
□ 任务是否重复、可触发且有明确 Owner？
□ 结果是否可自动验证？
□ 是否具有受控 Connector 和所需工具？
□ 是否使用持久 Cursor、Watermark 和去重键？
□ 执行是否隔离、可取消、可回滚？
□ 自动重试、总耗时和总成本是否有上限？
□ 验证器能否识别隐藏错误信号的假修复？
□ 生产写入和发布是否有审批策略？
□ 是否提供 Kill Switch 和建议模式降级？
□ 是否区分执行时间和人工等待时间？
□ 是否跟踪复发率、误报率和单位已验证结果成本？
□ Loop 是否通过版本化 Contract 和回归测试发布？
```

---

# 19. 七十八条硬规则

1. 已知业务规则使用确定性代码。
2. 使用能完成任务的最低 Agent 复杂度。
3. 新增 Agent、框架、多 Agent、耐久执行或自治 Loop 必须有 ADR。
4. 每个项目明确 Identity、Orchestration、Context、Gate、Recovery、Evolution 的责任。
5. Harness 组件可替换、可关闭、可评测。
6. P0 和 P1 规则具有稳定 ID、版本和机器执行机制。
7. 安全、权限、状态写入和不可逆动作采用 Fail Closed。
8. 高频违规优先转化为 Schema、Binding、Linter、Hook 或 Gate。
9. 单次失败不能直接升级为全局硬规则。
10. Context 按 Core、Scope、Phase、On-demand 和 Artifact 分层加载。
11. Required Read 和 Context Manifest 可记录、可 Diff、可评测。
12. 成本按 Run、Wave、Agent、模型、上下文来源和 Tool 归因。
13. 稳定指令位于动态任务数据之前，并记录前缀版本和 Hash。
14. 共享外部数据由上游采集一次，下游使用 Projection 或 Artifact。
15. 知识和代码检索先查版本化索引，必要时回退源搜索。
16. Runtime Protocol 与具体框架 Adapter 分离。
17. Agent、Thread、Run、Step、Event、Artifact 和 Checkpoint 具有稳定 ID。
18. 每个 Event、Artifact、Error 和 Approval 归属具体 Run。
19. Run 状态机支持取消、输入、审批、恢复、失败和完成。
20. Thread 与 Run 分离，并明确同一 Thread 的并发 Run 策略。
21. Checkpoint 具有 Schema 版本、Runtime 版本、迁移和兼容窗口。
22. Interrupt/Resume 持久化、幂等并校验权限与状态版本。
23. Event Stream 支持事件持久化、Cursor 续传和消费者去重。
24. Workspace 具有权限、Revision、生命周期和 Run 归属。
25. Runtime Adapter 必须通过协议一致性测试。
26. 所有模型调用经过 Model Gateway。
27. 模型分层路由具有能力门槛、预算、升级条件和 Eval。
28. Agent 具有输入输出 Schema、停止条件、预算和 Eval。
29. Tool 和 Connector 具有 Schema、权限、风险、超时、幂等和审计。
30. Agent 只加载当前任务所需的 Tool 和 MCP Server。
31. Tool 使用共享 Error Contract。
32. 可恢复 Tool 错误作为结构化数据；系统和安全错误由 Runtime 处理。
33. Connector 读取与写入权限分离。
34. 有副作用 Tool 需要审批或明确策略授权。
35. 对外通信和用户数据写入使用专用 Tool、确定性校验和交付证据。
36. 外部内容和 Tool 输出按不可信数据处理。
37. 精确 ID、数组和大数据通过 State、Binding 或 ArtifactRef 传递。
38. 同一来源在单次模型调用中只保留一种表示。
39. 模型调用前执行 Context 预算预检并预留输出空间。
40. Compaction 保留目标、进度、具体值、失败路径、审批和引用。
41. 阶段通过自包含 Contract 或 Handoff Artifact 交接。
42. 模型摘要不能替代规范字段和精确数据。
43. 领域知识、行为规范、工作流配置、能力和经验分开管理。
44. 知识条目具有来源、Owner、版本、范围和有效状态。
45. 自动抽取和用户纠正生成候选知识，审核后发布。
46. 知识变更执行影响分析、回归 Eval 和版本回滚。
47. 冲突或疑似过时知识不能用于高风险自动决策。
48. 步骤状态通过结构化控制表达。
49. Capability 按任务、权限和风险动态裁剪。
50. 确定性环境操作、批处理和测试优先使用版本化 CLI 或 Script。
51. MCP 用于动态能力和资源接入，并执行 Tool 白名单与结果裁剪。
52. Tool Schema 和 Tool Result Token 必须度量。
53. 确定性输出压缩保留错误、退出码、统计和原始引用。
54. Loop 承载方式与编排协议分开选择。
55. 控制平面拥有状态和阻断权；计算平面返回 Contract 与 Artifact。
56. 链路模式与单点模式复用相同能力和 Policy。
57. 流程重量匹配任务影响、可逆性和风险。
58. 小任务使用单 Agent 或 Fast Path；多 Agent 拆分需要收益证据。
59. 并行节点必须无数据依赖并隔离写入。
60. 无依赖 Tool、Connector、Subrun 和测试任务应批量并行。
61. 并行代码修改使用独立分支和工作区。
62. 多 Agent 需要明确准入条件、所有权和隔离。
63. 协调者不能重复 Specialist 的核心职责。
64. 嵌套 Agent、Workflow 和 Skill 执行深度、链路和环检测。
65. Session、Checkpoint、Memory、行为资产、Loop State 和业务状态分离。
66. 执行账本保存可恢复状态；传输层负责事件交付。
67. 错误映射到 Retry、Fallback、Rollback、Need Input 或 Abort。
68. 可恢复 Workflow 的副作用必须封装、可序列化和幂等。
69. 阻断 Gate 固化到 Workflow 或 CI。
70. 确定性检查优先于模型 Grader。
71. Harness 配置、规则、Skills、Workflow、Judge 和 Adapter 作为版本化被测对象。
72. Evaluator 只评估，不补做被测流程遗漏的步骤。
73. 缺失 Case、空产物、超时和依赖失败必须响亮失败。
74. GT、Judge 答案和候选版本标签不能进入被测 Agent 输入。
75. 精确指标由确定性 Grader 计算；LLM Judge 用于语义质量并经过校准。
76. 自报结果与独立环境结果必须比较并记录 Honesty Gap。
77. 成本优化在等价质量与安全门槛下通过多 Trial、消融或确定性对比验证。
78. AGENTS.md 保持简洁、分层、具体、可执行；Skill、Pack、Runtime Adapter、Judge 和 Loop Contract 具有版本与一致性测试。

---

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

## 附录：规范落地与演进

### A.1 规范落地链路

```text
本规范（工程实践总纲）
 → 结合项目 Intent / Plan / 技术栈 / 组织 Policy
 → Rule Resolver
 → 根 AGENTS.md + 模块 AGENTS.md + Skills + Doctor Checks + Evals
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
| 新增强制安全规则 | 必须升级并通过 Doctor |
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

# Agent 工程规范 · 核心原则

> **范围：** §1 核心原则：最简实现、Harness 设计、环境反馈、系统约束优先、约束分级、Loop Engineering、Runtime Protocol、规则编译与拦截、规则文件衰减
> 集合：`standards/` ｜ 导航：[知识库索引](../README.md)

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
| 操作流程 | Workflow、Skill、项目验证门槛 |
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

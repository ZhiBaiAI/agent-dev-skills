# 技术工具架构参考

**Agent 技术选型与架构设计的通用分析参考**

| 项目 | 内容 |
|---|---|
| 定位 | 通用参考，与具体脚手架无关 |
| 用途 | 做新 Agent 项目立项、技术选型、架构设计时，快速判断该选哪类工具、哪层抽象、何种范式 |
| 来源 | 综合业界主流框架（LangChain / LangGraph / Deep Agents、Mastra、AI SDK、OpenSpec 等）的分层理念与实战经验 |
| 与规范的关系 | 本文档是 `knowledge/standards/agent-engineering-standard.md` 的选型认知层；规范定规则，本文档帮你看清工具图谱 |

---

## 0. 文档定位

本文档不推荐具体框架品牌（那些会过期），而是沉淀**选型维度与判断框架**：面对一个新项目，该问哪些问题、按什么维度切分工具层、何时上重抽象、何时回归简单循环。框架和模型可替换，选型维度保持稳定。

---

## 1. 工具栈的三层抽象

业界主流 Agent 工具栈已收敛为三层抽象，控制力与抽象力此消彼长：

```text
控制力高 / 抽象低                              控制力低 / 抽象高
        Runtime  →  Framework  →  Harness
       (运行时)     (框架)        (脚手架)
```

| 层 | 职责 | 典型内容 | 何时选 |
|---|---|---|---|
| Runtime | 图执行引擎、耐久执行、状态机、HITL、容错、流式 | LangGraph、LangGraph 协议兼容实现 | 流程形状本身就是价值；确定性步骤多；需细粒度控制每一步 |
| Framework | 抽象与集成层、最小 agent loop、工具/模型/检索集成、middleware 钩子 | LangChain、Mastra、AI SDK | 要自己组装 harness；延迟敏感；需要精细控制每步的 tool 与 context |
| Harness | 开箱即用的 agent 脚手架，内置 context 工程最佳实践 | Deep Agents、Claude Code 式 harness | 大多数项目起点；要 context 管理、subagent、skills、memory 开箱即用 |

### 关键判据

- **harness 职责 = 在正确时间把正确 context 给到模型**。判断一个 harness 好不好，看它如何管理 context，而不是看它有多少功能。
- **三层完全可组合**：可以把 framework 的 `create_agent` 嵌进 runtime 的 graph，也可以把 runtime 的 graph 作为 subagent 嵌进 harness。不要把三层看作三选一。
- **核心 agent loop 标准化趋势**：随着模型变强，「模型规划 + 调用工具 + 反应结果」的 loop 已足够强大，可以标准化。harness 层因此出现并收敛最佳实践。

---

## 2. 确定性 vs 自主性谱系

三层抽象对应自主性谱系的不同位置。选型的本质是在这条谱系上定位：

```text
高确定性 / 低自主                              低确定性 / 高自主
   Runtime 编码拓扑  →  Framework agent loop  →  Harness 长时 fan-out
```

| 谱系位置 | 特征 | 适合 |
|---|---|---|
| 高确定性 | 业务状态编入图拓扑；模型只做单点抽取/判断 | 文档处理流水线、审批工作流、ETL |
| 中间态 | agent loop 主导；middleware 注入审批/合规/业务规则 | RAG 问答、Copilot、工具型助手 |
| 高自主 | 长时运行、subagent fan-out、内置 summarization/context 管理 | 深度研究、代码 Agent、开放式探索 |

### 判据

- **自主度上界 = 能廉价且可靠验证的量**。验证不了的就不要放给 agent 自主决策。
- **敏感/预设/可重复的任务用确定性**；动态/创造性/探索性任务用自主性。
- **middleware 是中间态的调节阀**：在核心 loop 上挂钩子注入确定性步骤（审批、合规、业务规则），不必上 runtime 全图。
- **runtime 是 escape hatch**：当 middleware 的内置钩子不够灵活时，才下沉到全自定义 graph。

---

## 3. 复杂度阶梯

不论选哪层抽象，都应优先选择能完成需求的最低复杂度：

```text
确定性函数 → 单次模型调用 → 结构化调用 → Tool Loop → 显式 Workflow → 耐久 Workflow → 多 Agent
```

### 选型判断

| 需求特征 | 推荐起点 |
|---|---|
| 输入输出明确、规则固定 | 确定性函数 |
| 单点语义抽取/分类/摘要 | 单次或结构化模型调用 |
| 下一步依赖中间结果、工具选择不能预定 | Tool Loop（framework 层） |
| 固定业务状态机、需事务/审批/审计 | 显式 Workflow（runtime 层） |
| 跨天恢复、人工中断、严格重放 | 耐久 Workflow（runtime 层） |
| 子任务可并行、上下文/权限需隔离 | 多 Agent（harness 层 subagent） |

**反模式：** 同步任务上耐久 Workflow；单轮抽取上多 Agent；「让 Agent 聪明一点」就上向量库。先验证需求，再上抽象。

---

## 4. Harness 的核心组件

一个成熟 harness 通常包含以下组件。选型时逐项对照，判断哪些开箱即用、哪些需要自己补：

| 组件 | 职责 | 选型关注点 |
|---|---|---|
| Context 管理 | 在正确时间把正确信息给到模型 | 是否支持 ArtifactRef、summarization、context budget 预检 |
| Subagent | 隔离上下文做专项工作，不污染主 context | 是否原生支持；fan-out 规模；状态交接 |
| Skills | 按需加载的指令与脚本 | 是否支持渐进加载；触发描述；Token 预算 |
| Memory | 跨运行学习与改进 | 持久化方式；召回策略；过期治理 |
| Filesystem / Artifact | 读写 context 到文件而非塞进窗口 | 是否支持外置存储 + 按需检索 |
| Hooks / Middleware | 在关键节点注入确定性步骤 | 触发时机分类；顺序/超时/幂等 |
| HITL | 人工审批与中断恢复 | 是否原生支持；持久化；幂等 resume |
| Observability | Trace、metric、cost | 是否内置；采样策略；脱敏 |

### 判据

- **harness 的价值在 context 工程，不在功能数量**。一个不管理 context 的 harness 等同于带工具的 chat。
- **优先选开箱组件多的 harness**，自己组装的边际成本高且易过时。harness 维护者会持续 survey 最佳实践。
- **能沉降到 harness 的就别自建**。团队自建的杠杆在组织特有流程与责任判断，不在重造 context 管理。

---

## 5. 规范驱动 vs 对话驱动

Agent 协作开发有两种范式，选型时需明确：

| 维度 | 对话驱动 | 规范驱动 |
|---|---|---|
| 上下文传递 | 即时口述，每次从零 | 结构化注入，一次写好永久生效 |
| 知识复用 | 不复用，每次重教 | 文档化，跨会话复用 |
| 质量保障 | 靠人盯 | 靠 hook + 规范自动检查 |
| 适合 | 一次性任务、探索 | 存量项目增量开发、团队协作 |

### 规范驱动的三层文档

复杂变更应按三层渐进细化，每层独立审查：

| 层 | 回答 | 审查者 |
|---|---|---|
| proposal | 为什么做、做什么、验收标准 | 产品 / 业务方 |
| design | 怎么做、架构、API、组件设计 | 开发负责人 |
| tasks | 任务拆解、优先级、依赖关系 | 执行 Agent |

- **proposal 不提前绑定技术方案**；**tasks 不混入需求变更**。
- **上游层变更时同步更新下游层并记录影响范围**，而非直接改代码。
- **规范层与执行层解耦**：规范是静态文档，执行是 workflow；更换执行工具不影响规范定义。

---

## 6. 「先看再写」原则

不论用哪类工具，Agent 在新建任何可复用产物前，必须先检索项目中已有的同类实现：

```text
正确：实现前 → 检索同类组件/函数/词条 → 命中则复用 → 未命中再新建
错误：拿到需求 → 直接生成新实现 → 联调时发现与项目惯例冲突
```

- **检索范围**：同目录同类组件、utils/composables/hooks、i18n 词条表、类型定义与枚举。
- **命中优先复用**，在新代码中引用而非复制。
- **通过 Skill 或 Hook 强制执行**，而非依赖模型自觉。
- **这是组织特有流程得以生效的前提**——只有先看见项目里已有什么，组织惯例才能被遵循。

---

## 7. 选型决策清单

面对新项目，按以下顺序自问：

### 7.1 复杂度定位

- [ ] 任务是否真的需要 Agent？单次结构化调用是否足够？
- [ ] 是否有固定业务状态机？是否需要事务/审批/审计？
- [ ] 是否需要跨天恢复或人工中断？
- [ ] 子任务是否可并行、上下文是否需隔离？

### 7.2 抽象层定位

- [ ] 流程形状本身就是价值？→ Runtime
- [ ] 要自己组装 harness、延迟敏感？→ Framework
- [ ] 要开箱即用的 context 管理 + subagent + skills？→ Harness
- [ ] 三者是否需要组合？（harness 内嵌 subagent 走 framework loop；framework 嵌进 runtime graph）

### 7.3 Context 工程能力

- [ ] 是否支持 ArtifactRef / 外置存储 + 按需检索？
- [ ] 是否支持 context budget 预检与 summarization？
- [ ] 是否支持 subagent 隔离上下文？
- [ ] 是否支持 skills 渐进加载与触发描述？

### 7.4 确定性注入

- [ ] 哪些步骤必须确定性？（审批、合规、业务规则）
- [ ] 用 middleware/hook 能注入吗？还是必须下沉到 runtime graph？
- [ ] hook 是否支持触发时机分类、顺序、超时、幂等？

### 7.5 可观测与验证

- [ ] 是否内置 Trace、metric、cost 归因？
- [ ] 是否支持 HITL 审批与持久化恢复？
- [ ] 验证能力是否匹配自主度？（验证不了的不要放给 agent）
- [ ] 是否支持 eval / regression 基线？

### 7.6 范式选择

- [ ] 是一次性探索还是存量项目增量开发？
- [ ] 是否需要跨会话知识复用？
- [ ] 是否需要规范层（proposal/design/tasks）与执行层解耦？
- [ ] 是否需要 hook 自动检查质量？

### 7.7 团队能力对照

选型决策之外，用吴恩达 AI 工程技能图谱（2026-08，基于职位发布、结构化专家访谈与问卷分析）对照团队缺口。构建和部署 AI 应用需要六项子能力：

| 能力 | 要点 |
|---|---|
| LLM 基础 | 分词与生成原理、上下文窗口取舍、缓存命中、采样参数、工具调用、模型选型与微调时机 |
| 用数据为模型提供依据 | 提示词内置 vs 工具按需检索、向量索引/知识图谱/语义层选型、文档向 LLM 输入的转换 |
| 构建智能体系统 | 架构选择（串联/并行/代码/LLM）、回退机制、工具（MCP/CLI/沙箱）、记忆架构、多智能体时机 |
| 基于评估的开发 | 评估/错误分析闭环、Grader 组合选型、评估评估本身 |
| 生产环境运维 | 可观测性、漂移检测、对抗性输入响应、统计化回归测试、成本/延迟优化 |
| 机器学习基础 | 偏差/方差、训练与推理权衡、训练与评估数据设计 |

吴恩达将"基于评估的开发"点名为决定 AI 工程师水平的**最重要特质**。选型启示：若团队尚无评估闭环，优先补验证能力（对应 §7.5），再谈放权给 agent 自主度——验证不了的不要放给 agent。

---

## 8. 选型反模式

- **默认上重栈**：同步任务上耐久 Workflow + 多 Agent。选最低复杂度。
- **为「智能」上重抽象**：关键词检索足够却上向量库；单轮抽取上多 Agent。先验证需求。
- **自建可被 harness 吸收的组件**：手写 context 管理、summarization、subagent 调度。用现成 harness。
- **把流程藏在 prompt 里**：固定业务状态机用自然语言描述而非代码。确定性流程用代码。
- **规范与执行耦合**：规范文档直接绑定执行工具。保持解耦，便于替换。
- **不「先看再写」**：拿到需求直接生成新实现。先检索复用。
- **自主度超过验证能力**：放开 agent 自主决策却没有廉价可靠的验证机制。验证上界即自主度上界。
- **middleware 能解决却上 runtime 全图**：escape hatch 是最后手段，不是默认选项。

---

## 9. 主流框架与技术栈对比

以下对比基于公开文档与生态观察，**仅作选型参考**。框架版本快速演进，具体能力以官方文档为准；选型时回归本文档第 1–3 节的维度判断，不要被品牌绑定。

### 9.1 Runtime 层（图执行 / 耐久 / HITL）

| 框架 | 定位 | 核心能力 | 适用场景 | 注意事项 |
|---|---|---|---|---|
| LangGraph | 图式 agent 运行时 | 状态机、耐久执行、HITL、容错、流式、subgraph | 流程形状即价值；确定性步骤多；需细粒度控制 | Python/JS 双栈；学习曲线偏陡；过度用图会变重 |
| Temporal | 通用耐久工作流引擎 | 跨天恢复、重放、计时器、worker 隔离 | 业务工作流、订单/支付、跨服务编排 | 非 Agent 专属，需自己接模型层；运维成本高 |
| Inngest | 事件驱动 step 函数 | step 编排、重试、并发控制、scheduled | 事件驱动 Agent、定时任务、轻量耐久 | 托管为主；step 粒度较粗 |
| Restate | 耐久执行 + RPC | 日志重放、状态持久、virtual object | 需要强一致恢复的后台 Agent | 生态较新 |

**选型提示：** Runtime 层是 escape hatch，不是默认选项。先确认 middleware 能否解决，再下沉到全图。

### 9.2 Framework 层（agent loop / 集成 / middleware）

| 框架 | 定位 | 核心能力 | 适用场景 | 注意事项 |
|---|---|---|---|---|
| Vercel AI SDK | 模型抽象 + 最小 agent loop | model gateway、streaming、tool calling、generateObject | TS 项目默认；Next.js 集成；延迟敏感 | harness 能力薄，需自己补 context 管理 |
| LangChain | 抽象与集成层 + 最小 harness | create_agent、middleware、integrations、工具/检索 | 自己组装 harness；多 provider；多检索源 | 抽象层多，早期 API 不稳定已改善 |
| Mastra | TS agent framework | agent、workflow、memory、RAG、eval | TS 全栈 Agent；需 workflow + memory 一体 | 生态较新；Elastic 协议 |
| OpenAI Agents SDK | OpenAI 官方 agent SDK | agent、tool、guardrail、session、HITL、trace | 深度用 OpenAI 栈；guardrail 一体化 | 与 OpenAI 服务绑定较深 |
| Google ADK | Google 官方 agent 开发套件 | scaffold、enhance、eval、deploy、observe | Google Cloud 部署；全生命周期 | 与 GCP 生态耦合 |
| PydanticAI | 类型优先的 Python agent 框架 | 强类型 IO、structured output、dependency injection | Python 项目；类型安全诉求 | 能力较精简，需自己补 harness |

**选型提示：** Framework 层的区分点在「集成广度 vs 抽象纯净度」。要开箱集成多选 LangChain/Mastra；要纯净控制选 AI SDK/PydanticAI。

### 9.3 Harness 层（开箱即用脚手架）

| 框架 | 定位 | 核心能力 | 适用场景 | 注意事项 |
|---|---|---|---|---|
| Deep Agents (LangChain) | 通用 agent harness | filesystem、subagents、skills、memory、summarization、middleware | 大多数项目起点；深度研究、代码、文档 Agent | 较新，最佳实践持续迭代 |
| DeepSeek Harness (DSH) | model-native、一切皆插件的 agent harness | Cordis 微内核；模型/Tool/Skill/Session/Sandbox/Storage/Loop/调度/UI 全部可替换插件；四运行模式（Standard 全量 / **Code 代码编排** / Minimal 两工具基准 / Creator 自定义预设）；append-only session log 支持 resume/fork/replay | 追求可替换性与模式分层的项目；同构批量调用走 Code mode（见标准 §3.14） | 2026-08 开源（MIT, TypeScript），生态尚新 |
| Claude Code / Cursor 式 harness | 编码 Agent harness | 项目理解、增量开发、工具调用、diff 验证 | 代码仓库内开发；存量项目增量 | 绑定具体产品；自建成本高 |
| CrewAI | 角色化多 Agent | role、task、crew、流程预设 | 多角色协作场景；流程演示 | 默认多 Agent，注意复杂度阶梯反模式 |
| AutoGen | 多 Agent 对话框架 | 角色对话、group chat、code execution | 多视角讨论、研究探索 | 易陷入多 Agent 过度化；验证能力要跟上 |
| OpenSpec + AI Workflows | 规范驱动协作体系 | proposal/design/tasks、skills、hooks、templates | 存量项目增量开发；团队协作 | 需自建执行层；规范层与执行层要解耦 |

**选型提示：** Harness 层的判据是 context 工程能力（见第 4 节），不是功能数量。开箱组件多的优先，自建成本高且易过时。

### 9.4 规范与脚手架层

| 工具 | 定位 | 核心能力 | 适用场景 | 注意事项 |
|---|---|---|---|---|
| OpenSpec | 变更规范管理 | proposal/design/tasks 三层、变更归档 | 存量项目增量开发；需审查对齐 | 仅规范层，需配执行层 |
| Agent Starter | CLI 脚手架 | intent、catalog、resolver、generator、eval | 立项生成项目骨架；确定性选型 | TS 优先；生成项目不依赖 CLI |
| Google agents-cli | 全生命周期 CLI | scaffold、enhance、eval、deploy、observe | GCP 部署；全生命周期 | 与 Google 生态耦合 |

### 9.5 周边能力对比

| 能力 | 主流选项 | 选型判据 |
|---|---|---|
| 模型网关 | Vercel AI SDK、LiteLLM、Portkey、自建 | 统一调用、路由、预算、缓存；优先用现成 |
| 可观测 | LangSmith、Langfuse、Phoenix、Arize | Trace + cost + eval 一体优先；云端 vs 自托管 |
| 向量库 | pgvector、Pinecone、Weaviate、Qdrant | 先确认是否真需要 RAG；pgvector 起步够用 |
| 沙箱 | E2B、Daytona、Modal、自建 Docker | 浏览器/shell 必备；网络策略 + 凭证隔离 |
| 队列 | BullMQ、Inngest、Temporal、SQS | 按耐久性 vs 简单性谱系选；全同步不要上 |
| Memory | Mem0、Letta、自建 | 先判断是否真需要跨会话记忆；过期治理是关键 |

### 9.6 跨层组合参考

- **轻量 Copilot**：AI SDK（framework）+assistant-ui，不上 runtime
- **深度研究 Agent**：Deep Agents（harness）+ LangGraph subgraph（runtime 作为 subagent）
- **业务工作流 + 单点 Agent**：LangGraph（runtime 主干）+ create_agent（framework 做单点判断）
- **存量项目增量开发**：OpenSpec（规范层）+ 任意 framework/harness（执行层）
- **强一致耐久后台**：Temporal（runtime）+ AI SDK（framework 调模型）

---

## 10. 参考来源

| 来源 | 重点借鉴 |
|---|---|
| LangChain / LangGraph / Deep Agents 分层 | Runtime / Framework / Harness 三层抽象；middleware 钩子；subagent 作为 context 隔离工具 |
| vivo OpenSpec + AI Workflows 实战 | 规范驱动三层文档；规范层与执行层解耦；hook 按触发时机分类；「先看再写」原则 |
| Anthropic Context Engineering | harness 职责 = context 工程；ArtifactRef；context budget；渐进加载 |
| Anthropic Building Effective Agents | 复杂度阶梯；Tool Loop vs Workflow 边界；环境反馈 |
| Anthropic Agent Skills | skills 渐进加载；触发描述；Token 预算 |
| 强模型时代 harness 演进 | 规则→判据；路径放开/验收收紧；L0–L3 归属分层；自主度上界 = 验证能力 |
| linux.do 个人开发者 T0→T3 演进（2026-08） | T3 代码编排：分离不确定性与确定性，N 次往返压缩为 1 次；DeepSeek Harness Code mode 产品化印证 |
| Google Antigravity Skills 系列（2026-07/08） | 结构化澄清与偏好水合回环；昂贵产物审阅回环；子 Agent 事件驱动协作契约（反应式唤醒 / 两级断路） |

# 选型参考 · 框架对比

> **范围：** §9 主流框架与技术栈对比、§10 参考来源
> 集合：`selection/` ｜ 导航：[知识库索引](../README.md)

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

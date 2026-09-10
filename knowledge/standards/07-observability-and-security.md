# Agent 工程规范 · 可观测与安全

> **范围：** §9 Observability 与 Trace、§10 安全、Containment 与最小权限（沙箱边界、Brain/Hands/Session、不可信数据、OWASP、AI-BOM、记忆投毒）
> 集合：`standards/` ｜ 导航：[知识库索引](../README.md)

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

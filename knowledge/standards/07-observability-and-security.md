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
- **MUST** 给规则和知识设置"升级门槛"，而不是违反一次就写成硬规则：问题第一次出现先记为记忆，同类问题反复出现才考虑升级为规则；规则每次会话都加载、每多一条就稀释其他规则，而按需检索的经验在未命中时不占注意力。升级决定权留在人手里，agent 只提交建议与证据（实测比例：13 条进化记录里 10 条沉淀为记忆、3 条升级为规则）。
- **SHOULD** 用命中数据驱动淘汰，而不是定期凭印象清理：为知识条目与规则埋点统计实际读取次数与最近读取时间，输出命中排行与从未命中的"僵尸条目"，逐条复核后决定删除、补检索关键词或晋升。埋点脚本必须在任何异常时静默退出，绝不阻塞 agent 的正常读写。

## 9.6 成本可观测与预算控制

- **MUST** 在 Run 开始时解析预算。
- **MUST** 在 Step 和 Tool 边界更新成本。
- **MUST** 在 Soft Limit 触发压缩、范围缩减或模型路由。
- **MUST** 在 Hard Limit 停止或请求审批。
- **MUST** 保留预算变更和批准记录。
- **MUST** 给每一类循环单独设天花板（如修复类循环统一取 3 轮）：修复轮数遵循衰减收益曲线，前几轮解决绝大多数问题，越往后边际收益越低、越修越乱的概率越高。触顶后不硬闯——暂停并把"继续 / 改方向 / 终止"交回人；检测到循环也不直接 kill，先注入反思给自纠机会。
- **SHOULD** 将总成本分解为可独立度量与优化的方程项（用户/会话数、每会话请求数、每请求 token、每 token 单价），并区分两类：采用与参与项目标是增长，agent 自耗项（agent 为自身目的消耗的轮次与 token）是优化对象——成本治理的主手段是消灭零价值 token 消耗，而非降单价或降级工具；工作负载与模型版本连续变化时，锁模型对比才能分离自身优化的收益。
- **SHOULD** 会话成本分析按"反模式 + 财务影响 + 针对性修复"三元组输出（次优模型路由、大 payload 常驻上下文逐轮重计费、缓存失效全价重建、初始化预载开销等），不做聚合指标——聚合数字无法指明行动，单条反模式的成本归因才能驱动修复。
- **SHOULD** 工具输出优化的判据是全任务成本而非单次调用：被删信息重要时 agent 会用恢复轮次（重读原件、重跑命令）找回，局部省、全局贵。压缩按输出类型分级——源码类与任意脚本结果原样保留，搜索结果重组但不丢内容，仅重复性噪声（构建/安装/测试日志）在节省可观时压缩，且保留完整原件与直接恢复路径；恢复路径使用率同时是压缩是否过度的评测信号。删格式先于删信息（无信息量的重复格式是无恢复需求的零风险优化）。
- **MUST** prompt/instructions 的压缩或重写须先为预期行为建立回归评测——未被测试的行为可能被更短的 prompt 悄悄删除（实际案例：压缩把"谨慎并行"指导改写成硬调度策略，子代理被静默串行化）；且优化证据是工作流局部的，同一改动换运行面（离线 benchmark / 在线实验 / 不同产品面）须重新评测后才可推广。
- **SHOULD** 成本优化优先动"模型从不需要的输入"，而非压缩信息本身：先量工具输出与常驻指令各占多少 token（工具输出通常是最大项），再按"删格式先于删信息、保留原件与恢复路径"的顺序处理；**恢复路径的使用率是压缩是否过度的评测信号**。
- **SHOULD** 按"业务成熟度"而非"一刀切"分配预算：成熟业务承担明确的降本目标，处于投入期的新业务保留充足预算以换取速度——统一限流会同时压制该省的和该花的；实测案例中某公司按此区分后整体日均成本降约 65%，而新业务的使用强度未受影响。
- **SHOULD** 用模型选型 + 系统限额两件工具共同治理：选型给出"这类任务该用哪档"的默认，系统限额给出"部门与个人每月最多多少"的硬边界，限额按月调整随业务变化——只靠选型无硬约束，只靠限额会压制合理使用。
- **SHOULD** 把模型选型沉淀成按任务类型的默认分工并让全员共用：轻量模型跑需求明确的实现与日常自动化、中间档跑标准设计与分析、最强档跑复杂判断与架构设计；**默认从轻量档起步、随复杂度升级**比"默认最强、需要时降档"更省，且选型成为共识后个体不必每次自行权衡。
- **SHOULD** 收紧高成本模式（如快速模式）的默认开关：改为按需单次申请而非全局开启，同时用并行等手段补回速度——这类改动常被担心拖慢开发，但实测可以平滑过渡。
- **SHOULD** 为长跑与无人值守运行配备成本护栏与显式停止条件：预算上限过低会产出沉默的结果而非报错，须同时定义"预算耗尽时如何显式暴露"，避免无声降级被误读为正常完成。

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
- **MUST NOT** 让沙箱内产生的值成为主机侧决策的权威输入：快照元数据、校验和、存储地址这类由客机内部产生的字段必须由主机侧重新推导或签名——把客机派生值当权威是可跨账户调换资源的信任边界缺陷。
- **MUST** 让同一决策只有一个执行点：出网管控、权限判定这类横切决策散落在多个子系统时会"彼此意见不一致"，须收敛为**每种数据类型只有一个规范化例程、每类决策只有一个组件**。
- **MUST NOT** 依赖秘密保守来维持边界：需要攻击者不知道某个密钥才成立的边界不构成边界（实践判据：即使未证明存在针对陌生人的攻击路径，也应按缺陷修复）。
- **MUST** 把隔离当作逐层验证的属性，而非架构层面的假设：虚拟机/容器边界成立，不代表其下的存储分配器、卷复用、快照缓存也成立——共享存储上的残留数据应视为首要隔离控制项，"分配时清零"或"释放时擦除"必须显式配置并经独立测试，不能依赖默认值（实测：精简配置启用"分配时不清零"后，64 KiB 块只被写入 4 KiB，其余保留上一租户内容，全部可测目录块均来自他人）。
- **MUST** 在边界设在模型改不动的位置：提示词、hooks、配置文件都是 Agent 能读、极端情况下能改写或无视的软约束；强制点应落在网络、会话、系统层这类绕不过的执行点。软约束只能提高先天合规率，兜底必须由确定性代码完成。
- **SHOULD** 用独立对抗性测试验证隔离边界，并把发现按"是否可访问他人数据"分级——这类测试的价值在于用真实攻击面换回结构缺陷清单，而不是拿到一个"通过了"的结论。
- **SHOULD** 无人值守运行中的凭证用占位符加保险库模式：沙箱内只有占位符，由平台在请求白名单目标时替换为真值；工具审批在无人值守场景不可用，因此凭证须限定为只读并显式放行。

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
- **MUST** 把**知识库与记忆检索结果**同样按不可信数据处理：条目可能被人为投毒、或沉淀了已过期的结论，一旦命中检索就顺着上下文放大——须做来源标注与可信度分级，与权威源交叉核验，禁止把检索片段当"可信内参"直接拼接进系统指令；更前置的做法是把关口左移到**入库前审核**（来源可追溯、准入评审、版本与过期下线），团队级多人可写的知识库尤其如此。
- **MUST** 对进入上下文前的输入做归一化再判定：先解码 Base64、HTML entity、URL 与 Unicode escape，剥掉零宽字符与 RTL 控制符，再交给规则与模型——否则"忽略之前的指令"这类内容可以藏在编码或不可见字符里绕过检测。归一化、来源分槽、检测分级三步叠加，单点检测器会被编码与语义改写绕过。
- **MUST** 把**训练与评测环境本身**当作风险面：含糊或不可能完成的任务会驱使模型追求非预期策略（利用软件缺陷执行命令、绕过访问控制取数据、规避工具限制），而**若训练奖励了我们没打算奖励的行为，模型就学到了那条绕路划算**——须审计奖励设计、把联网评测改为离线或加监控、并对模型行为做主动检测而非等待上报。
- **MUST NOT** 把模型对自己推理过程的叙述当作可靠证据：模型自述的动机与推理链不构成行为归因的证据，判断须依据可观察的动作与副作用记录。

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

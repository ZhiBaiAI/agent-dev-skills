# Agent 工程规范 · 编排范式与人工介入

> **范围：** §3 Agent 与 Workflow 选择（Tool Loop/Workflow/多 Agent 准入、任务分级、协作平面、代码编排）、§6 Human-in-the-loop（审批、Guardrail、结构化澄清、审阅回环）
> 集合：`standards/` ｜ 导航：[知识库索引](../README.md)

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
- **MUST** 交叉验证不能由同源 Agent 互审构成：上下文、脚手架与模型相同（或相似）的 Agent 是低方差的，会犯同一个错，单点坏决策会放大成系统性失败（Frontier Red Team 实测）——评审必须更换上下文起点、更换模型或引入异源证据，否则只是同一个 Agent 的重复抽样，不构成独立验证。

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

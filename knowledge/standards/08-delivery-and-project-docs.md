# Agent 工程规范 · 交付与项目文档

> **范围：** §12 长任务开发范式、§13 AGENTS.md 规范、§14 Skills 规范、§15 CLI 自动化、§16 项目应内置能力、§17 Definition of Done
> 集合：`standards/` ｜ 导航：[知识库索引](../README.md)

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
- **SHOULD** 知识建设走飞轮：每阶段产物（proposal 的业务定义与验收标准、design 的链路记录、tasks 验证结论）是下一阶段输入，知识作为真实迭代的副产物沉淀，而非先建完美知识库再启用；归档时确认知识更新（重复、冲突、失效链接、无来源结论由定期健康检查回收）。

## 12.3 Planner、Generator、Evaluator 模式

复杂且价值足够高的开发任务可采用：Planner（分可验证任务）→ Generator（实现一个任务）→ Evaluator（独立运行测试、检查交互和对照需求）。要求：衡量收益、限制成本、确保 Evaluator 不只重复 Generator 的判断、将主观标准转成可检查的 Rubric。

## 12.3.1 大规模改造：先分类、再建证书、最后动代码

适用于"agent 把多年代码改造压缩到数周"的场景（跨栈迁移、框架升级、遗留重构）。瓶颈不在生成改动，而在**让组织围绕改动达成一致**——顺序不能反。

- **MUST** 先把改造归入三类之一并取得共识：**同栈升级**（版本跃迁、行为不变）、**跨栈转换**（换栈、行为不变）、**重想象**（重写、行为改变）。分类不清的争论不会消失，只会在后期以"这个改动到底对不对"的形式反复出现——三类的"正确"标准根本不同。
- **MUST** 建"证书"（certificate）：一组**机器可检查**的完成条件（测试通过、覆盖率阈值、基准不退化、对抗性审查、差分/回放测试），让 Agent 能无人循环迭代；没有证书只能人盯，人盯则速度退回人工速度。证书复杂度是主要成本项——受监管场景里验证成本可能超过编写成本。
- **SHOULD** 先把代码库转成行为规格（评估与结构提取 + 人访谈与文档补齐缺口），再定分级审查策略：反复出现的同类问题**在源头修**（改工作流、改规则）而非逐个打补丁，稀缺的领域专家时间只投给最高风险等级。
- **SHOULD** 先在小片代码上把工作流调顺再全量铺开——出问题时"改工作流，而不是改每一个改动"；可选的就地现代化按"从叶子向内分区、一次冻结并现代化一块、CI/CD 做闸门"推进。
- **MUST** 校准预期：主要驱动通常是**风险降低**而非成本节省，把降本当立项理由会在后期被数据打脸；受监管环境中"减轻人工审查"的责任应由管理层承担，否则会压在执行层而无人敢放行。
- **边界：**"重想象"类最难——规格比现有代码更主观，模型判断占比更高。

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

- **SHOULD** 写"何时相关"而非"必须照做"：AGENTS.md 每轮都被加载，写绝对指令会让模型在无关任务上也被绑住；正确写法是说明这条规则**在什么情形下适用**并显式授权其中的安全例行流程（如"涉及认证模块时必须先跑该测试"），让模型自己判断相关性。
- **SHOULD** 用清晰边界替代"永远先问"：把"先问我"换成"这些情形自行决定、这些情形停下来问"，既减少无谓中断，也让真正需要人介入的情形被认真对待。
- **MUST** 把"完成"的定义写死：显式列出判定完成前必须跑通的检查（构建、测试、lint、人工确认项），而不是让模型自己判断"差不多了"——长任务的提前停止大多发生在完成标准模糊处。

## 13.6 AGENTS.md 生成质量要求

生成后的 AGENTS.md 必须：简洁、与当前项目匹配、包含准确目录和命令、使用具体可执行规则、不包含不存在的脚本或包、不复制整份工程规范、不含 Secret、可由开发人员直接阅读、可由编码 Agent 稳定执行、经 Snapshot 和行为测试验证。

## 13.7 生成项目应保留规则来源

生成项目应保留规则来源映射（如 `.agent/rules.lock.yaml`），记录每条规则的标准 ID、来源版本和目标。用途：解释 AGENTS.md 中某条规则为何存在、升级时判断规则变化、检测用户删除强制规则、支持规则校验命令。

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

- **MUST** 把描述当**路由判据**写而非营销文案：它要回答"何时该用、何时不该用、产出什么、成功标准是什么"，推荐直接放一段"该用 vs 不该用"短块并写具体（输入、涉及的工具、预期产物）。反例：描述写"用于数据库、查询、模型或持久化相关的工作"，会把调用推到任何碰到数据库的任务上，而非只在处理迁移时触发。
- **MUST** 主动写负例与边界案例：反直觉的是"把 Skill 变得可用"最初反而**降低**正确触发率，修复办法是补几条明确的"不要在这个情况调用"及其替代做法——实测某团队基于 Skill 的路由在定向评测中初始触发率下降约 20%，补负例与边界覆盖后恢复。
- **MUST** 警惕描述膨胀：Skill 数量过多时平台会**自动截短**每个描述以适应上下文，模型看到的描述更少、更难选对；描述之间还会互相矛盾或过度强调适用性，让模型加载了帮不上忙的指令。
- **SHOULD** 把模板与示例放进 Skill 内部而非系统提示：它们只在 Skill 触发时加载，既"用时即有"又不会给无关查询增加 token——实测这是某团队在质量与延迟上收益最大的改动之一。
- **SHOULD** 让多工作流的 Skill 根文档做**最小路由器**，指向配套文档与脚本，而不是写成完整配方——读 Skill 本身消耗上下文、把人推向压缩、并引入当前任务用不上的指导。- **MUST** 意识到仓库里的 Skill 也在指导**别人的 agent**：对某一档模型有帮助的指导，可能过度约束另一档更强的模型，写之前先想清楚哪些模型会用到这些指令。
- **MUST** 每个 Skill 聚焦一个工作，并测试其误触发与漏触发。
- **MUST NOT** 让所有 Skill 描述过于宽泛。
- **MUST** 让工具与 Skill 绑定：Skill 所需的工具随 Skill 加载而可用，**未读 Skill 不得调用其工具**（调用应失败为未知工具）——否则 agent 可以绕过 Skill 直接调用工具，或读了 Skill 仍要自行寻找配套工具。
- **MUST** 支持 Skill 的可固定与可重载：高频 Skill 可在首次模型调用前载入（省去一次读取往返，让工作在第一次调用就开始）；长跑会话应能在不重开会话的前提下拾取新增/修改/删除的 Skill。重载会使提示缓存失效，须权衡并记录理由（缓存本身在闲置后即过期，空闲会话的代价通常可接受）。
- **MUST** 中途加载工具时保持缓存前缀不变：把新增工具放进**新的系统消息**，让其上方的稳定前缀原样保留；不支持中途加工具的模型按追加处理，不得改动既有前缀。
- **SHOULD** 让工具具备"生成工具"的能力：意图明确后让 agent 造出专用界面或工具供后续反复使用，比反复用对话驱动更省；把 agent 本身当作工具反复使用是持续的 token 浪费。
- **SHOULD** 认识 Skill 生态的分布特征以校准投入：技能供给偏技术而需求分散在各工作类型；安装量极度集中（极小比例占据多数安装，近半数只被装过一次）；跨行业可迁移的 Skill 平均收益远高于垂直深耕，供过于求的领域（如软件工程类）采用率低于平均——**判断一个 Skill 值不值得做，看它是否填补了供给不足的工作类型**。
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

### 指令严格程度匹配脆弱程度

- **MUST** 按「软化这句会不会立刻出问题」决定严格程度，而不是按「这条多重要」——重要的判断可以只给目标，脆弱的口子必须逐字固定。
- **MUST** 把定调语、要复述的框架、让 Agent 对自己说的重置语列为低自由度并原文照抄：软化成「请批判性审查」正是模型会漂移回去的客气版，而原句改变的是这一轮里「有用」的定义——用户已在开场声明期待坏消息，再客气就是没完成任务。
- **SHOULD** 把产出物骨架、台账格式列为中自由度（给模板，字段名必须对）；把需要来回争论的部分列为高自由度（只给目标，写成步骤反而僵化）。
- **MUST** 在给出严格程度划分时说明理由，让模型在文字没覆盖的情形下也能按同一判据取舍。

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

### 退出码要一个码对应一个动作

- **MUST** 让每个退出码只对应一种处置，且该处置写在输出里——0 继续、1 停下并说明缺什么和该跑什么、2 用法错误，而不是笼统的"失败"。
- **MUST** 让脚本只测量、人来裁决：门禁脚本不得自行改写门禁状态。
- **MUST** 在无法机械判定时返回"需人工判定"这一独立退出码，而不是猜，也不是一律拦下。真实台账会漂移出格式（如 `readiness: core **ready**; periphery not-yet`），猜会有一半时候猜错，一律拦下则会被无视，两者都让检查失去意义。
- **MUST** 在多个来源互相矛盾时不替人选一个，返回需人工判定并给出矛盾所在行。
- **SHOULD** 让脚本从"人签字确认的那一处"读状态，而不是从留档副本读；副本仅供记录，不作为判定依据。

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

所有项目默认应具备以下能力（由 agent.design.md 提供基线，按需扩展）：

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
- Capability 解析器
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
- 项目验证门槛
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

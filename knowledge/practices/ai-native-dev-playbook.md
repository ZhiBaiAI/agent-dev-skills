# AI 驱动开发实践指南（Vibe Coding Playbook）

这是团队的 L2 开发实践：日常用 AI（Claude Code / Codex 等 Coding Agent）开发应用与 Agent 项目时怎么协作、怎么搭项目环境、怎么验收。内容提炼自 [agent-development-lessons.md](../cases/agent-development-lessons.md)（下称 lessons）与 [agent-engineering-standard.md](../standards/agent-engineering-standard.md)（下称标准），出处以（lessons §x.x / 标准 §x.x）标注，细节回原文件查。

***

## 1. 心智模型

- **三层公式：** `业务研发 Agent = Model + 通用 Coding Agent Harness + 项目 Harness`。模型和通用 Harness 是商品，项目 Harness（业务知识、流程约束、完成标准）才是自己的东西。（lessons §10.10）

- **瓶颈是上下文，不是模型能力：** 模型决定通用能力，上下文决定它在具体项目里走多远。靠人在对话里重讲，上限就是对话长度和个人表达能力。（lessons §10.10）

- **代码是中间产物，交付链才是目标：** Agent 写代码越快，计划、评审、测试、部署这些"人速步骤"越容易成为瓶颈——项目 Harness 要保证读对、拦住、接得上。（lessons §10.8, §10.7）

- **委派结果，不指挥步骤：** 描述最终状态让系统规划路径并回报 trade-offs；模型能力升级后重新校准委派边界，敢提上一代模型时代"不合理"的需求，但每次放大委派范围同步放大验证与运营控制。（lessons §10.6）

- **复杂度阶梯：** 优先选择能完成需求的最低复杂度；普通代码可表达的分支循环不用模型，不因为有大模型就上 Agent，不因为步骤多就上多 Agent。（标准 §1.1）

## 2. 人机协作范式

- **任务分级：** Minor（影响小，直接做）/ Standard（常规，按流程）/ Major（架构级，走完整方案评审），不因任务描述简短而降级处理。

- **结构化澄清：** 遇歧义用有限选项（≤7 项）+ 默认值 + 可跳过，不做自由反问；关键决策（数据迁移、API 破坏性变更）先确认再动手。

- **plan-then-execute：** 实现类任务先产出实现计划（变更文件清单、执行顺序、风险、验证方法），人审核计划后再放行执行——人的审核点放在计划层，不放在逐行代码层。（lessons §10.7）

- **工件链：** 复杂变更走 proposal（为什么做/做什么/验收标准）→ design（怎么做）→ tasks（拆解执行）三层渐进细化，每层独立评审；产物版本化、人机共读，是阶段交接契约而非一次性 prompt。**阶段不自动越过**：proposal 未确认不写 design。（标准 §12.2；lessons §10.7）

- **验收标准先行且机器可检查：** 测试绿、产物存在、diff 为空、退出码 0；"怎么做"留给 Agent，"做到什么程度算完成"必须事先说清。

- **回答 ≠ 负责：** Agent 说"已完成"只是声明；完成由外部证据确认（构建产物、测试报告、部署回查）。

## 3. 项目 Harness 三件事

### 3.1 可信上下文（读对）

- **AGENTS.md 是 Router 不是百科全书：** 只放项目约定总入口和导航，收敛为三跳（AGENTS.md → context/README → 专题文档/源码），每跳只收窄问题空间。好的 Context System 首先是 Navigation System。（lessons §10.8）

- **已存在 ≠ 已生效：** 本地代码含修复不等于线上消费的是它。重要结论带现场：claim / source / revision / scope / observed\_at / verification，判断以实际消费的 revision 为准。（lessons §10.8）

- **规则文件防膨胀：** 两级目录封顶、索引先行（title+description 一行一条）、不确定的显式标"待补齐"；规则变更需人审，对话按功能切分防上下文稀释。

- **prompt vs Skill 按频率分配：** ≥1/3 流量需要的指令进 system prompt（加载 Skill 要花一轮模型调用），其余进 Skill；可由已有信号（如用户来源页）预测的 Skill 由 harness 预载、跳过加载轮；安全/法务约束和关键用户事实永驻 prompt。多域能力优先单 Agent + Skills 而非按域拆子 Agent——跨意图强耦合的会话每次 handoff 都丢共享上下文还多花 token。（lessons §10.11）

### 3.2 可执行约束（拦住）

- **判据是违反后会发生什么：** 条件不满足仍能继续 = 提醒；条件不满足下一步无法进行 = 约束。可机械判断且出错代价高的要求（报告是否本次生成、测试目录是否一致、必测场景是否真跑到、是否推保护分支）必须接到执行链上（门禁脚本/动作前拦截），不依赖 Agent 记得——"把 prompt 写得更严厉"不改变流程性质。（lessons §10.8）

- **交付链 vs 判定链：** 交付链证据门禁严格（防假绿）；判定类任务单步失败可降级继续（输出带缺口标注的结论），前提是缺口显式暴露给下游。（lessons §10.9）

- **结论要不要负责：** 需对特定人/规则/检查负责的输出（审核、合规、权益判定）才为确定性付 Harness 工程成本；纯生成、探索性任务不上。（lessons §10.9）

### 3.3 可恢复流程（接得上）

- **换 Agent 检验：** 假设每个阶段结束后换全新 Agent，只靠项目状态、文档和真实工件（非会话历史）还能继续吗？不能则流程仍绑在对话上。（lessons §10.8）

- **交接合同：** 各阶段输入来自已确认事实或上一步产物（不从聊天记忆猜），输出落到文档/代码现场/报告（不只留一句总结）。

- **恢复任务 ≠ 恢复会话：** state 文件是书签（当前阶段/状态/路径）不复制正文；凭持久化信息重建上下文，不试图找回上一段对话。（lessons §10.8, §4.10）

## 4. 知识库与上下文管理

- **本地优先：** 项目需要的信息进同一个文件夹用 Git 管理（版本/评审/来源/回滚全齐），不建平台；**动静分离**——稳定知识进 Git，动态事实（工单状态、测试环境、日志）从权威系统实时读，配置写着某环境 ≠ 代码已部署。（lessons §10.10）

- **载体选文件 Wiki：** 判据是更新路径最短——知识、文档、测试、索引进同一个 MR 一起评审共享版本；RAG/图谱作检索补充不作主知识源。（lessons §10.10）

- **知识边界：** 只存"代码之外影响业务和技术判断的信息"（业务规则口径、跨系统链路、新旧切换、废弃状态），不复制代码可表达的内容——逐方法解释的文档生成快失效也快。文件带 Metadata（至少 status/version/source），读正文前先判适用性。（lessons §10.10）

- **知识飞轮：** 不追求完美知识库再启用；每个迭代的产物（业务定义、链路记录、验证结论）是下阶段输入，知识作为真实迭代的副产物沉淀，归档时确认更新。（lessons §10.10）

- **能力观察规则（何时沉淀）：** 只在出现明确信号（重复劳动、反复纠正、反复找同一上下文）时才建议沉淀一项（Runbook/Skill/Script/Gate），只建议不自动改；一次问题先修复，重复路径再沉淀，能机械判断的最后才变约束。（lessons §10.8）

- **记忆异步抽取：** 会话记忆（偏好、事实）由会话后的抽取器离线写入，不放在对话热路径上同步生成；写路径带验证——防止对话里的临时表述直接污染长期记忆。（lessons §10.11）

- **人工介入即缺口信号：** Agent 卡住求助或犯错时，问"我知道什么它不知道"——每次介入暴露一个知识/数据缺口，把缺口补进上下文或知识库，让这次介入成为最后一次（Kavak：辅导 Agent 过河的同时产生它可学习的 trace）。（lessons §10.13）

## 5. 验证与评审

- **分层验证一次跑全：** 静态检查 → 单测 → 构建 → 集成 → E2E 用一条命令执行；Agent 会话内自测自修直至通过（反馈循环），不靠人当测试员。（标准 §7）

- **假修复检测：** 验证须全部跑、顺序一致、引用日志原文；修复须定位根因（改前复现改后消除）、附原失败用例回归，禁止掩盖性修复（吞异常、放宽断言、跳过用例）。（标准 §7.6）

- **Code Review 只报可行动的 finding：** 开发者能采取行动的问题才报；风格偏好、假设性设计问题（"如果未来需求变了"）、无法验证的担忧不报；不做过度设计验证，不质疑已定案的架构。输出按「必须修 / 建议修」两级，每条附位置和修复建议。

- **评审对象从代码行转向意图与取舍：** 大型变更交付物附 intent 说明（改动意图、关键 trade-offs 及理由），人质询意图层面问题，Agent 代查代码细节。（标准 §9.x；lessons §10.6）

- **防刷指标：** 自优化环节禁单一指标——对冲指标配对、评测集冻结（变更走审批，被优化方无权改）、目标定期人复核。（标准 §7.10；lessons §5.12）

- **评测主指标判定：** 每类任务定一个主指标决定通过与否，其余指标单独衡量定位短板，不做全指标 AND；上游错误（如路由错）时下游指标记 skip 而非 fail，不因上游污染失真。（lessons §5.13）

- **扇出采集 + 对抗性审查：** 大范围找 bug、性能排查、方案调研类任务先扇出广撒网采集候选，再用独立的多视角审查逐个评估过滤误报，人只审过滤后的问题集——不直接消费扇出原始产出（信息量超出人的处理能力）；确定性循环保证每项被同一技术审查，这是信任来源。（标准 §3.14；lessons §10.12）

- **渐进信任验证：** PR 附测试 + 结果截图，必要时让 Agent 录屏演示使用过程；人从"每次亲自验证"逐步过渡到抽查——信任随验证闭环质量扩大而非一次性给足，验证质量稳定后可继续放大委派范围。（lessons §10.12）

## 6. 安全与部署

- **高风险动作前拦截：** 推保护分支、强推、跳过检查、删库级命令、生产数据写操作，执行前机械拦截；"Agent 说我会小心"不能代替检查。（lessons §10.8）

- **部署用授权 gate：** 敏感动作用 hook/脚本做机器强制审批（特定人员确认才放行），不依赖 prompt 约定。（lessons §10.7）

- **完成标准对齐外部证据：** 代码合入只是一步，还要回查工作项状态、监控指标、健康检查——代码版本、业务状态、交付证据一致才算完成。（lessons §10.7, §10.10）

- **秘密不进上下文：** 凭据走环境/密管，不进 AGENTS.md、不进知识库、不进会话记录。

- **副作用变更 stage/apply 分离：** 模型只生成带服务端 ID 的暂存变更（购物车、订单、配置），apply 仅对经真实界面或策略批准的 ID 生效，且 apply 时按当前状态与限额复查——多轮间环境已变时不沿过期前提直接落单。（lessons §10.11）

- **超时≠失败，先查状态再重试：** 写带副作用的工具调用时——服务端可能已成功但客户端报错，盲目重试=重复副作用（退款执行两次）。请求带幂等键，超时后先用请求 ID 查状态确认真实结果，未知状态清空前不重发；重试设上限、并行设扇出上限、持续失败熔断防级联。（标准 §5.6；lessons §6.6）

## 7. 成本与效率

- **子 agent 默认降档：** 定义明确的子任务用低成本模型（主模型/协调者用强模型兜底），是单项影响最大的成本杠杆；允许手动覆盖。（lessons §8.5, §3.9）

- **code-mode 批量编排：** 轮询、批量重复操作放子进程脚本只回摘要，多次模型往返压缩为一次（省 50-90%）。（标准 §3.14；lessons §8.5）

- **上下文瘦身：** MCP 工具 schema 按需加载不预载；长会话及时压缩；稳定内容做缓存前缀。

- **缓存分层：** 跨会话稳定的内容（system prompt、工具定义）做全局缓存前缀，会话内稳定内容做会话级前缀，易变内容放末尾——稳定内容最大化缓存命中。（lessons §10.11）

- **优化全任务成本而非工具调用成本：** 工具调用便宜但任务没完成，已花的 token 全是浪费——成本指标以"完成任务"为分母；prompt 压缩是行为变更，改动手动回归核心场景。（lessons §8.6）

- **成本可见：** 会话/任务成本实时可见，异常时能定位到反模式并给出针对性修复（反模式 + 财务影响 + 修复动作三元组）。（lessons §8.5）

- **留痕零开销：** 可解释性记录用开关控制（默认关、nil 降级、按需全量开），"需要时能查"与"平时不拖累"同时成立。（lessons §10.9）

## 8. 常见反模式与对策

| 反模式                      | 对策                                                          |
| ------------------------ | ----------------------------------------------------------- |
| 假绿/假修复（报告绿但没跑/掩盖失败）      | 阶段门禁查证据；验证全部跑、引用日志原文（标准 §7.6）                               |
| 把 prompt 当约束（写得严厉就以为拦住了） | 判据"违反后能否继续"，可机械判断的接执行链（lessons §10.8）                       |
| 流程绑在对话上（换会话就断）           | 换 Agent 检验 + 交接合同（lessons §10.8）                            |
| 知识库复制代码 / 越建越大           | 只存代码外信息；两级目录封顶；索引先行（lessons §10.10）                         |
| Harness 只增不减             | Add/Thin 都由运行证据决定，定期删无价值组件（lessons §10.8）                   |
| 复制别的项目长成的 Harness        | 带走方法和原则，节点/Gate/脚本重新生成（lessons §10.8）                       |
| 停不下来 / 转不起来              | 停止语义五层分解（模型流/工具批/turn/driver/Goal），idle ≠ 完成（lessons §4.10） |
| 刷指标（Goodhart）            | 对冲指标 + 评测集冻结 + 目标人复核（lessons §5.12）                         |
| 为降延迟牺牲智能                 | 错的是降到 Low，不是分层本身（lessons §8.1, §8.5）                        |
| 把 AI 采纳率当效果指标            | 用 spec→plan 转化率、PR 首次过 CI 率、自治修复闭环时间（lessons §10.7）         |

## 9. 团队落地路径

1. **从最小基线开始：** 先跑一个真实、简单、可验收的任务（含目标/范围/约束/完成条件）建基线，不先加多 Agent、长 Prompt、复杂工作流；失败后沿执行链定位问题最早出现在哪一层，不先改 Prompt。（lessons §10.8）
2. **观测驱动生长：** 重复失败→加证据门禁；无法恢复→加状态存档；目录污染→加工作区隔离——每个组件都有明确触发它的那次失败。（lessons §10.10）
3. **老系统冷启动：** submodule 关联代码仓、旧文档收进 raw/（原始事实，不当已确认规则）、AI 听记采访核心成员整理入库。（lessons §10.10）
4. **预判 Harness 收缩：** 文件搜索、代码理解会被模型和 Coding Agent 产品逐步吸收；不会消失的是四样——业务知识及事实治理、项目规则与决策边界、领域工具适配、验证标准与质量责任。角色向 FDE（深入现场连接问题、知识、系统、交付）演进。（lessons §10.10）
5. **共识边界：** 团队间需要统一的是数据协议与状态/证据/安全/度量口径，不是统一的流程实现。（lessons §10.10）
6. **每周交付+持续项目底盘：** 保持每周用新模型 ship 点东西、保持一个持续迭代的小项目作为测试每个新模型的底盘——模型不是可互换的（形状差异真实存在），选型直觉来自大量亲手使用而非 benchmark 表格。（lessons §10.13）

## 10. 参考来源

本指南提炼自以下案例，案例全文见 [agent-development-lessons.md](../cases/agent-development-lessons.md)（下称 lessons，章节号即下表对应案例），规范条目见 [agent-engineering-standard.md](../standards/agent-engineering-standard.md)（下称标准）。

| # | 案例 | 来源 | 链接 |
|---|------|------|------|
| 1 | 委派结果与意图评审（Anthropic Labs 工程与组织方法） | Mike Krieger 访谈 | https://www.bestblogs.dev/video/8d3ae5678 |
| 2 | AI-Native SDLC（intent/spec/plan 工件链 + hooks 审批 gate） | Anthropic | https://claude.com/blog/the-ai-native-sdlc-playbook |
| 3 | 项目 Harness 三件事（读对/拦住/接得上） | 蓝翔（腾讯） | https://mp.weixin.qq.com/s/fV8qN6qs9ac-VXDwZCuaxA |
| 4 | 确定性 Harness（阶段化编排 + 零开销留痕） | 左昊（腾讯） | https://mp.weixin.qq.com/s/rQYSuTF98-xGcdgGPuNHjQ |
| 5 | 项目 Harness 完整研发闭环（Price360-KB） | 默达（淘天） | https://mp.weixin.qq.com/s?__biz=MzAxNDEwNjk5OQ==&mid=2650545626&idx=1&sn=cfd0d3011972881686bbb9bedbc319da |
| 6 | Software Factory 成本工程（消灭零价值 token） | Uber Engineering | https://www.uber.com/blog/running-a-software-factory-efficiently-at-uber-scale/ |
| 7 | Agent Teams 协作机制（通信通道 ≠ 协作语义） | 蒋泽林（千问AI平台） | https://mp.weixin.qq.com/s/T_sYOS11KrOijp_aCEcgnQ |
| 8 | Loop 停止语义五层分解（DSH/Pi） | 若飞 | https://mp.weixin.qq.com/s/60H9httJacoMWPgbHG6SJg |
| 9 | Graph Engineering（Loops watching loops + 评测集冻结） | 姜剑（千问AI平台） | https://mp.weixin.qq.com/s/BSCzaVPaX7W5E8vrVrVC0g |
| 10 | 精细化评测（主指标判定 + Judge 四原则） | 砚东（AliExpress） | https://mp.weixin.qq.com/s?__biz=Mzg4NTczNzg2OA==&mid=2247511370&idx=1&sn=c9f4ff1d054cb229ac2f8c1462fcb05e |
| 11 | 成本效率（本地指标陷阱 + prompt 行为回归测试） | GitHub Copilot 团队 | https://github.blog/ai-and-ml/github-copilot/how-we-make-ai-coding-more-cost-efficient-without-sacrificing-task-quality/ |
| 12 | 四个获奖工程模式（双向 MCP / 事件驱动并发 / 同标准 fallback / 分层路由） | Sergio Villani（Google） | https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/ |
| 13 | LLM-as-Judge rubric 四原则 | Jan-Felix Schmakeit（Google AI） | https://dev.to/googleai/how-to-write-reliable-rubrics-for-llm-as-a-judge-evaluations-ndp |
| 14 | 商务 Agent 解剖（单模型+skills + stage/apply + 记忆异步抽取 + 缓存分层） | Anthropic | https://claude.com/blog/the-anatomy-of-effective-commerce-agents |
| 15 | 指挥目标与扇出对抗审查（70-80% 工作在 agent + 渐进信任验证） | Claude Code 团队 | https://www.bestblogs.dev/video/9899b4cdb |
| 16 | AI Agent 就是分布式系统（超时=未知 + 记忆当缓存 + 审批绑定参数） | Salman Munaf（TikTok SRE） | https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2651292485&idx=1&sn=6e8d3295322532b385527234117b84a4 |
| 17 | 级联循环组织观（Agent 攀登人类选山 + 人工介入即缺口信号 + 收益上限选型） | Anish Acharya（a16z） | https://www.bestblogs.dev/article/9f27a9b6d2 |


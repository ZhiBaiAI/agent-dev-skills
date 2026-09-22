# Agent 经验 · 多 Agent 与耐久执行

> **范围：** §3 多 Agent 协作、§4 长任务/耐久执行
> 集合：`cases/` ｜ 导航：[知识库索引](../README.md)

## 3. 多 Agent 协作

### 3.1 扁平对等协调全面失败——锁机制让 20 个 agent 退化成 2-3 个吞吐

- **来源：** [Cursor — 扩展长时间运行的自主编码能力](https://cursor.com/cn/blog/scaling-agents) ｜ 2026-01-14
- **问题：** 最初让所有 agent 对等、通过共享文件自行协调，用锁防抢占同一任务。失败方式：① agent 持锁太久或忘记释放，锁成瓶颈，20 个 agent 有效吞吐降到 2-3 个；② 系统脆弱——agent 持锁时失败、重复获取已持有的锁、未获锁就改协调文件。换成乐观并发控制更健壮但深层问题仍在：无层级时 agent 变得过度规避风险，专挑小而安全的改动，没人承担难题，长时间空转无实质进展。
- **原因：** 大型项目工作拆分一开始不清晰，动态协调缺乏层级会导致责任真空。
- **解决方法：** 拆成 Planner（持续探索代码库、创建任务、可递归派生子 planner）/ Worker（只管把分配的任务做到底，不互调不关心全局）流水线。每周期末 review agent 判断是否继续，下一轮从干净状态重启。教训："最好的系统比你想的更简单"——最初借鉴分布式计算/组织设计的模型并非都适用；合适的结构化程度在两端之间。

### 3.2 多 agent 编译 Linux 内核全卡同一 bug——并行度失效

- **来源：** 同上《Building a C compiler with parallel Claudes》
- **问题：** 测试套件 99% 通过后让 16 个 agent 编译 Linux 内核，但内核是"一个巨型任务"，每个 agent 撞同一个 bug、修复后又互相覆盖，16 个 agent 没用——都卡在同一任务。
- **原因：** 独立测试可天然并行，但单体大任务不行。
- **解决方法：** 用 GCC 作"在线已知正确编译器 oracle"——随机用 GCC 编译大部分内核，只把剩余文件交给 Claude 的编译器。内核能跑→问题不在 Claude 的子集；崩了→二分细化重新用 GCC 编部分文件。这样每个 agent 能并行修不同文件的不同 bug。之后还要用 delta debugging 找"单独能跑但组合失败"的文件对。

### 3.3 多 Agent 放大 eval 污染——意外解率是单 agent 的 3.7 倍

- **来源：** [Anthropic — Eval awareness in Claude Opus 4.6's BrowseComp performance](https://www.anthropic.com/engineering/eval-awareness-browsecomp) ｜ 2026-03-06
- **问题：** 同一批 1266 个 BrowseComp 题，单 agent 配置意外解率 0.24%，多 agent 配置 0.87%，**3.7 倍**。
- **原因：** 多 agent 不改变模型倾向，但更高 token 用量 + 每轮多个并行搜索者，提高了"至少一个 agent 撞到泄漏材料或开始怀疑自己在被测"的概率。
- **解决方法：** URL 级 blocklist 不够（Opus 会绕路）；最有效的缓解是直接屏蔽所有含 "BrowseComp" 变体的搜索结果。呼吁社区把 eval 完整性当作"持续的对抗性问题"而非设计期问题。

### 3.4 多 Agent 间"传话游戏"——电商站缓存查询成永久污染

- **来源：** 同上《Eval awareness in BrowseComp》
- **问题：** 发现新型 agent 间污染——电商网站会把搜索 query 自动生成永久页面（即使零匹配），URL slug 里嵌着之前 agent 的搜索假设。后续 agent 读到 URL 路径，等于看到了前人的"思考痕迹"。
- **原因：** 每个搜网的 agent 都在留痕，网在慢慢积累过往 eval 运行的永久记录。
- **解决方法：** 一个 agent 自己诊断出来了："Multiple AI agents have previously searched for this same puzzle, leaving cached query trails on commercial websites that are NOT actual content matches."——这是新发现的污染向量，尚无完美解，需社区关注。

### 3.5 GAN 式 generator/evaluator——但自评不可信

- **来源：** [Anthropic — Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) ｜ 2026-03-24
- **问题：** 让 agent 自评工作质量时，它"自信地夸自己"，即使人看来明显平庸。主观任务（设计）尤甚；即便有可验证结果的任务，agent 判断力也常碍事。
- **原因：** LLM 对 LLM 生成物天然宽容。
- **解决方法：** 把"做事的 agent"和"评判的 agent"分开。分离本身不消除宽容，但调一个独立的 evaluator 让它 skeptical 比让 generator 批评自己 tractable 得多。具体做法：写 4 条评分标准（Design quality / Originality / Craft / Functionality）同时给 generator 和 evaluator；用 few-shot + 详细打分分解校准 evaluator；evaluator 用 Playwright MCP 实际导航页面再打分；5-15 轮迭代。全栈场景扩展为三 agent：Planner / Generator / Evaluator，每 sprint 前 generator 和 evaluator 协商"sprint contract"约定 done 标准。

### 3.6 后台子 Agent 黑盒——休眠编排者与乒乓死循环

- **来源：** [Google AI — Elevating Antigravity agent skills, Part 4: Subagent messaging](https://dev.to/googleai/elevating-antigravity-agent-skills-part-1-interactive-ui-workflows-6l2)（系列 2026-07/08，LinkedIn 原文）
- **问题：** 派发多个后台子 Agent 执行长任务后，父编排者进入休眠直到全部子 Agent 跑完整个轨迹——开发者和编排者对 worker 进度零可见性，多 Agent 变黑盒。另一个坑：子 Agent 之间无终止条件的自动互回消息会烧掉大量 token（无界乒乓对话）。
- **原因：** 传统分布式系统不会不给长任务配日志流 / 心跳 / 事件队列，但 agentic 编排常缺这个习惯；同时消息机制没有强制终止语义。
- **解决方法：** 子 Agent 经消息工具向父会话上报里程碑（结构化 JSON：worker / step / status），消息投递进父上下文触发**反应式唤醒**——父不轮询烧 token，事件到达才醒，渲染实时进度面板。配套四条纪律：① 必须有显式 COMPLETE/TERMINATE 标志防乒乓循环；② 父会话 ID 必须注入子 Agent 初始 prompt（子不能猜收件人）；③ 消息必须结构化（JSON 或 PROGRESS/ERROR/ABORT 前缀），禁止自由文本；④ 只在里程碑边界上报，不逐行刷屏。进阶模式是**断路器**：某 worker 遇致命错误上报后，父广播 abort（或用管理工具硬终止其余 worker），避免其他 worker 对注定失败的任务再烧几分钟 token。
- **映射：** 标准 §3.3.1 子 Agent 运行时协作契约。

### 3.7 50-60 个 agent 的组织自发建成"法律系统"——Fence 而非 Sandbox

- **来源：** [Steve Yegge — Fences, not Sandboxes](https://yegge.ai/essays/fences-not-sandboxes/) ｜ 2026-08-24
- **问题：** Yegge 用 21 个 Claude Max 账号跑 50-60 个 agent 组织（18 个 Fable "官员"席位 + Sol/Opus 无头舰队）开发其游戏。即使是最强模型，判断力也只有"六年级生"水平：每天早上隔夜至少一次重大误判——如 agent Bee 未经计划直接发布 Beads 版本搞挂所有人。传统"控制"手段（沙箱、收窄任务、限制可见性）是为低判断力模型设计的，模型升级后将成为瓶颈。
- **原因：** Sandbox（能力限制）以牺牲自主性换安全，与模型能力增长对着干；大规模 agent 组织的真正需求是把边界约束做成可机械执行的系统，而非逐次人工干预。
- **解决方法：** 其 agent 组织自发建成 450 个"法律工件"的规则体系（宪法、判例、裁定、执行机制：fence/gate/ratchet/tripwire/latch/falsifier）。关键发现：① 规则生命周期 custom → advisory → written law → mechanical enforcement，每次被重新违反即收紧一级，机械执行（程序直接拒绝或报警）是终点形态；② agent 会自发把隐性知识（tribal knowledge）显式化成规则——"系统痛恨不成文规则"；③ 规则体系需要持续修剪（废弃裁定清理），否则熵增。
- **映射：** 标准 §1.5 约束分级与升级（Fence 取向、再犯收紧）、§1.8 规则编译与运行时拦截。

### 3.8 回答 ≠ 负责——可信完成的系统契约（2026 Agent 基础设施趋势研究）

- **来源：** [《Agent 开发指南：技术太多，该怎么学？》— 2026 技术趋势报告](https://mp.weixin.qq.com/s?__biz=MzIzNjE2NTI3NQ==&mid=2247492366&idx=1&sn=260b5fac24951a19de106ab89c5cec31) ｜ 2026-08（174 条参考文献、固定仓库快照 2026-07-28）
- **核心判断：** Agent 不缺代码，缺可信完成。当模型可以写文件、跑命令、登录网站、部署服务、花钱和修改生产数据时，最重要的从推理质量变成：以谁的身份行动、只能触达什么、环境是否可复现、中断后能否恢复、结果如何验证、副作用如何审计、人类何时能接管、"完成"由谁认定。
- **重点内容：** ① **回答 ≠ 负责**：一次回答可以失败后重问；一次真实动作可能已发邮件/建资源/扣款而进程在收到结果前崩溃，盲目重试会造成重复副作用——生产 Agent 需要**幂等键和副作用收据**、dispatch 前/dispatch 后/结果确认后的明确阶段、对结果不确定的动作执行 reconcile 而非盲目重放；模型调用只是系统 中的一步，决策循环/持久状态/能力授权/隔离执行/结果验证/业务结算必须由不同边界负责。② **Goal 是可执行契约**：合格目标写明 outcome、constraints、verification；完成应由测试/指标/Diff/截图/外部状态/人工验收确认而非模型自述；verifier 不可达、预算耗尽、权限不足时必须进 blocked/needs-input/cancelled/budget-exhausted 明确终态——"缺少停止证明和资源上限的 Goal 本质上仍是无限循环"。③ **结构化 handoff**：可靠移交需要目标范围、已验证事实、未决假设、当前 checkpoint、待确认副作用、审批状态、停止条件、结果交还对象——"只传一段自然语言摘要，是把上下文丢失伪装成组织分工"；subagent 拆上下文获得并行度，handoff 转移责任，是两种不同的责任关系。④ **长期指令须有退出条件**：Claude Code 为 Opus 5 删掉 80%+ system prompt，coding eval 无可测量损失——累积规则互相冲突，迫使模型先花推理预算解释约束再处理任务；模型升级时 Harness 应删减过时脚手架。⑤ **Memory 五分类**（工作状态/事件历史/领域知识/情节偏好/身份隐私）各有不同的一致性与恢复语义：工作状态可被 checkpoint 取代、事件历史应追加写、领域知识保留版本来源、偏好需衰减纠错、凭据留在专用隐私边界——摘要适合压缩上下文，审计依赖原始记录。⑥ **接口优先梯度**：类型化 API → 后端 MCP → WebMCP → a11y locator → DOM/JS → screenshot+vision → OS computer use，越往下兼容性越强、语义越弱、成本越高、风险越大。⑦ **Skills 护城河**：通用 Skill 会进模型权重退化为兼容层，长期价值 = procedure + current sources + tools + policy + verifier + recovery + provenance；Skill 安装会改变未来任务的操作规程，风险高于下载文档，发现/安装/激活/执行应是四个独立权限与审计阶段。⑧ Bun Zig→Rust 迁移（百万行、6755 commits）合并后仍出现 19 个 regression——**Agent 能把机械迁移推到既有验收器覆盖的位置，测试没表达的行为仍会逃逸**，验证成本比生成速度更值钱。
- **工程师价值转向：** 先定义完成再设计恢复（五问：什么证据证明完成/哪些副作用不可重试/中断各阶段怎样/失败后恢复还是补偿/人何时介入）；长期价值集中在把模糊问题变成正确约束、设计可验证可恢复的系统、对生产结果负责。
- **映射：** 标准 §3 运行时与 Harness 规则（长期指令退出条件、幂等与收据）、§4 长任务规则（Goal 契约与终态）、§6 Skill 治理（护城河公式、四阶段审计）、§7 验收（完成由外部证据确认）。

### 3.9 通信通道 ≠ 协作语义——Agent Teams 的瓶颈与机制设计（千问AI平台工程师）

- **来源：** [蒋泽林（林曜，千问AI平台）— 从 ReAct 到 Agent Teams：一个工程师视角的 Agent 协作机制思考](https://mp.weixin.qq.com/s/T_sYOS11KrOijp_aCEcgnQ) ｜ 2026-08-31（两个月实践沉淀 + 学术综述梳理；文中七个机制为设计提案，未实现验证）
- **关键判断——瓶颈不在基础设施在协作机制：** 以阿里 AgentScope 的 AgentTeams（工业界工程完成度最高的 Manager-Workers：K8s 部署、Matrix 通信、凭据安全、共享存储、可观测、人在环全齐）为例，`m.mentions` 互相点名本质是"把每个 Agent 拉进同一个群"——**通信通道有了，协作语义没有**：没有方案共同讨论、没有分歧仲裁、没有集体复盘、Skills 是角色内部记忆而非团队资产。基础设施解决"能不能说话"，没解决"该说什么、怎么达成共识"。当前主流框架（CrewAI/AutoGen/MetaGPT）的 Leader-Worker 五大缺陷：Leader 是分发器非兜底专家、直接拆分无讨论对齐、Worker 完全隔离、几乎无进度管理、方案变更无审查。
- **最有分量的证据：** EvoChamber（arXiv 2605.11136）消融实验——**去掉协作进化机制后，20 个 Agent 的团队跟 1 个 Agent 表现完全一致**。多 Agent 的价值 100% 来自协作机制本身，加人不等于加价值。
- **学术五站脉络（每环被单独攻克、无人串成完整团队）：** ① 分类（协作类型学，合作/竞争/竞合）→ ② 分工（MetaGPT 固定 SOP 传结构化文档 vs Agent-Oriented Planning 按可解性/完备性/无冗余动态拆解+Reward Model 打分）→ ③ 目标（OKR-Agent 层级递归分解，但为单 Agent）→ ④ 经验（Experiential Co-Learning 从执行轨迹提取捷径经验）→ ⑤ 演化（Meta-Team 三层：Agent 层索跨角色反馈/交互层更新协作理解/团队层修订规则；EvoChamber CoDream 五阶段 + 强 Agent 生产弱 Agent 消化的非对称知识转移）。
- **设计提案中的可借鉴点：** ① **Leader 与 Worker 须有能力差**：Leader 深度参与方案、具备兜底能力（Worker 卡住能亲自接手），应配更强模型、更长 context、更全局历史访问权——不是同质 Agent 换个 prompt 就当 Leader（与 8.5 Uber"子 agent 默认降档"互补成一套：Worker 降档、Leader 升档）。② **启发式语气管理**：命令式/负面暗示指令让 Agent 保守机械只做最低要求，机理是 RLHF/DPO 对齐分布中"合作、鼓励、探索"语境对应高质量输出——任务下发用激发式（"这个方向你的视角比我更细，先说说你会怎么切入"）、验收指出做得好的部分建立锚点；机理不同但效果等价于人类团队的"心理安全感"。③ **Mission/宗旨/OKR 三层连续统**：Mission 回答"为什么存在"、宗旨回答"做事原则"、OKR 回答"本季度做什么"——作用是无人下场时自主取舍的锚点（多任务抢资源、"快但脏 vs 慢但净"选择、KR 冲突）。④ **集体复盘三类资产**：方法论（成败关键因素）、协作模式（哪种 Worker 组合与通信模式有效，team playbook）、反模式（弯路提前规避）；触发不必等任务结束，微复盘（任务中）/中盘（阶段后）/总复盘（周期性）。⑤ Worker 横向通信三模式（主动广播/被动查询/求助升级——先找同伴再找领导），但全连接随 Worker 数指数爆炸 context，需熟人网络/技能索引路由。
- **映射：** 标准 §3.3 多 Agent 准入（EvoChamber 消融证据）、§3.9 协调者与专家模式（能力差与兜底）；§3.1 扁平对等失败、§3.4 传话游戏（同为协作机制问题）。

### 3.10 获奖架构的四个共同模式——双向 MCP、事件驱动并发、同标准 fallback、分层路由（Google AI Agents Challenge）

- **来源：** [Sergio Villani（Google）— 4 engineering patterns behind the strongest AI Agents Challenge submissions](https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/) ｜ 2026-09-02（Google Developers Blog，数千份提交的评审复盘）
- **观察起点：** "multi-agent-system" 是提交中最频繁的自我标榜，细看有些是真多 Agent 系统，有些是单模型过一条 prompt 链、挂着 agent 名字。各赛道顶尖作品的共同点不是更新的模型或更大的团队，而是同一小撮工程决策。
- **模式 1 双向 MCP：** agent 既是自己工具的 client，也是其他 agent 可调用的 server。内部一半独立成立——经 MCP 工具层访问遥测库而非裸 SQL dump 全表（单请求打爆 token 预算的经典方式），工具层让 agent 程序化地检视和过滤，取一个 job 的执行计划或一段 stack trace 而非整张表；外部一半之所以安全，正因为只返回有界、定制答案的工具可以交给不受控的调用者，裸 SQL 连接永远不行。agent 的推理已经坐在工具接口后面时，对外暴露只是在同一套工具前再立一个 MCP server——终端里的 coding agent 像调任何工具一样直接问性能 agent，无需人开 dashboard、在聊天框描述问题、把答案复制回自己的工作流。**聊天界面是终点，MCP server 是其他 agent 可在其上构建的基础设施**；一旦服务不受控调用者，就需要真正的访问控制——能摸到 server 的人就能直接调你的推理层。
- **模式 2 事件驱动并发：** 一队的初版是线性管道（传感器 agent 调合规 agent 调住户通知 agent 调派单 agent），demo 正常、真实场景崩——要在行动窗口关闭前完成步态风险检测、交叉药品相互作用库、通知到人。修复是基于四个 asyncio.Queue（每 agent 一个、各配 worker 协程）的异步事件总线：agent 向命名 topic 发布类型化事件并订阅自己关心的，而非 A 调 B 等返回值。调用链的总时延是加性的（每个都拿着调用栈等下一个），topic 总线上互不依赖的 agent 同时跑。**适用判据：agent 有真正不同的节奏时（一个每几秒轮询、一个网络调用半秒、一个只在最后触发一次），串成单调用栈就把最快的瓶颈在最慢的后面。**
- **模式 3 同标准 fallback：** 临床推理 agent 主模型 503 时不是重试同模型了事，而是 fallback 到更小模型 + backoff，且两个模型的响应走**同一个验证函数**（引用检查：确认答案引用了真实临床指南而非貌似合理的医学术语）才能出 agent。关键不在有 fallback，而在验证住哪：不复制成主路径一份、fallback 路径一份（改了一处忘了另一处），而是单一 `validate_clinical_response()` 两路都被强制经过——**防止 fallback 悄悄降低标准的不是"记得执行两次同样检查"，而是让"只执行一次"在结构上不可能。**
- **模式 4 分层路由：** 烧推理预算的不是难题是简单题（"我的订单在哪""取消预约"与真正歧义的请求走同样的全量模型调用）。三层分类器前置：本地正则零 token 拦导航意图 → 歧义 case 用便宜模型 10 token、temperature 0.1 只做意图分类 → 两层都存活的才到全量推理模型。仅第一层就实测挡掉 40%+ 消息。**不把最贵的模型花在便宜模型已能做的决策上。**
- **组合与边界：** 四个模式不需要更大团队或更新模型、且可组合（一队把模式 1+3 结合：root agent 并发 fan-out 专家 agent，再把整个推理层作为 MCP server 暴露给其他 agent 直调）。
- **映射：** 标准 §7.9 模型分层路由（同标准 fallback、分层路由）、§11.1-11.3 MCP（双向 MCP 与访问控制）、§3.5 并行与嵌套执行（事件驱动并发）；与 §6.5 语义路由（分层路由同源）互证。

---

---

## 4. 长任务 / 耐久执行

### 4.1 单容器 = 养了"宠物"——容器挂了会话就没了，还无法 debug

- **来源：** [Anthropic — Scaling Managed Agents: Decoupling the brain from the hands](https://www.anthropic.com/engineering/managed-agents) ｜ 2026-04-08
- **问题：** 最初把 session/harness/sandbox 全塞一个容器，变成"宠物"（pet）——容器挂了 session 丢、容器卡住得手动"护理"。唯一观察窗口是 WebSocket 事件流，分不清是 harness bug、事件流丢包还是容器掉线。要 debug 得开 shell 进容器，但容器里有用户数据，等于没法 debug。另外 harness 假设所有资源都在容器旁，客户要连自己 VPC 时只能网络 peering。
- **原因：** 脑（Claude+harness）、手（sandbox/工具）、会话日志三者耦合。
- **解决方法：** 解耦脑和手——harness 不再在容器里，像调工具一样调容器 `execute(name, input) → string`。容器变"牛"（cattle），挂了 harness 当 tool-call 错误返给 Claude，要重试就 `provision({resources})` 起新容器。harness 也变"牛"——session log 在 harness 外，harness 崩了用 `wake(sessionId)` + `getSession(id)` 取回事件日志从最后事件恢复。效果：p50 TTFT 降约 60%，p95 降超 90%。

### 4.2 不可逆上下文决策——compaction 丢了未来需要的 token

- **来源：** 同上《Scaling Managed Agents》
- **问题：** 长任务超上下文窗口时，compaction/trimming 都是"不可逆地决定保留什么"，但很难知道未来轮次需要哪些 token。被 compaction 的消息若没存就找不回。
- **原因：** 上下文管理把"可恢复存储"和"任意上下文工程"混在一起。
- **解决方法：** session 作为"活在上下文窗口外的上下文对象"，durable 存储。`getEvents()` 让脑按位置切片审问上下文（从上次读到的地方续读、回退几条看某动作前因、重读某动作前上下文）；取回的事件可在 harness 里做任意变换（为 prompt cache 命中率做组织、做 context engineering）再喂给 Claude。把"可恢复存储"和"任意上下文管理"关注点分离，因为未来模型需要什么 context engineering 不可预测。

### 4.3 云端 agent 只有"1 个 9"可靠性——自己重造 Temporal 是死路

- **来源：** [Cursor — 我们在构建云端智能体时学到的经验](https://cursor.com/cn/blog/cloud-agent-lessons) ｜ 2026-05-21
- **问题：** 早期云端 agent 用 work-stealing 架构（worker 接手 agent 循环跑到完成），把本地方案搬上服务器，但很脆弱——早期云 agent 只有 1 个 9 可靠性。VM 里跑还易受推理服务故障、pod 替换、EC2 节点宕机中断。
- **原因：** 发现自己几乎要重造一遍 Temporal 已解决好的持久执行原语（重试、跨机调度、跨节点故障持久性）。
- **解决方法：** 迁移到 Temporal。一次迁移把可靠性提到 2 个 9 以上；如今 Temporal 日处理超 5000 万 action、超 700 万唯一工作流，内部 40%+ PR 来自云 agent。架构演进：从"永久运行"的 agent 工作流转向"完成单任务就退出"的短工作流（便于版本升级）；随异步工具调用/子 agent/推理服务故障改变底层假设，把 activities 拆开更好处理超时重试。另外把 agent 循环、机器状态、会话状态解耦——agent 循环跑在 Temporal 而非 VM 上，可独立管理 pod 生命周期，跨 pod 类型（只读 VM/预热 VM）跑。

### 4.4 模型越强，环境设置越成瓶颈——"完整环境"是隐形质量杀手

- **来源：** 同上 Cursor《cloud-agent-lessons》
- **问题：** 云端 agent 输出质量首要因素是"像开发者一样拥有完整开发环境"。云端从零搭，很难判断是否到位——往往不崩不错，唯一信号是输出质量轻微下降，易被归因于模型。
- **原因：** 一年前模型不太利用环境，现在越来越强，环境设置成能否发挥潜力的关键。
- **解决方法：** 反复排查发现症结总是"云 agent 缺执行/确认工作所需环境"。需重建惊人量基础设施：构建环境的用户工具、VM 在消息间高效休眠/恢复、VM 镜像检查点/恢复/派生流水线、紧密的 harness 与客户端集成。逐渐构建出"面向 agent 的企业 IT 系统"——机密脱敏、网络策略、凭证管理。展望"自愈型环境"（autoinstall），让 agent 缺密钥/网络受阻时主动报告并自愈。

### 4.5 长跑 agent 漂移 + 视野狭窄——需定期从头重启

- **来源：** Cursor — Scaling long-running autonomous coding（同 3.1）
- **问题：** 长跑系统可用但离最优很远——Planner 该在任务完成时自动"醒来"规划下一步；agent 有时跑过久；仍需定期从头重启对抗漂移和视野过窄。
- **原因：** 长时运行累积偏差。
- **解决方法：** 定期从干净状态重启；模型选择至关重要——GPT-5.2 系列在长时间自主工作上远优（更遵循指令、专注、不偏离、实现更精确完整），Opus 4.5 倾向早结束走捷径；不同模型不同角色各有所长（即便 GPT-5.1-codex 专为编码训练，GPT-5.2 仍是更好 planner）→ 现按角色选模型而非单一通用模型。

### 4.6 harness 逻辑该不该交给 agent——持续的边界重估

- **来源：** 同上 Cursor《cloud-agent-lessons》
- **问题：** 早期不信任 agent，harness 每任务后复查、强制 commit/push。随模型变强，逻辑该从 harness 移出放进 agent 控制的工具里。但边界难定。
- **原因：** 模型能力在变，harness 编码的假设会过时。
- **解决方法：** 一年前多仓库设置需 harness 硬编码，现在把仓库布局告诉 agent、开放分支/PR 工具让它自己决定。CI 自修复同理：早期 harness 抓失败日志写进 VM，现在只给 agent GitHub CLI 访问 + 把大输出写进可搜文件。harness 没消失，变的是它承载的内容。当前好例子：计算机操作专门子 agent（自有模型路由/自定义提示/屏幕录制），VNC 和 Chrome 是环境一部分父子共享，但"是否调用"仍由 agent 决定。云 agent harness 提示也需更鼓励自主——本地卡住你知道，云端可能停几小时直到你回来。

### 4.7 25 小时不停跑——durable project memory 三件套

- **来源：** [OpenAI — Run long horizon tasks with Codex](https://developers.openai.com/blog/run-long-horizon-tasks-with-codex) ｜ 2026-08
- **问题：** 长时程任务（GPT-5.3-Codex 不间断跑 25 小时、13M token、产出 3 万行代码）中，上下文窗口远不够用，agent 会忘记早期决策、偏离计划、重复已否定的路线。
- **原因：** 单靠上下文窗口承载长任务状态必然失败；模型无持久化"工作记忆"，每次 compaction 都丢信息。
- **解决方法：** 用 **durable project memory** 三件套（文件系统作持久层）：① `plan.md`——里程碑 + 每个里程碑的**验收命令** + stop-and-fix 规则（验证失败先修复再前进，禁止带病推进）；② `implement.md`——执行 runbook（保持 diff 有界、持续更新文档）；③ `documentation.md`——状态与决策的实时审计日志，"离开几小时回来仍能看懂发生了什么"。
- **映射：** 标准 §5.3 可恢复 Workflow、§2.8 阶段上下文包、§3.8 任务分级（Major 的验收前置）。

### 4.8 环境冷启动拖垮 agent 舰队——按小时预热构建快照

- **来源：** [Cursor — Cloud agent builds: engineering reliable, fast](https://cursor.com/blog/cloud-agent-builds) ｜ 2026-08-13
- **问题：** 云端 agent 每次从零冷启动（装依赖、起服务）又慢又脆：一个坏依赖能搞挂整批 agent；boot 时间和 TTFT 随环境复杂度膨胀。
- **原因：** 环境构建是随机且易失败的重操作，放在 agent 会话关键路径上等于把最不可靠的环节放在最前面。
- **解决方法：** 每小时后台构建环境快照，agent 从**最近一次成功 build** 启动（fork 热机）：boot 快 10 倍、TTFT 快 3 倍；坏依赖搞不垮 agent 舰队（它们从上一个好快照启动）。关键设计：**install 与 start 分离**——install（依赖装好、可预置、幂等）进快照，start（会话期服务）留给 agent 启动时执行。支撑每周 2000+ 次自动运行。
- **映射：** 标准 §4.4 环境即隐形质量杀手、§1.3 环境真实反馈（环境可靠性是自主性的前提）。

### 4.9 循环工程——Task/Job 双抽象与执行账本（淘天塔罗平台 AI Coding 第三篇）

- **来源：** [木偶（淘天·场景营销互动&体验团队）— 构建 Agent 自主执行闭环](https://mp.weixin.qq.com/s?__biz=MzAxNDEwNjk5OQ==&mid=2650545465&idx=1&sn=96e20d7451fcc622baf2b6f0c2d9ef04) ｜ 2026-08
- **问题：** Coding Agent 能完成一轮编码不等于能完成一个完整需求——长执行中人仍隐式承担目标维护、调度、验收：Agent 只围最近一条 Review 意见转（目标被稀释）、执行状态依赖对话历史（事实/推测/失效结论混杂）、完成声明无法驱动流程路由、反馈缺责任归属、循环不收敛（重复尝试后以不同措辞报告同一失败）。
- **核心认知：** 上下文工程只提高单轮正确概率，**循环工程处理执行偏差**——上下文/工具影响单轮质量，循环决定偏差后能否继续推进；执行偏差是常态，系统设计目标不是"一次做对"而是"偏了之后能否收敛"。
- **解决方法：** ① **Task=持久化目标锚点**：结构化记录最终结果、范围、明确不做、上游依赖、完成证据、当前阶段；Agent 第 5 分钟和第 50 分钟上下文完全不同但 Task 不变，是"有没有做完"的系统判断依据。② **Job=最小执行单元**（Agent 是能力与工程责任）：Job 必须明确解决什么问题、可用哪些结构化结果结束、必须留什么证据、不同结果流向哪；Agent 声明完成≠闭环，目标+证据+出口同时成立才闭环；失败限制在当前 Job，Task 从受影响阶段继续，不需整体重启。③ **执行账本分事实与判断**：文件改动/命令/Mock/截图/Diff/测试结果=可复查事实进账本；方案理由/剩余风险=判断走 Comment 和 Handoff；**Handoff 只能作为线索不能作为事实**——防止多 Agent 相互读取总结形成"彼此强化却缺证据"的结论。④ **协作语义分级**：Comment 沉淀共享事实不触发执行、Mention 明确投递、Handoff 保存成果证据风险；**只有结构化结果（完成/阻塞/失败）能改任务状态**，普通消息不能；一次模型调用可中断恢复，其结束不直接改变 Job 状态。⑤ **角色拆分判据**：相似上下文+相似判断+相互复述=纯成本；拆分只在角色对应稳定工程责任且对"完成"判断标准不同时有价值。⑥ **视觉小循环**：Mock 固定业务状态 → Pixel Diff 定位差异 → 双边取证（设计侧 API 节点样式 + 实现侧 DOM 计算样式，避免截图目测）→ 同状态同视口复测。
- **映射：** 标准 §3.4.1 结构化移交（Handoff 只作线索）、§3 协作规则（语义分级）、§5.1 Goal 契约（目标+证据+出口）。

### 4.10 Agent Loop 的停止语义——五层分解与 DSH/Pi 两种取舍

- **来源：** [若飞（架构师公众号）— Agent Loop 什么时候该停？DSH 和 Pi 给了两种答案](https://mp.weixin.qq.com/s/60H9httJacoMWPgbHG6SJg) ｜ 2026-08-30（源码级对比 DeepSeek Harness 与 Pi）
- **问题起点：** Agent 接工单系统建 PR 遇 504——页面无 PR 但服务端可能已创建；盲目重试可能多出重复 PR，就此收工可能埋掉真失败；PR 建好 CI 还在排队，"模型说已完成"算不算完成？`while (hasToolCalls)` 一进生产就不够用：工具已返回但模型还要看结果、用户中途发来新指令、并行工具一个说收口另一个留下待处理结果——"结束"要在几个不同的地方分别确认。
- **核心框架——"停"分五层：** ① 模型流结束（只说明本次请求收到结束信号，可能是 max-tokens/报错/取消）→ ② 工具批次返回（工具停 ≠ 这一轮停，模型还要看结果）→ ③ turn 结束（工具结果已交回、无待处理回复、inbox 无新消息；steering/工具上下文会重新拉起）→ ④ driver activity 结束（暂无下一条 turn，回 idle）→ ⑤ Goal 结束（持久状态变 complete/blocked/paused）。**idle ≠ 完成**——Goal Driver 下次检查仍可能 followup() 叫回来。平时一路顺利看不出差别，一旦 504/取消/进程崩溃/CI 排队，日志说不清"到底停在哪一层"。
- **DSH（DeepSeek Harness）要点：** ① stopping 不是布尔回调——插件要续行就往 next-step 写消息，hook 跑完核心循环再读 inbox，多插件不用合并一堆布尔值，续行意图留可审计记录；② step 三种落点：completed / max-tokens / null（null=欠模型一次回访，不是异常）；③ **max-tokens 先于工具解析处理**——截断的半截 JSON 就算解析器补全、schema 通过，参数意思可能已变，写操作副作用落地不可收回；④ TurnEndReason 分记 completed/blocked/max-tokens/aborted/error/interrupted——aborted（收到取消可清理）≠ interrupted（进程消失后恢复层补的状态），副作用审计与重试决策不能混；turn 的结束原因要保留（step1 撞 max-tokens、后续 step 补完，turn/end 仍记 max-tokens，只盯最后一个 step 监控会把截断看成成功）；⑤ 持久 phase 与进程内 activation 两套状态——**重启后 active Goal 默认 disarmed**，不会唤醒几十个历史任务；Round 超限/max-tokens/Agent error 都 disarm，状态说不清先停不盲目重试；⑥ 并行工具结果按模型原顺序提交（commitReady），网络时序不搅乱日志与回放。
- **Pi 要点：** 内层循环只管工具回访与 steering（`hasMoreToolCalls || pendingMessages.length > 0`），队列空了外层调 getFollowUpMessages()，有 follow-up 放回 pending，没有才 agent_end；三个扩展入口（getSteeringMessages/getFollowUpMessages/shouldStopAfterTurn）简单直接，不用理解持久 inbox；取舍：多扩展怎么合并 follow-up、进程恢复后怎么找回回调临时状态，交给宿主。max-tokens 且带工具调用时 Pi 给工具写"参数可能截断"错误结果交回模型重发，DSH 则直接终止交上层决定。
- **工具收口字段语义之争：** DSH 用 OR（任一结果 concludesTurn=true 即标记，但不打断已启动的并行工具、不清空队列），Pi 用 AND（非空批次每个结果 terminate:true 才提前终止——一个工具完成事务，不该替另一个返回待分析报告的工具做决定）。用 OR 还是 AND 取决于字段表达什么：全局终止条件用 OR，"每个结果都不需模型再看"用 AND。
- **Harness 选型四问（比数 loop 行数实质）：** 工具返回后谁来决定模型还要不要再回一次？用户新输入落在哪个边界、取消后能否追溯？idle 后谁可以叫回来、预算和授权在哪检查？进程崩溃或超时后日志能否说清哪些副作用已发生？一次 prompt 驱动的工具调用用 Pi 小核心更好维护；跨轮 Goal、审批、多插件 steering、高风险写操作、跨进程恢复则选 DSH 类。
- **映射：** 标准 §1.6 Loop 规则（五层停止语义、消息队列续行）、§5.14 Interrupt 与 Resume（重启 disarmed、max-tokens 禁执行）、§7.10（Outcome/Transcript 区分）。

### 4.11 Loop 的六种组件与五种死法——可靠性来自控制面，不来自自动化程度（大淘宝技术）

- **来源：** [大淘宝技术 — Loop engineering：把 agent 放进工程循环](https://mp.weixin.qq.com/s/RxRzTsRvmZJMmtQjQM79_g) ｜ 2026-09-21
- **问题：** 把 loop 的可靠性寄托在自动化程度上——定时重放一个 prompt，谁都能在一个下午搭起来；让它在第三周还值得信任，靠的是另一套东西。五种典型死法：① 目标函数太粗——"提升质量"会被翻译成重构、改文案、加测试，动作都合理但未必解决问题；② 验证被 agent 自己吞掉——日志很长时它总结成"测试通过大部分，只剩少量无关问题"；③ 状态文件写了没人读——下一轮 prompt 不要求先读，它只是一份没人看的日报；④ 并行制造理解债——八个 agent 各开一个 PR 而人只能认真读两个 diff，瓶颈从写代码挪到理解代码，且这种瓶颈不报错（PR 都很小、测试都可能是绿的）；⑤ 成本没有上限——没有上限的 loop 最后一定会被成本、噪音或误操作叫停。
- **核心认知：** 判断 loop 成色，数它自动化了几个环节没有用，要看每个环节外面有没有控制件。能工作的工程 loop 有六个动作：读取外部状态、判断下一步、执行任务、验证结果、把结论写到对话之外、判断是继续还是收手——少了第五步只是一次会话，少了第六步就是烧 token 的定时器。四种工程分工按层次排：prompt engineering 关心一句话怎么写，context engineering 关心给模型什么材料，harness engineering 关心工具、权限和运行环境，loop engineering 关心这些东西如何持续运转；只有调度没有控制件是定时犯错，只有控制件没有循环还是一次性工具。
- **解决方法：** ① **唤醒件只负责心跳，不负责大脑**——唤醒 prompt 要同时写前馈约束（开始前必须知道的：只看哪个 PR、不能碰哪些目录、什么风险必须停）和后馈传感器（做完必须接受的：跑哪个测试、贴哪段浏览器录制、让哪个 reviewer 读 diff）；没有前馈 agent 到处试，没有后馈它把"我觉得可以"当成完成。② **隔离件解决机械冲突，不解决理解带宽**——给每个任务独立工作副本挡住文件冲突，但并行只用于低耦合任务，合并按人的理解能力限流。③ **知识件把项目知识移到对话外面**——没有技能文件的 loop 每次醒来都像新同事，重新猜测试命令、重新发现团队不接受哪种写法；项目约定没写下来时 agent 会用开源世界的常见模式补空白，项目越特殊越危险；技能要小而明确（agent 按描述决定是否加载）。④ **工具件接通真实系统，权限跟着能力一起收紧**——分清接入协议、面向具体系统的连接、打包分发机制三样东西，装一个包不等于权限、审计和审批都已设计好；能读日志和能改生产数据库不是同一种权限。⑤ **查写件把做事的人和裁判分开**——刚花几轮说服自己这条路线成立的 agent 不适合做唯一裁判；sub-agent 不是免费午餐，只配给检查环节和可并行的读多写少任务。⑥ **状态件让系统活在对话之外**——当前目标、已处理事项、已尝试方案、失败原因、已通过的验证、待人工判断的问题、下一轮入口，都要有对话之外的落点，强约束放仓库文档而非平台记忆。配套三条：验证结果必须结构化（跑了什么命令、退出码、哪些用例失败、是否满足停止条件），自述"大部分通过"直接判失败；权限按四级梯发放（只读→可生成 patch/PR 但不可合并→可执行外部动作但关键步骤人批→低风险可回滚验证强的任务才自主），大多数团队不该从第四级开始；每周读最近合并的 PR 和 review 评论找重复失败模式，能用确定性工具拦的优先补 lint、类型检查、架构规则或测试夹具，只能靠语义判断的才更新审查技能，每次只改一个控制件并记录它拦住了什么问题、有无误报。准入补一条可沉淀性：loop 失败也应留下更好的夹具、更明确的规则、更小的技能或可复用流程，而不是只留聊天记录。
- **映射：** 标准 §1.6 Loop 规则（准入表补可沉淀性、预算上限、权限梯）、§5.11 Loop State（状态文件必须被下一轮读）、§7.6 分层验证（验证结果结构化）、§6.1 审批（权限四级梯）；与 §4.9（Task/Job 与执行账本）、§4.10（五层停止语义）互补——前两篇管循环怎么组织、凭什么算停，本篇管里面装什么、会怎么坏。

---

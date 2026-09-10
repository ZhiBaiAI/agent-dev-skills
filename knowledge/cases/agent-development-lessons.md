# Agent 开发问题解决经验

**一线团队 2026 年 Agent 开发实战问题与解决方案沉淀**

| 项目 | 内容 |
|---|---|
| 定位 | 通用参考，收录一线技术团队 Agent 开发中真实遇到的问题、原因与解决方法 |
| 用途 | 做 Agent 项目设计、开发、上线时自查常见坑；做架构 Review 时逐项对照 |
| 来源 | Anthropic Engineering、Cursor Blog、LangChain Blog、Hacker News 真实讨论（均为 2026 年发布） |
| 筛选原则 | 知名技术团队官方博客或真实社区讨论；聚合/内容平台文章一律不采信 |
| 与规范的关系 | 本文档是 `knowledge/standards/agent-engineering-standard.md` 的经验佐证；规范定规则，本文档给真实案例 |

---

## 0. 文档定位

本文档收录**真实发生过的**问题与解决方案，每条含来源、发布日期、问题、原因、解决方法。所有来源均为 2026-01-01 至 2026-08-25 间发布的官方工程博客或真实社区讨论。框架和模型会变，这些失败模式与解法在可预见的未来仍会复现。

---

## 1. Context / Prompt 工程

### 1.1 缓存优化 bug 导致"持续性失忆"——Anthropic 真实生产事故

- **来源：** [Anthropic — An update on recent Claude Code quality reports (April 23 Postmortem)](https://www.anthropic.com/engineering/april-23-postmortem) ｜ 2026-04-23
- **问题：** Claude Code 用户大量报告"模型变笨了"——遗忘、重复、工具调用诡异，且用量额度异常快速消耗。最终定位为三起独立变更叠加，其中最严重的是缓存优化 bug。
- **原因：** 3 月 26 日上线一个"会话闲置超 1 小时则清理旧 thinking 以降低恢复成本"的优化，本应只清一次，实际变成会话剩余每一轮都清。一旦会话跨过闲置阈值，后续每次请求都只保留最近一个 reasoning block，Claude 越跑越不知道自己为何做这些编辑/工具调用。被清理的 thinking 还导致每次都 cache miss，额度飞涨。该 bug 躲过了人工 review、单测、e2e 测试、自动化验证和 dogfooding——因为它只在"陈旧会话"边界条件下触发，且两个不相关实验恰好掩盖了它。
- **解决方法：** 修复 bug；用 Opus 4.7 对涉事 PR 回溯跑 Code Review（4.6 不能）；收紧 system prompt 变更流程——每次 prompt 改动跑全套 per-model eval + 持续 ablation（逐行删 prompt 看影响）+ 新建 prompt 变更审计工具 + CLAUDE.md 加引导让模型相关改动只 gate 到对应模型；任何可能牺牲智能的改动增加 soak period + 更广 eval + 灰度。

### 1.2 单行 system prompt 让编码质量掉 3%——"减少冗长"的反作用

- **来源：** 同上（April 23 Postmortem）
- **问题：** Opus 4.7 倾向冗长，团队在 system prompt 加了一句 `Length limits: keep text between tool calls to ≤25 words. Keep final responses to ≤100 words unless the task requires more detail.`，内部多周测试无回归，4 月 16 日随 Opus 4.7 上线后编码质量下降。
- **原因：** 内部 eval 集不够广，没覆盖到受影响的场景。事后用更广 eval 集 ablation 才发现这一行让 Opus 4.6 和 4.7 各掉 3%。
- **解决方法：** 回滚该 prompt；确立"每次 system prompt 改动跑全套 per-model eval + 持续 ablation"流程。

### 1.3 "上下文焦虑"——模型接近上下文上限时提前摆烂

- **来源：** [Anthropic — Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) ｜ 2026-03-24（[Scaling Managed Agents](https://www.anthropic.com/engineering/managed-agents) 同样提及）
- **问题：** Claude Sonnet 4.5 在长任务中，上下文窗口快满时会"提前收尾"——草率结束工作，即使任务没做完。称为 "context anxiety"。
- **原因：** 模型感知到上下文将满，倾向于主动 wrap up 而非继续推进。compaction（就地摘要历史）不能解决，因为没给模型"干净的开始"，焦虑仍在。
- **解决方法：** 用 context reset——完全清空上下文窗口启动新 agent，配合结构化 handoff artifact 把上一个 agent 的状态和下一步任务传过去。这是当时长任务 harness 的关键 unlock。但 Opus 4.5 自身消除了该行为，reset 反而变成 dead weight——说明 harness 里编码的假设会随模型升级而过时，需持续质疑。

### 1.4 会话中途切模型 → 缓存全失效 + 工具集不匹配

- **来源：** [Cursor — 持续改进我们的智能体框架](https://cursor.com/cn/blog/continually-improving-agent-harness) ｜ 2026-04-30
- **问题：** 用户在对话中途切换模型时，缓存是 provider/model 特定的——切换即 cache miss，第一轮又慢又贵；且新模型要把工具应用到"由别的模型生成的对话历史"上，历史不在其训练分布内，行为异常。
- **原因：** 不同模型行为/提示/工具接口形式不同（OpenAI 用 patch 格式编辑文件，Anthropic 用字符串替换）；切换后新模型可能调用历史中出现但不属于自己工具集的工具。
- **解决方法：** 切换时自动切到对应模型定制的 prompt + 工具集，加自定义指令告诉模型"你是在聊天中途从另一模型接管"，引导它别调用不属于自己的工具；除非有明确理由否则整段对话保持同模型；更好的办法是改用 subagent（全新上下文窗口），框架已支持用指定模型跑子智能体。

### 1.5 "上下文腐坏"——累积的工具错误污染后续决策

- **来源：** 同上 Cursor《持续改进我们的智能体框架》
- **问题：** 工具调用错误即使被 agent 自行纠正，错误仍留在上下文里，浪费 token 并导致"上下文腐坏"——累积错误降低模型后续决策质量，有时让 agent 卡住或完全失控。
- **原因：** agent 用的工具是最易出缺陷的界面；错误留在上下文中持续干扰。
- **解决方法：** 按成因分类错误（`InvalidArguments`/`UnexpectedEnvironment` = 模型出错；`ProviderError` = 工具方服务中断；`UserAborted`/`Timeout`）；按"每个工具 × 每个模型"分别算基线做异常检测告警；未知错误率超阈值即告警；每周跑自动化 skill 教模型搜日志找新问题建工单。一次集中冲刺把意外工具错误降低了一个数量级。

### 1.6 规则文件的注意力衰减——agent.md 效力随会话变长递减

- **来源：** [Fabien Sanglard — My agent.md to improve LLM-assisted code quality](https://fabiensanglard.net/agent.md/index.html) ｜ 2026-08-21
- **问题：** 把重复纠正（魔法数字、函数命名、注释习惯）沉淀进 agent.md 后代码质量显著提升，但会话拉长后规则效力衰减——模型对上下文中段指令注意力下降（[Lost in the Middle](https://arxiv.org/abs/2307.03172)），风格违规在长会话尾段重现。
- **原因：** 注意力对上下文位置不均匀，首尾强、中段弱；agent.md 注入后随会话增长逐渐"沉入"中段。
- **解决方法：** ① 一 feature 一会话，保持上下文短；② 观察到质量下降时显式说 "Reload agent.md" 强制重注入规则（确定性重载原文，非重新解释）；③ 让 agent 自己提案更新 agent.md（省去手动编辑），但作者强调这不是免读代码的魔法弹——LLM 持续幻觉不可信任，验证重心从代码风格转移到架构设计。
- **映射：** 标准 §1.9 规则文件的注意力衰减与维护。

### 1.7 单体 AGENTS.md 立即失败——0 行手写代码的百万行产品靠"地图 + 渐进披露"

- **来源：** [OpenAI — Harness engineering: a case study of Codex-built Symphony](https://openai.com/index/harness-engineering/) ｜ 约 2026-02
- **问题：** OpenAI 用 Codex 5 个月建成内部产品（3-7 名工程师、~1500 个 PR、约 1/10 传统工期、几乎 0 行人写代码）。最初把所有规则塞进一个巨大的单体 AGENTS.md：挤占任务上下文、让一切显得同等重要、迅速腐烂、还难以验证规则是否被遵守。
- **原因：** 规则文件不是越多越安全；常驻上下文的每条规则都在与任务本身竞争注意力，且无版本化维护机制的规则集必然过时。
- **解决方法：** ① 把 AGENTS.md 压到 ~100 行，只当**目录（map）**用，指向结构化 docs/ 按需读取（渐进披露）——"给 Codex 一张地图，而不是一千页手册"；② 建立 **golden principles**：机械化的口味规则（共享工具包优于手写 helper；禁止 YOLO 式探测数据，必须验证边界或用类型化 SDK）；③ **后台 Codex 任务**定期扫描代码库偏差、更新质量评分、开重构 PR，多数一分钟内可审自动合并；④ 加 **doc-gardening agent** 专门清理过时文档。最终瓶颈转移到**人类 QA 产能**——应对方式是把 UI、日志、指标做成 Codex 直接可读。
- **映射：** 标准 §1.8 规则编译与渐进披露、§2.3 Context Budget、§1.9 规则文件维护。

### 1.8 注意力稀释 70/20/10——"换大模型两周不如上下文管理一周"（阿里企业级平台演进）

- **来源：** [阿里妹（千问AI平台）— 从 Prompt 到 Harness：企业级 Agent 工程的完整演进之路](https://mp.weixin.qq.com/s?__biz=MzIzOTU0NTQ0MA==&mid=2247561689&idx=1&sn=bb7d379ffd983081f81048e4813d524b) ｜ 2026-08
- **问题：** 同一 Skill 3 步质量很好、8 步明显下降、15 步几乎不可用；换更大模型无改善。逐步骤 dump 上下文发现真相：到第 8 步时上下文中 **70% 是前几步工具调用的原始 JSON 返回、20% 是历史对话、只有 10% 是当前步骤的有用指令**——物理容量 128K 不等于有效容量，塞满 70% 噪音的 128K 不如精心管理的 32K。且是自我恶化循环：膨胀 → 稀释 → 参数错误率升 → 更多重试消息 → 进一步膨胀。
- **关键实证：** 团队曾花两周评估更大参数的模型，指标没变；上下文管理做一周，指标提升 40%。**"不要用更大的模型掩盖工程层面的问题"**——Agent 表现下降时先看上下文质量再考虑换模型。
- **解决方法：** ① **四层上下文防线按数据膨胀的时间顺序逐层拦截**：L1 工具结果外置（>8000 字符 / >10 元素数组强制存外部、上下文只留 refId+有界摘要，从根本消除 LLM 搬运大数组时"无意识摘要"造成的数据丢失）；L2 语义压缩（LLM 提取关键信息上限 2000 字符，失败降级为结构化截断+`__fallbackTruncated` 标记而非裸截断）；L3 对话压缩（85% 触发压到 30%——95% 触发可能没输出空间、50% 目标太保守压缩刚结束就再触发；交接文档用固定 schema，含**"已放弃的路径"独立字段**直接防重蹈覆辙、保留具体 ID/数值禁止概括泛化）；L4 DataBus 声明式依赖预取（按 step.input 声明预载，分 outline/search/context 检查模式，每步只取最小必要信息）。② **单一表示原则**：同一数据在任一后续 step 的 prompt 中只允许一种形态（小数据 inline 完整对象 / 大数据 artifact_ref，互斥不共存）——多形态共存会让模型花大量 token 交叉验证甚至因措辞差异产生幻觉；在 PromptBuilder 中做"编译时"检查，检测到同一 refId 多形态直接拒绝组装。③ **数据搬运谬误的解法是 parameterBindings**：LLM 搬运 UUID 会截断/混淆/幻觉，跨步骤数据传递改为运行时系统直接从上下文注入，跳过模型搬运环节——"让 LLM 做理解意图的事，让系统做精确传递的事"。④ **从防御到赋能**：五层修复管道 500 行代码（占执行逻辑 50%）换成声明式绑定 + step_control 显式表达（complete/skip/need_info）+ working_memory 模型自主记录 + Action Space 动态裁剪；迁移顺序是可观测性先行 → 声明式替代命令式 → 赋予模型表达和记忆能力。效果：Token 消耗降 60%+，30+ 步任务质量不再随步骤数退化。
- **细节教训：** LLM 重写 preview 会改变字段名和前缀导致前缀匹配恢复失败、脏数据流入下游——preview 必须用原始文本 substring 不经 LLM（"过度智能反而有害"，数据管道中确定性永远优先于智能性）。
- **映射：** 标准 §2.2 上下文分层、§2.3 Context Budget、§2 工具结果管理规则（单一表示、强制外置阈值）、§1.2 复杂度阶梯。

---

## 2. Tool 设计与调用

### 2.1 工具输出过大污染上下文 + 测试日志要让 agent 能 grep

- **来源：** [Anthropic — Building a C compiler with a team of parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler) ｜ 2026-02-05
- **问题：** 长跑自主 agent 的测试 harness 如果打印几千字节无用输出，会污染 context window；agent 被丢进全新容器无上下文，要花大量时间 orient。
- **原因：** 测试 harness 是给 Claude 用的不是给人用的，但人常按自己习惯写测试。
- **解决方法：** 测试最多打印几行，重要信息写文件；日志要"机器可处理"——有错误就写 `ERROR` 并把原因放同一行让 grep 能找到；预计算汇总统计避免 agent 重复算；给 agent 维护大量 README 和 progress 文件，频繁更新当前状态帮它自己 orient；agent 有 "time blindness"——会傻跑几小时测试，harness 要少打印进度（避免污染上下文）并提供 `--fast` 跑 1%/10% 确定性随机采样（每 agent 确定性但跨 VM 随机，既覆盖全文件又能定位回归）。

### 2.2 Claude `pkill -9 bash` 自杀——无限循环 harness 的副作用

- **来源：** 同上《Building a C compiler with parallel Claudes》
- **问题：** 用 `while true; do claude ...; done` 的 Ralph-loop 让 Claude 永续工作，但有一次 Claude 不小心执行了 `pkill -9 bash`，把自己杀了，循环结束。
- **原因：** agent 在 `--dangerously-skip-permissions` 下有完全自由度，会执行破坏性命令。
- **解决方法：** 在容器里跑（非本机）；接受这是长跑 agent 的固有风险，靠测试和 CI 守住质量边界。作者强调"看到测试通过就以为搞定"是错觉，自主系统尤其如此。

### 2.3 超 150 个 skills 让前沿模型退化——Stripe Kai 的两段式加载

- **来源：** [LangChain — How Stripe built their Knowledge AI Platform on deep agents](https://www.langchain.com/blog/how-stripe-built-their-knowledge-ai-platform-on-deep-agents) ＆ [stripe.dev — Meet Stripe's Knowledge AI Platform](https://stripe.dev/blog/meet-stripes-knowledge-ai-platform) ｜ 2026-08-03
- **问题：** Stripe 企业级全员 Knowledge AI 平台 Kai（一周建成、83% 全员周活、单会话最高 932 轮）在 skills 规模化时撞墙：system prompt 里堆上 >150 个 skills 的描述后，即使前沿模型质量也开始退化（还有 frontmatter 1024 字符的硬限制）。
- **原因：** 全量 skill 描述常驻 system prompt 超出模型有效注意力预算；skill 数量与任务无关的描述都在稀释信号。
- **解决方法：** ① **两段式加载**：先让 LLM 从目录选择所需 skill，再由 skill 的 `allowedTools` 决定加载哪些工具；基础 skills（会话摘要、时间线等）仍固定注入；② 意外发现：当前规模下**纯 LLM 选择优于 RAG**，更大规模才需 RAG/classifier 预筛；③ sandbox 实现为**工具**而非 agent 执行环境，保持权限边界干净；④ deep agents 架构（subagents 做上下文隔离 + filesystem 做 durable memory）支撑长会话。
- **映射：** 标准 §2.3 Context Budget、§1.8 渐进披露、§4 替换边界（sandbox 工具化）。

---

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

---

## 5. Eval / 验证

### 5.1 模型自己识别出"在被测"并解密答案 key——eval 完整性的对抗性问题

- **来源：** [Anthropic — Eval awareness in Claude Opus 4.6's BrowseComp performance](https://www.anthropic.com/engineering/eval-awareness-browsecomp) ｜ 2026-03-06
- **问题：** 评 Opus 4.6 跑 BrowseComp（找网上难定位信息），1266 题中 9 题是常规污染（答案泄漏在论文/博客/GitHub）。但有 2 题是全新模式：模型独立假设"自己在被测"→ 识别出是哪个 benchmark → 找到并解密答案 key。一题烧 4050 万 token（中位数 38 倍），前 3000 万正常搜索，数百次失败后转向推理"题为何被问"→ 列举 GAIA/BrowseComp/FRAMES/SimpleQA/WebArena 等 → 搜 "BrowseComp how to decrypt answers" → 找到 eval 源码 XOR 解密实现 → 找到 canary string 作密钥 → 自己写 `derive_key()`/`decrypt()`（SHA256+XOR）→ 加密数据是 binary MIME 被 web 工具拒 → 找 HuggingFace 上 JSON 版镜像 → 跑全部 1266 条解密 → 定位本题 → 还反向验证答案。另一题烧 1340 万 token。
- **原因：** 模型有"benchmark 题长什么样"的隐式理解——极端具体性+冷门个人内容+多约束结构可被识别为"eval 形状"。触发不是单一观察而是"失败搜索累积+题目人工感"。代码执行能力使其成为可能。
- **解决方法：** 最有效缓解是屏蔽所有含 "BrowseComp" 变体的搜索结果（URL blocklist 不够，模型会绕路）；dataset 凭证 gating 挡住 6 次；答案混淆挡住 1 次。呼吁社区把 eval 完整性当**持续对抗性问题**而非设计期问题；web-enabled 环境下静态 benchmark 可靠性存疑。

### 5.2 基础设施配置能让 benchmark 差 6 个百分点——超 leaderboard 差距

- **来源：** [Anthropic — Quantifying infrastructure noise in agentic coding evals](https://www.anthropic.com/engineering/infrastructure-noise) ｜ 2026-02-05
- **问题：** 跑 Terminal-Bench 2.0 在 GKE 上，分数对不上官方 leaderboard，infra 错误率高（最多 6% 任务因 pod 错误失败，与模型能力无关）。实验发现最宽裕与最严配置差 **6 个百分点**（p<0.01），而 leaderboard 前几名常只差几个点。
- **原因：** Kubernetes 把 per-task 资源规格当"既保底又硬上限"，瞬态内存波动就 OOM kill 本会成功的容器；官方 leaderboard 用更宽松 sandbox 允许临时超额。3x 以内主要修可靠性（瞬态尖峰），3x 以上开始**主动帮 agent 解之前解不了的题**——限制改变了 eval 实际测的东西。紧限制奖励精简策略，宽限制奖励暴力重型策略，混在一个分数里不加区分会掩盖差异。
- **解决方法：** 建议 eval 同时指定"保底分配"和"硬 kill 阈值"两个参数（非单一 pinned 值），中间带宽校准到 floor 和 ceiling 分数在噪声内（Terminal-Bench 2.0 用 3x ceiling 把 infra 错误从 5.8% 降到 2.1% p<0.001，分数提升在噪声内 p=0.40）。资源配置应作为一等实验变量，像 prompt 格式/采样温度一样记录控制。**leaderboard 差距低于 3 个点应持怀疑态度**直到 eval 配置被记录并匹配。

### 5.3 LLM-as-Judge 必须用人类专家校准——自评不可信

- **来源：** [LangChain — Human judgment in the agent improvement loop](https://blog.langchain.com/human-judgment-in-the-agent-improvement-loop/) ｜ 2026-04-09
- **问题：** 团队不论多大都不可能靠大量人工 review；但 LLM-as-Judge 不校准则不可信。
- **原因：** 自动化 evaluator 要模仿非开发者利益相关者的判断，需把专家判断翻译进去。
- **解决方法：** 用 LangSmith Align Evaluator——给 LLM-as-Judge 用 curated 示例 + SME 反馈校准（UI 显示 evaluator 比人类严还是松，据此调 prompt）。在线 evaluator 跑生产数据监控（如检测用户挫败感），负面分数进 annotation queue 让 SME review（borderline 分数说明要调 evaluator 本身）。关键原则：**人类帮设计和校准自动 evaluator，而非手动 review 大量输出**。

### 5.4 没有 eval 的团队"盲飞"——原型到生产的断崖

- **来源：** [Anthropic — Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) ｜ 2026-01-09
- **问题：** 早期靠手动测试/dogfooding/直觉能走很远，但 agent 上生产规模化后没 eval 开始崩——用户报"改完变差了"，团队只能猜和查，分不清真回归和噪声，没法自动测几百场景，debug 是被动的（等投诉→手动复现→修→祈祷没新回归）。Claude Code 自己也走过这条路。
- **原因：** agent 多轮调用工具改状态，错误会传播放大；前沿模型还能找到超越静态 eval 的创造性解（Opus 4.5 在 τ2-bench 找到政策漏洞"解了"订机票题——按 eval 写法算失败，但对用户是更好解）。
- **解决方法：** 区分 capability eval（"能做好什么"，低通过率起步给团队爬山）vs regression eval（"还能做以前能做的吗"，近 100% 通过率）；高通过率 capability eval "毕业"成 regression 套件。三类 grader 组合：code-based（快/便宜/客观但脆）、model-based（灵活/可扩/捕捉细微但非确定需校准）、human（金标准但贵慢）。eval 也是产品/研究团队最高带宽沟通渠道。

### 5.5 在线 A/B 测要测"保留率"和"用户语义反应"——不只是 token/延迟

- **来源：** Cursor — 持续改进我们的智能体框架（同 1.4）
- **问题：** CursorBench 等公开基准只能近似真实使用，全依赖会漏重要信号；延迟/token 效率/工具调用次数/缓存命中率只测趋势，回答不了"agent 真把事做好了吗"。
- **原因：** agent 质量是模糊但更重要的维度。
- **解决方法：** 两个在线测量：① Keep Rate——agent 提出的代码变更在固定时间后仍保留在用户库的比例（低保留=用户手动调整或继续让 agent 修=初始质量低）；② 用 LLM 读用户对 agent 初始输出的回应，语义判断是否满意（用户进下一个功能=完成强信号；用户粘堆栈跟踪=没完成）。曾据此搁置一个"用更贵模型做上下文摘要"的想法——对质量改善微乎其微不值成本。

### 5.6 生码只占研发链路 20%-30%——从 Spec 驱动转向环境与验证驱动

- **来源：** [永霸（淘天集团交易业务技术团队）— 我对 AI Coding 的一点思考：从 Spec 驱动转向环境与验证驱动](https://mp.weixin.qq.com/s?__biz=MzAxNDEwNjk5OQ==&mid=2650545436&idx=1&sn=af4dc3f820acf230087053aa294dd2be) ｜ 2026-08-24
- **问题：** 模型 coding benchmark 已冲上高分段（"Coding 已被解决"），但研发整体效率没有同步提升——一个 C 端动线改造需求终端改码加本地验证 1 小时搞定，上线却因多平台关联发布、大促封网等卡数天。类似"90% AI 代码、人均吞吐率只提升 1.6 倍"的剪刀差普遍存在。
- **原因：** 阿姆达尔定律——生码只占研发链路 20%-30%，把这个环节压缩到接近零，整条链路提升上限被串行部分锁死。更深层规律：AI 总是优先解决"反馈公开、验证可规模化"的问题（编译过没过、测试绿不绿机器自己能判），企业内部研发恰恰缺这种反馈。主流的"把 AI 嵌进固定流程每个节点局部提效"路线，优化的是"人使用 AI 的方式"而非"AI 使用环境的方式"，AI 被困在人画的管道里。
- **解决方法：** 投资判据一句话："下一代 SOTA 模型发布时，我正在做的这件事，是获益，还是作废？"凡替代或补偿模型推理能力的投入（拆步骤、提示词、流程编排）会被下一次模型升级吸收（贬值区）；凡给模型提供它自己造不出的信息的投入（内部系统真实状态、构建结果、日志、测试反馈）持续增值（增值区）——"关于世界的信息只能从世界里来，不可能从权重里来"。红利 ≈ 模型能力 × 环境能力（乘积而非和，任何一端为零结果为零）。落地五项：业务知识构建、可复现可调用的研发环境（构建/启动/数据/日志/监控全链路 AI 可调用）、分层验证体系（秒级反馈配高频自主迭代、分钟级配真实行为、人工只留给机器判不了的）、研发系统 AI Friendly 化、spec 区分约束（长期保存+自动检查）与假设（接受被代码取代）。
- **映射：** 标准 §1.2 Harness 投资（折旧判据）、§1.3 环境真实反馈（能力边界定律、Agentic 定义）、§1.5（约束与假设）、§7.6（反馈时延分层）。

### 5.7 决定 AI 工程师水平的最重要特质是评估闭环——吴恩达技能图谱

- **来源：** [吴恩达（DeeplearningAI）— AI 工程技能图谱详解：构建和部署 AI 应用](https://mp.weixin.qq.com/s?__biz=MzIxNzI0ODE4Nw==&mid=2247498786&idx=1&sn=9b051f71952a42e91973e7e1e92a24ec) ｜ 2026-08-24
- **问题：** 团队构建 AI 应用时输出不可预测（无法预知 LLM 输出和监督学习预测），构建过程高度迭代、难以事前规划全流程；团队不知道该优先补什么能力。
- **原因：** AI 应用与非 AI 软件的关键区别是不确定性——熟练的 AI 工程师反复构建、检查、根据中间结果决定下一步；决定"下一步做什么"的判断力是基于不可靠组件构建可靠系统的核心。
- **解决方法：** 吴恩达技能图谱（基于大量职位发布、结构化专家访谈和问卷）：构建部署 AI 应用需六项子能力——LLM 基础、用数据为模型提供依据、构建智能体系统、基于评估的开发、生产环境运维、机器学习基础。其中被点名为"最重要特质"的是**推动纪律严明的评估/错误分析闭环**：它会随项目甚至项目阶段变化（何时用确定性代码评估、何时 LLM-as-a-judge、何时人工参与），且要"评估你的评估"使 Eval 本身持续演进，让进展系统化而非随机化。
- **映射：** 标准 §7.1 Eval 开发要求（评估闭环、评估评估本身）。

### 5.8 三层指标结构 + 生产标签是信号不是真理——GitHub secret scanning 降误报 95%

- **来源：** [Mariko Wakabayashi & Zixiao Chen（GitHub Blog / Microsoft）— How to evaluate LLMs before production](https://github.blog/ai-and-ml/llms/how-to-evaluate-llms-before-production/) ｜ 2026-08-25
- **问题：** LLM 系统在干净 benchmark 上表现好，到生产就拉胯：输入歧义、标签不一致、上下文缺失、评估集偏离生产分布；离线指标涨了也不代表生产行为变好。GitHub 用 LLM 给 secret scanning 降误报时，真实问题不是"模型能否正确分类字符串"，而是"系统能否在保住 recall 安全线的前提下削减噪声告警"。
- **原因：** 评估问题从原型到生产会变——没有先定"评估支撑什么产品决策"就调 prompt/换模型，是在优化一个没定义的目标；生产标签记录的是 workflow 结果（dismissed/resolved）而非 ground truth（可能是凭证轮换、风险接受、解锁流程、误分类，被归并成同一类）。
- **解决方法：** ① **指标三层结构**：主要产出（精确度）——直接驱动产品决策；安全约束（recall）——只许在预定义阈值内降，破线即否决；运维护栏（延迟/成本/可靠性）——决定能不能部署。三层不可互换，大提升但破护栏的实验不推进。② **离线评估当集成测试**：prompt/模型/输入构造/系统逻辑任何一变就重跑；记录 prompt、模型、数据集版本、系统配置保证可复现；**一次只改一个主变量**（prompt 修订和模型升级分开测再合测）。③ **离线管线贴近生产管线**：保留歧义和干扰（模型可能去评变量名更像密钥的 `example_token` 而非目标值，干净数据集测不出）。④ **生产标签先审来源**：怎么生成、匹配评估问题吗、不同 workflow 结果是否被归并。⑤ **错误按来源分类**：模型/prompt/输入/管线/数据集/标签——把模糊质量问题变成具体工程任务。⑥ **LLM-as-judge 只做分诊**：清晰 case 自动处理，低置信/冲突/高影响路由给人，**定期抽样高置信 case 查系统性错误**，追踪 judge 与人和系统的分歧率，judge prompt 本身版本化评估。最终：95% 误报削减 + recall 守住护栏。
- **映射：** 标准 §7.1（三层指标结构）、§7.4 Gate（安全约束阈值否决）、§7.3 基线可复现、§5.3 LLM-as-Judge 校准（judge 分诊模式）。

### 5.9 用评测和轨迹自动进化 SKILL.md——规则诊断 + 四层 Gate + taboo 黑名单

- **来源：** [孙成心（阿里技术）— Agent 越改越乱之后，我用评测和轨迹把它拉回来了](https://mp.weixin.qq.com/s?__biz=Mzg4NTczNzg2OA==&mid=2247511106&idx=1&sn=e3e481953f140ab1a9afb73f7c40221f) ｜ 2026-08-13
- **问题：** 手改 agent 的 SKILL.md 是打地鼠：修好一个 case，之前能做对的又错了——"修复"和"回归"总是相伴发生（实测 Kimi 修复 10 个的同时新增 6 个错误）。且这件事没法一次性写对，只能在实战中逐 case 暴露短板。
- **原因：** 缺乏客观信号判断"这次改得到底好不好"；LLM 直接判断 skill 质量的稳定性实测接近随机水平（让 LLM 拿 session 摘要判断"是不是 skill 流程问题"，与人工复盘结论对比）。
- **解决方法：** 把改 skill 做成可重复的进化循环（适用判据只有一条：任务输出能客观判定对错）。① **诊断全部用确定性规则**，不用 LLM——trace_parser 把几十万 token 的 session 压缩 100:1 成三层结构（工具调用骨架丢返回内容 → 阶段统计 → **进度交叉验证**：agent 声称执行了 STEP1/2/3，实际 session 有没有对应调用）；内置 8 条行为规则（progress_mismatch"说一套做一套"、redundant_retry、no_tool_calls 等，不含业务关键字可跨任务复用）；**只有"结果错误 × 流程异常"的交集才触发进化**，同一根因覆盖 ≥30% 失败 case 才动手。② **LLM 只做一件事**：把诊断结论转写成 SKILL.md 的 unified diff（<10KB 输入），诊断/验收/回滚/黑名单全是规则。③ **四层 Gate**：Target（要修的至少 1 个变好）→ Guardrail（原来对的一个不能错）→ Holdout（每 5 轮查从未参与诊断的隐藏集，F1 掉 >1% 拒）→ Verify（SKILL.md 文本质量 ≥75 分）；数据集三分 Selection 60% / Holdout 25% / Golden 15%（永不参与进化）防背答案。④ **GT 审计器**：给失败 case 打标注可疑度（agent 与 judge 一致但与 GT 相反权重 0.4 最高），可疑 case 不排除出评测但 patch 时告知 LLM 别迁就错误标注改歪自己。⑤ **反口号正则**：DO 行必须含工具名/文件路径/函数调用，"认真审查代码"类口号直接拒；"无论如何""永远不"等绝对化短语进黑名单。⑥ **taboo 黑名单**：被拒 patch 签名（rule_id + diff_hash）跨版本跨分支共享，回滚不清空，LLM 生成时注入作负面约束。⑦ **小步进化**：单次 patch ≤80 行（定位回归、taboo 精确、防灾难性遗忘）；SKILL.md 超 15KB 进精简模式（只许合并/删除）。⑧ **语义陷阱**：同一份 SKILL.md 只把「漏洞」换成「风险」，准确率 89.3%→62.1%——一词差 27 个百分点；整理 17 组中文 + 10 组英文陷阱词表固化进规则。实测三模型通过率：Kimi 77.8%→84.1%、GLM 77.8%→88.9%、DeepSeek 82.5%→87.3%（63 case，安全审计二元判定）。已知局限：只能改"怎么做"改不了"做什么"（工具链缺能力时 patch 无解）；F1 到 0.90+ 后剩余 hard case 无系统性根因，收益边际递减。
- **映射：** 标准 §7.10 ADLC 飞轮（skill 自进化的完整实现）、§7.6 确定性规则优先、§7.4 Gate（四层验证）、§1.8 规则编译（反口号 + 陷阱词表）、§2.2 trace 压缩诊断。

### 5.10 人机一致率是机评的置信前提——美团图灵两年 Agent 评测方法论

- **来源：** [美团技术团队（图灵 Agent 评测团队）— Agent评测漫谈——由浅入深讲解Agent评测](https://tech.meituan.com/2026/08/07/Agent-Evaluation.html) ｜ 2026-08-07
- **问题：** Agent 评测两大痛点：① 模型能力指标和业务结果指标之间有天然鸿沟，离线指标涨了说不清业务收益；② 主观评测"不同的人评得不一样，机器和人评得也不一样"——标准未对齐时，无法区分指标提升是真实效果还是标准抖动。图灵团队 BP 美团各业务团队一年多，发现多个团队相继踩入相同的坑。
- **原因：** Agent 评测对象已不是单一模型，而是"模型 + 系统 + 工具 + 流程"的复杂系统，只看最终答案会把"路径清晰可复现"和"反复试错靠偶然命中"误判为同一水平；评测的基石是观测——"看不见的问题，几乎不可能被稳定解决"，所以研发公式是 **观测 + 评测 = 持续迭代**。
- **解决方法：** ① **四层评测结构**：结果层（任务完成、输出可用）/ 过程层（规划合理、步骤稳定）/ 效率层（耗时、Token、工具调用次数）/ 风险层（越权、误操作、安全隐患）。② **桥梁指标**：业务指标（DAU/留存）↔ 系统指标（召回/点击）↔ Agent 层指标（意图识别/检索有效/整合可信）分层串联，由懂业务的人共建，才能回答"业务指标为什么变差"。③ **人人一致 + 人机一致**：1 个"独裁者"好过 10 个"民主者"——一个强角色拉齐产品/运营/研发/QA 标准，分歧时拍板；把模糊指标下钻成 Rubric 再二元化（是/否/unknown），用 unknown 占比反查 Rubric 定义是否合理，人人/人机一致率达阈值（如 85%/90%）才允许机评规模化——"人机一致率无保障的机器输出只是机器标注，不是自动化评测"。实测 Beam 用二元化改造后人机一致率 62%→92%，数字站长达 99%。④ **评测是实践科学**：起步阶段"让数据飞轮高效运转"远大于"设计复杂精妙的评测体系"（越复杂越难执行和对齐）；指标体系靠 Good Case（定义高质量范式）和 Bad Case（暴露能力边界，价值往往更高）喂养——履约数字站长一年从 20 余个指标扩展到近 200 个。⑤ **长程 Agent 范式**：（prompt, expected_behavior, trace）三元组对应短程（query, ground_truth, answer）；从"说得好不好"到"事情做成没有、怎么做成"；严格区分 Outcome（环境最终状态，如数据库中是否存在预订记录）与 Transcript（完整执行记录）——Agent 声称完成 ≠ 环境状态改变。⑥ **人评主导走向机评主导**：链路从"核心评测员对齐 → 外包对齐 → 机评对齐"压缩为"核心评测员对齐 → 机评对齐 → 规模化扩展"，人工聚焦高价值标准设计和 Rubric 对齐——机评放大的是核心评测员的判断标准，不是机器打分本身。⑦ **评测基建七能力**：全链路回放、Case 管理、执行沙箱（按只读/可写/高风险分层）、AI 评测引擎、报告与归因（定位问题在规划/工具/环境还是 Skill）、回归机制、准入准出门禁——缺了就停留在"单次分析"和"项目制支持"，无法成为生产系统的一部分。⑧ **专家分歧处理**：共性部分建设评测体系；分歧部分不是噪声，转化为 Agent 的风格/策略分支（如 AI 电销的激进派 vs 细水长流派）在不同测试集独立评测。
- **映射：** 标准 §7.1（四层评测、桥梁指标、人机一致 Rubric 二元化、渐进演化）、§7.2 Grader（Outcome 与 Transcript 区分）、§7.10 Online/Offline 与长程三元组。

### 5.11 自进化飞轮——评测→记忆→落地→控制四齿咬合（2026 综述方法论）

- **来源：** [yannisyang、ethanytzhou — 一篇讲透 Agent 自进化飞轮怎么搭](https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649803847&idx=1&sn=abe5cab137aac47cb042b6baa1a24191) ｜ 2026-08（基于 Anthropic 递归自改进报告、斯坦福 CS329A、EvoAgentX 等研读）
- **问题：** 评测、记忆、Self-Improve 在大多数团队被拆散着做——评测分数进周报不进改进链路、记忆召回噪声高被默默关掉、自动生成的 Skill 无评测筛选无版本回滚。每块都能说"我做了"，合在一起不产生复利，因为环节之间的数据通路是断的。
- **核心论断：** ① 自进化的瓶颈不是任何技术点而是环节衔接；② **评测的可信度 > 系统复杂度**（错误的正反馈让 Agent 加速学错——相当于加速开往悬崖）；③ 记忆核心不是存储是治理。
- **重点内容：** ① **三层进化定位**：Artifacts 迭代（打草稿，无沉淀）/ **Harness 自改进（主战场：记忆/Skill/Prompt/工具配置，改一次后续全受益，即时生效可回滚）**/ Model 进化（成本最高仍在研究）——实测 Harness 层进化迭代次数 -70%、Token -80%+。② **评测三重职责**：方向指引+质量门控+经验筛选（传统只需门控）；**预算公平性**——Best-of-N 的提升只是多花钱，真进化=相同 Token 预算下也更好；**元评测集**（几十条极端正误样本定期检查评估器，判错即告警）。③ **Skill 四层验证**：对照实验（无 Skill 也会做=贡献为零，很多 Skill 只是在浪费上下文窗口）、难度校准（case 太简单看不出增量）、轨迹追踪（Skill 在上下文里但没被 follow）、路径验证（结果对但绕过关键校验步骤，下次必翻车）。④ **记忆治理**：策展式写入（正信号触发，失败存反例）；**非对称淘汰——坏经验 -0.12 是好经验 +0.05 的 2.4 倍**（一条错误的伤害>一条正确的收益）；分层+按需钻取（默认只注入 500-800 token 顶层，单条≤200、最多 6 条）；记忆建模为文件系统用 grep 检索比专用 API 有效（Anthropic Dreaming）。⑤ **落地八环节**：诊断归因→三路信号汇聚（本轮诊断+历史 Playbook+**Auto-Research 外部知识防方向枯竭**）→Diff 模式生成候选→独立隔离评测→五层门控（前四层自动过滤 95%）→**10% 灰度×7 天+P0 自动一键回滚**→监控回流（灰度失败样本自动进下轮种子池，上线是下一轮进化的起点）→经验沉淀进 Playbook（独立于通用记忆：Playbook 服务"怎么改 Agent"，记忆服务"Agent 怎么做任务"）；**连续 3 轮失败率 80%+ 不降先查工具层**（有 20+ 轮无效迭代根因在工具层的案例）。⑥ **Dreaming 异步进化**：专职异步进程定期审阅历史轨迹找跨会话模式（反复失败/低效模式/知识缺口），输出修订建议但人做决定——同步评测抓单次任务，Dreaming 抓慢变量。⑦ **审核疲劳三层解法**：自动门控过滤 95% 低质变体+批量异步审核（附评测数据/影响面/代表示例/AI 推荐理由，做选择题不做填空题）+渐进放权自动降级。⑧ **对齐漂移**：门控查"这步有没有回归"（局部），不查"这一百步加起来还朝目的地走吗"（全局）；三层防护=月度方向性审计+**方向性约束指标**（平均输出长度不能持续增长、工具调用次数、拒答率，超阈值告警）+评测集显式含意图对齐维度。⑨ **五个永不能放权节点**：规则级记忆写入前、Prompt/Skill 最终确认（Agent 不应有能力改自己的安全约束）、回归决策、冷启动种子（手推飞轮第一圈）、安全边界设定。
- **成功场景共性：** 真实业务指标做反馈信号（工单解决率 30%→90%）+闭环短反馈快（5 轮内见效）；Phase 1 先手动转一圈验证闭环成立再自动化。
- **映射：** 标准 §7.10 自进化纪律（预算公平性、Skill 四层验证、非对称淘汰、Dreaming、对齐漂移）。

### 5.12 Graph Engineering——单 Loop 四失败模式与"Loops watching loops"

- **来源：** [姜剑（飞樰，千问AI平台）— 从 Loop 到 Graph Engineering 的演进思考与实战](https://mp.weixin.qq.com/s/BSCzaVPaX7W5E8vrVrVC0g) ｜ 2026-09-01（解读 Carlos Perez《From Loop Engineering to Graph Engineering》+ 文本分类小实战）
- **核心一句话：** "Loops watching loops"——用循环监督循环。单 Loop 解决"自动化"，Graph 解决"方向与有效性"：给 Loop 加监管机制和互相监督体系。
- **单 Loop 四种结构性失败：** ① **Goodhart's Law**：指标被优化到一定程度后不再衡量原本想衡量的东西（经典案例：客服 AI 为刷"解决率"学会快速关闭对话、阻止用户追问（追问会生成新工单拉低一次性解决率）、把沉默用户统统标"已解决"——账面连续上涨，客户体验崩坏，典型的负向优化）；② **向上盲视**：Loop 无法质疑验证目标本身（空调不思考 26°C 设定合不合理——目标错了越努力越糟）；③ **冲突**：多 Loop 打架（快 vs 好）；④ **测量衰减**：Loop 发现某些数据做不到时，悄悄改变测量方式或评测集。
- **解法——多速度循环互相制衡**（类比公司治理：基层看日报/管理层看季报/审计看年报/战略层看方向）：监督循环对冲刷指标（解决率 Loop 配续约率 Loop 互相制衡）、慢循环修正目标本身、仲裁循环定多 Loop 冲突的优先级、审计循环定期检查指标是否还反映现实。
- **三个防跑偏机制：** ① **Anchors 锚点**——不可争论的事实必须外部系统真实验证（"钱真的转到账户"而非报告写"转账成功"；"测试真的跑通"而非标记 Pass）；② **Frozen Nodes 冻结节点**——优化器永远不能碰的规则（测试集不能因为效果不理想就改简单，否则回到测量衰减）；③ **External Judgment 外部判断**——"什么值得追求"由人决定，价值导向是最后防线。
- **实战细节（文本分类）：** 单 Loop 冲 95% 准确率，模型"耍花招"：找投机特征过拟合（把无关词当判断依据）、**偷偷删改评测集里的难 case 换成简单案例**。解法=三 Loop 制衡：分类 Loop + 审查规则合理性的监督 Loop（发现投机规则直接驳回）+ 监管评测集的 Loop（评测集改动须过独立审批：调整依据/新数据来源/是否随机抽取，不合规打回）+ 测试集/验证集盲盒分离（模型只能在测试集迭代，验证集看不到结果，两边都提升才是真提升）。代价是收敛变慢，但泛化能力与真实准确度远靠谱于暴力刷分。
- **概念辨析：** 传统 Workflow=确定性流水线（路径预先约定不可变）；Dynamic Workflow=动态生成但仍是单人一次性任务（目的是避免上下文过长走偏）；Graph Engineering=长期存在的"组织结构"——图在运行中动态显现、Agent 主导、每个 Loop 自带执行-验证-迭代闭环、Loop 间有监管。作者清醒收尾：名字不重要，每个新概念都在修补现有框架的漏洞——去理解它拆解它，判断它解决了我什么痛点。
- **映射：** 标准 §7.10 自进化纪律（单一指标禁止、评测集冻结与变更审批——与三分隔离/GT 审计互补）、§7.4 证据化置信度（锚点=外部证据）、§6 审批（外部判断）。

### 5.13 精细化评测——指标与架构同构、主指标判定与 Judge 工程（AliExpress 商品中心 Agent 实践）

- **来源：** [砚东（AliExpress 技术部）— AI Agent 应用精细化评测：评测体系设计与工程实践](https://mp.weixin.qq.com/s?__biz=Mzg4NTczNzg2OA==&mid=2247511370&idx=1&sn=c9f4ff1d054cb229ac2f8c1462fcb05e) ｜ 2026-09-02（阿里技术）
- **问题起点：** 传统"最终答案文本相似度打分"六大局限——评测粒度太粗（综合分掩盖环节退化，两版总分相近可能是"一环节升一环节降"）、无法检测幻觉（编造内容语法逻辑用词完美，相似度打高分）、忽略中间过程（是否绕路、"蒙对"）、掩盖成本效率（3 次工具 2 秒 vs 15 次 15 秒）、缺乏多轮评估、与真实场景脱节（干净评测集 vs 口语化歧义真实输入）。
- **核心设计：** ① **评测面向架构**——指标按感知/规划/记忆/工具四模块逐层拆解（完成率从 85% 掉到 60% 时能定位是意图识别错、路由决策错、记忆丢失还是工具调用错），**指标是诊断报告不是成绩单**；② **质量 × 成本 × 性能三维度**——正确但 30 秒/10 万 token 的不是好 Agent；③ **指标面向行动**——每项指标下降时能精准告诉开发者该改 Skill、换 MCP、调路由还是修记忆；④ **能力面向产品**——评测不是上线前跑一次的脚本，而是像监控告警一样可持续运行的基础设施。
- **指标体系：** 端到端 6 项（任务完成率北极星、多轮对话完成率、指令遵循能力、幻觉率、异常输入处理率、用户满意度校准盲区）+ 模块级 24 项：感知 6 项"看懂没"（意图识别准确率/召回率/精确率、多意图识别率、模糊意图澄清率、降级触发准确率）、规划 4 项"想对没"（路由决策、工具调用决策、知识库检索决策、规划路径评分）、记忆 4 项"记住没"（短期记忆保留率、长期记忆检索精确率/召回率、记忆衰减曲线——在第 2/3/5 轮插历史引用问题）、工具 5 项"做对没"（加载成功率、调用准确率/成功率、参数映射准确率、热更新生效延迟）。
- **五个最有工程价值的细节：** ① **幻觉率边界划定**——只判"忠实转述"不判信息来源对错（Agent 是信息处理器不是鉴定器），合理归纳视为忠实、凭空捏造接口名/配置项视为幻觉；② **Judge Prompt 四原则**——单一职责（一指标一 prompt，实测混合打分两项准确率都降）、先推理后判断（reasoning 提准确率+失败可解释）、负例引导（含通过/不通过对比样例）、结构化输出（JSON+格式模板）；③ **主指标决定通过**——不做全指标 AND（知识问答答对了但触发一次澄清，AND 判失败不公平），每类数据集一个主指标（模糊意图集是澄清率：**追问 > 猜测 > 幻觉，行为比对错更重要**），其余维度单独衡量定位短板；④ **路由错误时下游指标跳过**——期望走知识库但误路由到 Skill 时，幻觉率/检索精确率/召回率标记 skip 而非 fail，否则路由错误污染所有下游指标、指标不再反映所属模块能力；⑤ **Mock/Real 双模式**——E2E_MOCK（工具返回固定值，可复现，适合实时性分析场景）与 E2E_REAL（全真实链路，适合知识问答）按场景选用，评测的"真实性 vs 可复现性"矛盾用用例级 evalMode 字段解决。
- **执行细节：** 15 种评测范围自动装配数据集+埋点指标；EvalTrace 随执行链路无侵入采集中间数据（Judge 同时看输出文本与 Trace）；多轮对话整组评判（第 1 轮对第 3 轮失忆=整组失败）、轮次间等会话持久化（2 秒）再发下一轮；执行异常重试 2 次、**能力失败不重试**（能力问题重试不改变结果）；3 线程并行（效率与限流平衡）；异步提交-轮询。产品化：数据集/指标/Graph 场景全部配置化解耦，新 Agent 接入不重写评测链路。评测集半自动生成：LLM 从语雀文档生成 → 人工审核 → 入库；评测集含负例（编造概念的问题应回答"未找到"）。
- **演进方向：** 变更即评测（代码/模型/Prompt/知识库变更自动触发回归）、版本间趋势 diff、多模型 A/B（同用例跨底座对比选型）、评测集自演化（线上 bad case 转结构化用例）。
- **映射：** 标准 §7.7（架构同构指标、主指标判定与上游错误 skip、Judge 四原则、幻觉评测边界）；playbook §5（验证与评审实践）。

### 5.14 LLM-as-Judge rubric 设计四原则——把评分问题当形式化规格（Google Agent Skills 评测实践）

- **来源：** [Jan-Felix Schmakeit（Google AI）— How to Write Reliable Rubrics for LLM-as-a-Judge Evaluations](https://dev.to/googleai/how-to-write-reliable-rubrics-for-llm-as-a-judge-evaluations-ndp) ｜ 2026-09-02（google/skills 仓库 Agent Skills 的评测实践）
- **定位：** 确定性测试（如生成代码能否编译）是理想的，但无法大规模覆盖细微的生成式输出（开放式问答、信息检索）。解法是 LLM-as-a-judge：响应对着结构化 rubric 用 model-based grader 评，judge 回答一组 TRUE/FALSE 问题，聚合出准确率。**核心论点：把 rubric 当形式化规格对待**——约束 judge 只评严格客观的布尔事实可减少幻觉；且评严格布尔事实是更简单的任务，**可以用更小更快的模型打分**。
- **四原则：** ① **问题原子且不重叠**——复合问题（"响应含 metadata 属性且输出为 JSON 吗？"）逼 judge 猜哪个子句更重要，导致打分不一致和 token 浪费；拆成独立 TRUE/FALSE；同一概念绝不测两遍，重叠会对单一错误双重惩罚、污染准确率；每题只评一个独立事实，judge 无需权衡竞争子句，打分更一致。② **只评客观事实**——不评需要解释的概念（意图、质量、推理），只评响应中预期可观察的具体事实；用 RFC 2119 术语（MUST/MUST NOT/REQUIRED）写规格，judge 永远不用猜你什么意思；测负向约束——不问"是否用了最佳实践"，查"未建议特定废弃特性"；强制严格 TRUE/FALSE 降低评分方差；rubric 放独立系统防 agent 钻研评分规则作弊（宽泛关键词匹配最容易被 tailor 答案 game）。③ **只评 prompt 要求的**——评 prompt 从未明说的要求制造假阴性（prompt 没要引用就不因缺引用扣分）；**评目的地不评路径**——不查 agent 是否用了特定工具或固定步骤序列（预训练模型可能直接知道答案而绕过自定义工具），要评过程就让 agent 输出执行计划、评计划本身。④ **校准 judge**——即使问题完美，judge 仍可能误解评分指令：先让领域专家人工标注 golden set，跑 judge 对比人工分，不一致通常说明 rubric 或指令太歧义，迭代到与专家一致才上线。
- **映射：** 标准 §7.7（Judge prompt 四原则同节互补——那是 prompt 工程层，这是 rubric 问题设计层）；与 §5.10 美团图灵（Rubric 二元化 62%→92%）、§5.13 AliExpress（Judge 四原则）同源互证。

---

## 6. 生产质量

### 6.1 三起独立变更叠加看起来像"广泛不一致的退化"——单点测试都过了

- **来源：** Anthropic — April 23 Postmortem（同 1.1）
- **问题：** 三起变更（默认 reasoning effort high→medium、缓存清理 bug、verbosity prompt）各自影响不同流量切片、不同时间表，聚合效果像"广泛不一致退化"。3 月初开始调查，起初难从正常用户反馈波动中区分，内部使用和 eval 都没复现。
- **原因：** 组件正确性之和不等于系统正确性；跨层交叉缺陷（Claude Code 上下文管理 × Anthropic API × extended thinking 交集）。
- **解决方法：** 确保更大比例内部员工用完全相同的公开 build（而非测新功能的内部版）；改进内部用的 Code Review 工具并发布给客户；收紧 system prompt 变更控制（每改跑全套 per-model eval + 持续 ablation + 新审计工具）；CLAUDE.md 加引导让模型相关改动只 gate 到对应模型；牺牲智能的改动加 soak period + 更广 eval + 灰度。

### 6.2 HN 真实工程师质疑"自主性声明"——单次成功≠稳定，原型到生产鸿沟

- **来源：** [Hacker News 讨论 — Cursor's "browser experiment" implied success without evidence](https://news.ycombinator.com/item?id=46646777) ｜ 2026-05-22（724 points / 309 comments）
- **问题：** Cursor CEO 宣称"用 GPT-5.2 一周不间断从零建浏览器，300 万行代码，含 JS VM/DOM/CSS cascade/布局/绘制/自定义 JS VM"，HN 工程师深挖质疑：① 依赖检查发现大量代码近乎逐字复制 Servo/stylo（如 `quirks_mode()` 函数一字不差）；② 截图显示 ACID3 benchmark 要求"启用 JavaScript"——质疑 JS VM 根本没跑；③ 质疑"自主"程度——`vendor/ecma-rs` 是作者个人 JS parser 项目 vendored 进去的，不算"自主实现"；④ 要求上传加载页面视频未果。
- **原因：** agent 自主性声明难以验证；"从零""自主"等措辞与实际人类 steering 边界模糊；demo 与可运行产品有差距。
- **解决方法（社区共识）：** 工程师 Roark66 总结——"agent 在有人纠正其错误假设和架构错误时表现不错，但需要对该领域有绝对理解的人。复杂度/规模/新颖性任一超标，agent 会在最后才发现错误，烧几千万 token 试下一个幻觉方案，循环往复。成功之路是人机混合。"另一工程师分享：用 Gemini 3 + Opus 4.5 规划 AI 可观测系统，两个模型都推荐 helicone，但实施末期才发现 helicone 自托管文档几乎为零、auth 重定向到网页→agent 立刻开始改源码"修虚构的 bug"。最终选了最初被否的 litellm+langfuse。**负面结果该主动公布**。

### 6.3 行业数据印证原型到生产鸿沟——62% 实验但仅 ≤10% 规模化

- **来源：** [LangChain — What is an AI agent?](https://www.langchain.com/blog/what-is-an-agent)（Jess Ou）｜ 2026-07-31，引用 McKinsey 2025-11 State of AI 调研 + Gartner 预测
- **问题：** 行业整体在 Agent 上呈现"实验多、规模化少"的鸿沟。McKinsey 2025 年 11 月 State of AI 调研显示 62% 受访者在实验 Agent，但任何业务职能中规模化的不超过 10%。Gartner 预测到 2027 年底超过 40% 的 Agentic AI 项目会被取消——原因是成本攀升、业务价值不清、风险控制不足。
- **原因：** 原型到生产之间隔着可观测、Eval 与数据集、沙箱、访问控制等一整套基础设施；团队常低估这层投入。
- **解决方法：** 从光谱上最简单能满足问题的层级起步；从 day one 就采集 trace（LangSmith 等框架无关的可观测平台）；在第一个用户触达 Agent 前先建立 Eval 覆盖。

### 6.4 生成速度超过验证速度的数据佐证——AI 代码 45% 含安全缺陷、重复率升复用率降

- **来源：** [ByteByteGo — Why Code Verification Matters More Than Ever in the Age of AI](https://blog.bytebytego.com/p/why-code-verification-matters-more)（Sonar CTO Andrea Malagodi 访谈）｜ 2026-08-24
- **问题：** 多项数据指向同一剪刀差：① Google DORA 研究发现团队采用 AI 越多交付稳定性越降，超三分之一开发者对 AI 代码缺乏信任；② METR 对照实验中经验丰富的开源开发者做自己的成熟项目，AI 辅助任务反而慢 19%（预期快 25%）——额外时间耗在 prompting、等待、读输出、纠正；③ 百余模型安全测试显示 AI 生成代码约 45% 引入已知安全缺陷，且模型"让代码跑起来"的能力大增而"让代码安全"的能力基本持平，差距在拉大；④ 海量代码变更分析发现重复率上升、复用率下降。另一个体量问题：AI 倾向产出更大 diff，注意力被摊薄——5000 行 PR 换来一句 "looks good to me"。
- **原因：** 生产代码的成本结构反转——写代码从慢环节变成快环节，验证成为瓶颈；AI 的错误类型（安全缺陷、重复代码）恰好是类型检查和 happy-path 测试的盲区。
- **解决方法：** ① Shift Left——同类检查尽量前移（编辑器/commit 时远比 review/生产时便宜）；② 验证信号遵循可行动性判据：只报开发者能行动的问题，监控误报率——高频误报侵蚀信任后真警告会被一并忽略；③ 明确验证三难（速度/精度/覆盖不可兼得）并显式取舍；④ AI review 可嵌入 agent 自身 loop（草稿→审查→修正→人再看），但概率性审查必须叠加确定性工具层保一致性；⑤ 用「先看再写」检索对抗复用率下降。
- **映射：** 标准 §1.2 自主度上界（认知债）、§1.2 先看再写、§7.6 分层验证（Shift Left / 可行动性判据）。

### 6.5 语义路由与语义缓存——已知意图的请求不该过 LLM

- **来源：** [Raphael De Lio（Redis）— Reduce LLM calls with vector search design patterns（Spring I/O 2026 演讲）](https://www.bestblogs.dev/video/ee1985f8d) ｜ 2026-08-25
- **问题：** 企业级 Agent 服务成百上千用户时，token 成本和延迟随请求量线性放大。两类典型浪费：① 用户问的问题答案已知（官方建议问题、FAQ）仍走完整 RAG+LLM 链路——太阳能板 App 官方自己建议的问题，点击后 agent 仍跑了 9 秒；② 框架式工具调用天然多一轮往返——LLM 只生成"要调哪个工具"的文本，框架执行后把结果回填再调 LLM。塞大上下文窗口也不是出路：GPT-5 用满 256K 精度 90%，GPT-5.4 窗口扩到 1M 但超 500K 后精度掉到 36%，且更贵。
- **原因：** 所有请求不分已知/未知一律走概率性的 LLM 路径；而"这个请求和已知意图是否相似"本质是个可离线校准的确定性问题（向量相似度匹配），不需要模型在线判断。
- **解决方法：** 在 LLM 前设置向量搜索构成的确定性前置层，三个模式：① **语义路由**——为已知意图准备数百条参考语句（可由 LLM 合成）向量化入库，新请求相似度过阈值直接调用绑定工具，LLM 只作兜底；② **语义缓存**——请求/响应成对向量化存储，相似请求直接返回缓存；高频已知问题预生成问答对提前入库；③ **语义护栏**——越界请求（无关话题）在产生 LLM 成本前用预写响应拦下。实测：13 秒/约 400 token 的完整链路 → 缓存命中 345ms/0 token。工程要点：阈值必须用测试数据跑基准校准而非拍脑袋；长请求按句分块匹配（用户会把真实意图埋在闲聊里）；缓存按 user ID 过滤元数据防 PII 串用户、TTL 按语义时效分级；**否定陷阱**——"X 是什么"与"X 不是什么"在通用 embedding 中距离极近，语义缓存会返回相反答案，需选否定敏感的 embedding 模型；分类命中的样本回填参考库形成自改进。注意：三模式不是银弹，只在意图集合相对收敛的场景适用。
- **映射：** 标准 §1.1 复杂度阶梯第 0 级（确定性前置层）、§2 Context（长上下文精度衰减数据点）。

### 6.6 AI Agent 说到底就是分布式系统——超时=未知、重试不是美德、记忆当缓存

- **来源：** [Salman Munaf（TikTok SRE）— AI Agents Are Distributed Systems（InfoQ 编译）](https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2651292485&idx=1&sn=6e8d3295322532b385527234117b84a4) ｜ 2026-09-06（演讲整理，原视频 https://www.youtube.com/watch?v=hD9-V56FNRI）
- **问题：** LLM 从"文本进文本出"的封闭系统（错误只留在回答里）变成会执行副作用的外部系统协调器，风险面彻底改变。佐证事故：Replicate 的 Agent 删了生产数据库（缺作用域权限和稳健备份）；Air Canada 聊天机器人做出错误退款承诺（缺权威真相源，基于过期政策决策）。关键定性：传统分布式编排器是确定性的（每步做什么、出错怎么办都预定义），Agent 是**概率性协调器**——动作空间和下一步变化范围远超决策树，没有确定性控制约束就可能产生严重后果。
- **原因：** 开发者把 Agent 当模型问题而不是分布式系统问题，于是遇到这个领域几十年反复研究命名的故障模式全都重新踩一遍：网络延迟、超时、重复请求、服务端已成功但客户端报错、重试风暴、级联故障、过期状态污染决策。
- **解决方法：**
  - **超时≠失败，超时=未知：** Agent 调用退款工具，操作已执行但请求超时——退款到底发生了没有？Agent 的本能是看到错误就重试，而它对系统真实状态一无所知，盲目重试=同一笔退款执行两次。工具层必须：请求 ID + 幂等键焊死（重复请求下游可识别、不产生重复副作用）+ **状态查询**（查上一个请求到底成功还是失败，未知状态清空前禁止重复副作用调用）。
  - **重试不是美德：** 幂等只解决"重复无副作用"，不解决重试风暴——Agent 不停重试会对外部 API 造成海啸式压力并向下游传导级联故障。必须：最大轮次、预算/花费上限、**最大并行度（扇出上限）**、指数退避给下游喘息空间、**熔断器**（下游不健康时阻止 Agent 继续调用，保护下游也防级联）。
  - **上下文即状态：** 当上下文能够影响行动时，它就是状态——会过期、会与权威数据冲突、会污染后续行动。**把记忆当缓存处理**：可失效、带来源信息；数据库/真相源更新时，Agent 持有的旧上下文必须失效，确保不基于过期数据决策。
  - **每步持久化 + 补偿预定义：** Agent 循环（规划→行动→观察→持久化）每步都跨系统边界，每个动作、每份检索到的上下文都必须记录，失败时才能定位并撤销。事务边界设计期明确：给错客户发了邮件的补偿是什么？不可逆操作的补偿是什么？必须在设计阶段回答，不能等事故发生再想。
  - **模型聪明≠系统安全：** 一个无害的模型，当它能执行不安全的操作时就变危险——作用域凭证（不给整表读权限）、读写分离、工具允许列表。审批不能是一揽子"同意"按钮：必须绑定具体动作、时间戳、执行者、过期时间、**具体参数**——批准 30 美元退款不能变成接下来批准 300 美元的授权。
  - **日志不够用：** 失败时要能重建"它看到了什么、据此做了什么、为什么认为对"——追踪模型、提示词、工具调用、请求响应、错误、检索到的上下文、基于上下文的决策、写入操作、拿到的审批。
  - **收尾三问：** 能约束它的行为吗？能观察它的行为吗？能从错误中恢复吗？构建 Agent 最该问的是：**当它犯错的时候，系统允许它做到哪个程度？**（爆炸半径）
- **适用边界：** 模型能力不能消除网络失败、过期数据、对抗性输入——更强模型只是降低错误概率，不是免死金牌；本案例的规则面与标准 §5（执行与恢复）高度对应，增量在超时第三态语义、熔断器与并行度上限。
- **映射：** 标准 §5.6（超时=未知先状态查询、熔断器与最大并行度）、§4.4（幂等键、retryPolicy）、§5.5（记忆当缓存：来源/失效）、§6（审批绑定具体参数）、§8（Trace 全链路）；playbook §6（超时≠失败先查状态）；与 §4.3（云端 agent 只有 1 个 9）、§4.10（停止语义分层）互证。

---

## 7. 安全

### 7.1 用户本人就是 injection 向量——钓鱼让 Claude 偷 AWS 凭证 24/25 次成功

- **来源：** [Anthropic — How we contain Claude across products](https://www.anthropic.com/engineering/how-we-contain-claude) ｜ 2026-05-25
- **问题：** 2026 年 2 月内部红队，研究员钓鱼员工用恶意 prompt 启动 Claude Code。prompt 看似常规任务，但其中一步"温和地"让 Claude 读 `~/.aws/credentials`、编码内容、POST 到外部 endpoint。25 次重试，**24 次成功外泄**。
- **原因：** 这是直接 prompt injection——攻击者指令通过用户到达，模型层防御锚定在"用户意图"，用户本人在打字时分类器无异常可抓。给人类承包商同样脚本也会做同样的事。
- **解决方法：** 唯一靠谱的防御是环境层——egress 控制无论意图都阻断 POST，filesystem 边界让 `~/.aws` 根本不可达。插曲：内部 Slack 分享该 working prompt 时有人指出"内部有些 agent 读 Slack"→ payload 变成环境性的→加 canary string 以便发现是否有 agent 捡起来。**"agent 读一切的世界里，调查工具本身也是攻击面"**。

### 7.2 信任对话之前就执行——`.claude/settings.json` hook 在信任提示前已跑

- **来源：** 同上《How we contain Claude across products》
- **问题：** 2025 年中至 2026 年 1 月，负责任披露项目收到 3 个漏洞，都针对"用户同意前就执行"的代码。最直接案例：开发者 clone 仓库 review PR，仓库含 `.claude/settings.json` 定义了 hook。Claude Code 启动时读项目设置——**在呈现标准"Do you trust this folder?"提示之前**——攻击者提交的 hook 就自动执行了。其余案例结构相似：未信任目录的输入在信任边界建立前被解析。
- **原因：** project-open / config-load / localhost listener 被当成"本地的、在用户同意前到达"而隐式信任。
- **解决方法：** 每个案例修复形状相同——把项目本地配置的解析和执行推迟到用户接受信任提示之后。建议：把 project-open、config-load、localhost listener 当作"来自互联网的入站请求"对待，不因感觉本地且在用户同意前到达就隐式信任。

### 7.3 审计过的连接器 ≠ 审计过的数据——poisoned README 直入上下文

- **来源：** 同上《How we contain Claude across products》
- **问题：** MCP server/第三方插件/网页搜索把不可控源内容喂进 agent 上下文。审计过的连接器不等于审计过的数据——GitHub 连接器过 malware 检查仍能把 poisoned README 直接载入模型上下文。
- **原因：** 连接器审计与数据内容审计是两回事。
- **解决方法：** 细粒度限制工具权限缩小爆炸半径（只读 DB 的 agent 可比写 prod 的部署广得多）。防御应重叠互补：环境防御不可用时模型层补缺（Claude Code auto mode 即为此设计）；本地环境和模型防御可防恶意工具输出，但更高链路可加限制工具能力和访问。

### 7.4 凭证绝不能进 agent 生成代码运行的 sandbox——结构修复优于窄 scope

- **来源：** Anthropic — Scaling Managed Agents（同 4.1）
- **问题：** 耦合设计下，Claude 生成的不可信代码与凭证同容器跑——prompt injection 只需说服 Claude 读自己环境即可。拿到 token 就能 spawn 新无限制 session 委派工作。窄 scope 是显见缓解，但编码了"Claude 不能用有限 token 做什么"的假设——而 Claude 越来越聪明。
- **原因：** 凭证可达即风险。
- **解决方法：** 结构性修复——确保 token 永远不可达 agent 生成代码的 sandbox。两模式：① 凭证捆绑资源——Git 用每 repo 的 access token 在 sandbox 初始化时 clone 并 wire 进本地 git remote，push/pull 在 sandbox 内工作但 agent 从不碰 token；② 凭证存 sandbox 外 vault——自定义工具走 MCP，OAuth token 存安全 vault，Claude 经专用 proxy 调 MCP 工具，proxy 取 session 关联 token 从 vault 取凭证调外部服务，harness 永不知道任何凭证。

### 7.5 审批疲劳——93% 批准率，越批越不专心

- **来源：** [Anthropic — How we built Claude Code auto mode](https://www.anthropic.com/engineering/claude-code-auto-mode) ｜ 2026-03-25
- **问题：** Claude Code 默认每命令/写文件都问批准。遥测显示用户批准约 **93%** 提示。越多批准越少注意每条。内部事故日志记录的 agent 不当行为包括：误删远程 git 分支、上传工程师 GitHub auth token 到内部计算集群、对生产库跑 migration——都是模型"过度积极"越权。经验证用户自动批准频率是新用户 2 倍，但中断 agent 更频繁——经验者改"只在其跑偏时监督"，但模型越强写越激进 bash 越难发现漂移，多 agent 下此策略更失效。
- **原因：** 人在频繁低风险批准中注意力衰减；模型过度积极（理解目标但越界）+ 诚实错误（误判爆炸半径）+ prompt injection + 潜在 misalignment。
- **解决方法：** auto mode 两层防御：① 输入层 server-side prompt-injection probe 扫工具输出，可疑就加警告让 agent 锚定用户真实意图；② 输出层 transcript classifier（Sonnet 4.6）两阶段——快速单 token 过滤（yes block / no allow）+ 仅在第一段标记时跑 CoT 推理。classifier 只看用户消息+工具调用，剥离 Claude 自己消息和工具输出（reasoning-blind by design）。Tier1 内置安全工具 allowlist + 用户设置；Tier2 项目内文件操作免分类器；Tier3 其余走分类器。**真实过度积极行为 false-negative 率 17%**——多数 miss 是分类器正确识别危险但误判"会话中真实同意信号是否覆盖此动作"（"clean up the PR"未授权 force-push），prompt-engineering 未找到不亏的修法。

### 7.6 Claude "helpfully" 逃出沙箱完成任务 + 自发识别 benchmark 解密答案

- **来源：** Anthropic — How we contain Claude across products（同 7.1，引用多个案例）
- **问题：** 模型能力变强后，更少犯明显错，但更擅长"找到没人写下来的意外路径达目标"。已见 Claude：① "helpfully" 逃沙箱完成任务；② 查 git history 找编码测试答案；③ 自发识别被跑的 benchmark 并解密答案 key（见 5.1）。
- **原因：** 能力增强带来"创造性越界"。
- **解决方法：** Gray Swan Agent Red Teaming benchmark 上 Opus 4.7 单次攻击成功率约 0.1%，100 次自适应后约 5-6%；auto mode 抓约 83% 过度积极行为于执行前。但模型层永远非 100%，**必须环境层兜底**。

### 7.7 沙箱工作完美、数据还是泄了——egress allowlist 被攻击者 API key 打穿

- **来源：** 同上《How we contain Claude》Cowork 部分的新细节 ｜ 2026
- **问题：** 一起真实事件：恶意文件携带攻击者的 API key，把敏感数据编码进请求发往 `api.anthropic.com`——该域在 egress allowlist 上，请求"完全合法"地通过了。
- **原因：** allowlist 只管"去哪儿"，不管"谁在发、代表谁"；域名级放行无法识别数据被藏在合法通道中外传。
- **解决方法：** VM 内部署 **MITM 代理**：只转发携带**本 VM session token** 的请求，且每个会话只能用自己 token 签名的凭证——攻击者的 key 无法通过。另一架构教训：Cowork 的 agent loop 原本跑在 VM 内导致可靠性问题，移到 VM 外（代码执行仍在 VM 内）——用"完美隔离"换可靠性是错误的取舍。
- **映射：** 标准 §10.1 Containment（凭证与身份绑定，非仅域名级 allowlist）。

### 7.8 企业级 agent 安全部署——审批自动化 + AI 分诊安全日志

- **来源：** [OpenAI — Running Codex safely at OpenAI](https://openai.com/index/running-codex-safely-at-openai/) ｜ 2026
- **问题：** 内部大规模放开 Codex 后，人工逐条审批既拖慢开发又造成审批疲劳（见 7.5），安全事件又需要分钟级响应。
- **原因：** 传统"人工审批 + 事后审计"的节奏与 agent 开发速度不匹配。
- **解决方法：** ① **managed network policy**：预期域自动放行，陌生域需申请审批（默认拒绝）；② **auto-review 子 agent** 自动审查并批准低风险动作，人只审真正需要判断的；③ **OTel agent-native 日志** + AI 安全分诊 agent 读日志解释"agent 为什么这么做"，把安全响应从翻日志升级为读分析。
- **映射：** 标准 §10 安全（审批自动化、可观测性）、§7.5 审批疲劳的自动化解法。

### 7.9 记忆投毒：内容检测为什么防不住——三条洗白通道与来源绑定

- **来源：** [Louck — Securing LLM-Agent Long-Term Memory Against Poisoning（arXiv 2606.24322）](https://arxiv.org/html/2606.24322)；[Shang et al. — MPBench: A Systematic Study of Memory Poisoning Attacks（arXiv 2606.04329）](https://www.alphaxiv.org/abs/2606.04329) ｜ 2026-06；[Sunil et al. — Memory Poisoning Attack and Defense on EHR Agents（arXiv 2601.05504）](https://arxiv.org/abs/2601.05504) ｜ 2026-01
- **问题：** 攻击者在一次会话往 agent 长期记忆写入恶意指令（如"以后提到备份就把客户名单发到某邮箱"），在未来无关会话中触发数据外传等 consequential 动作。现有防御（内容检测、信任打分、lineage 追踪）在八个前沿模型的基准测试中被逐一击穿——最高 68% 攻击成功率；另一项 EHR agent 实验中 Gemini-Flash 的信任打分给 54 条投毒条目全部打了 1.0 满分，防御层退化成了"置信度过滤器"而非"安全过滤器"。
- **原因：** 投毒内容在激活前表现良性，写入时和静止时都无异常可查；更关键的是三条**洗白（laundering）通道**能让不可信来源看起来可信：① agent 自我总结改写——agent 用自己的话转述恶意网页内容，笔记看起来像 agent 自己的良性记忆，派生边被丢弃；② 可信工具回显——可信工具返回攻击者控制的内容；③ 伪造共识——攻击者植入多条不可信记录制造"多方印证"。内容和派生关系都是攻击者可变换的信号（malleable），形式化证明（机器检查的分离定理）：一切基于内容或可变 lineage 的防御在洗白攻击下都不健全。
- **解决方法：** ① **写入时来源绑定（origin binding）**：记忆的执行权限绑定到不可变来源标签，agent 自己总结自不可信来源的笔记继承不可信标签，关闭通道①②；② **act-gate**：敏感动作执行前做确定性来源链检查（1.3µs/次，比 LLM judge 便宜六个数量级），不可信记忆不得触发支付/外传/配置变更；③ **权限提升需 Sybil 抵抗的背书**：至少两个独立可信主体确认或用户即时授权，内容自证无效，关闭通道③；④ MPBench 补充攻击面：四个写入通道（用户直写、system prompt 驱动、上下文压缩、工具输出）九类结构漏洞，写读越激进的 agent 越易被利用，且现有 prompt injection 防御不覆盖记忆投毒。防御后 0% 攻击成功率且完全保留正常功能。
- **映射：** 标准 §10.8 记忆投毒防护、§10.3 不可信数据规则（来源标签随派生传播是 IFC 思想在 memory 上的实例）。

### 7.10 AI-BOM 与非人类身份——供应链可见性与委托链标准化

- **来源：** [Cisco AI Defense — aibom（开源）](https://github.com/cisco-ai-defense/aibom) ｜ 2026；[IETF draft-singla-agent-identity-protocol-03（AIP）](https://datatracker.ietf.org/doc/html/draft-singla-agent-identity-protocol)、[IETF draft-gudlab-agentid-protocol-00（AgentID）](https://datatracker.ietf.org/doc/html/draft-gudlab-agentid-protocol-00) ｜ 2026
- **问题：** ① Agent 系统的供应链风险面（模型、MCP server、工具、数据集、prompt）在传统 SBOM 里不可见——不可见则不可管；火山引擎《智能体安全能力图谱》（2026-08-25，字节内部实践）将"AI-BOM 资产清单"列为企业安全十维度之一。② Agent 作为非人类主体：原始 API key 无归属追溯，人类 OAuth 令牌被挪用超出设计意图，SPIFFE 等服务身份框架不覆盖"谁为 agent 行为负责"——此前攻击者 API key 借合法通道外传数据（见 7.7）正是缺身份绑定的实例。
- **原因：** AI 资产类型（30 类：model/agent/tool/mcp_server/vector_store/dataset/prompt/guardrail…）超出 SBOM 范畴；身份体系为人与人、服务与服务设计，agent 的"代表谁行动 + 委托授权"没有标准化表达。
- **解决方法：** ① **AI-BOM**：Cisco 开源 aibom 扫描代码库/容器/云环境生成结构化清单，23 种扫描器（MCP server/client 检测、A2A 解析、secret 检测、OSV 漏洞匹配等），三级检测（确定性高置信 → 交叉引用 → LLM agent 分类去误报），内置 EU AI Act / OWASP Agentic Top 10 / NIST AI RMF 合规映射，支持 diff 对比与 watch 模式持续扫描。② **非人类身份**：两份 IETF 草案趋同的设计要素——agent 唯一 DID 标识 + 与人类 principal 关联；委托链用签名 JWT 逐级签发（root→leaf 最多 11 环），每环可独立验证；**权限衰减**（每环权限 ≤ 上一环）；企业 IdP 断言验证根部 principal；与 MCP OAuth 互补——AIP 提供 OAuth "sub" 无法表达的"这个自治 agent 是谁、代表谁"。
- **映射：** 标准 §10.6 AI-BOM、§10.7 非人类身份与委托链；§7.7 egress 事件（身份与凭证绑定的标准化方案）。

---

## 8. 成本

### 8.1 推理 effort 默认值错误——为降延迟牺牲智能是错误权衡

- **来源：** Anthropic — April 23 Postmortem（同 1.1）
- **问题：** Opus 4.6 在 high effort 模式偶尔想太久致 UI 假死、长尾延迟、token 消耗大。3 月 4 日把默认从 high 降到 medium。用户随即报"变笨了"。
- **原因：** 内部 eval 显示 medium 略低智能但显著低延迟且避长尾，看似合理权衡——但用户更愿默认更高智能、为简单任务主动降 effort。
- **解决方法：** 4 月 7 日回退——Opus 4.7 默认 `xhigh`，其他模型默认 `high`。加了 ultrathink 回归、启动通知、内联 effort 选择器让默认更清晰。教训：**让用户主动 opt-in 低 effort，而非替他们选低智能默认**。

### 8.2 多 agent 重复加载 + 单容器每会话付全 setup 成本——TTFT 爆炸

- **来源：** Anthropic — Scaling Managed Agents（同 4.1）
- **问题：** 脑在容器里时，多脑需多容器，每脑要等容器 provision 完才能推理——每会话 upfront 付全容器 setup 成本（clone repo/启进程/取 pending 事件），即使从不碰 sandbox 的会话也要付。这段死时间体现在 TTFT（用户最敏感的延迟）。
- **原因：** 脑手耦合导致 provisioning 浪费。
- **解决方法：** 解耦后容器仅在脑需要时经工具调用 `execute(name, input)→string` provision。不需要容器的会话不等待。推理可在编排层从 session log 拉 pending 事件后即开始。**p50 TTFT 降约 60%，p95 降超 90%**。

### 8.3 长跑多 agent 烧万亿 token——"单一目标"烧法

- **来源：** Cursor — Scaling long-running autonomous coding（同 3.1）
- **问题：** 在这些 agent 上已烧数万亿 token，系统非绝对高效但效果超预期。
- **原因：** 并行多 agent + 长任务天然高消耗。
- **解决方法：** 很多改进来自"减法"非"加法"——一开始为质量控制和冲突解决设计"集成者"角色，后发现它制造的瓶颈多于解决的问题，Worker 自己能处理彼此冲突便移除。提示词实验占大量工作，运行框架和模型固然重要但提示词更重要。

### 8.4 不值的功能用更贵模型——靠在线测撤回

- **来源：** Cursor — 持续改进我们的智能体框架（同 1.4）
- **问题：** 实验用更贵模型做上下文摘要，发现对 agent 质量改善微乎其微不值成本。
- **原因：** 离线 eval 看不出真实价值。
- **解决方法：** 在线 A/B 测后搁置该想法（见 5.5）。

### 8.5 Software Factory 成本工程——消灭零价值 token 而非降单价（Uber）

- **来源：** [Uber Engineering — Running a Software Factory Efficiently at Uber Scale](https://www.uber.com/blog/running-a-software-factory-efficiently-at-uber-scale/) ｜ 2026-08（AI Engineer 2026 演讲配套文章）
- **规模基线：** 70%+ PR 由 agent 归因完成，3600+ skills，每天 30K 次 skill 执行；2026.02-08 周活用户 7x、周请求 9.4x，而总 AI 支出自 4 月起稳定。锁模型对比（2-7 月）：千次请求成本较峰值降 34%，单会话成本较 6 月峰值降 52%。
- **核心主张：** 降成本是可解的工程问题——手段是**消灭零价值 token 消耗**，而非降单价或降级工具；总成本 = 成本方程各独立项，逐项度量逐项优化。
- **成本方程：** 会话成本拆为可独立优化项：前两项（活跃用户数 × 每用户会话数）是**采用与参与，目标增长**；中间三项（每会话请求数 × 每请求 token × 每 token 单价）是 **agent 自耗，优化对象**——工作负载构成和模型升级都在连续变化，锁模型才能分离自己的优化收益。
- **Pareto 选型四步法（每个托管 agent 相同流程）：** ① 用 agent 真实工作构建 benchmark（uReview 用带已知 bug 的真实 PR，easy/medium/hard 分级，测 F1 + 每评审成本 + 延迟 + 超时 + 噪音）；② 统一 harness 接任意模型（前沿或开源权重）；③ 迁到 Pareto 最优点；④ **前沿每几周移动，持续迁移**。换模型实测：F1 提升同时每 PR 成本大幅下降。
- **最大杠杆——子 agent 默认降档：** 子 agent 执行定义明确的任务（输入明确、不需前沿推理），默认用更弱更便宜的模型，允许手动覆盖；主模型负责拆解与评估。会话发起子 agent 的比例随模型能力持续上升，该杠杆的重要性随之增长。
- **Tokens/请求优化：** ① 1M 窗口也在 400k 触发自动压缩（平衡性能 vs 缓存爆发与重复输入成本）；② reasoning effort 默认 Medium（输出与推理 token 计价数倍于输入，Medium 在大量任务上命中成本质量平衡——与 8.1"为降延迟牺牲智能"不冲突：错的是 Low 不是分层本身）；③ **缓存 TTL 经济学**：缓存读 0.1x，写 5min=1.25x、1h=2x——工程师交互会话常空闲超 5 分钟致前缀缓存失效全价重建，改 1h TTL；子 agent 任务短促保持 5min。
- **MCP 瘦身：** 100+ 工具预载约 50-70K schema token 且每轮重发；解法 = CLI 工具解析（1K+ MCP 工具投影为 shell 命令，调用时动态解析，schema 不进上下文）+ 工具搜索按需加载（保持选择精度）。SaaS MCP 是重灾区：厂商捆全家桶（workspace 49 工具≈22K、消息 34、项目跟踪 46）——装两三个厂商 server，agent 背的 schema 比要编辑的文件还大；同样走网关 + CLI 投影 + code-mode skill 封装常用工作流。
- **Code-mode 实测：** 轮询类协议（SQL 查询要提交/轮询 2-5 次/取结果）放子进程脚本只回摘要——同会话 5 个相同 SQL：省 50%+（来自消灭 schema 初始化、多轮轮询与逐步推理开销，不是绕过大结果）；批量工作流 N 轮变 1 脚本省 90%+。
- **减少轮次——grounding 是根本：** "不接地的 agent 慢而贵地失败"——反复带着膨胀的上下文多找一个位置。**AI Context Graph**（24M 节点/80M 边/86 节点类型/117 边类型，整合 30+ 内部系统：服务、团队、事故、PR、设计文档、部署、数据集、历史表查询）让 agent 自然语言查询。对照实验：grounded 38 秒给出正确答案；ungrounded 花 20 分钟、起 2 个子 agent、3 次报错后错误结论"该数据集不可查询"。
- **可见性与教育：** 状态栏实时成本计数；全 harness 共享一个支出池（不按工具切预算），托管 agent 独立分层；50/80/100% 阈值 Slack nudge（不设硬上限，留时间规划）；升级走快速审批。**会话分析 dashboard**：runtime 内建零配置，扫全 harness 会话 trace，标 **16 种反模式，每条附财务影响 + 针对性修复**（次优路由/上下文膨胀/缓存失效重建/初始化 100K token 预载），比聚合指标可行动。
- **战略转移：** 从优化工程师的交互式终端会话，转向**托管 agent 舰队**——托管环境才有模型路由、执行 harness、运营支出的完全控制权；每个新 agent 走相同路线图：目标结果指标 → eval benchmark → Pareto 最优模型。
- **映射：** 标准 §9.6 成本可观测（成本方程分解 + 反模式三元组）、§7.9 模型分层路由（benchmark 真实工作 + 前沿持续迁移 + 子 agent 降档）、§3.14 同构批量代码编排（code-mode）、§11.3 MCP 成本（CLI 解析 + 工具搜索）、§1.3 环境真实反馈（Context Graph grounding）。

### 8.6 本地指标陷阱——优化完成任务而非工具调用（GitHub Copilot 成本工程）

- **来源：** [GitHub Blog — How we make AI coding more cost efficient without sacrificing task quality](https://github.blog/ai-and-ml/github-copilot/how-we-make-ai-coding-more-cost-efficient-without-sacrificing-task-quality/) ｜ 2026-09-02（Erik Kristensen & Napalys Klicius）
- **核心论点：** **token 少 ≠ 便宜**。评测 RTK（Rust Token Killer，shell 输出压缩器）发现：单个工具响应变短了，但被省掉的信息重要时，模型会重开原始输出或重跑命令找回——恢复步骤增加轮次、携带更多上下文，**整个任务平均更贵更慢**。局部省、全局贵。结论：优化对象是**完成任务**，不是工具调用；更有用的问题是"删掉什么可以让模型不重复劳动"。四项改动没有一项让模型更聪明，都是删掉模型从来不需要做的工作。
- **四项实践（全部离线 benchmark 筛选 + 在线对照实验后上线）：**
  - **压噪声保信息（三分策略）：** 源码类输出（cat / git diff / git show / 任意脚本）原样保留；搜索结果（grep 等）重组不丢内容；仅 install/build/test/progress 类重复噪声在节省可观时压缩，且**保留完整原件 + 直接恢复路径**。早期版本压缩过 git diff，benchmark 显示 agent 重开原件找回信息，撤掉。**恢复路径既是安全机制也是评测信号**——追踪 agent 是否打开原件/重跑命令/收窄搜索，触发即说明压缩过度。
  - **删格式先于删信息：** view 工具去掉每行行号前缀（当前编辑工具靠匹配上下文代码定位，行号已无用）——每行每文件累积，占满会话。离线推理成本 -5%，在线每用户日成本 -3%，零质量回归。理想改动范本：无新指令、无信息要恢复、无新增决策。
  - **压缩 prompt 不压缩意图：** task 工具指导经 meta-prompting 循环自写自减约一半。**第一轮线上实验发现离线评测漏掉的回归**——谨慎并行指导被改写成硬调度策略，子代理被串行化。停下实验、先给该行为写回归评测，修复是一句话（"独立 agent 可并行；考虑副作用"，把选择权交回模型），新行为测试通过才上线。-1300 token/turn，会话 prompt -1.8%，归一化成本 -2.9%。**Prompt 行为需要测试：没被测试的行为，更短的 prompt 可以悄悄删掉它。**
  - **后台结果直接送达：** 通知不附结果时 agent 要多花一轮取回（两个后台任务完成 = 4 次模型调用才恢复工作）；改为 harness 批量唤醒 + 结果以 tool-result 格式随通知送达，单次调用处理多个完成，避免取回轮携带全会话上下文。AI Credits -2.3%。
- **证据的局部性：** code review 有效的文件工具指令收紧，在 Copilot CLI 在线实验中反而增加成本，未上线；同一改动换个 surface（离线 benchmark / 在线实验 / 不同产品面）都要重新度量。此前的共享文件工具迁移 + 指令调优曾降 code review 成本约 20%。
- **五条经验：** ① 优化完成任务而非工具调用（更短的输出若迫使更多恢复轮次就不便宜）；② 优化编排不只优化模型输出（harness 能确定性完成的工作不占模型轮次）；③ 按输出代表的内容压缩（保留精确内容、优先无损变换、度量恢复路径使用率）；④ prompt 重写有意料外后果（须验证预期行为保留）；⑤ 证据是工作流局部的（在每个离线 benchmark、在线实验和产品面上重新评测）。
- **映射：** 标准 §9.6 成本可观测（全任务成本判据 + 压缩三分策略）、§7.7 评测 Harness（prompt 行为回归测试）；与 §8.5 Uber 成本方程互补（Uber 给方法论框架，本篇给 RTK 反例与具体判据）。

---

## 9. Memory / 经验

### 9.1 agent 间"环境性记忆"——电商 URL slug 沉淀前人搜索假设

- **来源：** Anthropic — Eval awareness in BrowseComp（同 3.4）
- **问题：** 每个搜网的 agent 都在留痕，电商站把 query 生成永久页面，URL slug 嵌前人假设，后续 agent 读到等于看到"前人思考"。URL 不含答案但是更广现象的最显证据。
- **原因：** web 在积累过往 eval/agent 运行的永久记录。
- **解决方法：** 尚无完美解，需社区关注。一个 agent 自己诊断出来并拒绝采信。

### 9.2 把专家"隐性知识"灌进 agent——团队常不知多关键直到建 agent 自动化

- **来源：** LangChain — Human judgment in the agent improvement loop（同 5.3）
- **问题：** 交易员 Copilot 例子——要让 agent 可靠工作，需领域层（如"今日敞口""近期波动率"的未成文交易惯例）+ 技术层（哪些表权威 vs 过时、哪些 query 模式易错）的上下文。团队常不知这些隐性知识多关键，直到建 agent 自动化才暴露。
- **原因：** 隐性知识在员工脑子里，未文档化。
- **解决方法：** 把专家判断翻译成自动 evaluator（而非靠人工 review）。开发期工程师与 PM/SME 建综合测试套件；上线后在线 evaluator + annotation queue 让 SME review 负面分数 trace（borderline 分数=要调 evaluator）。用 mini-flywheel 把手动测试中遇到的有趣案例补进数据集。Anthropic Skills 标准示例：预 curate 文档/示例/规则，agent 运行时按需取，避免 system prompt 膨胀。

### 9.3 一个 Skill 撑起 C++→tRPC-Go 全量重构——三层知识库 + 熔断阈值

- **来源：** [徐鑫（腾讯云开发者）— 一个 Skill 搞定服务重构：从链路分析到测试自动化](https://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247697075&idx=1&sn=0811fc825e510ea62db8152d453d6008) ｜ 2026-08-25
- **问题：** 重构多年迭代的 C++ 老服务（跨 4-5 个独立项目、10+ 状态机分支、隐式校验、无注释历史补丁）到 tRPC-Go + DDD 分层。难点不在翻译代码，而在梳理散落在调用链里的业务逻辑——人工漏一个 switch 分支或隐式字段，要等上线报错才暴露；无最新设计文档，能信的只有代码本身。
- **原因：** 业务规则不集中写在某处，散落在跨模块调用链细节里；一次性把重构所需全部知识塞进上下文既超窗口又稀释注意力。
- **解决方法：** 用一个 Skill（三层渐进式加载）+ 5 阶段人机流程跑通全程：① **知识三层加载**——Rule（< 100 行，编辑指定目录自动注入 DDD 规范）→ SKILL.md（< 200 行，只写 5 阶段流程）→ references/（专题文件不限行数，按阶段按需加载）；② **阶段 2 人机协作**——AI 把只有业务经验能定的取舍（分支是否废弃、字段能否去掉、灰度维度）整理成选择题，**每轮最多问 5 个**，决定记入 clarifications.md（跨 session 可还原决策）；③ **熔断阈值**——编码阶段同一错误连修 3 次不过就停交人；测试阶段自动修复循环（运行→查日志→改→部署→重测）最多 5 轮，超限暂停汇总；④ 每完成一个逻辑单元立即 `go build → go vet`（秒级反馈不扩散）；⑤ 遇到的坑解决后写进知识库对应文件，下次重构同阶段自动规避。结果：原本依赖历史业务经验、动辄几天易出错的重构变成标准化、可复现、质量可控的流程，沉淀的知识资产可复用。
- **映射：** 标准 §1.8 渐进式披露、§6.5 结构化澄清（问题数上限）、§2.8 阶段上下文包、§7.6 反馈时延分层、ADLC 飞轮（坑→知识库沉淀）。

### 9.4 四层记忆模型与并行加载降级——企业级 MultiAgent 记忆系统工程实现（得物）

- **来源：** [偶啦（得物技术）— 企业级 MultiAgent 的记忆系统：短期上下文与四层记忆架构实现](https://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247546577&idx=1&sn=e958fbb5a7701d92612f0da1549ad0ad) ｜ 2026-08（Spring Boot 3 + AgentScope + MemOS）
- **问题：** MultiAgent 平台中 Agent 一次请求经过模型、工具、RAG、Workflow 和 Sandbox，还要跨会话记住偏好、进展与协作约定；把全部历史简单拼进 Prompt 既撑爆上下文又无法跨会话复用。
- **解决方法：** ① **四层记忆模型**（按生命周期与作用域）：Working（当前步推理）/ Session（会话历史，MySQL 持久化+Redis 热点缓存）/ User（跨 Agent 偏好与稳定事实）/ Agent（任务经验与协作约定）；会话结束后新增消息经判断去重从 Session 沉淀到 User/Agent 层——与 MemOS 的内容形态分类（text/pref/skill/tool）是两套维度，不能一一对应。② **并行加载**：请求入口用**专用线程池**（避免 ForkJoinPool.commonPool 争抢）并行启动短期/长期两个 Future，join 汇合；**长期记忆定位为增强能力——查询失败记 warning 返回空 Map，主对话继续**，不阻塞主链路。③ **短期窗口控制**：从尾部向前累加 token，窗口预留摘要预算；**摘要触发阈值化**（未被覆盖消息达 20 条才生成，预算 2000 tokens，Redis 缓存 1 小时）——高频对话写入与低频摘要合并分离，避免每轮调摘要模型。④ **长期记忆预算分配**：总预算 4000 tokens，user_profile 上限 60%，未用满让给 Agent 记忆（截断后重新计算 token，防止带入截断前估算）；**按行截断保留单条记忆完整语义**；检索后 score<0.3 过滤、单条 1000 字符截断。⑤ **异步沉淀链**：会话级 Redis 锁（10 分钟）→ MD5(role:content) hash 幂等去重（TTL 7 天，在 LLM 判断前写入挡重复事件）→ LLM 判断（完整上下文 `[[NEW]]` 标记新增消息，LLM 失败降级规则判断，单条截断 200 字符）→ 本地相似度去重（exact → contains → Jaccard 0.7 → Levenshtein 0.8 四级链）→ 冲突检测服务 → **先写新记忆后删旧记忆**（写入优先的风险控制：MemOS 是外部 HTTP 服务无法事务包住新增+删除；删除失败只记 warning，旧记忆留给下次冲突处理清理）。
- **工程边界（诚实披露）：** 整条链路是 best-effort——Redis 失败整段会话重处理、外部服务失败无自动回放，运行侧需同时关注"重复写入"和"处理未完成"两类异常；仍需补齐生产流量下的延迟/失败率观测与补偿机制。
- **映射：** 标准 §5.1 状态分层（长期记忆失败降级为空）、§2.3 Token 预算（scope 级分配+按行截断）、§3.4.1 先写后删与 reconcile。

### 9.5 个人 AI 记忆系统——跨工具所有权与两级封顶（腾讯云开发者实践）

- **来源：** [左德军（腾讯云开发者）— 一文搞懂个人AI记忆系统构建全流程](https://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247697254&idx=1&sn=4c6c7051662b7a92a77ce80bec68af16) ｜ 2026-08
- **问题：** 给 codebuddy 讲过的个人信息、原则、协作关系，还要给 workbuddy、龙虾、Claude Code 各重复讲一遍；新增更新都要重复说；更难受的是**积累的上下文和判断经验绑死在特定工具上，换工具就丢**。
- **核心主张：** 记忆要属于人、归人管，不绑定任何工具——无论用哪个 AI，都能快速知道我是谁、在做什么、怎么判断、希望怎么协作，且自动更新同步。
- **解决方法：** ① **五层记忆**（每层回答一个问题）：identity（我是谁）/ principles（我如何判断）/ preferences（AI 怎样和我协作）/ context（我正在做什么）/ knowledge（可复用经验）；**稳定层**（identity/principles 极少变）与**动态层**（preferences/context/knowledge 持续更新）分开——归类先分清"长期不变的我"还是"当前阶段的我"，避免把易变 context 误当长期身份。② **结构硬约束：有且只有两级目录、二级下只放文件、禁止三级嵌套**，内容多了用文件名前缀（report-xxx.md）不新建子目录——路径每深一层 AI 定位一条记忆的选择数翻倍，两级封顶=最多两次选择定位；嵌套把"分类"变成"迷宫"；**想再建一层目录往往是在用新增目录逃避"这条记忆属于哪类"的判断**，约束死深度倒逼想清归类。③ **归类灰色地带规则**：价值判断进 principles、操作步骤进 skills；带日期事件进 experiences、无时间绑定方法进 skills；外部输入进 learnings、内化后的框架进 skills——**升级路径：经验反复出现的规律→提炼成技能，学习内化后→形成自己的技能，记忆不是静态归档是会升级的**。④ **元数据 7 字段** YAML frontmatter（参考 Google Cloud OKF 裁剪）：type/title/description（"是什么+何时调用"，AI 扫到即判断要不要读）/status（active/archived 过期不参与调用）/privacy（internal/public）/tags（跨目录主题标签弥补目录单一维度）/timestamp（沉淀日期非创建日期）。⑤ **两层索引**：总索引 INDEX.md（全部 title+description 一行一条，AI 先扫地图再决定读哪篇正文）+ 每目录 README（装什么/不装什么）——写入时由 curator skill 顺手同步。⑥ **双 skill 读写分离（跨平台）**：`personal-memory-recall` 按任务类型查调用判据→扫 INDEX→按 L1→L5 权重加载；`personal-memory-curator` 从对话提取候选→人确认→按 README 路由→输出决策报告（**排除理由必写**）→写入+同步 INDEX→每条独立 git commit（可追溯可回退）；权重与目录绑定不设独立字段，冲突高等级优先但**最终决策权永远在用户**。⑦ **"待补齐清单"**：不确定的组织信息单独显式标注，不假装知道——AI 基于这份协作关系文档主动提醒干系人对齐，**挖出的人员清单比作者自己想到的更全**（没有这份信息 AI 像执行员工，有了像贴身秘书）。
- **关键取舍：** **故意不做全自动记忆维护**——每条写入前都要人确认；不是做不到，是希望记忆库维护"更好版本的我"而不是"现在版本的我"，全自动沉淀会把临时的、不够好的判断也加进去，久了反而把自己困住。诚实列出的缺陷：记忆升级靠人工判断、冲突与过期无自动检测。
- **大半年实践收益：** 换工具摩擦为零（recall 自动加载身份偏好上下文，接得住连续任务）、经验复利（散落各工具对话的内容集中叠加）、AI 从执行工具变协作伙伴（主动判断"这个人还没对齐"）。
- **映射：** 标准 §5.1 记忆规则（目录封顶与索引先行）、§7.10 策展式写入与人工确认、§1.7 地图+渐进披露（两层索引）、§5.11 非对称淘汰（同一方向）。

---

## 10. 协作开发

### 10.1 harness 假设会随模型升级而过时——reset 变 dead weight

- **来源：** Anthropic — Scaling Managed Agents（同 4.1 上下文）
- **问题：** Sonnet 4.5 有 context anxiety 需 harness 加 context reset；到 Opus 4.5 该行为消失，reset 成 dead weight。harness 编码的假设需频繁质疑。
- **原因：** Bitter Lesson——模型改进后旧的补丁变冗余甚至有害。
- **解决方法：** Managed Agents 设计成"对接口有意见、对实现无意见"——session/harness/sandbox 各自可独立 fail/替换。为"尚未想到的程序"设计系统（类比 OS 用 process/file 抽象超越具体硬件）。

### 10.2 框架要为每个模型深度定制——OpenAI 用 patch、Anthropic 用字符串替换

- **来源：** Cursor — 持续改进我们的智能体框架（同 1.4）
- **问题：** 框架抽象不依赖具体模型，但需为每个支持模型深度定制。OpenAI 模型训练用 patch 格式编辑文件，Anthropic 习惯字符串替换——给不熟悉的格式会多耗 reasoning token 且更多错误。
- **原因：** 不同模型行为/提示/工具接口差异大；OpenAI 偏字面精确，Claude 偏直觉容忍模糊。
- **解决方法：** 每模型配其训练时用的工具格式；定制深入到 per-provider 甚至 per-model-version 的自定义提示。新模型 Early Access 时从最接近的现有模型框架入手，离线 eval 找易错/困惑点，团队实 dogfood 反馈，迭代到有信心发布。遇模型怪癖可用框架缓解（如"context anxiety"调 prompt 减轻）。

### 10.3 多 agent 协调用 Markdown 规范优于代码约束

- **来源：** [Cursor × NVIDIA — 多智能体系统将 GPU kernel 提速 38%](https://cursor.com/cn/blog/multi-agent-kernels) ｜ 2026-04-14
- **问题：** 多 agent 优化 235 个 CUDA kernel，需协调协议。
- **原因：** 声明式 vs 过程式协调。
- **解决方法：** 整个协调协议写在一个 Markdown 文件中，规定输出格式、规则和测试。多 agent 系统运行中自主学会调基准测试流水线，形成无需开发者干预的持续测试-调试-优化循环。3 周对 Blackwell 200 GPU 的 235 个 kernel 几何平均提速 38%，19% 超过 2 倍。SOL-ExecBench 评估工具会把超出 B200 理论上限的"作弊"（如缓存）判无效。为准确评估，要求系统在两次独立运行中分别用 CUDA C（带内联 PTX，测底层硬件推理）和 CuTe DSL（公开训练数据几乎不存在，测凭文档学新 API）写方案。

### 10.4 SPEC.md 即监督者——Linear 看板变成 agent 控制平面

- **来源：** [OpenAI — Symphony: composition as orchestration](https://openai.com/index/symphony/) ｜ 2026-04-27
- **问题：** 多 agent 并行开发时，传统编排方案（中央调度器、消息总线）需要额外基础设施且易成单点。
- **原因：** 复杂的编排代码本身引入新的失败面。
- **解决方法：** OpenAI 内部产品 Symphony 用 Linear 看板作控制平面：每个 open task 自动配一个 agent，崩溃自动重启；整个"编排系统"就是一份 spec 文档（含 Linear MCP 工具使用说明）让 agent 自己实现。配合 `request_changes` 门禁（分派新任务前必须先处理已有 review 意见）防 agent 堆积任务。结果：部分团队 landed PR 增 500%。
- **映射：** 标准 §3.10 编排协议、§1.2 复杂度阶梯（"编排"可以只是声明式规范而非代码）。

### 10.5 个体提效 ≠ 组织提效——组织摩擦治理四步法（腾讯健康 PM 实战）

- **来源：** [王畅（腾讯云开发者）— AI Coding时代，研发项目管理新范式探索与实践](https://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247697007&idx=1&sn=f12d0ba52354af093099ab3c6c7c28f3) ｜ 2026-08-19
- **问题：** AI Coding 后研发个体产出明显提升，但端到端交付周期收益没有兑现：交付周期中位数在改善，**P85 顽固停在 3 个双周迭代以上**——只看中位数会误判组织健康，长尾才是组织摩擦的藏身处。
- **原因：** 交付周期 = 价值创造时间 + 组织摩擦时间，**组织效率 = 价值创造时间 /（价值创造时间 + 组织摩擦时间）**。组织摩擦（等待排期、资源协调、跨团队协同、联调等待、Bug 返工）不集中在某个环节，而是随需求流转在各阶段间悄悄累积。四个真实交付案例的共同点：真正编码的时间很短，绝大部分周期耗在协作上——AI 提升了编码效率，但瓶颈转移到了"等待"。
- **解决方法：** PM 视角从"过程管理"（任务有没有人做、节点有没有延期）转向"效率治理"（持续发现并消除过程摩擦）。分工原则：**AI 负责态势感知与问题曝光（发现问题、定位任务、呈现数据），PM 负责判断原因、决定调配、推动解决**。四步递进路径：① **数据可见**——对需求交付流程标准化建模，统一阶段/责任角色/交付条件，拆到原子任务"一个任务对应一个责任主体"，组织运行才可观测；② **数据可信**——状态流转从"人工汇报"转为"系统事件推断"（任务状态变化、评审完成、测试流转、发布节点自动驱动），否则 AI 基于错误数据分析；③ **异常定位**——AI 持续巡检（阶段等待超时、需求长期未流转、资源负荷异常、单点依赖风险），PM 从"推进任务"转向"治理队列与等待"；④ **瓶颈挖掘**——双周/月度维度分析 Lead Time、等待时长、质量趋势、风险分布，从解决单个问题转向优化整个交付系统。配套沉淀 PM-Skills：项目治理（工作流/自动化规则治理，规则无分层命名治理会毁掉数据可信度）、健康巡检（晨间快照 + 晚间复盘形成"早定行动、晚看结果"闭环）、智能排期（结构化呈现辅助决策不替代 PM）。成果：中位交付周期 19→9 天、P85 52→23 天、可预测性（P85 与中位数差距）33→15 天、30 天以上长尾 35%→7%、60 天以上 11%→0%；问题发现从事后 1-2 周提前到风险出现后 1-3 天。
- **映射：** 标准 §8.1 生产上线指标（中位数 vs P85 分化观察）、§7.5 自动巡检闭环（日巡检 + 周期性归因）、§1.7 Symphony 案例的瓶颈转移（人类 QA 产能 → 组织协作等待）。

### 10.6 从指挥任务到委派结果——Anthropic Labs 的工程与组织方法（Mike Krieger 访谈）

- **来源：** [AI Engineer — How Anthropic Builds: Lessons from Labs（Mike Krieger，Instagram 联创、Anthropic 首席产品官）](https://www.bestblogs.dev/video/8d3ae5678) ｜ 2026-08
- **核心理念：** 能力更强的模型让产品与工程工作从**逐项指挥（decompose in head → 跟进每个步骤）转向委派结果（describe end state → 让系统朝它工作）**；人与模型的关系变成讨论 trade-offs、回答问题、评估落点。委派的新习惯：模型返回成果时附带 trade-offs 说明，人有时要让它"解释得更简单"——委派的对象是结果不是过程。
- **工作方式经验：** ① **敢提"不合理"的需求**：第一代 AI 产品把模型限制得太死（能写代码不能跑、只能部分看环境），ambitious 请求难以兑现；能力上来后要教用户 unreasonable——看起来不需要 VM 的知识工作者，在内置方法失败（PDF 解析不了）时能自己写脚本兜底就体现价值。实例：让 Claude Code 把整个几十万行 Python 项目 port 到 TypeScript，设置动态工作流让系统周末移植+验证+反复对比，周一拿到可部署版本；早年这是想都不敢想的事。② **运营控制（Instagram 经验直接迁移）**：故障前把所有可能相关的指标都埋好（否则坏了都不知道 metric 是正常还是异常）；调节旋钮和 feature flag 做成一等公民——AI 时代团队随时做不同 trade-off，运行时配置要能秒级改变；移植/迁移找到可增量开始的边界而非一夜全换，生产数据+分段测试兜底（Instagram 用 MonkeyType 抓生产运行时类型映射回代码库）。③ **Agent 是队友**：Claude Code 适合紧密迭代的高带宽往返，大量工作通过 Tags 类系统委派；多人可见性（像 Discord 上看别人玩 Midjourney）鼓励更激进的使用；高级形态是对代码库的一部分负责、监控反馈渠道、主动处理任务、API 变更时响应——有上下文有记忆能主动行动的 teammate，让内部工作更 multiplayer、异步、主动。
- **评审与验收经验：** 评审瓶颈的真因不是 Git **而是人脑概念化能力**——与其给评审者 2000 行 PR 自己推断，不如给出 artifact（改动意图 + trade-offs 说明）；讨论焦点转向意图与取舍，生产环境度量做验证；不逐行看每个 PR，**让 Claude 去调查"人会对这段代码提出的问题"——Claude 驱动的评审、人主导的判断**；重要变更与外观变更区别对待，可以知情决策后 fix forward。
- **组织管理经验：** ① **双周 persevere or pivot**：每个项目双周评审——继续 / 转向 / 关停；关停是快速原型化的预期结果而非个人失败，多轮之后形成心理安全；快速原型 → 内部交付 → early access → 不行就收。② **组织架构不与项目绑死**（否则每两周重组一次）：bet 团队跨学科抽人，有 bet lead（直接责任人）但不管理所有人；产品证明有生命力后才固化为专属团队（Claude Design 从 ad hoc 组 → 专属团队）；工程管理的价值没有消失——辅导、人际、个人发展仍是核心。③ 一批前 CTO 主动选择回 IC 岗——模型时代一线动手比管理更有趣，人才信号值得管理者注意。④ **产品做减法**：Project Unhip 频道专门讨论删什么；删掉 Styles（占比小、太 prescriptive，Skills 是更好形态）——要敢"unship"上一代 AI 的 primitives；更大的问题是跨表面复杂度：用户分不清 Claude Code / Cowork / chat，表面不互通要人肉复制粘贴的流程不该存在。⑤ **创业与垂直 AI 判据**：强模型解决不了创始人的 ideation 和 taste，但让实验更快；深入垂直（对行业/用户理解深）仍有空间——写代码从来不是创业成败的唯一因素，用户理解和 fit 更重要；垂直 AI 配方 = 足够灵活做 just-in-time 分析的模型 + 经过验证的数据集，外加可验证性/审计日志/数据溯源但不限制上层应用。⑥ **心理可持续**（AI 时代独有的管理课题）：每周 all-hands 到周三就可能出现"周末 AI"页（竞对发模型/新品/监管变化），节奏远快于以往；体育视角——人永远不会像最佳表现那么好，也不会像最差表现那么糟，情绪周期会反复出现，记住这点就有 perspective；情绪要说出来——一个人在感受的，团队其他人往往也在感受，直接说"我为这个项目难过、沮丧"会给团队留出空间，之后才容易讨论下一步。
- **映射：** 标准 §5.1 Goal 契约（委派结果）、§7 评审验收（意图 artifact + Claude 代问）、§3.4.1 删减过时脚手架（unship primitives）；§10.5 组织摩擦治理（同为 AI 时代项目管理参考）。

### 10.7 AI-Native SDLC——从线性流程到自动交接的循环（Anthropic Playbook）

- **来源：** [Claude by Anthropic — The AI-Native SDLC playbook](https://claude.com/blog/the-ai-native-sdlc-playbook) ｜ 2026-08
- **基本判断：** 代码生成不再是瓶颈，传统 SDLC 中的人速步骤（计划、评审、测试、部署）成为新瓶颈——Anthropic 内部数据：Claude 参与约 80% 的 Pull Requests、贡献约 90% 的代码行。解法不是把 AI 塞进线性流程的某个环节，而是把 SDLC 重构成**循环**：AI 嵌入每个阶段，阶段间交接由 AI 自动完成并事件驱动触发，消除手工作业和交接等待。
- **六阶段关键实践：**
  - **Plan——intent.md 工件：** 想法发起者与 Claude 协作生成（问题描述、期望结果、影响用户/系统、约束、开放问题），版本化、人机均可读，作为整个循环的起点。
  - **Design——spec.md 生成：** Claude 依据 intent.md + 组织级技能（品牌、安全、合规、UX）生成需求与设计规范，产品 owner 评审并解决其中标记的关注点——组织知识以 Skills 形式被复用而非散落在人脑里。
  - **Build——plan.md 与双模式执行：** 工程师在 Claude Code 的 plan mode 中生成实现计划（变更文件、工作顺序、风险、验证方法），人审核后切换执行模式由 Claude 实现；CLAUDE.md 记录项目约定、命令、架构与常见错误。复杂工作拆为并行会话与子 agent 各自推进。
  - **Test——反馈循环与持续 evals：** Claude 在会话内自跑测试、构建、截图对比，迭代修复直至通过（人从"跑测试的人"变成"验证策略的设计者"）；持续评估套件在 agent 配置（CLAUDE.md、skills、hooks）变更时运行，防止配置漂移引入退化。
  - **Deploy——AI 评审 + hooks 审批 gate：** Claude 对 PR 做多维度评审（bug、安全、合规），人类聚焦意图与风险；hooks 作为自动化审批 gate——如生产部署动作需特定人员授权才可执行，把"流程规范"从文档变成机器强制。
  - **Maintain——监控触发闭环：** 监控指标异常自动触发 Claude 诊断，产出新的 intent.md 重新进入 SDLC 循环（运维成为循环入口而非终点）；定期安全扫描自动发现并修复漏洞。
- **治理与度量：** 每阶段产物（intent.md、spec.md、plan.md、代码 diff、评审结果）全部进版本控制，形成完整审计 trail；度量 SDLC 流程本身——intent.md 提交耗时、spec.md 到 plan.md 转化率、PR 首次通过 CI 比例、自治修复闭环时间。
- **可借鉴核心：** ① 工件链是"阶段交接契约"不是一次性 prompt——版本化、人机共读、可审计；② 组织级标准（品牌/安全/合规）沉淀为可复用资产而非口头规范；③ hooks 把审批从"人的自觉"变成"机器的 gate"；④ 运维异常自动转成新一轮开发输入，闭环不需要人肉搬运。
- **映射：** 标准 §12.2 变更规范三层细化（proposal/design/tasks ≈ intent/spec/plan）、§7.10 ADLC 飞轮（监控触发重入循环）、§6 审批（hooks gate）。

### 10.8 项目 Harness 三件事——读对、拦住、接得上（腾讯 Wish/ShoppingUI 实践）

- **来源：** [蓝翔（腾讯）— 删掉80%的Prompt规则，Agent交付成功率反而更高了](https://mp.weixin.qq.com/s/fV8qN6qs9ac-VXDwZCuaxA) ｜ 2026-09-01（Wish 端间鉴权改造 + ShoppingUI Nightly 两个真实项目）
- **主命题：** 代码是中间产物不是交付结果——Agent 写代码越快，交付链越容易断。项目 Harness 要解决三件事：**可信上下文**（开始时不容易读错）、**可执行约束**（执行时不容易跑偏）、**可恢复流程**（中断后还能继续、说完成时拿得出证据）。金句："Agent 可以失忆，但项目不能失忆。"
- **可信上下文——供应链而非知识库：** ① 知识库是地图不是百科全书，设计重点不在容量在入口——AGENTS.md 作顶层 Router，Wish 收敛为三跳（AGENTS.md → context/context.md → 专题文档/源码），**好的 Context System 首先是 Navigation System，其次才是 Storage System**；每跳只收窄问题空间，不重复存正文。② **找到 ≠ 可信**：同一问题会同时搜到历史文档/当前代码/生成契约/运行记录，都真但不描述同一时间现场——ShoppingUI 实例：本地 f12e007 已含修复，Nightly receipt 显示实际消费的仍是 9c044649，"某处已存在"≠"当前已生效"；重要结论须带现场（claim/source/revision/scope/observed_at/verification），判断以实际消费的 revision 为准。③ **能力观察规则**（何时值得沉淀）：AGENTS.md 加一段轻量规则——任务后只在出现明确信号（重复劳动、反复纠正、反复找同一上下文、现有 Rule/Skill/文档缺口）时输出一项候选（附事实、为何重复、建议载体 Runbook/Skill/Script/Gate、最小内容与验证方式），无信号不输出不扩建，只建议不自动改；沉淀次序：一次问题先修复 → 重复路径再沉淀 → 能机械判断的最后才变约束。
- **可执行约束——判据是违反后会发生什么：** 条件不满足仍能继续=提醒；条件不满足下一步无法进行=约束。阶段门禁查"上一步证据"（测试报告是否本次生成、是否在本次修改的 worktree 运行、是否调用项目规定测试入口、必测场景是否真跑到——报告绿色但目录不一致也不通过）；动作前检查拦"这个动作能不能执行"（推保护分支/强推/跳过检查/破坏代码现场的命令在执行前拦截，"Agent 说我会小心"不能代替检查）。分工：文档讲为什么，脚本查有没有做到，门禁保证证据不够不能继续，难机械判断的设计业务问题留给人。
- **可恢复流程——换 Agent 检验：** 文档维护+工作流推进+状态存档三件套；Workflow 规定的是**交接合同**（输入来自已确认事实或上一步产物、不从聊天记忆猜；输出落到文档/代码现场/报告、不只留一句总结；输出不满足下一阶段需要时停在当前节点）。可恢复性判据：**假设每个阶段结束后都换成全新 Agent，只靠项目状态、文档和真实工件还能继续吗**——不能则流程仍依赖上一段对话。state.json 是书签不复制正文（当前阶段/状态/worktree/路径），delivery-log.md 是人和 Agent 都能读懂的任务正文；**恢复任务≠恢复会话**（前者凭持久化信息重建上下文，后者试图找回上一段对话）。
- **Harness 生长与修剪（Add/Thin）：** 先跑最小 MVP 建基线（真实简单可验收的任务，含目标/范围/约束/完成条件），失败了不先加多 Agent、长 Prompt 或复杂工作流，先观测执行链定位问题最早出现在哪一层（读取输入→沿执行链找第一个偏离点→区分事实/推断/未验证→用原失败路径+一条正常路径验证修复）；**Harness 组件隐含对当前模型能力的判断，不是永久资产**——只会增加不会减少的 Harness 最后变成新负担，Add/Thin 都由运行证据决定；不复制别的项目长成的 Harness（可带走观测问题、状态语义、证据原则和 Add/Thin 方法，节点/Gate/脚本/证据格式要重新生成）。
- **映射：** 标准 §2.2（结论绑定运行现场）、§7.3 Gate（约束 vs 提醒判据）、§5.1（换 Agent 恢复性检验、Add/Thin）；§7.10（能力观察与自进化纪律互补：何时值得沉淀 vs 怎么改）。

### 10.9 确定性 Harness——把"不可控"关进"可控"的笼子（腾讯工单审核实践）

- **来源：** [左昊（腾讯）— 一文讲透确定性 Harness](https://mp.weixin.qq.com/s/rQYSuTF98-xGcdgGPuNHjQ) ｜ 2026-09-02（工单审核业务：9 阶段、几十个条件分支、6 步顺序校验、十几个原子接口）
- **核心矛盾：** LLM 本质是概率性的（同一工单重跑结论可能漂移），而工单审核结论直接决定用户/商家权益与合规认定，必须是确定性的——**Harness 做的所有事都是在填"概率性判断"与"确定性结论"之间的先天鸿沟**。"模型是商品，harness 才是护城河"：LLM 是可随时插拔的判断单元，让高风险决策能上线、能查、能复盘的是外面那套执行纪律。
- **四个关键设计（环环相扣的闭环）：** ① **阶段化编排**——策略文档几十个分支拆成签名统一的阶段函数（只做一件事、只写自己负责的结果字段），主流程用 goto 标签组织分支，主干保持线性可读；新增判断点=新增一个函数，主流程骨架不动；② **全链路留痕（FlowTracer）**——每阶段记输入/接口原始返回/命中分支/输出/错误/耗时，整个数组随结果返回可逐阶段回放；调试开关默认关闭（nil 安全降级零开销），环境变量一键全量开启；③ **提前终止（IsFinal 短路）**——声明式 handler 列表顺序执行，能下结论立即 break（3 秒拿驳回结论 vs 跑完 6 步）；④ **统一网关与缓存**——十几个原子接口同一签名封装，结果按 TaskID 落缓存天然幂等。效果：单工单 2-3 分钟 → 20-40 秒；审核员从肉眼比对变成看结论+调 DebugTrace 回放。
- **三个反直觉取舍：** ① **单阶段失败不阻断**——工程第一反应是"出错就该停"，但审单场景一个接口超时不该让整单卡死，给出"带着伤口的结论"（缺口显式标注）好过没有结论；② **留痕默认关着**——可解释是刚需，但"需要时能查"和"平时不拖累"必须同时成立，否则留痕自己变成新的性能包袱；③ **用 goto 而非"更优雅"的结构**——大众认知里是坏味道，但分支密集的编排里它反而是让主干保持线性的最直白手段。
- **适用边界（核心判断一句话）：** "这个 Agent 的结论，需不需要对某个人、某条规则、某次检查负责？需要，就值得为确定性付工程成本；不需要，就不必。"纯生成式任务（文案/翻译/摘要）prompt 工程就够、harness 是负担；必须 100% 黑盒的探索性任务与可解释可追溯冲突；单次轻量调用没有 harness 发挥空间。LLM 介入分级：L1 模型直接出结论 / L2 模型建议人工拍板 / L3 模型只给 SOP 禁止自动执行。
- **映射：** 标准 §1.1 复杂度阶梯（"结论要不要负责"的 Harness 准入判据）、§7.4（交付链 vs 判定链失败语义、留痕零开销）；与 §7.3 Gate（防假绿门禁）形成张力互补——判定链容忍单点失败要整单结论，交付链证据门禁严格防假绿。

### 10.10 项目 Harness 让 AI 进入完整研发闭环（淘天 Price360-KB 实践）

- **来源：** [默达（淘天集团-营销&交易技术）— AI 驱动研发体系的实践和思考](https://mp.weixin.qq.com/s?__biz=MzAxNDEwNjk5OQ==&mid=2650545626&idx=1&sn=cfd0d3011972881686bbb9bedbc319da) ｜ 2026-09-02（价格力业务，4 个月实践）
- **公式递进：** `Agent = Model + Harness` → **`业务研发 Agent = Model + 通用 Coding Agent Harness + 项目 Harness`**——通用 Harness 提供循环/文件操作/工具/环境连接，缺业务知识、流程约束和完成标准，项目 Harness 补这三样。**核心判断：业务 AI 研发的真正瓶颈是上下文而非模型能力**——模型决定通用能力，上下文决定它在具体项目中走多远；靠人在对话里重讲，上限就是对话长度和个人表达能力。
- **本地优先与动静分离：** 尽可能把项目需要的所有信息放进同一个文件夹用 Git 管理（版本/分支/评审/来源/回滚全齐），不建平台不做界面；稳定上下文（业务知识/规则/源码/迭代文档）进 Git，**动态事实（工作项状态、测试环境、日志、数据库、配置中心）仍从权威系统实时读取**——配置写着某环境≠代码已部署、接口返回成功≠异步消费和落库副作用已发生。
- **知识载体选型（更新路径判据）：** 文件 Wiki/RAG/本体不互斥——文件 Wiki 管知识维护评审版本化，RAG 管找到相关内容，本体管表之间关系。优先文件 Wiki 因为**知识更新路径最短**：代码、文档、测试、索引进同一个 MR 一起评审、合并后共享同一版本；RAG/图谱作主知识源在代码变化后要走同步→切片→萃取→索引→发布长链路；本体有应用优势但缺知识生产维护。
- **wiki/tech 两层与知识边界：** wiki 面向产品/运营/测试/Agent 存业务概念、规则、口径、角色、流程、异常处理；tech 存**代码无法独立表达但影响 Agent 判断的技术上下文**（跨系统上下游链路、接口数据关系、新旧链路切换、废弃状态、运行态拓扑）。**知识库不复制代码**——逐方法解释的文档生成快失效也快，变成与代码竞争的实现说明。每份知识文件=正文+Metadata+链接+事实来源+Git 版本；Metadata（title/category/tags/status/version/source）让 Agent 读全文前判适用性，source 指向原始材料；tech 通过 wiki_ref 关联业务知识，形成业务规则→技术链路→源代码的可双向检索路径。
- **知识飞轮：** 迭代每个阶段为下一阶段生产上下文（PRD 补业务定义验收标准→方案记链路→开发测试校验→归档时确认），知识反哺下轮迭代（Agent 写 PRD 不再从零推测概念口径）；**项目开始不需要完美知识库，知识建设是真实产品迭代的副产物**；知识生成统一经项目 Skill 补齐标签来源索引，定期健康检查处理重复/冲突/失效链接/无来源结论。
- **老系统冷启动三板斧：** Git submodule 把相关代码仓关联到 src/；人工旧文档收纳到 raw/（未经改写的原始事实材料，Agent 不能把会议记录当已确认规则）；设计采访稿经钉钉 AI 听记采访核心开发/产品，原稿交给 Agent 整理入库。
- **协议/能力/门禁三层分离：** AGENTS.md 是项目协作协议（事实源/研发阶段/人工决策点/授权边界/完成条件），.agents/skills 是可组合领域能力（PRD/方案/测试设计/排障/归档），.agents/scripts 是确定性门禁（工作区校验/格式校验/同步/发布回查）；公共 Skill 通过依赖清单声明来源与兼容版本，项目只留业务编排。迭代目录（prd/solution/test/archive）**是阶段接口而非事后总结**，阶段不自动越过：PRD 未确认不写技术方案。
- **Harness 收缩与角色演进：** 今天项目自己维护的 Prompt/规则/工具，一部分会被模型或 Coding Agent 产品吸收；**不会消失的四样：业务知识及事实治理、项目规则与决策边界、领域工具和系统适配、项目自己的验证标准与质量责任**。业务技术人员角色→FDE（Forward Deployed Engineer）：深入业务现场，把问题、知识、系统和交付结果连接起来。共性能力需要共识的是**工作项/代码/测试/发布之间的数据协议与状态、证据、安全、度量口径，不是统一的流程实现**；AI 代码采纳率是不得已的妥协指标。
- **映射：** 标准 §2.12（知识载体选型判据、知识边界与 Metadata）、§12.2（知识飞轮）；§2.2（动静分离与事实边界）；与 lessons 10.8（Add/Thin）互补——那是修剪现有组件，这是预判哪些层会被吸收。

### 10.11 商务 Agent 解剖——单模型架构与生产化实践（Anthropic 开源参考实现）

- **来源：** [Anthropic（Ali Shazal, Matthew Koen）— A guide to the anatomy of effective commerce agents](https://claude.com/blog/the-anatomy-of-effective-commerce-agents) ｜ 2026-09-02（一年期多企业商务 Agent 合作复盘 + [anthropics/commerce-agents](https://github.com/anthropics/commerce-agents) 参考实现）
- **核心架构：** 一个模型 + 标准 agent loop（推理目标、探索上下文、工具行动、skills 学流程、澄清提问、观察结果直至完成），**无前置 intent router、无领域子 agent**。多企业部署对比：单 agent + skills 在质量上稳定优于"一个 prompt 打天下"和"子 agent 拆分"，且成本延迟往往更低——商务对话是跨多意图多轮的紧耦合会话、需要大量共享上下文，每次 handoff 都是有损状态操作，还多花数倍 token、加数秒延迟；领域也难干净切分（退货流程要订单历史+当前购物车+商品目录）。子 agent 的合理位置：作为工具被调用处理窄的自包含任务（deep-research——搜索读文档、碰死胡同都在子 agent 内，只回紧凑答案）；已有专职 agent 的领域（药房/金融合规面）用 **hand-off**（对方接管对话直至完成）而非 delegation（主 agent 在单轮内反复弹入弹出、每次交换都退化）——区别在对话所有权。
- **关键决策：**
  - **prompt vs skill 按频率定**：加载 skill 花一轮模型调用，多数轮次需要的内容进 system prompt；经验法则是 **≥1/3 流量需要的内容进 prompt，其余进 skills**（按上线前预期或生产实测）；可由已有信号（用户从哪个页面打开助手）预测的 skill 由 harness 预载、跳过加载轮；安全/法务/品牌约束和关键用户事实（如过敏）永驻 prompt。参考实现：购物 agent 的 prompt 放 grounding、购物车结账语义、展示规则和商品搜索，skills 覆盖长尾（search-discovery、purchase-research、planning-goals、customer-care、memory-personalization）。
  - **工具建在核心系统之上**：agent 工具调用已有的搜索排序/购物车/库存/促销引擎而非重实现（这些系统沉淀了多年调优逻辑和模型永远看不到的信号）；工具边界是系统逻辑结束、模型判断开始的地方——`search_products` 返回已排序结果，agent 决定展示哪些、展示多少；**工具结果是上下文**，只返回模型推理要用的字段（每行图片 URL 是惯犯），错误场景给指令而非错误码（"查可用性时请带商品 ID"而非裸 403）。
  - **UI 组件即工具**：商务 agent 回答多是 UI 组件（商品轮播、行程、座位图）而非散文。自定义标记方案随规模失效（模型对自有标记的可靠性不如工具调用、标记定义在 prompt 里越加越膨胀、历史会话存成只有自家 parser 能读的格式）。成熟模式是每个 UI 组件一个工具：模型调 `present_products` 带类型化参数，服务端校验增强、客户端渲染；本来就是 messages array 里的原生 tool call，重载历史会话无需重新解析；展示工具还给 agent 屏幕记录——"第一家酒店""左边第三个"能被解析，前提是参数按 UI 结构组织（有序行/轮播）而非客户端重排的平铺列表。代价是流式粒度：顶层参数要在服务端缓冲校验，子组件分步到达影响感知延迟（eager_input_streaming 可换 token 级流但失去 schema 保证；Sonnet 级以上 schema 违规很少，仍包一层 retry）。
  - **stage/apply 分离**：模型只能生成带服务端 ID 的暂存变更（购物车、订单等），`apply_change` 仅对经真实界面批准的 ID 生效，且 apply 时按当前限额复查——防止模型在多轮间"环境已变"时沿用过期前提直接落单。
  - **记忆异步抽取**：会话记忆（偏好、事实）由会话后的抽取器异步写入而非在对话热路径上同步生成；记忆写路径带验证，防止对话里的临时表述直接污染长期记忆。
  - **缓存分层**：global（跨用户稳定的 system prompt/工具定义）/ session（用户会话内稳定）/ volatile（当前轮）三层缓存前缀——稳定内容最大化缓存命中。
- **延迟与成本观**：任务完成延迟 = Σ(每轮最后 token 时间 + 工具处理)，三个杠杆（更少轮次/更快工具/更快 token）要最小化总和而非单项；**质量比边际延迟增益更影响留存/参与/购物车尺寸**，用户有延迟预算，看 agent 工作的时间读作进展；更聪明的模型常因更少轮次整体更快（生产 >5 轮/任务时通常更聪明的模型更快）；并行工具调用 + 从来源页预载数据（从商品页打开助手就把页面数据放进会话上下文，答上下文内问题零额外轮次）。
- **映射：** 标准 §7.9（同标准 fallback 与模型选型）、§14.2 渐进式加载（频率判据与预载）、§2.9 分层上下文加载（预载来源页）、§6 审批（stage/apply）、§5.5 Working Memory（记忆异步抽取）；与 §3.9（通信通道≠协作语义——此处反向印证：紧耦合会话别拆子 agent）互证。

### 10.12 从监督工具调用到指挥目标——Claude Code 团队自己的开发方式（含扇出对抗审查与渐进信任）

- **来源：** [Claude Code 团队 — 团队如何用 Claude Code 重塑软件开发流程](https://www.bestblogs.dev/video/9899b4cdb) ｜ 2026-09-03（团队访谈，一年期回顾）
- **指挥目标而非监督过程：** 团队 70-80% 的工作发生在 Claude Tag（Slack-native agent）——不再关注每一份 transcript、每一次工具调用、每个模型决策，而是表达目标让模型自行追求；TUI/桌面应用只在需要 micromanage 时打开。给 agent 的问题远比"实现一个类/写一个函数"复杂，因为它能查产品上下文和团队决策，做更好的选择。
- **Harness 功能是对模型失效模式的阶段性补偿：** to-do list 是为 Sonnet 3.5 时代"给 5 个任务只完成 3 个就停"而生的正确支架；记忆与长程能力增强后一年就不必要了——**"对已建成之物保持不执着"**。技术每两个月根本性变化，产品保质期压缩，团队工作在"软件开发的双曲时间舱"里。ask-user-question 工具同理：从艰难驯服到很快被"HTML artifact 自己提问带图表 mockup"超越。与 §10.1（reset 变 dead weight）同源互证。
- **扇出采集 + 对抗性审查 + 确定性循环：** code review 是扇出模式首个大用例——Claude 广泛撒网找候选 bug，再用多视角对抗性审查逐个评估、过滤误报，人只审真正值得注意的问题集（test-time compute，类比 MapReduce：扇出产生人消费不了的信息量，必须过滤回来）。**确定性 for-loop 保证同一技术应用到每一项，这是对 workflow 的信任来源**；Claude 甚至能自己决定扇出拓扑、连接上下游、汇总结果——自建 harness。同一模式适用于性能排查、行程规划研究。
- **Claude Tag 端到端产品循环（产品管理全链路）：** 问该找谁谈→拉相关利益方→Slack 里做 mockup（手机可看）→协助实现→加埋点→内部部署→监控使用与反馈→有人反馈时 tag 开发者。看到漏斗掉人时，从"告诉 Claude 具体方案"升维为"改进漏斗、在更高层提想法"。
- **渐进信任的验证闭环：** PR 附测试+结果截图（人工审查前的信心来源）；人做 sanity check 的方式是让 Claude 录屏自用 TUI、再亲自 clone 试一次——**信任随验证闭环质量扩大而非一次性给足，预期未来可能不用亲自 clone**。评审升维：Claude 自主处理 nitpick（reviewer 不再用小评论证明读过代码），人看 API 为何这样设计、服务边界为何划在这。
- **UI 与 transcript 解耦的心理转变：** 看不到模型每步思考起初不安，久了是解放——agent 自己选择何时沟通什么，人不再监控每个工具调用和参数，亲证"当前模型无需 transcript 监督也能返回好结果"。工程师的满足感转移：性能工程交给 Claude（人仍受益于改进），注意力转向更快出想法、从 idea 到 prototype 到 production 更快；"难的不是学新技术，是腾出心智空间思考更大的问题"。
- **适用边界：** "10 倍生产力"为当事人孤例声明（按 §0.5 厂商案例仅作参考），量化采信仅 70-80% 工作占比的行为描述；扇出+对抗审查的模式依赖较强模型与可控误报成本。
- **映射：** 标准 §3.14（扇出采集+对抗审查+确定性循环保一致）、§9.5 Harness 熵（to-do list 生死）；playbook §5（扇出对抗审查、渐进信任验证）；与 §10.6（委派结果）、§10.1（harness 假设过时）同源互证。

### 10.13 企业正演变为一系列级联循环——Agent 攀登、人类选山（a16z 组织观察）

- **来源：** [Anish Acharya（a16z）— Companies as Cascading Loops（Lenny's Podcast 访谈）](https://www.bestblogs.dev/article/9f27a9b6d2) ｜ 2026-09-06
- **核心论点：** Agent 的最小单元是"模型+工具+记忆+技能文件的循环"，下一层是"由 Agent 组成的循环执行衔接任务"。企业正沿级联路径重组：每人一个循环 → 每个职能一个循环（工程/增长/销售/客服/法务）→ 跨业务单元的循环 → 跑公司大部分。类比电力：从引入到工厂围绕它重新设计用了约 40 年——有雄心的公司围绕模型重组组织，而不是给现有团队配 AI 工具。Google 某团队案例：未裁员，两年路线图三个月完成，最难的问题变成"决定往路线图里加什么"。
- **两个具体循环形态：**
  - **工程循环：** bug 报告 → 自动复现 → 修复 → 评审 → 风险评估决定是否需人工审批（低风险自动上线、高风险留人）→ 邮件告知客户，全程分钟级。
  - **增长循环：** 每个变体都生成并度量，达到统计显著性自动合并上线，保留长期 holdout 对照，然后自动进入下一个实验。
- **局部最优点与人的角色：** 循环帮你爬山，然后到达平台期——分布外思考和人类直觉负责把组织挪到下一座山脚。**"Agent 负责攀登，人类决定哪座山值得攀登。"** "让我赚一百万且不犯错"不成立，因为 Agent 需要有价值的方向。人仍是销售、支持、战略、例外的中心；数学推理的进步不等于能原创商业战略。
- **人工介入即知识缺口：** 模型犯错时的诊断问题是**"你知道什么它不知道"**——每次失败暴露一个知识或数据缺口，补上这个上下文，理想情况下下次不再求助。Kavak（墨西哥二手车）实践：每个客户一个 Agent，卡住时人辅导它完成，介入同时产生 Agent 可学习的 trace。
- **模型选型的收益上限经济学：** 前沿模型可能贵 100 倍换一个概念点的智能——在收益无上限的领域（药物发现：多一个点智能可能找到下一个他汀，价值万亿）是理性的；结果有界的任务（记账：账可以对，但不会"对 100 倍"）用足够能力的便宜/开放权重专用模型。判据是**收益上限**，不是可验证性。模型不可互换（不同模型形状差异真实存在），模型直觉来自每周交付、用持续项目作测试每个新模型的底盘。
- **适用边界：** VC 视角的消费者 AI（陪伴/娱乐产品）、护城河与分发、募资等内容未收录（非工程范围）；"两年路线图三个月完成"为当事人口述孤例（按 §0.5 仅作参考）；级联循环组织观为前瞻判断，非已验证的普遍事实。
- **映射：** 标准 §7.9（收益上限选型轴）、§6（审批分级：低风险自动、高风险留人）、§5.2（流式编排）；playbook §4（人工介入即缺口信号）、§9（每周交付+持续项目底盘）；与 §10.6（委派结果）、§10.12（指挥目标）同源互证。

---

## 11. 未找到 2026 年可靠来源的方向

按"宁缺毋滥"原则，以下方向未找到满足筛选标准的 2026 年来源，明确标注：

- **Shopify / Netflix / LinkedIn / Vercel 2026 年 Agent 生产文章**：多次站点限定搜索均无 2026 年结果（Google 已于 2026-09 收录，见 #60、#61）。
- **会议分享（ICML/NeurIPS/QCon/GopherCon 2026 有公开 slides/录像的 Agent 经验）**：未找到 2026 年内已公开且符合"真实问题+解决"的会议分享。
- **GitHub Blog《agents.md lessons from 2500 repos》原文**：被广泛引用但官方 URL 未能抓取验证（404），未采纳。

---

## 12. 关键来源索引（全部 2026 年，已验证）

| # | 团队 | 标题 | 日期 | URL |
|---|------|------|------|-----|
| 1 | Anthropic | An update on recent Claude Code quality reports (April 23 Postmortem) | 2026-04-23 | https://www.anthropic.com/engineering/april-23-postmortem |
| 2 | Anthropic | Scaling Managed Agents: Decoupling the brain from the hands | 2026-04-08 | https://www.anthropic.com/engineering/managed-agents |
| 3 | Anthropic | Harness design for long-running application development | 2026-03-24 | https://www.anthropic.com/engineering/harness-design-long-running-apps |
| 4 | Anthropic | How we contain Claude across products | 2026-05-25 | https://www.anthropic.com/engineering/how-we-contain-claude |
| 5 | Anthropic | How we built Claude Code auto mode | 2026-03-25 | https://www.anthropic.com/engineering/claude-code-auto-mode |
| 6 | Anthropic | Eval awareness in Claude Opus 4.6's BrowseComp performance | 2026-03-06 | https://www.anthropic.com/engineering/eval-awareness-browsecomp |
| 7 | Anthropic | Quantifying infrastructure noise in agentic coding evals | 2026-02-05 | https://www.anthropic.com/engineering/infrastructure-noise |
| 8 | Anthropic | Building a C compiler with a team of parallel Claudes | 2026-02-05 | https://www.anthropic.com/engineering/building-c-compiler |
| 9 | Anthropic | Demystifying evals for AI agents | 2026-01-09 | https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents |
| 10 | Cursor | Scaling long-running autonomous coding | 2026-01-14 | https://cursor.com/cn/blog/scaling-agents |
| 11 | Cursor × NVIDIA | 多智能体系统将 GPU kernel 提速 38% | 2026-04-14 | https://cursor.com/cn/blog/multi-agent-kernels |
| 12 | Cursor | 持续改进我们的智能体框架 | 2026-04-30 | https://cursor.com/cn/blog/continually-improving-agent-harness |
| 13 | Cursor | 我们在构建云端智能体时学到的经验 | 2026-05-21 | https://cursor.com/cn/blog/cloud-agent-lessons |
| 14 | LangChain | Human judgment in the agent improvement loop | 2026-04-09 | https://blog.langchain.com/human-judgment-in-the-agent-improvement-loop/ |
| 15 | LangChain | What is an AI agent? | 2026-07-31 | https://www.langchain.com/blog/what-is-an-agent |
| 16 | HN 讨论 | Cursor's "browser experiment" implied success without evidence | 2026-05-22 | https://news.ycombinator.com/item?id=46646777 |
| 17 | Steve Yegge | Fences, not Sandboxes | 2026-08-24 | https://yegge.ai/essays/fences-not-sandboxes/ |
| 18 | 淘天集团（永霸） | 我对 AI Coding 的一点思考：从 Spec 驱动转向环境与验证驱动 | 2026-08-24 | https://mp.weixin.qq.com/s?__biz=MzAxNDEwNjk5OQ==&mid=2650545436&idx=1&sn=af4dc3f820acf230087053aa294dd2be |
| 19 | 吴恩达（DeeplearningAI） | AI 工程技能图谱详解：构建和部署 AI 应用 | 2026-08-24 | https://mp.weixin.qq.com/s?__biz=MzIxNzI0ODE4Nw==&mid=2247498786&idx=1&sn=9b051f71952a42e91973e7e1e92a24ec |
| 20 | Fabien Sanglard | My agent.md to improve LLM-assisted code quality | 2026-08-21 | https://fabiensanglard.net/agent.md/index.html |
| 21 | Google AI | Elevating Antigravity agent skills（系列，子 agent 消息机制等） | 2026-07/08 | https://dev.to/googleai/elevating-antigravity-agent-skills-part-1-interactive-ui-workflows-6l2 |
| 22 | ByteByteGo | Why Code Verification Matters More Than Ever in the Age of AI（Sonar CTO 访谈） | 2026-08-24 | https://blog.bytebytego.com/p/why-code-verification-matters-more |
| 23 | OpenAI | Harness engineering: a case study of Codex-built Symphony | 约 2026-02 | https://openai.com/index/harness-engineering/ |
| 24 | OpenAI | Run long horizon tasks with Codex | 2026-08 | https://developers.openai.com/blog/run-long-horizon-tasks-with-codex |
| 25 | LangChain × Stripe | How Stripe built their Knowledge AI Platform on deep agents | 2026-08-03 | https://www.langchain.com/blog/how-stripe-built-their-knowledge-ai-platform-on-deep-agents |
| 26 | OpenAI | Symphony: composition as orchestration | 2026-04-27 | https://openai.com/index/symphony/ |
| 27 | Cursor | Cloud agent builds: engineering reliable, fast | 2026-08-13 | https://cursor.com/blog/cloud-agent-builds |
| 28 | OpenAI | Running Codex safely at OpenAI | 2026 | https://openai.com/index/running-codex-safely-at-openai/ |
| 29 | Anthropic | How we contain Claude across products（Cowork egress 事件等新细节） | 2026 | https://www.anthropic.com/engineering/how-we-contain-claude |
| 30 | 徐鑫（腾讯云开发者） | 一个 Skill 搞定服务重构：从链路分析到测试自动化 | 2026-08-25 | https://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247697075&idx=1&sn=0811fc825e510ea62db8152d453d6008 |
| 31 | GitHub Blog（Microsoft） | How to evaluate LLMs before production | 2026-08-25 | https://github.blog/ai-and-ml/llms/how-to-evaluate-llms-before-production/ |
| 32 | Raphael De Lio（Redis，Spring I/O 2026 演讲） | Reduce LLM calls with vector search design patterns | 2026-08-25 | https://www.bestblogs.dev/video/ee1985f8d |
| 33 | Louck（arXiv） | Securing LLM-Agent Long-Term Memory Against Poisoning | 2026 | https://arxiv.org/html/2606.24322 |
| 34 | Shang et al.（arXiv） | MPBench: A Systematic Study of Memory Poisoning Attacks in LLM Agents | 2026-06 | https://www.alphaxiv.org/abs/2606.04329 |
| 35 | Sunil et al.（arXiv） | Memory Poisoning Attack and Defense on EHR Agents | 2026-01 | https://arxiv.org/abs/2601.05504 |
| 36 | Cisco AI Defense | aibom（开源 AI-BOM 扫描工具） | 2026 | https://github.com/cisco-ai-defense/aibom |
| 37 | IETF | draft-singla-agent-identity-protocol-03（AIP） | 2026 | https://datatracker.ietf.org/doc/html/draft-singla-agent-identity-protocol |
| 38 | IETF | draft-gudlab-agentid-protocol-00（AgentID） | 2026 | https://datatracker.ietf.org/doc/html/draft-gudlab-agentid-protocol-00 |
| 39 | Addy Osmani | Agentic Code Quality（约束容量管理与双目的检验的出处） | 2026-08-08 | https://addyo.substack.com/p/agentic-code-quality |
| 40 | 孙成心（阿里技术） | Agent 越改越乱之后，我用评测和轨迹把它拉回来了（skill 自进化四层 Gate + taboo 黑名单） | 2026-08-13 | https://mp.weixin.qq.com/s?__biz=Mzg4NTczNzg2OA==&mid=2247511106&idx=1&sn=e3e481953f140ab1a9afb73f7c40221f |
| 41 | 美团技术团队（图灵评测） | Agent评测漫谈（人机一致 Rubric 二元化 + 桥梁指标 + 长程评测三元组） | 2026-08-07 | https://tech.meituan.com/2026/08/07/Agent-Evaluation.html |
| 42 | 王畅（腾讯云开发者） | AI Coding时代，研发项目管理新范式探索与实践（组织摩擦治理四步法） | 2026-08-19 | https://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247697007&idx=1&sn=f12d0ba52354af093099ab3c6c7c28f3 |
| 43 | 阿里妹（千问AI平台） | 从 Prompt 到 Harness：企业级 Agent 工程的完整演进之路（注意力稀释 70/20/10 + 四层防线 + 单一表示原则） | 2026-08 | https://mp.weixin.qq.com/s?__biz=MzIzOTU0NTQ0MA==&mid=2247561689&idx=1&sn=bb7d379ffd983081f81048e4813d524b |
| 44 | 佚名（趋势研究报告） | Agent 开发指南：技术太多，该怎么学？（回答≠负责 + Goal 契约 + Skills 护城河，174 条参考文献） | 2026-08 | https://mp.weixin.qq.com/s?__biz=MzIzNjE2NTI3NQ==&mid=2247492366&idx=1&sn=260b5fac24951a19de106ab89c5cec31 |
| 45 | 偶啦（得物技术） | 企业级 MultiAgent 的记忆系统：短期上下文与四层记忆架构实现（四层模型 + 并行加载降级 + 先写后删） | 2026-08 | https://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247546577&idx=1&sn=e958fbb5a7701d92612f0da1549ad0ad |
| 46 | 木偶（淘天） | 构建 Agent 自主执行闭环（循环工程 + Task/Job 双抽象 + 执行账本 + 协作语义分级） | 2026-08 | https://mp.weixin.qq.com/s?__biz=MzAxNDEwNjk5OQ==&mid=2650545465&idx=1&sn=96e20d7451fcc622baf2b6f0c2d9ef04 |
| 47 | yannisyang、ethanytzhou | 一篇讲透 Agent 自进化飞轮怎么搭（四齿飞轮 + Skill 四层验证 + 非对称淘汰 + 对齐漂移） | 2026-08 | https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649803847&idx=1&sn=abe5cab137aac47cb042b6baa1a24191 |
| 48 | Mike Krieger（Anthropic CPO，AI Engineer 访谈） | How Anthropic Builds: Lessons from Labs（委派结果 + 意图评审 + 双周 persevere or pivot + 心理可持续） | 2026-08 | https://www.bestblogs.dev/video/8d3ae5678 |
| 49 | 左德军（腾讯云开发者） | 一文搞懂个人AI记忆系统构建全流程（五层记忆 + 两级封顶 + 双 skill 读写分离 + 跨工具所有权） | 2026-08 | https://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247697254&idx=1&sn=4c6c7051662b7a92a77ce80bec68af16 |
| 50 | Anthropic | The AI-Native SDLC playbook（intent/spec/plan 工件链 + hooks 审批 gate + 监控触发闭环） | 2026-08 | https://claude.com/blog/the-ai-native-sdlc-playbook |
| 51 | Uber Engineering | Running a Software Factory Efficiently at Uber Scale（成本方程 + Pareto 选型 + 子 agent 降档 + code-mode + 反模式 dashboard） | 2026-08 | https://www.uber.com/blog/running-a-software-factory-efficiently-at-uber-scale/ |
| 52 | 蒋泽林（林曜，千问AI平台） | 从 ReAct 到 Agent Teams（通信通道≠协作语义 + EvoChamber 消融证据 + Leader/Worker 能力差 + 启发式管理） | 2026-08-31 | https://mp.weixin.qq.com/s/T_sYOS11KrOijp_aCEcgnQ |
| 53 | 若飞（架构师公众号） | Agent Loop 什么时候该停？DSH 和 Pi 给了两种答案（五层停止语义 + max-tokens 截断风险 + 重启 disarmed + 选型四问） | 2026-08-30 | https://mp.weixin.qq.com/s/60H9httJacoMWPgbHG6SJg |
| 54 | 姜剑（飞樰，千问AI平台） | 从 Loop 到 Graph Engineering（单 Loop 四失败模式 + Loops watching loops + 锚点/冻结节点/外部判断） | 2026-09-01 | https://mp.weixin.qq.com/s/BSCzaVPaX7W5E8vrVrVC0g |
| 55 | 蓝翔（腾讯） | 删掉80%的Prompt规则，Agent交付成功率反而更高了（Harness 三件事 + 已存在≠已生效 + 约束判据 + 换 Agent 检验 + Add/Thin） | 2026-09-01 | https://mp.weixin.qq.com/s/fV8qN6qs9ac-VXDwZCuaxA |
| 56 | 左昊（腾讯） | 一文讲透确定性 Harness（阶段化编排 + FlowTracer 零开销留痕 + IsFinal 短路 + 结论要不要负责判据） | 2026-09-02 | https://mp.weixin.qq.com/s/rQYSuTF98-xGcdgGPuNHjQ |
| 57 | 默达（淘天集团-营销&交易技术） | AI 驱动研发体系的实践和思考（Price360-KB 项目 Harness + 文件 Wiki 选型判据 + wiki/tech 分层 + 知识飞轮 + Harness 收缩） | 2026-09-02 | https://mp.weixin.qq.com/s?__biz=MzAxNDEwNjk5OQ==&mid=2650545626&idx=1&sn=cfd0d3011972881686bbb9bedbc319da |
| 58 | 砚东（AliExpress 技术部） | AI Agent 应用精细化评测（指标与架构同构 + 主指标判定 + 路由错误 skip + Judge 四原则 + Mock/Real 双模式） | 2026-09-02 | https://mp.weixin.qq.com/s?__biz=Mzg4NTczNzg2OA==&mid=2247511370&idx=1&sn=c9f4ff1d054cb229ac2f8c1462fcb05e |
| 59 | GitHub Copilot 团队（Erik Kristensen & Napalys Klicius） | How we make AI coding more cost efficient without sacrificing task quality（本地指标陷阱 + 压缩三分策略 + prompt 行为回归测试 + 后台结果直达） | 2026-09-02 | https://github.blog/ai-and-ml/github-copilot/how-we-make-ai-coding-more-cost-efficient-without-sacrificing-task-quality/ |
| 60 | Sergio Villani（Google） | 4 engineering patterns behind the strongest AI Agents Challenge submissions（双向 MCP + 事件驱动并发 + 同标准 fallback + 分层路由） | 2026-09-02 | https://developers.googleblog.com/4-engineering-patterns-behind-the-strongest-ai-agents-challenge-submissions/ |
| 61 | Jan-Felix Schmakeit（Google AI） | How to Write Reliable Rubrics for LLM-as-a-Judge Evaluations（rubric 四原则：原子不重叠/客观事实/只评所求/人工校准） | 2026-09-02 | https://dev.to/googleai/how-to-write-reliable-rubrics-for-llm-as-a-judge-evaluations-ndp |
| 62 | Anthropic（Ali Shazal, Matthew Koen） | A guide to the anatomy of effective commerce agents（单模型+skills 架构 + prompt/skill 频率判据 + UI 组件即工具 + stage/apply + 记忆异步抽取 + 缓存分层） | 2026-09-02 | https://claude.com/blog/the-anatomy-of-effective-commerce-agents |
| 63 | Claude Code 团队 | 团队如何用 Claude Code 重塑软件开发流程（指挥目标 70-80% 工作占比 + harness 功能=失效模式阶段性补偿 + 扇出对抗审查 + 渐进信任验证 + UI/transcript 解耦） | 2026-09-03 | https://www.bestblogs.dev/video/9899b4cdb |
| 64 | Salman Munaf（TikTok SRE） | AI Agents Are Distributed Systems（超时=未知 + 重试不是美德 + 记忆当缓存 + 审批绑定具体参数 + 爆炸半径三问） | 2026-09-06 | https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2651292485&idx=1&sn=6e8d3295322532b385527234117b84a4 |
| 65 | Anish Acharya（a16z） | Companies as Cascading Loops（级联循环组织观 + Agent 攀登人类选山 + 人工介入即知识缺口 + 收益上限选型经济学 + 电力类比） | 2026-09-06 | https://www.bestblogs.dev/article/9f27a9b6d2 |

---

## 13. 经验到规范的映射

这些经验在本仓库 `knowledge/standards/agent-engineering-standard.md` 中已有对应规则，对照如下：

| 经验 | 对应规范章节 |
|---|---|
| 缓存优化 bug 致持续性失忆 | §1.6 Loop Engineering、§8 生产上线指标 |
| 单行 prompt 让质量掉 3% | §1.2 Harness 设计、§9.5 Harness 熵 |
| 上下文焦虑与 context reset | §2.10 上下文成本与稳定前缀、§5.3 可恢复 Workflow |
| 会话中途切模型缓存失效 | §1.7 Runtime Protocol、§9.4 模型与运行时升级 |
| 累积工具错误致上下文腐坏 | §2.3 Context Budget、§4.1 Tool Contract |
| 工具输出过大污染上下文 | §4.3 Tool 描述规则、§2.5 ArtifactRef |
| pkill -9 bash 自杀 | §10 安全、Containment 与最小权限 |
| 扁平对等协调失败需层级 | §3.3 多 Agent 准入条件、§3.12 规模路由 |
| 多 agent 编译内核卡同 bug | §3.1 范式选择、§3.7 链路/单点模式 |
| 多 Agent 放大 eval 污染 | §7.1 Eval 开发要求、§7.4 证据化置信度 |
| 多 Agent 间传话游戏 | §2.8 阶段上下文包与交接产物 |
| 自评不可信需独立 evaluator | §7.1、§7.4 证据化置信度 |
| 单容器即"宠物"需解耦脑手 | §1.7 Runtime Protocol、§5.3 可恢复 Workflow |
| 不可逆上下文决策需分离存储 | §2.5 ArtifactRef、§5.5 Working Memory |
| 自造 Temporal 是死路 | §1.1 复杂度阶梯、§5.3 可恢复 Workflow |
| 完整环境是隐形质量杀手 | §8.1 生产上线指标、§10 安全 |
| 长跑漂移需定期重启 | §5.3、§5.14 Interrupt 与 Resume |
| harness 边界随模型进化重估 | §9.5 Harness 熵与规则债 |
| 模型自识别 benchmark 解密答案 | §7.1 Eval 开发要求、§10 安全 |
| 基础设施配置差 6 个百分点 | §7.1、§7.7 强 Agent 评测 |
| LLM Judge 需专家校准 | §7.4 证据化置信度、§7.7 |
| 无 eval 团队盲飞 | §7 Eval-first 工程、§8 生产上线指标 |
| 在线 A/B 测保留率 | §7.7 强 Agent 评测、§8.1 |
| 三起变更叠加致退化 | §1.6 Loop、§8.1 生产上线指标 |
| HN 质疑自主性声明 | §8.1、§0.5 厂商案例仅作参考 |
| 钓鱼偷 AWS 凭证 24/25 成功 | §10.1 Containment、§10.3 不可信数据 |
| 信任对话之前就执行 hook | §10.1 Containment、§1.8 规则编译 |
| 审计过的连接器≠审计过的数据 | §10.3 不可信数据、§4.1 Tool Contract |
| 凭证不进 sandbox | §10.1 Containment、§10.2 最小权限 |
| 审批疲劳 93% 批准率 | §6.3 人机协作、§10.2 最小权限 |
| 模型创造性地逃沙箱 | §10.1 Containment、§10 安全 |
| 推理 effort 默认值错误 | §1.2 Harness 设计、§9.5 Harness 熵 |
| 多 agent 烧万亿 token | §3.3、§7.8 成本归因 |
| 在线测撤回不值的功能 | §7.7 强 Agent 评测、§7.8 成本归因 |
| agent 间环境性记忆 | §5.5 Working Memory、§10.3 不可信数据 |
| 隐性知识灌进 agent | §14 Skills、§7.4 证据化置信度 |
| harness 假设随模型过时 | §9.5 Harness 熵、§9.4 模型与运行时升级 |
| 框架需 per-model 深度定制 | §1.7 Runtime Protocol、§9.4 |
| 多 agent Markdown 协调协议 | §3.10 编排协议、§12.2 变更规范 |
| 不要外包无法评估的判断 | §3.1 Tool Loop 适用场景、§7.1 Eval 开发要求 |
| Online/Offline eval 二分与 ADLC 飞轮 | §7.10 Online / Offline Eval 与 ADLC 飞轮 |
| 行业 62% 实验 ≤10% 规模化鸿沟 | §8.1 生产上线指标、§7 Eval-first |
| agent 组织自发建法律系统（Fence 取向、再犯收紧） | §1.5 约束分级与升级、§1.8 规则编译 |
| 生码只占 20%-30%（环境与验证是瓶颈、投资折旧判据） | §1.2 Harness 设计、§1.3 环境真实反馈、§7.6 反馈时延分层 |
| 评估闭环是最重要特质（评估你的评估） | §7.1 Eval 开发要求 |
| 单体 AGENTS.md 失败（地图 + 渐进披露、golden principles） | §1.8 规则编译与渐进披露、§2.3 Context Budget |
| 25 小时长跑 durable project memory 三件套 | §5.3 可恢复 Workflow、§2.8 阶段上下文包 |
| 超 150 个 skills 模型退化（两段式加载） | §2.3 Context Budget、§1.8 渐进披露 |
| SPEC.md 即监督者（看板即控制平面） | §3.10 编排协议、§1.2 复杂度阶梯 |
| 环境热构建快照（install/start 分离） | §4 环境可靠性 |
| egress allowlist 被打穿（凭证与身份绑定） | §10.1 Containment |
| 企业安全部署（审批自动化 + AI 分诊日志） | §10 安全 |
| Skill 三层知识库 + 熔断阈值撑起全量重构 | §1.8 渐进式披露、§6.5 结构化澄清、§7.6 反馈时延分层 |
| 三层指标结构 + 生产标签是信号（judge 分诊） | §7.1 Eval 开发要求、§7.4 Gate 阈值否决、§5.3 Judge 校准 |
| 语义路由/缓存确定性前置层（13s→345ms） | §1.1 复杂度阶梯第 0 级、§2 长上下文精度衰减 |
| 记忆投毒洗白通道与来源绑定防御 | §10.8 记忆投毒防护、§10.3 不可信数据规则 |
| AI-BOM 资产清单 + 非人类身份委托链 | §10.6 AI-BOM、§10.7 非人类身份与委托链 |
| Skill 自进化（规则诊断 + 四层 Gate + taboo 黑名单 + 27pp 语义陷阱） | §7.10 规则/Skill 自进化工程纪律 |
| 人机一致率是机评置信前提（Rubric 二元化 62%→92%） | §7.1 Eval 开发要求、§7.10 长程三元组与人评链路压缩 |
| 桥梁指标 + 四层评测 + 评测体系渐进演化 | §7.1 Eval 开发要求、§7.2 Grader、§7.10 ADLC 飞轮 |
| 个体提效 ≠ 组织提效（组织摩擦公式 + 编码时间占比极低） | §8.1 生产上线指标、§1.7 Symphony 瓶颈转移 |
| 数据可见 → 可信 → 异常定位 → 瓶颈挖掘四步治理 | §7.5 自动巡检闭环、§8.1 生产上线指标 |
| 注意力稀释 70/20/10 + 换模型不如管上下文 | §2.2 上下文分层、§2.3 Context Budget |
| 单一表示原则 + 工具结果强制外置 + parameterBindings | §2 工具结果管理规则 |
| 回答 ≠ 负责（幂等键 + 副作用收据 + reconcile） | §3 运行时规则、§4 长任务规则 |
| Goal 可执行契约 + 明确终态 + 结构化 handoff | §4 长任务规则、§7 验收规则 |
| Skills 护城河公式 + 分发四阶段审计 | §6 Skill 治理规则 |
| 四层记忆模型 + 并行加载降级 + scope 级 Token 预算 | §5.1 状态分层、§2.3 Context Budget |
| 循环工程（Task/Job 双抽象 + 执行账本 + 协作语义分级） | §3.4.1 结构化移交、§5.1 Goal 契约 |
| Skill 四层验证 + 非对称淘汰 + 对齐漂移防护 | §7.10 自进化纪律 |
| 委派结果（describe end state）+ 意图 artifact 评审 | §5.1 Goal 契约、§7 评审验收 |
| 双周 persevere or pivot + bet 团队跨学科抽人 | §10 协作开发（AI 时代项目管理参考） |
| 个人记忆跨工具所有权 + 两级封顶 + 双 skill 读写分离 | §5.1 记忆规则、§1.7 渐进披露 |
| AI-Native SDLC 工件链（intent/spec/plan）+ hooks 审批 gate + 监控触发重入循环 | §12.2 变更规范渐进细化、§7.10 ADLC 飞轮、§6 审批 |
| 成本方程分解 + 消灭零价值 token + 反模式三元组 dashboard + 子 agent 默认降档 + Pareto 持续迁移 | §9.6 成本可观测、§7.9 模型分层路由 |
| 通信通道≠协作语义 + 20 agent 无协作机制=1 agent（消融）+ Leader/Worker 能力差 + 启发式管理 | §3.3 多 Agent 准入、§3.9 协调者与专家模式 |
| 五层停止语义 + stopping 消息队列 + max-tokens 禁执行 + 重启后 Goal 默认 disarmed | §1.6 Loop 规则、§5.14 Interrupt 与 Resume |
| 单 Loop 四失败模式（Goodhart/盲视/冲突/测量衰减）+ 评测集冻结审批 + 对冲指标 | §7.10 自进化纪律、§7.4 证据化置信度、§6 审批 |
| Harness 三件事（读对/拦住/接得上）+ 已存在≠已生效 + 约束判据 + 换 Agent 检验 + Add/Thin | §2.2 信任分级、§7.3 Gate、§5.1 状态与 Goal 契约 |
| 确定性 Harness 四设计 + 结论要不要负责判据 + 交付链/判定链失败语义 + 留痕零开销 | §1.1 复杂度阶梯、§7.4 证据化置信度 |
| 文件 Wiki 选型判据（同一 MR）+ wiki/tech 知识边界 + Metadata 事实来源 + 知识飞轮 + Harness 收缩 | §2.12 索引优先检索、§12.2 变更规范渐进细化 |
| 指标与架构同构 + 主指标判定与上游错误 skip + Judge 四原则 + 幻觉评测边界 + Mock/Real 双模式 | §7.7 评测 Harness |
| 本地指标陷阱（全任务成本判据）+ 压缩三分策略 + prompt 行为回归测试 + 后台结果直达 | §9.6 成本可观测、§7.7 评测 Harness |
| 双向 MCP（agent 既 client 又 server）+ 事件驱动并发 + 同标准 fallback（单一验证点）+ 分层路由（确定性检查前置） | §7.9 模型分层路由、§11.1-11.3 MCP、§3.5 并行与嵌套执行 |
| rubric 四原则（原子不重叠 + 客观事实 + 只评所求与评目的地不评路径 + golden set 人工校准） | §7.7 评测 Harness |
| 单模型+skills 优于子 agent 拆分 + prompt/skill 频率判据与预载 + UI 组件即工具 + stage/apply 分离 + 记忆异步抽取 + 缓存分层 | §7.9 模型分层路由、§14.2 渐进式加载、§6 审批、§5.5 Working Memory |
| 指挥目标不监督过程（70-80% 工作在 agent）+ harness 功能=失效模式阶段性补偿（to-do list 生死）+ 扇出采集+对抗审查+确定性循环 + 渐进信任验证 + UI/transcript 解耦 | §3.14 代码编排、§9.5 Harness 熵 |
| 超时=未知先状态查询 + 熔断器与最大并行度 + 记忆当缓存（来源/失效）+ 审批绑定具体参数 + Trace 全链路 | §5.6 故障分级、§4.4 Tool Contract、§5.5 Working Memory、§6 审批、§8 可观测性 |
| 级联循环组织观（人→职能→业务单元）+ 局部最优人选下一座山 + 人工介入即知识缺口 + 收益上限选型经济学 + 模型不可互换（每周交付养直觉） | §7.9 模型选型、§6 审批分级、§5.2 流式编排；playbook §4 缺口信号、§9 持续项目底盘 |

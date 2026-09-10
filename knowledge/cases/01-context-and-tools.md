# Agent 经验 · Context 与 Tool

> **范围：** §1 Context/Prompt 工程、§2 Tool 设计与调用
> 集合：`cases/` ｜ 导航：[知识库索引](../README.md)

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

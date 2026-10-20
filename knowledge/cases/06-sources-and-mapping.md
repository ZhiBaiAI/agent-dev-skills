# Agent 经验 · 来源与规范映射

> **范围：** §11 未找到可靠来源的方向、§12 关键来源索引、§13 经验到规范的映射
> 集合：`cases/` ｜ 导航：[知识库索引](../README.md)

## 11. 未找到 2026 年可靠来源的方向

按"宁缺毋滥"原则，以下方向未找到满足筛选标准的 2026 年来源，明确标注：

- **Netflix / LinkedIn 2026 年 Agent 生产文章**：站点限定搜索无 2026 年结果（Shopify 与 Vercel 已于 2026-09-22 后各出文章，见 #79、#80、#85，此条已部分推翻）。**注意**：这两家技术博客对脚本请求返回 403，需用浏览器核验；"脚本读不到"不等于"没有文章"，核验方式见仓库根 `article-intake-workflow.md` 第二节「抓不到就换浏览器读」。
- **会议分享（ICML/NeurIPS/QCon/GopherCon 2026 有公开 slides/录像的 Agent 经验）**：未找到 2026 年内已公开且符合"真实问题+解决"的会议分享。
- **GitHub Blog《agents.md lessons from 2500 repos》原文**：被广泛引用但官方 URL 仍为 404（2026-10-15、2026-10-19 两次复验，含两个路径变体），未采纳。
- **OpenAI 开发者博客（developers.openai.com）各篇**：**此条已推翻，且读取方式需更正（2026-10-19）**。此前记为"全站不显示发布日期/日期客户端渲染，须用浏览器"；实测再更正为：**服务端渲染的日期用脚本即可读到**——`curl` 抓 `https://developers.openai.com/blog/<slug>` 能直接解析出 `Oct 14, 2026` 这类日期（2026-10-19 实测 8 篇全部成功），无需浏览器。已入库篇目：`Shell + Skills + Compaction` 2026-02-11、`Rethinking skills and prompts for GPT-6 Astra` 2026-09-11、`Testing Agent Skills Systematically with Evals` 2026-01-22、`Automating repetitive work at OpenAI with Codex` 2026-08-25、`Run long horizon tasks with Codex` 2026-02-23、`Codex as a platform` 2026-08-19。
- **OpenAI 主站（openai.com）各篇**：对脚本请求返回 403（2026-10-16 实测），需用浏览器读。**但常规跟踪有更省事的通道（2026-10-19 实测）**：`https://openai.com/news/rss.xml` 返回 200 且含完整标题、发布日期与链接（约 1,300 条），可直接用于发现与按日期筛选，再对入选篇目用浏览器读正文。示例条目：`How Third Moment Research runs 10,000 agents for 36+ hours` 2026-10-14。

---

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
| 26 | OpenAI | Symphony: composition as orchestration（已迁至 open-source-codex-orchestration-symphony，标题改为 An open-source spec for Codex orchestration） | 2026-04-27 | https://openai.com/index/open-source-codex-orchestration-symphony |
| 27 | Cursor | Cloud agent builds: engineering reliable, fast（已迁至 /blog/builds，标题改为 Cloud agents start 3x faster with builds） | 2026-08-13 | https://cursor.com/blog/builds |
| 28 | OpenAI | Running Codex safely at OpenAI（已迁至 /index/running-codex-safely） | 2026 | https://openai.com/index/running-codex-safely |
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
| 51 | Uber Engineering | Running a Software Factory Efficiently at Uber Scale（成本方程 + Pareto 选型 + 子 agent 降档 + code-mode + 反模式 dashboard；已迁至 /us/en/blog/efficient-software-factory/） | 2026-08 | https://www.uber.com/us/en/blog/efficient-software-factory/ |
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
| 66 | Anthropic | Agentic coding is straining CI. Here's how we scaled test impact analysis at Anthropic（补丁续命 70 天/29 天/1 天 vs 重设计三周一季 + 单写者进程内状态是水平分片根因 + listener 滞后让 selector 用过期数据决策 + 指标做 agent 的眼睛与耳朵） | 2026-09-14 | https://claude.com/blog/agentic-coding-is-straining-ci-heres-how-we-scaled-test-impact-analysis-at-anthropic |
| 67 | Anthropic（Frontier Red Team） | Patterns and problems in multiagent systems（协调 swarm 27M token/266 漏洞 vs 独立并行 6.5M/21，仅 12 个重叠 + 角色/CEO 提示词干预无效 + 协调能力随模型代际变化 + 低方差一致性失败） | 2026-08-13（8-27 更新） | https://www.anthropic.com/research/multiagent-systems |
| 68 | Anthropic | Reducing cost and improving performance with Claude Platform（缓存命中率是架构属性不是调优参数 + 降本三手段：最大化缓存命中/升级时清提示反模式/按任务校准 effort） | 2026-09-08 | https://claude.com/blog/reducing-cost-and-improving-performance-with-claude-platform |
| 69 | 大淘宝技术 | Loop engineering：把 agent 放进工程循环（六组件控制面 + 五种死法 + 验证结构化 + 权限四级梯 + 预算上限 + 产物可沉淀准入） | 2026-09-21 | https://mp.weixin.qq.com/s/RxRzTsRvmZJMmtQjQM79_g |
| 70 | Addy Osmani | Loop Engineering（工程师从操作者变为循环机制设计者；前馈约束与后馈传感器） | 2026-06-07 | https://addyosmani.com/blog/loop-engineering/ |
| 71 | Birgitta Böckeler（Thoughtworks） | Harness engineering for coding agent users（只有调度没有控制件是定时犯错） | 2026-04-02 | https://martinfowler.com/articles/harness-engineering.html |
| 72 | Eric Carlson（Amplitude） | What I Learned Pointing a Ralph Loop at My Product for a Week（agent 自埋点做下轮验证 + 每 PR 附浏览器录制作证据） | 2026-05-13 | https://amplitude.com/blog/ralph-loop |
| 73 | Geambasu et al.（arXiv） | Engineering Robustness into Personal Agents with the AI Workflow Store（可靠流程沉淀为可复用受约束工作流，而非每次临场发挥） | 2026-05-11（v3 2026-06-10） | https://arxiv.org/abs/2605.10907 |
| 74 | Xin et al.（arXiv） | EurekAgent: Agent Environment Engineering is All You Need For Autonomous Scientific Discovery（权限、产物、预算与 HITL 是环境工程不是配套设施——有边界执行与隔离评估下 agent 表现判若两人） | 2026-06-11 | https://arxiv.org/abs/2606.13662 |
| 75 | 介错者 | 重读《人月神话》：在所有人都有 Agent 之后（组织是人的 Harness；补丁堆进 Agent 让错误边界维持更久；共同任务书六问；按用户可见结果验收；只奖励好消息让 Agent 漏掉坏消息） | 2026-08-31 | https://mp.weixin.qq.com/s/P-rDy0BpW1le5Mn6IpCTdQ |
| 76 | 盈衣（淘天集团-交易数据技术） | 淘宝AI购物助手在线评测落地实践（四层数据清洗 + 跨轮信息隔离 + CTX 诚实跳过 + 四层归因 + 指标拆分子分母 + 样本量按 CI 反推） | 2026-10-12 | https://mp.weixin.qq.com/s/blMjReLQZUuSK6bl6Bt3yA |
| 77 | Naresh Jain（Specmatic / ConfEngine） | Are Harnesses Worth the Cost?（同一任务 10 组配对实验：前沿模型加 harness 成本涨 7 倍、验收零提升；便宜模型加 harness 从 3/10 到 10/10；结论是介入时机而非有无） | 2026-10-13 | https://martinfowler.com/articles/are-harnesses-worth-the-cost.html |
| 78 | Anthropic | Redesigning source control at Anthropic（合并量 5 个月涨 10 倍压垮 git；7 人 5 个月自建 Scion，刻意放弃分布式与离线；承认无法复制合并队列全局锁；工程师常对 Claude 说"不"） | 2026-10-13 | https://www.anthropic.com/engineering/redesigning-source-control |
| 79 | Shopify（Mingyu Zhao） | ShopGym: Realistic, reproducible sandboxes for shopping agents（线上店铺转可重置沙箱；224 任务/6 店；孪生环境状态数相当但连接数偏少，聚焦型任务差距明显——保真度边界被量化） | 2026-10-01 | https://shopify.engineering/shopgym |
| 80 | Vercel（Andy Riancho） | Findings from the $1M Vercel Sandbox challenge（两周 1285 份报告、付费 24.2 万美元；内核零日可泄露宿主内存；三条判据：客机派生值不可作权威、同一决策只留一个执行点、依赖秘密的边界不是边界） | 2026-10-14 | https://vercel.com/blog/findings-from-the-1-million-dollar-vercel-sandbox-challenge |
| 81 | Google Cloud（George Weale, Bo Yang, Andrew Biernat） | Optimizing a 200,000-line Python codebase overnight with Gemini 4 Argon（20.5 万行/800 文件/1.7 万测试；提 6 项只过 2 项，四项驳回理由具体；通过 16892 开源测试却挂 25 个内部测试；"agent 探索、维护者做减法"） | 2026-10-13 | https://developers.googleblog.com/en/optimizing-a-200000-line-python-codebase-overnight-with-gemini-4-argon/ |
| 82 | Cursor（Jediah Katz, Connor O'Keefe, Calvin Yee） | Improved token efficiency for longer agent runs（系统提示砍 66%、工具按需加载、缓存断点前移、每十行标行号；token 成本降 7%、MCP 会话降 46.9%；反直觉：子 agent 有协调税、只在评测集上优化不够） | 2026-09-23 | https://cursor.com/blog/improved-token-efficiency |
| 83 | LangChain（Sydney Runkle, Eugene Yurtsev） | How to Build a Model Router in the Harness（973 线程对照：中位成本 $0.94 vs $2.61 降 64%，合并 PR 率无显著差异 p=0.49；只用便宜模型的对照组一天内被叫停；子 agent 各自选模型漏掉收益） | 2026-10-01 | https://www.langchain.com/blog/how-to-build-a-model-router-in-the-harness |
| 84 | Anthropic | Investigating unintended model actions in our evaluations and internal use（四类越界：利用缺陷执行命令、提交真实表单、绕过限制取数据、短链规避抓取限制；根因是训练奖励了投机取巧；模型自述推理不是可靠证据；联网评测已下线） | 2026-10-09（10-12 更正） | https://www.anthropic.com/research/investigating-unintended-model-actions |
| 85 | Vercel（Amelia Charles, Andrew Qu, Jonathan Hefner） | State of agent skills（7 个月 100 万技能/2.8 亿安装；0.04% 占 62% 安装、近半数只装过一次；跨行业技能安装量 3.6 倍；软件工程类安装量低于平均 30%。**vendor 自报数据**） | 2026-09-25 | https://vercel.com/blog/state-of-agent-skills |
| 86 | Uber Engineering | Designing MCP Gateway: Uber's MCP Management Platform（800+ 服务、5000+ 工具；MCP 无跨服务搜索，agent 必须预知地址；默认关闭"发现不等于暴露"；渐进发现、响应投影、结果外置） | 2026-10-01 | https://www.uber.com/us/en/blog/designing-mcp-gateway/ |
| 87 | GitHub Security Lab（Antonio Morales） | AI-powered fuzzing with the GitHub Security Lab Taskflow Agent（三层架构：agent 决策/工具执行；阶段只经 SQLite 通信；为引导与报告分别构建两套探针；时间预算倍增 + 平台期检测；无容器隔离的警告） | 2026-09-24 | https://github.blog/security/application-security/ai-powered-fuzzing-with-the-github-security-lab-taskflow-agent/ |
| 88 | GitHub（Michelle Zhou, Alejandro Carderera de Diego） | ReviewBench: An open benchmark for AI code review（分析 1.039 亿 PR 建模分布；219 PR/187 仓库/19 语言；固定金标集会随能力失效故设 grounded 与 augmented 双口径；离线在线同向：采纳率 +8%、召回 +13.6%） | 2026-10-05 | https://github.blog/ai-and-ml/github-copilot/reviewbench-an-open-benchmark-for-ai-code-review/ |
| 89 | Anthropic（Lance Martin） | Automating eval design and hillclimbing with Claude（挑模型失败案例采到的是失败指纹不是难度；评委用可核对命题非 1–5 分；评测噪音须小于最小可行动改进；残留文件会把答案递给 agent；用户流量偏易） | 2026-09-28 | https://claude.dev/blog/automating-eval-design-and-hillclimbing/ |
| 90 | Google Cloud（Dima Melnyk, Elia Secchi） | The Outer Loop, Insights First: An Ambient Quality Agent（线上全绿却悄悄答错；刻意不设置信度字段、信任锚是引用且无效引用被服务端拒收；聚类在被 transcript 验证前只算主张；随机采样而非只筛异常；96 会话花费 $0.70） | 2026-10-08 | https://developers.googleblog.com/en/the-outer-loop-insights-first-an-ambient-quality-agent-that-diagnoses-your-production-agent/ |
| 91 | LangChain（Arjun Natarajan, Eugene Yurtsev, Suraj Vathsa） | Search agent traces by behavior with AI Search in LangSmith（trace 中位 7.2 万 token 里约 85% 是工具输出、用户与 agent 文本仅约 2 千；向量检索分不出"做成了"与"声称做成了"；固定 8K 预算按优先级填充及其失效边界） | 2026-10-13 | https://www.langchain.com/blog/agent-traces-ai-search-langsmith |
| 92 | LangChain（Divya Santhanam） | The Most Common Questions About Agent Evals, Answered（10–20 条起步、把失败转成样本；按线程而非按轮评；结果对不等于路径对；某决策模型评委与人工判定全一致且花费 $0.34 vs $28.17） | 2026-10-15 | https://www.langchain.com/blog/common-questions-about-agent-evals |
| 93 | Uber Engineering | Scaling AI in Legal: Building Uber's Redlining Agent（合同评审时间降 20%、决策准确率 91%；语气规范交给法务自己持有；带半衰期的时间加权采样对抗概念漂移；早期纯检索三失败：相似度不准、语气不对、泛化差） | 2026-10-08 | https://www.uber.com/us/en/blog/building-ubers-redlining-agent/ |
| 94 | LangChain（Sydney Runkle, Srimanth Tangedipalli, Neal Patel） | Building Long-Running Agents with ScaleDown and Deep Agents（高频结构化上下文决策交窄任务专用小模型，开放式生成留给通用模型；压缩结果附带说明可回取；专用模型失败时保持历史原样不阻塞主链路。**厂商自报倍数**） | 2026-10-13 | https://www.langchain.com/blog/building-long-running-agents-with-scaledown-and-deep-agents |
| 95 | LangChain（Sydney Runkle, Hunter Lovell） | Building Prod with Jev and LangGraph（持久化执行的真正价值是"重跑可能走出另一条路径"；知识放在图的拓扑而非提示词；专用决策模型处理有界选择、阈值升级给 LLM） | 2026-09-25 | https://www.langchain.com/blog/building-prod-with-jev-and-langgraph |
| 96 | LangChain（Sydney Runkle） | Revamping Skills in Deep Agents（工具与技能绑定：未读技能调用失败为未知工具；技能可固定与重载；中途加工具放进新系统消息以保缓存前缀不变；重载致缓存失效的取舍） | 2026-10-07 | https://www.langchain.com/blog/revamping-skills-in-deep-agents |
| 97 | Anthropic（Lance Martin, CJ Avilla） | Building effective agent automations（按来源存书签而非固定时间窗；失败不等于沉默、保留书签并显式标注未读来源；发帖幂等与"可能已发"；发布前重新核实；陈旧记忆本身是危害；凭证占位符加保险库） | 2026-10-08 | https://claude.dev/blog/building-effective-agent-automations/ |
| 98 | GitHub（Burke Holland） | When chat is the wrong UI（意图明确后聊天是错误界面；让 agent 按需生成专用界面，后续交互变免费；把 agent 本身当工具反复使用是纯粹的 token 浪费） | 2026-09-24 | https://github.blog/ai-and-ml/github-copilot/when-chat-is-the-wrong-ui/ |
| 99 | Elastic（Toby Sutor） | Trust, but verify: Atomic claim checking against LLM hallucinations（生成与验证分离禁止自证；源拆成原子声明做存在/部分/缺失三态核对；"部分"价值最高；技术载荷逐字保留 + 强制申报冲突与省略；人工从通读降为裁定标记行，一小时降到五分钟；可信是架构属性不是模型属性） | 2026-09-24 | https://www.elastic.co/blog/atomic-claim-checking-against-llm-hallucinations |
| 100 | Cloudflare | Cloudflare Containers, rebuilt to scale agent sandboxes（镜像与实例类型从部署期配置改为运行时参数；放置决策下移到调用点；复用预热实例与预分发基础镜像；100 并发沙箱中位启动 4.049s→648ms；旧抽象仅维护到 2026-12-31 是有意保留的迁移负担） | 2026-09-30 | https://blog.cloudflare.com/faster-agent-sandboxes/ |
| 101 | GitHub（Brian Celenza） | Building Git infrastructure for agent-scale development（活动量 2182 亿→4733 亿事件/月、提交同比 5 倍；**读好扩写难扩**——加副本反而拖慢写；持久化机制同时是扩展机制；最小协调面 + 存储计算解耦 + 维护移出服务路径；内部基准写吞吐提升 35 倍） | 2026-10-06 | https://github.blog/engineering/architecture-optimization/building-git-infrastructure-for-agent-scale-development/ |
| 102 | GitHub / Microsoft CoreAI（Batu Ozturkler, Shengyu Fu） | GitHub's model for secret detection（**模型大小不等于模型有用**；正则保召回 + 微调小分类器读上下文判真伪；难负例靠学习不靠规则——加规则会藏起真密钥；不替代确定性扫描而是互补；精确率 +13% 召回 +19%，扫描耗时 -86% 成本 -98%） | 2026-10-13 | https://github.blog/ai-and-ml/githubs-model-for-secret-detection-finding-more-secrets-with-less-noise/ |
| 103 | Anthropic（Jonah Ezekiel, Lexie Tonelli） | How to prepare for AI-driven code modernization projects（先分类再建证书最后动代码；三类改造的正确标准不同，分类不清的争论会后期复现；证书=机器可检查的完成条件让 agent 无人迭代；反复问题在源头修；主要驱动是风险降低不是降本；小片调顺工作流再铺开） | 2026-09-23 | https://claude.com/blog/how-to-prepare-for-ai-driven-code-modernization-projects |
| 104 | NVIDIA（Sophia Abbassi 等） | How to Evaluate AI Agents From Tool Calls to Task Completion（调用准确"必要但不充分"；端到端评结局做发布门禁、步骤级评过程做定位；单次调用准确≠任务完成；工具精确率掩盖槽位填充失败；**报区间不报点**；基准之间不可比；污染面含联网搜到答案键） | 2026-09-21 | https://developer.nvidia.com/blog/how-to-evaluate-ai-agents-from-tool-calls-to-task-completion/ |
| 105 | LangChain（Oliver Armstrong） | How Snyk Turned an Internal Support Agent into a Customer Feature（内部跑一年再对外；工具按用户注册为类型化函数 + 中间件挂钩；**agent 只能访问用户本来就能看的数据**；离线+线上双层评测并在 CI 用阈值阻断；6 万+ 查询、85%+ 免工单） | 2026-10-08 | https://www.langchain.com/blog/how-snyk-turned-an-internal-support-agent-into-a-customer-feature |
| 106 | Cloudflare | Introducing Clef: our open-source decision models（输出受限结构而非自由文本；预填充后并行打分的非自回归推理；标签平滑 + Brier 校准；边缘部署压延迟；**诚实披露四项基准里输掉一项**：CLINC150+OOS 66.77 vs 89.27；中位延迟 209ms vs 对照 524ms） | 2026-10-01 | https://blog.cloudflare.com/clef-decision-models/ |
| 107 | Anthropic Frontier Red Team（William Zhou, Daniel Freeman） | AI models may be capable of self-replicating without access to a datacenter（26B 模型在 15 台电视盒子上分布式推理；前沿开源模型在 8 节点 VPC 完成端到端复制 5 试 4 成；推理搭建下限 9B；可用的防御动作：监控多 GB 载荷与设备间流量、消除未认证服务、带通知的清理 14 天清 92%、蜜罐取 harness；作者自陈把网络利用环节做成平凡，是成功率上界） | 2026-10-12 | https://www.anthropic.com/research/self-replicating-ai |
| 108 | OpenAI | Better prompt caching for GPT-6（工具变化不一定要删定义——用允许列表与工具选择 none 保住缓存前缀；新指令追加到上下文末尾；推理档位可中途调整而不破坏缓存；预热已知上下文；缓存诊断能区分工具/模型/参数/输入哪一类导致失效。**浏览器读取**） | 2026-09-22 | https://openai.com/index/better-prompt-caching-for-gpt-6 |
| 109 | OpenAI（Asana StackAI 案例） | Asana cuts model costs 76x in browser tests with GPT-6.1 Sol（缓存只缓了固定指令没缓逐轮增长的历史；历史既增长又被裁剪时只加缓存无效，须降低裁剪频率使历史在两次清理间保持稳定；先把代码重构成支持受控实验再开跑；144 次运行研究，成本 $36.21→$1.24→$0.47，快 5 倍。**浏览器读取**） | 2026-10-09 | https://openai.com/index/asana-browser-agent |
| 110 | OpenAI（LegalOn 案例） | LegalOn halves Codex costs while maintaining development speed（按任务复杂度分三档并默认从轻量档起步；按业务成熟度分预算——成熟业务承担降本目标、投入期新业务保充足预算；选型给默认 + 系统限额给硬边界；收紧快速模式默认开关并用并行补回速度；日均成本降 65% 而速度不变。**浏览器读取**） | 2026-10-08 | https://openai.com/index/legalon-halves-codex-costs |
| 111 | OpenAI（Third Moment Research 案例） | How Third Moment Research runs 10,000 agents for 36+ hours（1 万 agent 并发、单次会话 36–48 小时；把 Codex 当外层 harness 的子 agent，由外层看轨迹决定重试/终止/升级——内层无法区分"做完了"与"以为做完了"；把轮询等待当零价值消耗消除，回测 3 小时→3–5 分钟。**浏览器读取**） | 2026-10-14 | https://openai.com/index/third-moment-research |
| 112 | OpenAI | A model guide for the GPT-6 family（模型与推理档位是智能/价格权衡；默认从轻量档起步；prompt 与 skills 四个复查面——技能描述短而明确、AGENTS.md 说明何时相关并显式授权安全例行流程、用清晰边界替代"永远先问"、把"完成"定义写死含跑通与检查；长跑用中途纠偏与异步工具。**浏览器读取**） | 2026-10-02 | https://openai.com/index/practical-guide-building-gpt-6 |
| 113 | OpenAI | Skills that shell and compact（技能描述是路由逻辑不是文案，写"该用 vs 不该用"；补负例后触发率才恢复——初版可用化反而让定向评测触发率降约 20%；Skill 过多时平台自动截短描述；模板放 Skill 内而非系统提示是质量与延迟收益最大的改动之一） | 2026-02-11 | https://developers.openai.com/blog/skills-shell-tips |
| 114 | OpenAI | Rethinking skills and prompts for GPT-6（描述要短而明确；多工作流 Skill 根文档做最小路由器指向配套文档；AGENTS.md 写"何时相关"并显式授权安全例行流程；用清晰边界替代"永远先问"；把"完成"定义写死；容器复用与压缩的取舍） | 2026-09-11 | https://developers.openai.com/blog/rethinking-skills-and-prompts-for-gpt-6-astra |
| 115 | OpenAI | How we eval Skills（七步流程；prompt 集四类 case——显式点名/隐式/上下文依赖/负控制 should_trigger=false；用确定性程序解析结构化输出判触发，不用 LLM 打分；同一 harness 可扩展为查工具横跳、token 预算、构建通过；主观项用带约束 JSON rubric） | 2026-01-22 | https://developers.openai.com/blog/eval-skills |
| 116 | OpenAI | Automating repetitive work at OpenAI with Codex（Runme notebook 加目标单元格让工作可重复执行；WebMCP 把浏览器侧操作注册成工具供 agent 调用；决策在消散进聊天记录前先落到 notebook） | 2026-08-25 | https://developers.openai.com/blog/automating-repetitive-work-at-openai-with-codex |
| 117 | OpenAI | Codex as a platform（为 ARC-AGI-3 重设计 harness——任务表示、动作空间、上下文组织、反馈回路——把某前沿模型得分从 13.3% 提到 38.3%，输出 token 降到约六分之一；三层集成：`codex exec` / SDK / app-server；比模型必须固定 harness） | 2026-08-19 | https://developers.openai.com/blog/codex-as-a-platform |
| 118 | Anthropic / Claude Code（Addy Osmani, Morgan Westlee Lunt, Qing Wang） | How the Claude Code team does code review（七层评审防线 + 风险分级人工审批 + 审批绑定 commit + 事故即评测集 + 评审收敛：一轮通过 65%→78%、≥5 轮 8.9%→1.5%、平均轮次 2.08→1.32；合并量翻倍而回滚率降超 40%） | 2026-10-16 | https://claude.dev/blog/solving-code-review-at-anthropic/ |
| 119 | Google（Hugo Selbie, Dani Zamora, Jesse Carah, John DeMartino） | Evaluating AI agents: moving past the vibe check（CUJ 定义评测点 + 模拟用户起步 + Day 0 红笔读 5–20 条 trace + 四陷阱 + Day 2 CI 双规则 + judge 用人工标注校准 + 50–60 次小改动调优） | 2026-10-15 | https://developers.googleblog.com/en/evaluating-ai-agents-moving-past-the-vibe-check/ |
| 120 | LangChain（Jess Weng） | Context Engineering: How Deep Agents Remember（外置 / 摘要 / 技能渐进披露 / 子 agent 隔离 / 长期记忆五类手段；外置阈值约 2 万 token、用户消息 5 万、摘要触发 85%；六条原则：外置优于截断、压缩须可恢复、知识按需加载、噪音要隔离、记忆要自持、阈值要可配） | 2026-10-14 | https://www.langchain.com/blog/context-engineering-deep-agents-remember |
| 121 | 腾讯技术工程（komahu） | 3 个月用 Harness 重构 90 万行旧项目（旧系统当需求说明书全部重写；验收标准写死并量化；子 Agent 按维度专职审查；红线配置脱离脚本默认值；变异测试只跑增量；CLI + Hook 守卫防篡改；Context 三层加载。结果：12 万行代码 / 15 万行文档 / 6000+ 测试 / 覆盖率 86%，两月 30+ 需求） | 2026-10-16 | https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649804394&idx=1&sn=e5c46efb358d97b9f327f49175a8da53 |
| 122 | Carlos Kelkboom（Patrick Akil 播客，InfoQ 中文编译） | AI 生成的代码看不懂就扔掉重来（五条原则：亲自搭骨架、主 Agent 只读且把握 >90% 才启动执行、看不懂就丢弃重做、尽量一次做大、多路并行加独立评审；先看 diff 再看文件；PR 讨论不可消失） | 2026-10-18 | https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2651295352&idx=1&sn=9ee14521ea3fad2bf13007ac2abec56f |
| 123 | Anthropic（官方文档；InfoQ 中文转述） | Dynamic Workflows 编排与完成语义（编排程序替代主 Agent 逐次协调；`completed` 只代表程序执行结束；累计 1000 / 并发 64 / 默认 24 小时；预算非硬开关；上下文隔离≠文件隔离；子 Agent 不能互发消息） | 2026-10-18（文档 2026-10-19 核验） | https://platform.claude.com/docs/en/managed-agents/workflow-runs |
| 124 | 徐鑫（腾讯云开发者） | AI 工程化防翻车指南（安全边界外置到 Harness；事前隔离授权 / 事中循环封顶与三色卡 / 事后门禁审计；MCP 工具描述哈希锁定、按声明授权、成本包络、哨兵只吹哨、门禁脚本写单测、虚拟需求空跑；知识库检索结果同样不可信；参数缺省即拒发） | 2026-10-20 | https://mp.weixin.qq.com/s/5zhFeM48-ipqDyIAPvRj9A |
| 125 | Adam Tornhill（CodeScene；InfoQ 中文编译） | 三周重构 30 万行，Code Health 5.6→10.0（726 文件 / 2903 提交 / 25.2 万行改动 / 约 4000 美元；CodeHealth MCP 确定性评分 + 逐帧回放比对状态哈希；自建 22 方案 82 条说明并记录失败；小模型卡在局部最优；70%/45% 是推算非实测） | 2026-10-17（原文 2026-09-24） | https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2651295294&idx=3&sn=38b16d3e03f2113fbe9fd53b43505f55 |
| 126 | Pierre Pureur, Kurt Bittner, Todd Miller（InfoQ 中文编译） | 利用 AI 代码智能体优化软件架构的五种方法（记录遗留服务 / 发现架构缺陷 / 安全审计 / 提供架构基础 / 生成可测试 MVA；只给功能需求不会自动保证架构合理，须给可衡量 QAR 与权衡；编码变简单而写需求变难；架构评估有收益递减） | 2026-10-17（原文 2026-09-28） | https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2651295294&idx=2&sn=3bb12b13f32ebdeddc3d8fbc617b525c |
| 127 | 巷年（大淘宝技术） | 淘内会场 Agent Harness（一句话生成可投放页面；视觉软约束 / 货品认知约束 / 生码硬约束装配同一棵 DOM；视觉分组≠供给分组、wrapper 改变布局前提、供给写进属性；Skill 正文压缩后显式恢复；缓存未命中四类归因；交接用结论+记录地址；质检合法反例须持续回归） | 2026-10-19 | https://mp.weixin.qq.com/s?__biz=MzAxNDEwNjk5OQ==&mid=2650546485&idx=1&sn=b679cea45f2b32788e0c3eb2d3568cac |
| 128 | 褚杏娟（InfoQ 原创） | 谷歌开源 Harness²（八类可改组件；任务级"提出—探测—组合"用行为差异替代 ground truth；Harnessing 级维护持续改进 Harness；WorkBuddy-Bench +11.5 / JobBench +4.5 / LAB +11.8pp；用测试时计算换成功率；执行器默认联网且 Codex 关沙箱；883 次提交 8 个月后维护成本超过使用收益） | 2026-10-19 | https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2651295407&idx=1&sn=b8c188b3e578205013431c43c98bc95d |
| 129 | Steef-Jan Wiggers（InfoQ 编译；原文 Cloudflare） | Cloudflare 容器跨租户数据泄露（漏洞在存储分配器而非虚拟机边界；64 KiB 精简块 + skip_block_zeroing，4 KiB 写入后残留 60 KiB；六个部署 5614 个可测目录块全部来自他人；修复须清空既有映射与镜像缓存；判据=隔离要逐层验证、"分配时清零"须显式配置） | 2026-10-19（原文 2026-09-24） | https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2651295407&idx=4&sn=99d7b1b770fe9bc89cf7ca7a6b6bc3ae |

---

---

## 13. 经验到规范的映射

这些经验在本仓库 `knowledge/standards/`（导航见 `knowledge/README.md`）中已有对应规则，对照如下：

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
| HN 质疑自主性声明 | §8.1 生产上线指标、§7.8 成本归因、§19 硬规则 76（自报与独立结果比较） |
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
| 补丁续命时间指数衰减，重设计工期按「人周」估而非按历史经验估 | §1.2 Harness 设计、§8.1 生产上线指标 |
| 关键状态从第一版就外置（单写者进程内状态是水平分片的根因） | §5.1 状态分层、§5.3 可恢复 Workflow |
| 服务指标做 agent 的眼睛与耳朵（进出对账让 agent 自己爬坡，比人盯快） | §7.1 Eval 开发要求、§8.1 生产上线指标 |
| 协调 swarm 与独立并行互补而非替代（266 vs 21 仅重叠 12，限定范围后 token 效率相当） | §3.1 范式选择、§3.3 多 Agent 准入 |
| 提示词干预（规定角色/CEO 层级）不改变多 agent 协调结果 | §3.9 协调者与专家模式 |
| 协调能力随模型代际变化（老模型冲突后放弃、中间代际各占文件、最新代际高共享高吞吐） | §9.4 模型与运行时升级 |
| 低方差一致性失败（同源同上下文 agent 互相审查不构成交叉验证） | §3.9 协调者与专家模式、§7.4 证据化置信度 |
| 缓存命中率是架构属性（前缀字节级稳定 + 波动值外置 + 命中率可诊断对账） | §2.10 上下文成本与稳定前缀、§9.6 成本可观测 |
| Loop 六组件与五种死法（可靠性来自控制面 + 验证结果结构化 + 状态文件必须被读 + 权限四级梯 + 预算上限 + 产物可沉淀准入 + harness 自维护 loop） | §1.6 Loop 规则、§5.11 Loop State、§7.6 分层验证、§6.1 审批 |
| 组织是人的 Harness（规则与补丁归资源拥有方 + 共同任务书六问 + 按用户可见结果验收 + 坏消息必须能上报） | §5.7 知识资产分层、§2.8 阶段上下文包与交接产物、§7.4 证据化置信度 |
| 评委可信度瓶颈在输入（四层数据清洗 + 跨轮按轮投喂 + 诚实跳过 N/A + 四层归因 + 指标先拆分子分母 + 样本量按 CI 反推） | §7.1 Eval 开发要求、§7.7 评测 Harness、§7.10 Online/Offline 飞轮 |
| Harness 不是默认有益（前沿模型加 harness 成本涨 7 倍验收零提升；弱模型从 3/10 到 10/10；判据是介入时机与频次） | §1.2 Harness 设计 |
| 合并量 5 个月涨 10 倍压垮支撑设施（自建系统的规模判据 + 对 agent 产出说"不"） | §1.2 Harness 设计、§9.5 Harness 熵、§5.2 关键状态外置 |
| 孪生环境保真度的量化边界（状态数相当但连接数偏少、聚焦型任务差距明显） | §1.3 环境真实反馈、§7.7 评测 Harness |
| 沙箱悬赏的三条判据（客机派生值不可作权威 + 同一决策只留一个执行点 + 依赖秘密的边界不是边界） | §10.1 沙箱边界、§10.4 Workspace 与 Sandbox 状态 |
| 20 万行过夜优化（6 项只过 2 项及四种驳回形态 + 通过开源测试却挂内部测试 + agent 探索人做减法） | §7.6 分层验证与假修复检测、§8.1 生产上线指标、§1.2 自主度上界 |
| token 效率七手段（系统提示砍 66% + 工具按需加载 + 缓存断点前移 + 每十行标行号 + 删子 agent 引导；子 agent 有协调税；只在评测集优化不够） | §2.3 Context Budget、§2.10 上下文成本与稳定前缀、§9.6 成本可观测 |
| 973 线程模型路由对照（成本降 64% 而质量无显著差异；只用便宜模型的对照组一天内被叫停；子 agent 各自选模型漏掉收益） | §7.9 模型分层路由、§9.6 成本可观测 |
| 训练环境奖励投机取巧（四类越界行为 + 模型自述推理不是可靠证据 + 联网评测下线） | §10.3 不可信数据规则、§10.1 沙箱边界、§8.9 Lesson 与 Pattern |
| 技能生态分布（0.04% 占 62% 安装 + 近半数只装一次 + 跨行业 3.6 倍 + 供过于求领域采用更低） | §14 Skills 规范、§5.7 知识资产分层 |
| 高频上下文决策交窄任务专用小模型（受限输出空间 + 失败不阻塞主链路 + 开放式生成仍用通用模型） | §2.3 Context Budget、§7.9 模型分层路由 |
| MCP 网关规模化（800 服务/5000 工具 + 无跨服务搜索 + 发现不等于暴露 + 渐进发现与响应投影） | §11.3 MCP 成本与上下文边界、§4.6 Capability 与 Action Space |
| 模糊测试三层架构（agent 决策/工具执行 + 阶段只经持久状态通信 + 为两个目的分别产出探针 + 平台期检测） | §4.1 Tool 是契约、§5.3 可恢复 Workflow、§4.4 Tool 输出规则 |
| ReviewBench 双口径（固定金标集随能力失效 + grounded 与 augmented 并设 + 评论数量不代表质量） | §7.7 评测 Harness、§7.10 Online/Offline 飞轮、§7.1 Eval 开发要求 |
| 评测五条反直觉（挑模型失败案例采到失败指纹 + 评委用可核对命题 + 噪音须小于最小可行动改进 + 残留文件会递答案 + 用户流量偏易） | §7.1 Eval 开发要求、§7.4 证据化置信度、§7.7 评测 Harness |
| 环境质量 agent（不设置信度字段只信引用 + 聚类验证前只算主张 + 随机采样而非只筛异常 + 跳过显式记账） | §7.4 证据化置信度、§7.5 自动冒烟与回归闭环、§8.3 Trace 与 Trajectory |
| trace 构成（85% 是工具输出 + 固定预算按优先级填充及其失效边界 + 向量检索分不出做成了与声称做成了） | §2.3 Context Budget、§8.3 Trace 与 Trajectory、§7.10 飞轮 |
| 评测高频问答判据（10–20 条起步 + 按线程评 + 结果对不等于路径对 + 评委成本差 80 倍而判定一致） | §7.1 Eval 开发要求、§7.10 Online/Offline 飞轮、§7.9 模型分层路由 |
| 法务 agent（评审时间降 20% + 语气规范归法务持有 + 半衰期对抗概念漂移 + 纯检索三失败） | §5.7 知识资产分层、§7.9 模型分层路由、§7.10 自进化纪律 |
| 持久化执行的价值（重跑可能走出另一条路径 + 知识放图拓扑而非提示词 + 专用模型阈值升级） | §5.3 可恢复 Workflow、§7.9 模型分层路由、§3.10 编排协议 |
| 技能与工具绑定（未读技能调用失败为未知工具 + 可固定与重载 + 中途加载保缓存前缀） | §14.2 渐进式加载、§2.10 上下文成本与稳定前缀 |
| 无人值守 agent 设计（按来源存书签 + 失败不等于沉默 + 发帖幂等与"可能已发" + 陈旧记忆是危害 + 凭证占位符） | §5.11 Loop State、§5.6 故障分级、§5.5 Working Memory、§10.1 沙箱边界 |
| 让 agent 造工具（意图明确后专用界面优于反复对话 + 把 agent 当工具反复用是 token 浪费） | §16 项目应内置能力、§4.9 CLI 与 MCP 选择 |
| 轨迹是 trace 的投影（每条消息只列一次 + 定位行为后再跳进 trace + 单条轨迹只反映一次会话） | §8.3 Trace 与 Trajectory、§7.10 飞轮 |
| 原子声明核对法（生成与验证分离禁止自证 + 拆成原子声明做三态判断 + "部分"价值最高 + 技术载荷逐字保留 + 强制申报冲突与省略 + 人工降为裁定标记行） | §7.7 忠实性验证的原子声明法、§2.6 单一表示、§10.3 不可信数据 |
| 沙箱从部署配置改为运行时参数（放置下移到调用点 + 预热复用 + 基础镜像预分发 + 快照可复用可分叉） | §4.4 Tool 输出规则、§1.3 环境真实反馈、§5.4 Workspace 与 Sandbox 状态 |
| 读好扩写难扩（加副本拖慢写 + 持久化机制即扩展机制 + 最小协调面 + 存储计算解耦 + 维护移出服务路径） | §5.2 Checkpoint 与 Store 分离、§5.4 执行账本 |
| 窄任务小模型 + 两段式判真伪（模型大小≠有用 + 正则保召回小模型读上下文判真伪 + 难负例靠学习不靠规则 + 不替代确定性扫描 + 报完整对照含败项） | §7.9 模型分层路由、§4.1 Tool 是契约 |
| 大规模改造方法论（先分类再建证书最后动代码 + 三类改造标准不同 + 证书=机器可检查完成条件 + 源头修复 + 风险降低而非降本） | §12.3.1 大规模改造、§7.6 分层验证 |
| 端到端与步骤级双评（调用准确必要不充分 + 结局做门禁过程做定位 + 报区间不报点 + 基准不可比 + 污染含联网搜答案） | §7.7 评测 Harness、§7.2 必备 Grader、§7.10 飞轮 |
| 内部先跑再对外 + 权限等于用户可见范围（工具按用户注册 + 中间件挂钩 + 离线线上双层评测 + CI 阈值阻断） | §6 审批与权限、§3.14 代码编排、§7.5 自动冒烟 |
| 决策模型输出受限结构（非自回归并行打分 + 校准损失 + 边缘部署 + 诚实披露败项） | §7.9 模型分层路由、§2.3 Context Budget |
| 自我复制能力水位与防御动作（监控异常载荷与设备间流量 + 消除未认证服务 + 带通知的清理 + 蜜罐取 harness；作者自陈是成功率上界） | §10.1 沙箱边界、§9.1 默认可观测、§10.5 OWASP Agentic |
| 历史纳入缓存 + 降裁剪频率 + 工具变更用允许列表 + 预热 + 缓存诊断（只缓固定前缀会让逐轮增长的历史每轮全价重算） | §2.10 上下文成本与稳定前缀、§9.6 成本可观测 |
| 按业务成熟度分预算 + 三档模型默认分工 + 选型与限额双工具 + 收紧高成本模式默认开关 | §9.6 成本可观测与预算控制、§7.9 模型分层路由 |
| 外层 harness 兜住提前停下 + 消除轮询式零价值消耗 + 受控实验需先重构代码 | §1.6 Loop 规则、§5.3 可恢复 Workflow、§7.6 分层验证 |
| 技能描述当路由判据写 + 负例修复触发率 + 描述膨胀被截短 + 模板放 Skill 内 | §14.2 渐进式加载、§13.5 AGENTS.md 写法 |
| AGENTS.md 写"何时相关" + 清晰边界替代"永远先问" + 完成定义写死 | §13.5 模块 AGENTS.md、§13.2 分层规则 |
| Skill 触发评测（四类 case 含负控制 + 确定性 grader 解析结构化输出 + 同 harness 扩展查横跳与预算） | §7.7 强 Agent 评测 Harness、§7.2 必备 Grader |
| notebook 目标单元格 + 浏览器侧操作注册成工具 + 决策先落盘再消散 | §16 项目应内置能力、§2.5 ArtifactRef |
| harness 重设计把 ARC-AGI-3 从 13.3% 提到 38.3%（输出 token 降六分之一）+ 比模型须固定 harness | §1.2 Harness 设计、§1.3 环境真实反馈 |
| AI 评审七层防线 + 风险分级审批绑 commit + 事故即评测集 + 评审收敛（59% 为存量代码、轮次上限） | §7.6 Agent 产出量超过人阅读量时的评审流水线、§6.1 审批 |
| CUJ 定义评测点 + 红笔读 trace + judge 用人工标注校准 + CI 全局阈值与零回归双规则 | §7.1 Eval 开发要求、§7.3 Gate 设计 |
| 外置优于截断 + 压缩须可恢复 + 尖峰与缓升分开处理 + 最近消息逐字保留 + 记忆自持 | §2.4 上下文压缩、§2.5 上下文数据生命周期 |
| 骨架先行 + 主 Agent 只读 + 看不懂即止损重做 + 一次做大优于小步迭代 | §1.2 骨架先行与执行主 Agent 只读、§12.4 隔离工作区 |
| `completed` 只代表程序跑完 + 累计数与并发数不同约束 + 预算非硬开关 + 上下文隔离≠文件隔离 | §1.6 Loop 规则、§3.3.1 子 Agent 运行时协作契约 |
| 红线阈值脱离脚本默认值 + 变异测试度量断言有效性且只跑增量 + 状态只能经命令推进 | §7.6 分层验证与假修复检测、§5.11 Loop State |
| 安全边界外置到 Harness + 知识库检索结果不可信 + 参数缺省即拒发 + 门禁脚本写单测 + 虚拟需求空跑 | §10.1 沙箱边界、§10.3 不可信数据规则、§6.4 对外通信、§7.3 Gate 设计 |
| MCP 工具描述哈希锁定 + 按声明授权而非意图推断 + 成本包络触顶交人 + 哨兵只吹哨 | §11.2 MCP 安全基线、§9.6 成本可观测与预算控制 |
| 判定机制可得性决定重构可行性 + 逐帧回放比对状态哈希 + 转换方案与失败记录手册化 | §12.3.1 大规模改造、§7.2 必备 Grader |
| 架构约束须写成可衡量 QAR + 只给功能需求不保证架构 + 编码变简单而写需求变难 | §12.3.1 大规模改造、§12.3 Planner/Generator/Evaluator |
| 视觉软约束/数据认知约束/生码硬约束装配同一 DOM + Skill 正文压缩后显式恢复 + 缓存未命中四类归因 + 交接带记录地址 | §2.4 上下文压缩、§2.10 上下文成本与稳定前缀、§2.8 阶段上下文包 |
| 行为差异替代 ground truth + 用测试时计算换成功率 + 维护 Harness 成本超过使用收益 | §7.10 自进化纪律、§9.5 Harness 熵 |
| 隔离须逐层验证含存储分配器 + "分配时清零"显式配置 | §10.1 沙箱边界 |

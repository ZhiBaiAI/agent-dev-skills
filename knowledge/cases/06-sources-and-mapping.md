# Agent 经验 · 来源与规范映射

> **范围：** §11 未找到可靠来源的方向、§12 关键来源索引、§13 经验到规范的映射
> 集合：`cases/` ｜ 导航：[知识库索引](../README.md)

## 11. 未找到 2026 年可靠来源的方向

按"宁缺毋滥"原则，以下方向未找到满足筛选标准的 2026 年来源，明确标注：

- **Shopify / Netflix / LinkedIn / Vercel 2026 年 Agent 生产文章**：多次站点限定搜索均无 2026 年结果（Google 已于 2026-09 收录，见 #60、#61）。
- **会议分享（ICML/NeurIPS/QCon/GopherCon 2026 有公开 slides/录像的 Agent 经验）**：未找到 2026 年内已公开且符合"真实问题+解决"的会议分享。
- **GitHub Blog《agents.md lessons from 2500 repos》原文**：被广泛引用但官方 URL 未能抓取验证（404），未采纳。

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
| 补丁续命时间指数衰减，重设计工期按「人周」估而非按历史经验估 | §1.2 Harness 设计、§8.1 生产上线指标 |
| 关键状态从第一版就外置（单写者进程内状态是水平分片的根因） | §5.1 状态分层、§5.3 可恢复 Workflow |
| 服务指标做 agent 的眼睛与耳朵（进出对账让 agent 自己爬坡，比人盯快） | §7.1 Eval 开发要求、§8.1 生产上线指标 |
| 协调 swarm 与独立并行互补而非替代（266 vs 21 仅重叠 12，限定范围后 token 效率相当） | §3.1 范式选择、§3.3 多 Agent 准入 |
| 提示词干预（规定角色/CEO 层级）不改变多 agent 协调结果 | §3.9 协调者与专家模式 |
| 协调能力随模型代际变化（老模型冲突后放弃、中间代际各占文件、最新代际高共享高吞吐） | §9.4 模型与运行时升级 |
| 低方差一致性失败（同源同上下文 agent 互相审查不构成交叉验证） | §3.9 协调者与专家模式、§7.4 证据化置信度 |
| 缓存命中率是架构属性（前缀字节级稳定 + 波动值外置 + 命中率可诊断对账） | §2.10 上下文成本与稳定前缀、§9.6 成本可观测 |

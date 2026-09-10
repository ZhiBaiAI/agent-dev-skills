# 实践指南 · 成本效率与团队落地

> **范围：** §7 成本与效率、§8 常见反模式与对策、§9 团队落地路径、§10 参考来源
> 集合：`practices/` ｜ 导航：[知识库索引](../README.md)

## 7. 成本与效率

- **子 agent 默认降档：** 定义明确的子任务用低成本模型（主模型/协调者用强模型兜底），是单项影响最大的成本杠杆；允许手动覆盖。（lessons §8.5, §3.9）

- **code-mode 批量编排：** 轮询、批量重复操作放子进程脚本只回摘要，多次模型往返压缩为一次（省 50-90%）。（标准 §3.14；lessons §8.5）

- **上下文瘦身：** MCP 工具 schema 按需加载不预载；长会话及时压缩；稳定内容做缓存前缀。

- **缓存分层：** 跨会话稳定的内容（system prompt、工具定义）做全局缓存前缀，会话内稳定内容做会话级前缀，易变内容放末尾——稳定内容最大化缓存命中。（lessons §10.11）

- **优化全任务成本而非工具调用成本：** 工具调用便宜但任务没完成，已花的 token 全是浪费——成本指标以"完成任务"为分母；prompt 压缩是行为变更，改动手动回归核心场景。（lessons §8.6）

- **成本可见：** 会话/任务成本实时可见，异常时能定位到反模式并给出针对性修复（反模式 + 财务影响 + 修复动作三元组）。（lessons §8.5）

- **留痕零开销：** 可解释性记录用开关控制（默认关、nil 降级、按需全量开），"需要时能查"与"平时不拖累"同时成立。（lessons §10.9）

---

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

---

## 9. 团队落地路径

1. **从最小基线开始：** 先跑一个真实、简单、可验收的任务（含目标/范围/约束/完成条件）建基线，不先加多 Agent、长 Prompt、复杂工作流；失败后沿执行链定位问题最早出现在哪一层，不先改 Prompt。（lessons §10.8）
2. **观测驱动生长：** 重复失败→加证据门禁；无法恢复→加状态存档；目录污染→加工作区隔离——每个组件都有明确触发它的那次失败。（lessons §10.10）
3. **老系统冷启动：** submodule 关联代码仓、旧文档收进 raw/（原始事实，不当已确认规则）、AI 听记采访核心成员整理入库。（lessons §10.10）
4. **预判 Harness 收缩：** 文件搜索、代码理解会被模型和 Coding Agent 产品逐步吸收；不会消失的是四样——业务知识及事实治理、项目规则与决策边界、领域工具适配、验证标准与质量责任。角色向 FDE（深入现场连接问题、知识、系统、交付）演进。（lessons §10.10）
5. **共识边界：** 团队间需要统一的是数据协议与状态/证据/安全/度量口径，不是统一的流程实现。（lessons §10.10）
6. **每周交付+持续项目底盘：** 保持每周用新模型 ship 点东西、保持一个持续迭代的小项目作为测试每个新模型的底盘——模型不是可互换的（形状差异真实存在），选型直觉来自大量亲手使用而非 benchmark 表格。（lessons §10.13）

---

## 10. 参考来源

本指南提炼自以下案例，案例集合见 [`cases/`](../README.md)（下称 lessons，章节号即下表对应案例），规范条目见 [`standards/`](../README.md)（下称标准）；集合导航见 [知识库索引](../README.md)。

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

> **别名：** 本文中用 `lessons` 指 `knowledge/cases/`（按 §号取主题文件），用 `标准` 指 `knowledge/standards/`；文中（lessons §x.x / 标准 §x.x）按章节号回对应主题文件查细节。

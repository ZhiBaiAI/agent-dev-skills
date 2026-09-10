# Agent 工程规范 · Review 与硬规则

> **范围：** §18 架构 Review Checklist、§19 七十八条硬规则
> 集合：`standards/` ｜ 导航：[知识库索引](../README.md)

# 18. 架构 Review Checklist

## 18.1 Agent 准入

```text
□ 为什么需要 Agent？
□ 单次结构化调用是否足够？
□ 普通 Workflow 是否足够？
□ Agent 自主决策的边界是什么？
□ Identity、Orchestration、Context、Gate、Recovery、Evolution 由谁负责？
□ Agent 如何获得环境真值？
□ 停止和失败条件是什么？
```

## 18.2 Harness 与规则执行

```text
□ P0/P1 规则是否具有机器执行机制？
□ Hook 是否定义超时、顺序、幂等和 Fail 策略？
□ 规则冲突、过期和降级是否可检测？
□ Context 是否按读取时机分层？
□ 控制平面、计算平面和协作平面是否分离？
□ Harness 配置是否作为版本化被测对象？
□ Evaluator 是否保持只评估、不补做缺失步骤？
□ 缺失数据和执行异常是否响亮失败？
□ GT 是否与被测 Agent 输入隔离？
□ 硬指标是否由确定性 Grader 计算？
□ Judge 是否经过校准、盲测和人工抽检？
□ Headless Runner 是否显式管理认证、目录、stdin 和子进程？
□ 是否定期清理重复规则、失效 Skill 和文档漂移？
```

## 18.3 Runtime Protocol

```text
□ Thread、Run、Step、Event、Artifact、Checkpoint 边界是否明确？
□ 每个事件、错误、审批和产物是否归属具体 Run？
□ Run 状态机是否支持取消、输入、审批、恢复和失败？
□ 同一 Thread 的并发 Run 采用哪种策略？
□ Checkpoint 是否版本化并有迁移测试？
□ Interrupt/Resume 是否持久化、幂等并校验权限？
□ Event Stream 是否支持断线续传和消费者去重？
□ Workspace 是否具有 Revision、权限和生命周期？
□ Tool 错误和系统错误是否使用不同处理边界？
□ Runtime Adapter 是否通过协议一致性测试？
```

## 18.4 成本与上下文效率

```text
□ 成本能否按 Run、Wave、Agent、模型、上下文来源和 Tool 拆分？
□ 是否存在重复加载、无效加载和未使用上下文？
□ 稳定前缀是否与动态数据分离？
□ 共享外部数据是否由上游采集一次并生成 Projection？
□ 长知识和代码是否先查索引？
□ 小任务是否走单 Agent 或 Fast Path？
□ 多 Agent 拆分是否有质量与单位成功成本证据？
□ Agent 是否只看到所需 Tool 和 MCP Server？
□ 确定性环境操作是否使用 CLI 或 Script？
□ 无依赖调用是否批量并行？
□ 模型分层是否经过 Eval 并有升级条件？
```

## 18.5 Tool

```text
□ Tool 是否面向任务语义？
□ 是否与其他 Tool 功能重叠？
□ 输入输出是否有界？
□ 是否声明副作用？
□ 是否幂等？
□ 是否需要审批？
□ 是否有选择和参数 Eval？
```

## 18.6 Context

```text
□ 各上下文来源是什么？
□ 哪些是不可信数据？
□ Token 预算和输出预留是什么？
□ 大结果如何外置和按需取回？
□ 是否执行单一表示检查？
□ 压缩后如何追溯？
□ 长期事实如何验证和过期？
□ 领域知识、行为规范和工作流配置是否分开？
□ 知识条目是否有来源、Owner、版本和有效期？
```

## 18.7 Workflow

```text
□ 业务状态是否在代码和数据库中？
□ 外部调用是否在事务外？
□ Task 是否可重试和幂等？
□ 中断后是否可能重复执行？
□ 是否需要 Checkpoint？
□ 是否同时支持链路和单点执行？
□ 流程重量是否匹配任务影响？
□ 执行账本是否独立于传输连接？
□ 并行步骤是否无依赖并隔离写入？
□ 嵌套执行是否有深度、链路和环检测？
```

## 18.8 Security

```text
□ 文件系统边界是什么？
□ 网络出口边界是什么？
□ 凭证作用域是什么？
□ 外部内容能否影响权限？
□ MCP/Skill/依赖来源是否可信？
□ 是否有审计？
□ 是否做过 Prompt 注入和数据外泄测试？
```

## 18.9 Evals

```text
□ 成功标准是否可执行？
□ 是否测试结果和轨迹？
□ 是否运行多 Trial？
□ 是否包含回归用例？
□ 是否衡量成本、耗时和步数？
□ Grader 是否经过人工校准？
□ 置信度是否包含证据覆盖和确定性检查？
□ 冒烟失败是否区分能力、基础设施和测试缺陷？
□ 阻断 Gate 是否优先使用确定性检查？
□ 生成与评估是否需要独立上下文或角色？
```

## 18.10 Production Quality 与 Experience

```text
□ 是否只看单次结果或平均分？
□ 是否报告首次完成率、All-pass@k 和质量下限？
□ 是否计算每个成功任务的综合成本？
□ Trace 是否经过脱敏和 Trajectory 标准化？
□ 候选经验是否来自多个已评估 Trajectory？
□ 经验是否有范围、版本、证据、过期和回滚？
□ 经验召回是否先做权限和适用范围过滤？
□ 经验是否可能绕过安全、审批和业务规则？
□ 是否通过对照 Eval 验证质量和成本收益？
□ 模型、Tool 或规则变化后是否触发重新评估？
```

## 18.11 Loop Engineering

```text
□ 任务是否重复、可触发且有明确 Owner？
□ 结果是否可自动验证？
□ 是否具有受控 Connector 和所需工具？
□ 是否使用持久 Cursor、Watermark 和去重键？
□ 执行是否隔离、可取消、可回滚？
□ 自动重试、总耗时和总成本是否有上限？
□ 验证器能否识别隐藏错误信号的假修复？
□ 生产写入和发布是否有审批策略？
□ 是否提供 Kill Switch 和建议模式降级？
□ 是否区分执行时间和人工等待时间？
□ 是否跟踪复发率、误报率和单位已验证结果成本？
□ Loop 是否通过版本化 Contract 和回归测试发布？
```

---

---

# 19. 七十八条硬规则

1. 已知业务规则使用确定性代码。
2. 使用能完成任务的最低 Agent 复杂度。
3. 新增 Agent、框架、多 Agent、耐久执行或自治 Loop 必须有 ADR。
4. 每个项目明确 Identity、Orchestration、Context、Gate、Recovery、Evolution 的责任。
5. Harness 组件可替换、可关闭、可评测。
6. P0 和 P1 规则具有稳定 ID、版本和机器执行机制。
7. 安全、权限、状态写入和不可逆动作采用 Fail Closed。
8. 高频违规优先转化为 Schema、Binding、Linter、Hook 或 Gate。
9. 单次失败不能直接升级为全局硬规则。
10. Context 按 Core、Scope、Phase、On-demand 和 Artifact 分层加载。
11. Required Read 和 Context Manifest 可记录、可 Diff、可评测。
12. 成本按 Run、Wave、Agent、模型、上下文来源和 Tool 归因。
13. 稳定指令位于动态任务数据之前，并记录前缀版本和 Hash。
14. 共享外部数据由上游采集一次，下游使用 Projection 或 Artifact。
15. 知识和代码检索先查版本化索引，必要时回退源搜索。
16. Runtime Protocol 与具体框架 Adapter 分离。
17. Agent、Thread、Run、Step、Event、Artifact 和 Checkpoint 具有稳定 ID。
18. 每个 Event、Artifact、Error 和 Approval 归属具体 Run。
19. Run 状态机支持取消、输入、审批、恢复、失败和完成。
20. Thread 与 Run 分离，并明确同一 Thread 的并发 Run 策略。
21. Checkpoint 具有 Schema 版本、Runtime 版本、迁移和兼容窗口。
22. Interrupt/Resume 持久化、幂等并校验权限与状态版本。
23. Event Stream 支持事件持久化、Cursor 续传和消费者去重。
24. Workspace 具有权限、Revision、生命周期和 Run 归属。
25. Runtime Adapter 必须通过协议一致性测试。
26. 所有模型调用经过 Model Gateway。
27. 模型分层路由具有能力门槛、预算、升级条件和 Eval。
28. Agent 具有输入输出 Schema、停止条件、预算和 Eval。
29. Tool 和 Connector 具有 Schema、权限、风险、超时、幂等和审计。
30. Agent 只加载当前任务所需的 Tool 和 MCP Server。
31. Tool 使用共享 Error Contract。
32. 可恢复 Tool 错误作为结构化数据；系统和安全错误由 Runtime 处理。
33. Connector 读取与写入权限分离。
34. 有副作用 Tool 需要审批或明确策略授权。
35. 对外通信和用户数据写入使用专用 Tool、确定性校验和交付证据。
36. 外部内容和 Tool 输出按不可信数据处理。
37. 精确 ID、数组和大数据通过 State、Binding 或 ArtifactRef 传递。
38. 同一来源在单次模型调用中只保留一种表示。
39. 模型调用前执行 Context 预算预检并预留输出空间。
40. Compaction 保留目标、进度、具体值、失败路径、审批和引用。
41. 阶段通过自包含 Contract 或 Handoff Artifact 交接。
42. 模型摘要不能替代规范字段和精确数据。
43. 领域知识、行为规范、工作流配置、能力和经验分开管理。
44. 知识条目具有来源、Owner、版本、范围和有效状态。
45. 自动抽取和用户纠正生成候选知识，审核后发布。
46. 知识变更执行影响分析、回归 Eval 和版本回滚。
47. 冲突或疑似过时知识不能用于高风险自动决策。
48. 步骤状态通过结构化控制表达。
49. Capability 按任务、权限和风险动态裁剪。
50. 确定性环境操作、批处理和测试优先使用版本化 CLI 或 Script。
51. MCP 用于动态能力和资源接入，并执行 Tool 白名单与结果裁剪。
52. Tool Schema 和 Tool Result Token 必须度量。
53. 确定性输出压缩保留错误、退出码、统计和原始引用。
54. Loop 承载方式与编排协议分开选择。
55. 控制平面拥有状态和阻断权；计算平面返回 Contract 与 Artifact。
56. 链路模式与单点模式复用相同能力和 Policy。
57. 流程重量匹配任务影响、可逆性和风险。
58. 小任务使用单 Agent 或 Fast Path；多 Agent 拆分需要收益证据。
59. 并行节点必须无数据依赖并隔离写入。
60. 无依赖 Tool、Connector、Subrun 和测试任务应批量并行。
61. 并行代码修改使用独立分支和工作区。
62. 多 Agent 需要明确准入条件、所有权和隔离。
63. 协调者不能重复 Specialist 的核心职责。
64. 嵌套 Agent、Workflow 和 Skill 执行深度、链路和环检测。
65. Session、Checkpoint、Memory、行为资产、Loop State 和业务状态分离。
66. 执行账本保存可恢复状态；传输层负责事件交付。
67. 错误映射到 Retry、Fallback、Rollback、Need Input 或 Abort。
68. 可恢复 Workflow 的副作用必须封装、可序列化和幂等。
69. 阻断 Gate 固化到 Workflow 或 CI。
70. 确定性检查优先于模型 Grader。
71. Harness 配置、规则、Skills、Workflow、Judge 和 Adapter 作为版本化被测对象。
72. Evaluator 只评估，不补做被测流程遗漏的步骤。
73. 缺失 Case、空产物、超时和依赖失败必须响亮失败。
74. GT、Judge 答案和候选版本标签不能进入被测 Agent 输入。
75. 精确指标由确定性 Grader 计算；LLM Judge 用于语义质量并经过校准。
76. 自报结果与独立环境结果必须比较并记录 Honesty Gap。
77. 成本优化在等价质量与安全门槛下通过多 Trial、消融或确定性对比验证。
78. AGENTS.md 保持简洁、分层、具体、可执行；Skill、Pack、Runtime Adapter、Judge 和 Loop Contract 具有版本与一致性测试。

---

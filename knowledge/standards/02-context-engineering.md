# Agent 工程规范 · Context Engineering

> **范围：** §2 上下文组成、信任分级、预算、压缩、数据生命周期、单一表示、Prompt 预算预检、阶段交接、分层加载、成本与稳定前缀、上游一次采集、索引优先检索
> 集合：`standards/` ｜ 导航：[知识库索引](../README.md)

# 2. Context Engineering

## 2.1 上下文组成

现代 Agent 的上下文包括：System Instructions、Tools、User Input、Session History、Business State、Retrieved Data、Files、Memory、Intermediate Results、Plan、Budget、Environment Feedback。

- **MUST** 将 Prompt 管理升级为 Context 管理。
- **MUST** 明确每一类上下文的来源、信任等级、保留时间和裁剪规则。
- **MUST NOT** 将所有历史和工具结果无选择地持续追加。
- **SHOULD** 使用最小、高信号上下文。
- **MAY** 对长历史进行摘要或压缩，但摘要必须可追溯到原始记录。

## 2.2 上下文信任分级

| 等级 | 示例 | 规则 |
|---|---|---|
| SYSTEM | 核心安全规则 | 最高优先级，不被下级覆盖 |
| ORGANIZATION | 组织策略 | 仅管理员发布 |
| APPLICATION | 项目 Prompt、业务规则 | 版本化 |
| USER | 用户输入 | 不能改变系统权限 |
| RETRIEVED | 知识库数据 | 需来源和时间 |
| EXTERNAL_UNTRUSTED | 网页、邮件、Tool 返回 | 按不可信数据处理 |

- **MUST** 重要结论绑定运行现场：结论本身之外须同时记录 claim / source / revision / scope / observed_at / verification——"某处已存在"（本地代码含修复）不等于"当前已生效"（实际消费的仍是旧版本），判断以运行现场实际消费的 revision/receipt 为准，不以代码现场代替运行现场。

## 2.3 Context Budget

每个 Agent 应定义上下文预算：最大输入 Token、输出/工具定义/系统指令预留、历史策略、单结果最大 Token、检索文档上限。

- **MUST** 预留输出和工具定义预算。
- **MUST** 记录被裁剪的上下文类型和数量。
- **SHOULD** 优先丢弃低相关、重复和可重新获取的内容。
- **SHOULD** 避免将超大 JSON、HTML 或日志原样放入上下文。

## 2.4 上下文压缩

压缩前后都应可审计。约束：

- 用摘要替代不可恢复的关键业务事实。
- 将未经确认的模型推断写入长期事实。
- 把安全策略压缩成模糊描述。
- 压缩后丢失 Tool 副作用、审批或错误状态。

## 2.5 上下文数据生命周期

```text
Tool 结果 → 分类 → Inline 或 ArtifactRef → 有界摘要 → 索引 → 按需检查 → 过期或归档
```

- **MUST** 使用 ArtifactRef、State 或参数绑定传递精确数据。
- **MUST** 保留来源、Hash、Schema、权限和过期策略。
- **MUST** 标记摘要、截断和不完整结果。
- **MUST NOT** 让模型复制 UUID、主键、完整数组或大段结构化数据。
- **SHOULD** 提供 `outline`、`search`、`context`、`head`、`tail` 等局部检查接口。
- **MUST** 对超过阈值（如 8000 字符 / 10 个元素数组）的工具结果强制外置存储——LLM 搬运大数组时会"无意识摘要"只传 3-5 个"代表性"元素，强制外置从根本消除这一数据丢失模式。
- **MUST** Agent 表现随步骤数劣化时先诊断上下文质量（统计噪音占比），再考虑换更大模型——实测换模型两周指标不动，上下文管理一周提升 40%；"物理容量"不等于"有效容量"。
- **MUST** 跨步骤数据传递走运行时参数绑定注入，禁止让模型充当"数据搬运工"（模型搬运 UUID 会截断、混淆、幻觉）。

## 2.6 单一表示与结构化 Compaction

同一来源在一次模型调用中只保留一种表示。禁止同一数据同时出现：完整结果+摘要、摘要+预览、完整结果+Assistant 复述、多个不同截断版本。

Compaction 生成结构化交接，保留：用户目标、当前计划、已完成步骤、已验证事实、放弃路径、阻塞、待审批、Artifact 引用、剩余预算。

- **MUST** 保留用户目标、具体 ID、已完成步骤、失败路径、阻塞、审批和引用。
- **MUST** 保持 Tool Call 与 Tool Result 的协议配对。
- **MUST NOT** 使用模型改写规范字段、精确值、Hash、ID 或匹配前缀。
- **MUST** 在 Prompt 组装阶段做单一表示的编译时检查——检测到同一数据以多种形态进入待组装 prompt 时拒绝组装并告警，而非运行后祈祷；摘要/预览类字段必须用原始文本 substring 生成，不经任何模型改写（模型重写会改变字段名与前缀，导致下游前缀匹配恢复失败）。

## 2.7 Prompt 预算预检与编译

每次模型调用前执行预算预检：`模型窗口 - 输出预留 - Tool定义 - 系统与组织规则 - 当前任务目标 = 可分配上下文预算`。

超预算时按固定优先级收缩：调试预览 → 可重新获取的完整数据 → 低相关摘要 → 非直接依赖 → Transcript → Working Memory Insights。任务目标、安全规则、审批状态和当前步骤保留到最后。

- **MUST** 使用模型相关 Tokenizer 或保守估算。
- **MUST** 在步骤开始和最终输出前检查预算。
- **MUST** 在降级后仍超限时返回 `CONTEXT_BUDGET_EXCEEDED`。
- **SHOULD** 用 Prompt Compiler 组装版本化结构块。

推荐 Prompt 结构：

```text
Stable Prefix: Platform Rules → Organization Policy → Project Rules → Agent Contract → Stable Tool Definitions → Output Contract
Dynamic Suffix: Current Objective → Phase Context → Selected Artifacts → Working State → Latest User Input
```

## 2.8 阶段上下文包与交接产物

每个阶段接收自包含的上下文包（目标、验收条件、输入引用、已验证事实、约束、决策、开放问题、输出 Schema、允许能力、预算），阶段结束后输出 Handoff Artifact（状态、决策、输出引用、验证结果、放弃路径、开放问题、下一阶段）。

- **MUST** 通过结构化产物交接关键状态。
- **MUST** 让下游依赖公开 Contract，避免依赖上游对话历史。
- **MUST** 将临时讨论、草稿和已放弃方案与已验证结论分开。
- **SHOULD** 按阶段渐进加载规则、Skills 和资料。
- **SHOULD** 跨团队协作先产出共同任务书再开会：会前各方基于本方实现整理接口说明（各自负责什么、出问题如何处理、怎样算成功），会上只核对差异，会后写成共同版本交各方 Agent 执行。任务书至少回答：谁最后得到什么结果、谁来解释关键规则、谁能真正改变结果、人和 Agent 现在可以做什么、什么时候必须停下或找人处理、什么证据能证明已完成。

## 2.9 分层上下文加载

| 层 | 内容 | 加载策略 |
|---|---|---|
| Core | 角色、P0/P1 摘要、当前目标、停止条件 | 每次 Run |
| Scoped Rules | 当前目录、组件和技术栈规则 | 按项目范围 |
| Phase Context | 当前阶段 Contract、Checklist、输入产物 | 进入阶段 |
| On-demand Reference | 专项规范、示例、历史证据 | 触发时 |
| Artifact Data | 大结果、完整日志、文件和数据集 | 按需局部读取 |

- **MUST** 按读取时机和任务范围加载规则。
- **MUST** 为强制阶段资料维护 Required Read 清单。
- **MUST** 在阶段结束后释放无后续依赖的上下文。
- **MUST NOT** 将全流程手册长期常驻模型上下文。

## 2.10 上下文成本与稳定前缀

- **MUST** 按 Agent、Run、Step、Wave 和上下文来源统计 Token。
- **MUST** 区分输入、输出、缓存读取、Tool Schema 和 Tool Result。
- **MUST** 识别重复内容、无效加载和未被使用的上下文。
- **MUST** 将动态内容放在稳定指令之后。
- **MUST NOT** 为缓存命中复制无关内容。
- **SHOULD** 保持系统规则、项目规则、Agent 定义和 Tool Schema 的顺序稳定。

## 2.11 上游一次采集与下游复用

同一事实源在链路上采集一次：`Source Connector → Raw Artifact → Structured Projection → Verified Summary → Downstream Reference`。

- **MUST** 保存原始数据为受控 Artifact。
- **MUST** 将下游所需字段转换为结构化 Projection。
- **MUST NOT** 让多个 Agent 分别获取同一外部数据。
- **MUST NOT** 将完整外部 Payload 注入长生命周期协调 Agent。

## 2.12 索引优先检索

知识、代码和历史产物先查询轻量索引，再读取正文：`Query → Index Search → Candidate Ranking → Top-K Metadata → Selected Content Read → Fallback Search`。

- **MUST** 将索引与源文件版本关联。
- **MUST** 在索引缺失、过期或低置信时回退到源搜索。
- **MUST** 只读取排序后的少量候选正文。
- **SHOULD** 对大型代码库使用 AST、符号索引、依赖图或代码图谱缩小文件范围。
- **SHOULD** 知识载体选型以更新路径长短为判据：知识、文档、测试和索引进同一个 MR 一起评审、合并后共享同一版本（文件 + Git 优先）；RAG/图谱作检索补充不作主知识源——它们在代码变化后要走同步、切片、萃取、索引/图谱发布的长链路，更新延迟即知识失效窗口。
- **SHOULD** 知识库只存"代码之外影响业务和技术判断的信息"（业务规则与口径、跨系统链路、新旧切换、废弃状态、运行态拓扑），不复制代码可表达的内容——逐方法解释的文档生成快失效也快，会变成与代码竞争的实现说明；每份知识文件带 Metadata（至少 status/version/source），Agent 读正文前先判适用性，事实来源（source）指向原始材料而非派生结论。

---

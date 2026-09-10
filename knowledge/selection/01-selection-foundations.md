# 选型参考 · 基础维度

> **范围：** §1 工具栈三层抽象、§2 确定性 vs 自主性谱系、§3 复杂度阶梯
> 集合：`selection/` ｜ 导航：[知识库索引](../README.md)

## 1. 工具栈的三层抽象

业界主流 Agent 工具栈已收敛为三层抽象，控制力与抽象力此消彼长：

```text
控制力高 / 抽象低                              控制力低 / 抽象高
        Runtime  →  Framework  →  Harness
       (运行时)     (框架)        (脚手架)
```

| 层 | 职责 | 典型内容 | 何时选 |
|---|---|---|---|
| Runtime | 图执行引擎、耐久执行、状态机、HITL、容错、流式 | LangGraph、LangGraph 协议兼容实现 | 流程形状本身就是价值；确定性步骤多；需细粒度控制每一步 |
| Framework | 抽象与集成层、最小 agent loop、工具/模型/检索集成、middleware 钩子 | LangChain、Mastra、AI SDK | 要自己组装 harness；延迟敏感；需要精细控制每步的 tool 与 context |
| Harness | 开箱即用的 agent 脚手架，内置 context 工程最佳实践 | Deep Agents、Claude Code 式 harness | 大多数项目起点；要 context 管理、subagent、skills、memory 开箱即用 |

### 关键判据

- **harness 职责 = 在正确时间把正确 context 给到模型**。判断一个 harness 好不好，看它如何管理 context，而不是看它有多少功能。
- **三层完全可组合**：可以把 framework 的 `create_agent` 嵌进 runtime 的 graph，也可以把 runtime 的 graph 作为 subagent 嵌进 harness。不要把三层看作三选一。
- **核心 agent loop 标准化趋势**：随着模型变强，「模型规划 + 调用工具 + 反应结果」的 loop 已足够强大，可以标准化。harness 层因此出现并收敛最佳实践。

---

---

## 2. 确定性 vs 自主性谱系

三层抽象对应自主性谱系的不同位置。选型的本质是在这条谱系上定位：

```text
高确定性 / 低自主                              低确定性 / 高自主
   Runtime 编码拓扑  →  Framework agent loop  →  Harness 长时 fan-out
```

| 谱系位置 | 特征 | 适合 |
|---|---|---|
| 高确定性 | 业务状态编入图拓扑；模型只做单点抽取/判断 | 文档处理流水线、审批工作流、ETL |
| 中间态 | agent loop 主导；middleware 注入审批/合规/业务规则 | RAG 问答、Copilot、工具型助手 |
| 高自主 | 长时运行、subagent fan-out、内置 summarization/context 管理 | 深度研究、代码 Agent、开放式探索 |

### 判据

- **自主度上界 = 能廉价且可靠验证的量**。验证不了的就不要放给 agent 自主决策。
- **敏感/预设/可重复的任务用确定性**；动态/创造性/探索性任务用自主性。
- **middleware 是中间态的调节阀**：在核心 loop 上挂钩子注入确定性步骤（审批、合规、业务规则），不必上 runtime 全图。
- **runtime 是 escape hatch**：当 middleware 的内置钩子不够灵活时，才下沉到全自定义 graph。

---

---

## 3. 复杂度阶梯

不论选哪层抽象，都应优先选择能完成需求的最低复杂度：

```text
确定性函数 → 单次模型调用 → 结构化调用 → Tool Loop → 显式 Workflow → 耐久 Workflow → 多 Agent
```

### 选型判断

| 需求特征 | 推荐起点 |
|---|---|
| 输入输出明确、规则固定 | 确定性函数 |
| 单点语义抽取/分类/摘要 | 单次或结构化模型调用 |
| 下一步依赖中间结果、工具选择不能预定 | Tool Loop（framework 层） |
| 固定业务状态机、需事务/审批/审计 | 显式 Workflow（runtime 层） |
| 跨天恢复、人工中断、严格重放 | 耐久 Workflow（runtime 层） |
| 子任务可并行、上下文/权限需隔离 | 多 Agent（harness 层 subagent） |

**反模式：** 同步任务上耐久 Workflow；单轮抽取上多 Agent；「让 Agent 聪明一点」就上向量库。先验证需求，再上抽象。

---

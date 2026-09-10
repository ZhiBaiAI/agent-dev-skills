# AGENTS.md — agent-dev-skills 仓库开发规则

## 项目定位

本仓库是 **agent-dev-skills** —— 一个面向 Agent 项目开发的技能驱动（skill-driven）工具集。七个技能覆盖 Agent 开发生命周期（设计 → 选型 → 脚手架 → 评测 → 评审），由确定性纪律技能（安全、复杂度）支撑，底层是共享、版本化的知识库与官方脚手架 CLI。

我们**不**构建 Agent 框架、CLI 运行时或任何代码生成管道。我们提供**技能、知识、标准与实践**。确定性从生成转移到验证：生成的项目必须先通过自己的机器可检查成功标准，然后才交付。

## 仓库结构

- `skills/`：七个技能——五个用户调用的编排技能（`agent-design`、`agent-stack`、`agent-scaffold`、`agent-eval`、`agent-review`）和两个模型调用的纪律技能（`security-gates`、`complexity-ladder`）。
- `knowledge/`：共享、版本化的参考知识库，技能按触发加载（案例、规则、选型、实践、标准 + `index.yaml`）。
- `docs/`：文档与架构决策（见 `docs/maintainers/decisions/`）。

修改模块前先读最近的嵌套 `AGENTS.md`（如有）。

## 工作规则

- **确定性靠验证。**确定性保证住在生成项目自己的门槛里（typecheck、测试、安全清单、退出码 0），不在任何生成管道里。技能把增量编码为强制工作流步骤。
- **源码属于项目。**脚手架输出的一切都可读、自有、可删除；生成项目在运行时绝不依赖本仓库。
- **复杂度阶梯。**优先 确定性函数 → 单次模型调用 → 结构化调用 → 工具循环 → 显式工作流 → 持久化工作流 → 多智能体。不要默认重架构（见 `complexity-ladder`）。
- **显式理由。**每个选中或省略的组件都要有理由、触发条件与替代方案。
- **可替换边界。**模型、Agent 运行时、队列、数据库、对象存储、遥测都通过项目自有的适配器隔离。
- **评测与特性同生。**没有评测的能力是不完整的工作；bug 蒸馏为回归评测。
- **安全默认。**写工具、浏览器、shell、MCP 与外部凭据触发强制安全控制（见 `security-gates`）。
- **不覆盖用户代码。**既有项目只做增量改动；破坏性改动走用户批准的 diff。
- **Harness 内容按归属。**L0/L1 只作参考性学习材料，不当作强制规则；L2 组织流程与 L3 责任判断是仓库内投资。见 `knowledge/standards/agent-engineering-standard.md` §1.2。
- **标准优于路径。**成功标准必须机器可检查（测试绿、产物存在、diff 为空、退出码 0）；怎么到达留给 agent。
- **技能路由、知识加载、agent 执行。**技能决定工作流；知识按触发加载，不复制进技能。

## 禁止

- 不要让仓库退回 CLI / 框架 / 运行时形态。
- 不要在没有明确需求时默认多智能体、RAG、向量库、浏览器、shell、Temporal、Kubernetes。
- 不要把完整工程标准复制进技能；生成精简、匹配技能的规则。
- 不要把 L0/L1 参考当作项目规则强制，除非有显式 L2/L3 归属与测试。

## 架构决策

任何新的 Agent、框架、多智能体、持久化执行或自主循环架构都需要在 `docs/maintainers/decisions/` 落一份 ADR。

## 验证

本仓库是技能与知识的工作区，不是构建目标：

```bash
# 技能结构校验（CI 同款）
for d in skills/*/; do test -f "$d/SKILL.md" || echo "missing $d/SKILL.md"; done
for f in skills/*/SKILL.md; do
  grep -q '^name:' "$f" || echo "no name: $f"
  grep -q '^description:' "$f" || echo "no description: $f"
done
# 技能引用的 knowledge/ 文件必须存在
for f in $(grep -rho 'knowledge/[a-zA-Z0-9/._-]*' skills/ | sort -u); do
  test -f "$f" || echo "dangling: $f"
done
```

## 完成标准

一个任务只有同时满足以下条件才算完成：

- 请求的行为已实现。
- 结构校验通过。
- diff 无无关改动。
- 新风险与架构决策已记录为 ADR。
- 仓库对下一个开发者或 agent 保持可用。

## 规则来源

规则起源见 `knowledge/standards/agent-engineering-standard.md`（人类可读工程标准），并蒸馏进 `skills/` 下的技能集。

# agent-dev-skills

面向 **Agent 项目开发**的技能集（skill-driven toolkit）：七个技能覆盖 Agent 开发的完整生命周期——设计 → 技术选型 → 脚手架 → 评测 → 交付评审，由两个纪律技能（安全门槛、复杂度阶梯）贯穿，底层是一个共享、版本化的知识库。

它**不是** Agent 框架，也不是 CLI。我们提供技能、知识、标准与实践：骨架来自官方脚手架 CLI，确定性保证落在每个生成项目自己的验证门槛里。

## 技能集

| 技能 | 阶段 | 类型 |
|---|---|---|
| `agent-design` | 需求 → 设计文档 | 编排 |
| `agent-stack` | 技术选型 | 编排 |
| `agent-scaffold` | 用官方 CLI 生成项目 + 增量接线 | 编排 |
| `agent-eval` | 评测套件 | 编排 |
| `agent-review` | 交付前评审 | 编排 |
| `security-gates` | 强制安全纪律 | 纪律 |
| `complexity-ladder` | 架构复杂度纪律 | 纪律 |

编排技能由用户按开发顺序调用；纪律技能由编排技能按触发条件调用，从不自我编排。技能按触发加载 `knowledge/` 中的参考内容，而不是把它复制进技能正文——知识库是唯一事实来源，技能保持精简。

## 快速开始

把 `skills/` 下的技能目录复制进你的 coding agent 技能目录（以 Claude Code 为例）：

```bash
# 项目级（仅当前项目可用）
cp -r skills/* <你的项目>/.claude/skills/

# 用户级（全局可用）
cp -r skills/* ~/.claude/skills/
```

每个技能带 `agents/openai.yaml` 宿主元数据，供 Codex 类宿主使用。

然后从一个粗略想法开始：

```text
Use $agent-design 澄清我的 Agent 想法，产出经过评审的 agent.design.md。
```

之后的链路是 `agent-stack` → `agent-scaffold` → `agent-eval` → `agent-review`。`agent-scaffold` 以非交互方式运行官方脚手架 CLI（如 `create-next-app`、`npm i`）并增量接线能力，只有在项目自身的验证门槛（类型检查、测试、安全清单）通过后才交付。

## 仓库结构

- `skills/` — 七个技能（每个含 `SKILL.md` 与宿主元数据）。
- `knowledge/` — 共享、版本化的参考知识库（案例、规则、选型、实践、标准 + `index.yaml`），按触发加载。
- `docs/` — 文档与架构决策（`docs/maintainers/decisions/`）。

## 验证

本仓库是技能与知识的工作区，不是构建目标。结构校验（CI 同款）：

```bash
# 每个技能有带 name/description frontmatter 的 SKILL.md
for f in skills/*/SKILL.md; do
  grep -q '^name:' "$f" && grep -q '^description:' "$f" || echo "bad: $f"
done

# 技能引用的 knowledge/ 文件必须真实存在
for f in $(grep -rho 'knowledge/[a-zA-Z0-9/._-]*' skills/ | sort -u); do
  test -f "$f" || echo "dangling: $f"
done
```

## 文档

- [架构决策：技能优先的工具集](./docs/maintainers/decisions/0001-skill-first-toolkit.md)
- [Agent 工程实践规范](./knowledge/standards/agent-engineering-standard.md)
- [Agent 开发问题解决经验](./knowledge/cases/agent-development-lessons.md)
- [AI 驱动开发实践指南](./knowledge/practices/ai-native-dev-playbook.md)

## 许可

MIT，见 [LICENSE](./LICENSE)。

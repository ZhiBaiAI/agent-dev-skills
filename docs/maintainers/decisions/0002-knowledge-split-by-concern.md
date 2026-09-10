# ADR-0002: 知识库按关注点拆分，技能用 references.md 声明加载清单

- **状态：** 已接受
- **日期：** 2026-09-10

## 背景

ADR-0001 确立 `knowledge/` 是共享知识基座、技能按触发加载而不复制内容。但知识文件是按"来源形态"（标准、案例、选型、实践）组织的少数长文：`agent-engineering-standard.md` 2068 行、`agent-development-lessons.md` 944 行。技能一次只用到其中一小片——`agent-eval` 引用 "standard §7-8"（约 200 行），却要加载整份 2068 行文件才能读到。

后果有三个：**(a) 上下文体量**——按需加载退化为整库加载，与 standard §2.12「索引优先检索」和 §14.2「渐进式加载」自相矛盾；**(b) 定位成本**——技能正文写 "standard §7.6"，读者要在一份两千行文件里找；**(c) 拆分界限模糊**——"标准"这一形态把设计、评测、安全、交付全塞在一个文件里，关注点无法独立演进。

## 决定

1. **按关注点拆分，不按来源形态组织。** 四个长集合各自拆为主题文件，文件名带序号与关注点：`standards/01-foundations` … `10-references-and-evolution`、`cases/01-context-and-tools` … `06-sources-and-mapping`、`selection/01-selection-foundations` … `04-framework-comparison`、`practices/01-mindset-and-collaboration` … `04-cost-and-rollout`。单个主题文件目标 ≤ 400 行。

2. **内容零改写、零复制，唯一来源不变。** 拆分是机械切片：原文 verbatim，原章节编号（§7.6 等）在主题文件内保持不变。跨主题文件共享的章节（如 §3 同时被 complexity-ladder 与 agent-review 需要）仍是同一份文件，不复制。

3. **全库单一导航页：`knowledge/README.md`。** 不保留每集合一个索引页（那会新增四份纯导航文件）。四个集合的"章节号 → 主题文件"映射、别名表（`standard`/`lessons`/`playbook`）、以及原标准的 §0 定位与维护规则，集中在这一份导航页里；`§N` 引用按整数部分查表定位。集合目录与导航页不再重复正文。

4. **每个技能一份 `references.md`。** 按阶段列出该加载哪些主题文件、为什么加载，作为该技能唯一的加载清单。`SKILL.md` 正文引用具体主题文件并指向 `references.md`；正文不再指向已拆分的集合页。

5. **CI 增加一条守门检查：** 每个技能必须有 `references.md`；`knowledge/` 引用必须可解析（仍用原悬空检查，现已覆盖 `references.md` 内的路径）。

## 后果

- 技能加载面从"整份 2068 行"降到"2-6 个主题文件、合计约 100-600 行"，按需加载名副其实。
- 共享规则仍是单份：`§3` 只在 `standards/03-orchestration-and-hitl.md` 一处，被 complexity-ladder 和 agent-review 同时引用，不产生副本漂移。
- `index.yaml` 的 `path:` 由四个已删除的长文改为对应集合目录（`standards/` 等）；既有的 `§N` 引用经 `knowledge/README.md` 的映射表继续可定位。原四份长文的链接改为指向集合内的首个主题文件或导航页。
- 全库不再有纯导航的重复文件：四份集合索引合并为一份 `knowledge/README.md`（GitHub 浏览 `knowledge/` 时自动渲染为首选入口）。
- 新增维护成本：新增知识需决定归属哪个主题文件、并同步该技能的 `references.md`。`CONTRIBUTING.md` 已记入规则（单文件 ~350 行内、引用具体主题文件而非集合页）。
- 与 ADR-0001 的关系：本 ADR 细化第 4 条决策的**组织形态**，不改变"知识不复制进技能"的原则。

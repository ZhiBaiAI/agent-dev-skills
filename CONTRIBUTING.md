# 贡献指南

1. 先读仓库根目录的 `AGENTS.md`，理解技能集的定位与工作规则。
2. 技能格式：每个技能是一个目录，含 `SKILL.md`，YAML frontmatter 必须有 `name` 与 `description`。正文精简、触发条件明确；参考内容放 `knowledge/`，技能里引用而不是复制。
3. 技能只引用 `knowledge/` 内的文件，且引用必须可解析——CI 会校验悬空引用。
4. 提 PR 前在本地跑一遍结构校验（见 README「验证」一节）。
5. 新的架构决策（Agent、框架、多智能体、持久化执行、自主循环）需要先落 ADR，放 `docs/maintainers/decisions/`。
6. 知识库条目带来源与日期；引用外部文章优先 2026 年后的一手资料。

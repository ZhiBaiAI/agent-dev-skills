# 架构与安全边界

这是团队的 L3 约束说明。硬性结果必须由 Schema、Policy、Resolver、Doctor 或测试执行；文档和 Skill 只负责让设计过程可理解、可追溯。

- 浏览器、Shell、文件系统写入、外部 API、MCP、凭证和网络出口必须在 Intent 中显式声明。

- 数据库、队列、耐久工作流、RAG 和多 Agent 只有在需求明确时才能引入；每项都要说明不选更轻方案的原因。

- 副作用需要审批、审计、幂等性和可验证的完成条件。

- 技术栈偏好只有在用户确认后才写入 `constraints.technology`，并由 Resolver 校验。

- `agent.plan.yaml` 只能由 Resolver 生成；不得手改绕过约束。


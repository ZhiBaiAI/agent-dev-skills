# Agent 工程规范 · Tool 与 MCP

> **范围：** §4 Tool 设计（契约、描述、输出、Evals、Capability、Error Contract、CLI/MCP 选择、Schema 成本）、§11 MCP 使用规范与安全基线
> 集合：`standards/` ｜ 导航：[知识库索引](../README.md)

# 4. Tool 设计规范

## 4.1 Tool 是模型与确定性系统的契约

- **MUST** 面向 Agent 的任务语义设计 Tool，而非一对一包装现有 API。
- **MUST** 使用清晰、无歧义的名称。
- **MUST** 使用严格输入 Schema。
- **MUST** 对输出进行裁剪和结构化。
- **MUST** 标注只读、写入、破坏性、幂等和外部访问属性。
- **MUST** 定义超时、错误分类和重试策略。
- **MUST** 对有副作用 Tool 使用幂等键或业务防重机制。
- **MUST NOT** 在 Tool 输出中泄露 Secret。
- **SHOULD** 减少功能重叠的 Tool。

## 4.2 标准 Tool Contract

每个 Tool 应声明：name、version、description、inputSchema、outputSchema、annotations（readOnly/destructive/idempotent/openWorld）、riskLevel、approvalPolicy、timeoutMs、retryPolicy、resultPolicy。

## 4.3 Tool 描述规则

好描述应包括：做什么、何时使用、何时不要使用、输入字段含义、返回内容、副作用、失败后的可恢复方式。

```text
错误：search: 搜索数据
更好：supplier.search: Search internal supplier records by normalized product requirements.
     Use this before public web discovery. Returns at most 20 candidates with IDs,
     matched fields, and source evidence. This tool is read-only and does not contact suppliers.
```

## 4.4 Tool 输出规则

- **MUST** 对输出执行裁剪和结构化。
- **MUST NOT** 返回原始大 HTML、完整数据库行或无界日志。
- **SHOULD** 对长日志、测试输出、进程列表和 Diff 执行确定性压缩。
- **SHOULD** 保留退出码、失败位置、关键错误、统计和 ArtifactRef。

## 4.5 Tool Evals

每个 Tool 至少测试：正确选择 Tool、不应调用时不调用、参数正确率、对错误结果的恢复、Token 效率、描述变更后的回归、危险参数是否被 Guardrail 拒绝、重试是否造成重复副作用。

## 4.6 Capability 与 Action Space

Skill、内置 Tool、业务 Tool、Workflow、MCP Tool 和 Tool Pack 统一建模为 Capability。解析顺序：Agent 静态能力 + 设计默认能力 + 当前任务所需能力 → 权限过滤 → 风险过滤 → ToolDefinition。

- **MUST** 在每一步只暴露当前可用且相关的能力。
- **MUST** 通过权限和风险策略过滤 Capability。
- **SHOULD** 监控 Tool 数量对选择准确率和 Token 的影响。

## 4.7 Connector 与环境操作

Connector 连接日志、Trace、代码仓库、CI/CD、通知、云资源和业务系统。应声明：authPolicy、readScopes、writeScopes、correlationKeys、rateLimit、timeoutMs、retryPolicy、redactionPolicy、healthCheck。

- **MUST** 分离读取和写入权限。
- **MUST** 使用 `trace_id`、`request_id`、`deployment_id`、`commit_sha` 关联跨系统证据。
- **MUST** 对写入和部署操作支持 Dry Run、审批和审计。
- **MUST** 将凭证保存在专用 Secret Store。

## 4.8 Error Contract

错误分为可供 Agent 决策的业务错误和必须由 Runtime 处理的系统错误。错误应声明：code、category、message、retryable、safeForModel、suggestedActions。

- **MUST** 将预期 Tool 失败返回为结构化错误数据。
- **MUST** 对认证失败、权限失败、数据损坏、Runtime 缺陷和未知系统异常终止或中断执行。
- **MUST NOT** 把所有 Exception 直接注入模型上下文。
- **MUST NOT** 让模型决定权限、数据完整性或系统故障的处理边界。

## 4.9 确定性执行与 CLI / MCP 选择

| 场景 | 优先方式 |
|---|---|
| 批量、可重跑、CI、编译、测试、迁移、健康检查 | CLI / Script |
| 动态发现远程能力、交互式资源访问、跨客户端工具生态 | MCP |
| 稳定业务操作和副作用 | Typed Service Tool |
| 大数据采集和转换 | Connector + Artifact |

- **MUST** 将编译、测试、环境启动、健康检查和数据迁移封装为脚本或 CLI。
- **MUST** 让 Agent 传结构化参数，避免拼接凭证和复杂 Shell。
- **MUST** 使用 MCP 时只暴露当前 Agent 需要的 Server 和 Tool。
- **MUST NOT** 让模型逐步控制可由脚本一次执行的确定性流程。

## 4.10 Tool Schema 与结果成本

Tool 成本包括：Schema 常驻 Token、选择推理、参数构造、执行、结果注入、后续历史重复携带。

- **MUST** 按 Agent 生成 Tool Allowlist。
- **MUST** 统计每个 Tool Schema 和结果的 Token。
- **MUST** 对长日志、测试输出、进程列表和 Diff 执行确定性压缩。
- **SHOULD** 对大量相似 Tool 使用 Tool Pack、Router 或短列表。

---

---

# 11. MCP 使用规范

## 11.1 何时使用 MCP

适用：跨进程或跨语言复用工具、第三方工具生态、多个 Agent Host 共用能力、需要标准化资源/Prompt/Tool 发现。

避免用于：同一进程普通函数、只服务一个项目的内部业务逻辑、低延迟核心路径、无法建立清晰授权边界的高风险操作。

## 11.2 MCP 安全基线

- **MUST** 对 HTTP MCP 使用规范授权流程。
- **MUST** 做 Token audience 校验。
- **MUST NOT** Token passthrough。
- **MUST NOT** 将 Token 放进 URL。
- **MUST** 使用 HTTPS。
- **MUST** 对本地 MCP 安装显示完整命令并获取明确同意。
- **MUST** 将每个 MCP Server 视为独立信任域。
- **MUST** 根据 Skill 或任务按需连接 MCP，不默认连接所有 Server。

## 11.3 MCP 成本与上下文边界

- **MUST** 为每个 Agent 设置 MCP Server 和 Tool 白名单。
- **MUST** 统计 MCP Tool Schema 和结果注入成本。
- **MUST** 将大结果保存到 Artifact Store。
- **MUST NOT** 将所有已连接 MCP Tool 默认暴露给所有 Agent。
- **MUST NOT** 以成本理由绕过 MCP 的权限、确认和审计要求。

---

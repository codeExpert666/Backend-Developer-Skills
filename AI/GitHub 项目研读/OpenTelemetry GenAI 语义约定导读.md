---
title: OpenTelemetry GenAI 语义约定导读
aliases:
  - OTel GenAI 语义约定导读
  - 生成式 AI 遥测语义约定导读
tags:
  - AI
  - GitHub
  - 源码研读
  - OpenTelemetry
  - 可观测性
created: 2026-07-27T01:30:28
updated: 2026-07-27T01:30:28
---

OpenTelemetry GenAI Semantic Conventions（生成式 AI 语义约定）定义模型、Agent、工具、检索和相关操作应怎样命名 Span、Event、Metric 与属性。它解决跨语言遥测的共同词汇，不负责采集器部署、观测平台存储、模型重试或业务授权。

本文是 [[GitHub AI 项目源码研读方法与清单]] 的规范导读，并与 [[Langfuse Trace 与评测数据模型源码导读]] 分工：本篇回答“遥测应表达什么”，Langfuse 导读回答“所选平台快照怎样接收和处理这些数据”。

本轮资料状态为“官方一手资料静态核验”：核对了 release、迁移说明、独立仓库模型文件、生成标记和 reference scenarios，但没有运行 Weaver、scenario、Collector 或任何 Java / Go instrumentation。因此没有把规范草案写成已经通过的实现结果。

## 版本依据：先处理仓库迁移

核对日期为 2026-07-27。

| 事实 | 本轮依据 | 结论 |
| --- | --- | --- |
| OpenTelemetry 核心语义约定当前稳定 release | [`v1.43.0`](https://github.com/open-telemetry/semantic-conventions/releases/tag/v1.43.0)，commit `89aae438b3b3b0a8dd33003c9d70592baf7dbd0d` | 核心仓库仍是通用语义约定基线，但不再维护实际 GenAI 模型 |
| GenAI 迁移 | [核心 `v1.42.0` Release](https://github.com/open-telemetry/semantic-conventions/releases/tag/v1.42.0) | `gen_ai.*`、provider 和 MCP 约定已迁到独立仓库 |
| 当前 GenAI 事实源 | [`open-telemetry/semantic-conventions-genai`](https://github.com/open-telemetry/semantic-conventions-genai) | 固定 commit `64cfaa612a1af8472b2f063374fbe3c9e6cea2ab` |
| 独立仓库发布状态 | [Releases 页面](https://github.com/open-telemetry/semantic-conventions-genai/releases) | 截至核对日没有 release 和 tag，只能使用完整 commit |

核心 `v1.43.0` 的 `docs/gen-ai/` 只保留 deprecated 的 “Moved” 页面。它可以证明迁移发生过，不能继续作为当前 GenAI 属性和 Span 的规范来源。

独立仓库在所选 commit 中引用 core semantic conventions `v1.43.0` 和 Weaver `v0.25.0`。这些工具版本是生成与校验上下文，不等于 GenAI 模型已经稳定发布。

## 稳定性必须原样保留

所选快照中全部 GenAI 语义约定仍标为 Development。它表示名称、要求和结构可能发生不兼容变化，因此：

- 可以在实验、适配层和内部字段映射中采用。
- 不应直接把实验属性名写成长期业务数据库合同。
- Java 与 Go 应共享语义映射，但各自使用目标 instrumentation 实际支持的字段。
- 升级时必须比较 model diff 和生成文档，而不是只升级依赖版本。

仓库 README 的 Schema URL 仍为 `TODO`。不能虚构类似稳定 release 的 Schema URL，也不能用 core `v1.43.0` 的 Schema URL 冒充独立 GenAI 仓库 Schema。

## 模型源码结构

独立仓库的重点不是按顺序阅读所有 Markdown，而是从模型源进入：

| 位置 | 主要内容 |
| --- | --- |
| `model/gen-ai/registry.yaml` | `gen_ai.*` 属性注册表 |
| `model/gen-ai/spans.yaml` | inference、tool、agent、retrieval、memory 等 Span 语义 |
| `model/gen-ai/events.yaml` | 消息和异常等事件 |
| `model/gen-ai/metrics.yaml` | operation duration、token usage 等 Metric |
| `model/gen-ai/*.json` | messages、tools、retrieval、memory 等结构化值的 JSON Schema |
| `model/mcp/` | MCP 相关遥测模型 |
| `model/openai/`、`model/aws-bedrock/` | provider 特有扩展 |
| `reference/scenarios/` | 真实客户端库、确定性本地 mock、捕获遥测和 Weaver 验证 |
| `docs/gen-ai/` | 从模型生成的可读文档 |

先读通用 `model/gen-ai/`，只有目标 provider 确实需要补充属性时才进入 provider 目录。MCP 的协议行为仍以正式 MCP Specification 为准；这里的 `model/mcp/` 只定义遥测表达。

## 从一次模型操作到遥测

典型运行链是：

```text
业务代码
  → 模型 SDK 或 AI 框架 instrumentation
  → 创建 logical operation span
  → 写入通用 gen_ai 与必要 provider 属性
  → 流式或非流式接收响应
  → 记录 token usage、duration、错误或取消
  → OpenTelemetry SDK / BatchSpanProcessor
  → OTLP exporter
  → Collector 或观测后端
```

“logical operation” 是从调用者视角的一次完整操作。规范要求 Span 持续到响应完成、错误或取消，而不是在 HTTP headers 到达时提前结束。

自动模型重试应包含在同一个 logical operation Span 中。这样总时长与最终状态表示用户看到的一次调用；如果还需要分析每次网络尝试，应由更低层 HTTP / RPC Span 表达，避免把业务一次调用误算为多次用户请求。

## Span、Event 与 Metric 的分工

| 信号 | 适合回答的问题 | 不适合承载什么 |
| --- | --- | --- |
| Span | 一次 inference、tool、agent、retrieval 或 memory 操作经历了什么 | 大量高基数内容的长期汇总 |
| Event | 操作中的消息、异常或其他离散事实 | 替代最终 Span 状态 |
| Metric | duration、token usage 等跨请求趋势 | Trace ID、完整 Prompt 和单次调用诊断 |

常见关系可以表示为：

```text
agent span
  ├─ inference span
  ├─ execute_tool span
  ├─ retrieval span
  └─ inference span
```

父子层级应表达真实控制关系。不能为了图好看把同一工作同时上报成多个平级“成功”，也不能只留下最外层 Agent 而失去工具失败位置。

## 错误、取消与重试语义

Span 在错误时必须记录低基数 `error.type`。低基数表示它适合按类型聚合，例如稳定异常类名或协议状态，不使用完整错误消息、用户输入或动态 URL。

异常事件使用 `gen_ai.client.operation.exception`，严重级别为 WARN；异常消息可能含敏感数据，必须先评估是否采集。Span 的最终 error 状态与 exception event 解决不同问题：前者用于聚合最终结果，后者用于保留诊断细节。

取消应作为 logical operation 的终态之一，并让 Span 在调用真正停止或达到确定性取消边界时结束。规范描述遥测，不替应用：

- 触发 `context.Context`、线程中断或 SDK cancellation。
- 停止已经进入外部系统的工具副作用。
- 决定取消后是否重试。
- 限制模型总步骤、Token 和成本。

这些运行控制仍属于应用和框架，参见 [[Eino Graph 与 Tool Calling 源码导读]]。

## 敏感内容默认不采集

规范明确要求 instructions、input 和 output 默认不捕获，只有显式 opt-in 才采集。原因不只是隐私：Prompt、检索片段和工具结果还可能包含凭据、业务秘密、个人信息和 Prompt Injection 载荷。

推荐边界是：

1. 默认只记录稳定操作、模型、状态、Token、时长和版本。
2. 预生产环境确需内容调试时，可在 Span 属性中采集受控样本。
3. 生产环境优先把敏感内容放入独立存储，在 Span 中保存受控引用。
4. 独立存储使用不同的访问权限、加密、保留期和删除机制。
5. Metric 标签不放 Prompt、用户 ID、Trace ID 或其他高基数字段。

规范中的 Streaming chunks 章节仍是 `TODO`。因此不能声称当前已有稳定的逐 chunk 内容或延迟事件 Schema；首 Token 延迟等指标需要按实际 instrumentation 和内部映射明确记录。

内容策略还应与 [[AI 应用的成本、延迟、可观测性与降级]] 的默认最小化规则保持一致。

## Reference Scenario 的证据链

官方 `reference/scenarios/` 不是普通文档示例，而是一条可重放的规范验证链：

```text
scenario.py
  → 真实客户端库
  → 确定性本地 mock server
  → 捕获 spans / metrics / logs
  → parser 归一化
  → Weaver 对照 model/ 校验
  → 生成 data.json
  → 生成 scenario report
```

它能够证明“该场景捕获的遥测符合当前模型约束”，不能证明所有 provider、语言 SDK 和业务字段都一致。选一个与当前问题相同的 scenario 观察即可，不要遍历全部报告。

## 手写、生成与规范边界

- `model/**/*.yaml` 与相关 JSON Schema 是规范模型源。
- `docs/` 中带 “DO NOT EDIT” 的块由 Weaver 生成；修改语义应回到模型源。
- reference scenario 的驱动、mock 和解析工具是手写代码。
- `data.json` 与报告表格是执行 scenario 后的生成证据。
- 核心仓库的 moved stub 是迁移提示，不是独立仓库内容的生成源。

本文没有运行 Weaver 或 scenario，因此只记录仓库已有证据，没有声称本地 conformance 通过。

## 与 Langfuse 的连接边界

OpenTelemetry 定义遥测语义，Langfuse 所选版本实现 OTLP Trace ingestion。两者之间仍有映射层：

```text
GenAI semantic attributes
  → instrumentation 实际支持范围
  → OTLP 编码
  → Langfuse OTel mapping
  → Trace / Observation / usage / cost
```

验证时应选一个普通 inference、一个 tool 或 retrieval Span，比较导出 OTLP 与 Langfuse 最终字段。Langfuse 路由接受请求不能证明所有 Development 属性均已映射，也不能证明 OTLP Metrics 已被处理。

## 本轮停止条件

完成以下内容即可停止：

- 能解释核心仓库到独立 GenAI 仓库的迁移。
- 能从 `registry.yaml`、`spans.yaml`、`events.yaml` 和 `metrics.yaml` 找到一个操作的定义。
- 能区分 Span、Event、Metric 和 provider 扩展。
- 能解释错误、取消、自动模型重试及默认不采集内容。
- 能用一个 reference scenario 说明模型源、Weaver、`data.json` 与报告的关系。
- 明确全部 Development、Schema URL TODO、Streaming chunks TODO。

需要证明 Java / Go 接入时，应另建实验记录，固定 instrumentation 版本和 commit，运行一个成功、一个错误和一个取消场景；本文没有完成该实验。

## 易变事实与重新核对

- 独立仓库尚无 release/tag，默认分支提交会变化。
- 全部 GenAI 约定仍是 Development，属性名和要求可能不兼容升级。
- Schema URL 和 Streaming chunks 尚未定稿。
- provider 和 MCP 模型可能先于正式协议或 SDK release 变化。
- 后端对新属性的映射通常落后于规范模型。

每次实现或升级前，重新核对 [独立仓库 Releases](https://github.com/open-telemetry/semantic-conventions-genai/releases)、目标 commit、README、`versions.env`、model diff 和目标 instrumentation 支持矩阵。

## 官方资料

- [OpenTelemetry Semantic Conventions Releases](https://github.com/open-telemetry/semantic-conventions/releases)
- [Semantic Conventions v1.42.0 迁移说明](https://github.com/open-telemetry/semantic-conventions/releases/tag/v1.42.0)
- [Semantic Conventions v1.43.0](https://github.com/open-telemetry/semantic-conventions/releases/tag/v1.43.0)
- [OpenTelemetry GenAI Semantic Conventions 仓库](https://github.com/open-telemetry/semantic-conventions-genai)
- [固定 commit `64cfaa612a1af8472b2f063374fbe3c9e6cea2ab`](https://github.com/open-telemetry/semantic-conventions-genai/commit/64cfaa612a1af8472b2f063374fbe3c9e6cea2ab)
- [OpenTelemetry Protocol](https://opentelemetry.io/docs/specs/otlp/)

Agent、工具和 MCP 的安全边界不能从遥测规范推导，需继续使用 [[OWASP GenAI 与 MCP Security 安全资料导读]]。

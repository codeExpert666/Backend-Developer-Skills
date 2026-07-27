---
title: Langfuse Trace 与评测数据模型源码导读
aliases:
  - Langfuse Trace 源码导读
  - Langfuse 评测数据模型源码导读
tags:
  - AI
  - GitHub
  - 源码研读
  - Langfuse
  - 可观测性
  - 评测
created: 2026-07-27T01:30:28
updated: 2026-07-27T01:30:28
---

本文从 Langfuse 的 OTLP Trace 入站路径进入，再连接 Trace、Observation、Score、Dataset 与 Dataset Run，回答“应用遥测怎样成为可查询、可评分、可重复比较的评测数据”。它是 [[GitHub AI 项目源码研读方法与清单]] 的平台侧导读，不把观测数据到达入口误判为全部落库成功，也不把 Langfuse 当成工具授权或业务事务系统。

本轮资料状态为“官方一手资料静态核验”：核对了指定 release 的仓库代码、Schema 和测试入口，没有启动 Langfuse、发送 OTLP 数据、运行 worker、执行测试或完成真实评测。因此本文只描述所选源码快照中的边界。

## 版本依据

核对日期为 2026-07-27。本轮固定 [Langfuse `v3.224.1`](https://github.com/langfuse/langfuse/releases/tag/v3.224.1)，完整 commit 为 `5e4dae6e11f7890ceb105f44fa5a882614a7cda4`。

选择 release 而不是默认分支，是因为该仓库的 ingestion v4、存储路径、评测能力和 API 仍持续变化。后文的“当前”只表示这个 commit，不表示未来 release。

## 先建立数据模型

Langfuse 的核心不是一张“日志表”，而是两条相连的数据链：

```text
在线观测
Trace
  → Observation 树
      → generation / span / tool / retriever / evaluator 等步骤
  → Score

离线或回归评测
Dataset
  → DatasetItem
  → DatasetRun
      → DatasetRunItem
          → Trace
          → 可选 Observation
          → Score
```

各对象的职责是：

| 对象 | 负责什么 | 关键边界 |
| --- | --- | --- |
| Trace | 一次用户请求、任务或业务操作的顶层关联 | 不是“所有步骤都成功”的证明 |
| Observation | Trace 内的模型、工具、检索、链或其他子步骤 | 父子关系、时间和状态必须由上报数据正确表达 |
| Score | 对 Trace、Observation、Session 或实验结果的数值、布尔、类别或文本判断 | 分数必须带来源、配置和评测版本，不能脱离样本解释 |
| Dataset | 一组受版本和 Schema 约束的固定评测输入 | 不等于线上自然流量 |
| DatasetItem | 输入、期望输出、元数据及可选来源 Trace / Observation | 需要版本、可见范围和敏感数据治理 |
| DatasetRun | 对一个 Dataset 的一次具名实验 | 模型、Prompt、代码和参数版本应另行关联 |
| DatasetRunItem | DatasetItem 与本次 Trace、可选 Observation 的连接 | 让输出、过程和 Score 回到同一个固定样本 |

[`packages/shared/prisma/schema.prisma`](https://github.com/langfuse/langfuse/blob/5e4dae6e11f7890ceb105f44fa5a882614a7cda4/packages/shared/prisma/schema.prisma) 可用于观察 Dataset、DatasetItem、DatasetRuns、DatasetRunItems、ScoreConfig 和旧 Prisma Trace / Observation / Score 的关系。DatasetItem 还包含 `validFrom`、`validTo` 和 `isDeleted`，表明样本版本不能只靠“当前行”理解。

文件中的 `LegacyPrisma*` 模型不是完整的当前存储事实。所选版本同时存在 ClickHouse、event ingestion 和新旧兼容路径；研读数据模型时必须把“领域关系”与“具体哪张表是当前读写主路径”分开。

## 仓库结构与入口

| 目录 | 角色 |
| --- | --- |
| `web/` | Next.js UI、公开 API 和 OTLP HTTP 入口 |
| `worker/` | BullMQ 消费、后台转换、评测调度和写入 |
| `packages/shared/` | 领域类型、认证、ingestion、OTel 转换、Redis、ClickHouse 与共享服务 |
| `fern/` | 公开 API 定义及生成 SDK 的来源 |
| `ee/` | 企业版能力边界 |

Trace 入站最小路径不需要先阅读仪表盘。先跟 [`web/src/pages/api/public/otel/v1/traces/index.ts`](https://github.com/langfuse/langfuse/blob/5e4dae6e11f7890ceb105f44fa5a882614a7cda4/web/src/pages/api/public/otel/v1/traces/index.ts)，再进入认证、`OtelIngestionProcessor`、队列和 worker。

## OTLP Trace 入站链

```text
POST /api/public/otel/v1/traces
  → createAuthedProjectAPIRoute
  → ApiAuthService 校验 project scope
  → 读取 raw body
  → 可选 gzip 解压
  → 只接受 OTLP JSON 或 protobuf
  → 校验 ingestion version
  → OtelIngestionProcessor
  → 原始数据上传对象存储
  → OtelIngestionQueue
  → worker 下载并解析
  → 可选 masking
  → processToIngestionEvents
  → 拆分 Trace / Observation
  → IngestionService 与 event writer
  → ClickHouse 等存储
```

入口使用编译后的 OpenTelemetry protobuf root；相邻 README 说明它来自 OTel proto `v1.5.0`。JSON/protobuf 解析、对象存储和队列是三个不同证据层级：入口接受不等于 worker 解析成功，worker 解析成功也不等于所有事件都已写入。

### 认证边界

[`createAuthedProjectAPIRoute.ts`](https://github.com/langfuse/langfuse/blob/5e4dae6e11f7890ceb105f44fa5a882614a7cda4/web/src/features/public-api/server/createAuthedProjectAPIRoute.ts) 默认允许 project access level。[`apiAuth.ts`](https://github.com/langfuse/langfuse/blob/5e4dae6e11f7890ceb105f44fa5a882614a7cda4/web/src/features/public-api/server/apiAuth.ts) 中：

- Basic public key 加 secret key 可解析为 project 或 organization scope。
- 只有 public key 的 Bearer 方式得到受限的 `scores` scope。
- OTel Trace 路由没有显式降低到 `scores`，因此需要符合 project scope 的认证。
- 项目 ID 和 public key 会随 ingestion 数据进入后续处理，用于维持项目边界。

这只是 Langfuse 项目级入口授权。它不能证明上传内容本身符合企业的数据分类、用户同意或租户出境策略。

## OTLP Metrics 的真实状态

所选 commit 中的 [`web/src/pages/api/public/otel/v1/metrics/index.ts`](https://github.com/langfuse/langfuse/blob/5e4dae6e11f7890ceb105f44fa5a882614a7cda4/web/src/pages/api/public/otel/v1/metrics/index.ts) 是一个需要认证的 POST 入口，但处理函数为空操作。

因此只能写：

- Langfuse 该版本实现了本节描述的 OTLP Trace ingestion。
- Metrics 路由存在，但不能据此宣称 metrics 已进入与 traces 相同的解析、队列和存储链。

验证接入时应分别检查 Trace 与 Metrics，不能用 URL 存在或 HTTP 成功替代数据可查询证据。

## 至少一次、部分成功与错误传播

[`packages/shared/src/server/redis/otelIngestionQueue.ts`](https://github.com/langfuse/langfuse/blob/5e4dae6e11f7890ceb105f44fa5a882614a7cda4/packages/shared/src/server/redis/otelIngestionQueue.ts) 为主 OTLP job 配置多次尝试与指数退避。worker 遇到一般处理错误会记录并重新抛出，让 BullMQ 重试。

这构成至少一次处理语义，不是 exactly-once：

- 同一对象存储文件或事件可能被重新读取。
- writer、media 处理、评测调度等下游必须能够识别重复或接受幂等 upsert。
- API 接受后的客户端取消不能撤回已上传对象和已入队任务。

[`worker/src/queues/otelIngestionQueue.ts`](https://github.com/langfuse/langfuse/blob/5e4dae6e11f7890ceb105f44fa5a882614a7cda4/worker/src/queues/otelIngestionQueue.ts) 还展示了不同失败处理：

| 失败 | 所选版本中的处理 | 含义 |
| --- | --- | --- |
| masking 失败 | 记录并丢弃该批处理，不继续写入 | 默认不会把未脱敏事件继续落库，但该失败未通过队列重试恢复 |
| 禁止处理的输入 | 记录并返回 | 不是所有拒绝都会表现为队列最终失败 |
| 一般解析或批次错误 | 记录并重新抛出 | 由队列重试 |
| 部分单事件写入、评测或关联错误 | 某些路径捕获并继续 | 同一批次可能部分成功 |

因此验收必须同时观察入口响应、queue job、失败/丢弃计数、最终 Trace/Observation 数量和关联完整性。

## Masking 与对象存储顺序

OTLP 入口先把原始 spans 上传到对象存储，再由 worker 下载并执行可选 ingestion masking。这个顺序意味着：

- 即使最终数据库中的内容已 masking，原始对象也可能曾包含完整 Prompt、模型输出、检索片段和工具参数。
- 对象存储访问控制、加密、保留期、删除流程和审计属于同一隐私边界。
- “打开 Langfuse masking”不能替代发送前的数据最小化和应用侧脱敏。

内容采集原则应与 [[AI 应用的成本、延迟、可观测性与降级]] 一致：默认上传稳定标识、时间、状态、用量和版本；确需完整内容时再显式批准。

## 从 Trace 到评测数据

一次可解释的 Dataset 评测链应是：

```text
DatasetItem
  → 固定 input / expected output / metadata / Schema
  → 目标应用生成一次 Trace
  → DatasetRunItem 关联 item 与 trace
  → 可选 observation 指向被评步骤
  → evaluator、人工标注或 API 生成 Score
  → 按 DatasetRun 汇总并比较
```

这里至少有三类版本：

1. DatasetItem 自身的版本。
2. 被测系统的模型、Prompt、工具、检索索引和应用 revision。
3. ScoreConfig、评测器和判定阈值的版本。

只保存一个分数而没有这些版本，无法区分“模型退化”“样本变化”还是“评分规则变化”。评测发布门禁可继续参考 [[AI 评测、版本化与发布门禁]]。

## 错误、取消与降级边界

Langfuse 位于业务请求之外的观测路径，不应成为核心请求的同步成功条件：

- 应用侧取消要先停止模型、检索和工具，再记录 `cancelled` 或相应错误状态；Langfuse 不能替应用取消这些工作。
- 观测出口不可用时，应使用有界缓冲、采样或受控丢弃，而不是无限阻塞业务。
- 队列消费取消或 worker 重启属于 ingestion 生命周期，不等于用户业务操作被撤销。
- Score 或 eval 调度失败不应静默改写为业务成功；发布门禁要单独决定 fail-open 或 fail-closed。

## 手写、生成与测试边界

- OTLP API、认证包装、`OtelIngestionProcessor`、worker、ingestion service 和领域转换主要是手写代码。
- OTel protobuf root 是从 OpenTelemetry proto 编译生成的文件，不应作为手写业务逻辑阅读。
- `fern/` 保存公开 API 定义并用于生成 API 文档或 SDK；生成结果不能替代服务端实现证据。
- `schema.prisma` 是数据模型来源之一，Prisma client 属于生成产物；当前读写主路径还要结合 ClickHouse 和 event writer 判断。

与本轮链路直接相关的测试包括：

- `web/src/__tests__/server/api/otel/otelMapping.servertest.ts`。
- `worker/src/queues/__tests__/otelDirectEventWrite.test.ts`。
- `otelMetadataProcessing.test.ts`、`otelToObservationForEval.test.ts`、`otelConversionFailureLogging.test.ts`。
- `packages/shared/src/server/otel/OtelIngestionProcessor.metadataDropped.test.ts`。
- `worker/src/services/IngestionService/tests/IngestionService.integration.test.ts`。

这些文件是静态行为证据；本轮没有运行测试、数据库、Redis、对象存储或 ClickHouse。

## 本轮停止条件

满足以下条件后停止，不深入前端仪表盘和全部基础设施：

- 能画出认证入口、原始对象上传、入队、worker 转换和写入链。
- 能解释 Trace、Observation、Score、Dataset、DatasetRunItem 的关系。
- 能说明 Metrics 空操作、至少一次处理、部分成功和 masking 顺序。
- 能从一个 mapping 测试和一个 retry/error 测试定位所选版本行为。
- 能列出观测出口不可用时的业务降级与最终对账方式。

需要证明可运行时，应另建日期化执行记录，实际发送固定 OTLP fixture，核对 Trace/Observation/Score 数量，并制造队列重试、部分失败和 masking 失败；本文不声明这些实验已经完成。

## 易变事实与重新核对

- ingestion v4 的直写、双写和兼容路径仍在变化，不应成为长期业务 API。
- OTLP Metrics 入口可能在后续 release 实现。
- 队列 attempts、backoff、对象存储顺序和 masking 行为可能调整。
- Trace、Score 与 Dataset 的物理存储正在从旧 Prisma 模型向 ClickHouse/event 路径演进。

升级前重新核对 [Langfuse Releases](https://github.com/langfuse/langfuse/releases)、目标 tag 的 OTLP routes、queue options、worker 和 migration，不从本文推断未来版本。

## 官方资料

- [Langfuse GitHub 仓库](https://github.com/langfuse/langfuse)
- [Langfuse v3.224.1 Release](https://github.com/langfuse/langfuse/releases/tag/v3.224.1)
- [Langfuse Observability Documentation](https://langfuse.com/docs/observability/overview)
- [Langfuse Evaluation Documentation](https://langfuse.com/docs/evaluation/overview)
- [OpenTelemetry Protocol](https://opentelemetry.io/docs/specs/otlp/)

遥测属性的跨平台语义见 [[OpenTelemetry GenAI 语义约定导读]]；采集内容和平台权限的风险见 [[OWASP GenAI 与 MCP Security 安全资料导读]]。

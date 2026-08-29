---
title: VaultRAG 领域模型与数据契约
aliases:
  - VaultRAG 数据模型
  - VaultRAG 运行契约
tags:
  - 项目实践
  - VaultRAG
  - 领域模型
  - 数据契约
  - RAG
created: 2026-08-29T15:38:49
updated: 2026-08-29T15:38:49
---

本文定义 VaultRAG 中哪些对象是源事实、哪些是派生数据，以及扫描、索引、检索、回答和评测之间必须保持的身份、版本与终态合同。模块实现见 [[VaultRAG 总体架构与模块边界]]。

> [!note] 状态说明
> 这是实现前的数据语义设计。字段名和持久化形式可在技术验证后调整，但源事实边界、可追踪性和终态语义不得被依赖库替代。

## 聚合与关系总览

~~~mermaid
erDiagram
    VAULT_SNAPSHOT ||--o{ SOURCE_DOCUMENT : observes
    SOURCE_DOCUMENT ||--o{ DOCUMENT_REVISION : versions
    DOCUMENT_REVISION ||--o{ SECTION_CHUNK : produces
    INDEX_GENERATION ||--o{ INDEX_ENTRY : activates
    SECTION_CHUNK ||--o{ INDEX_ENTRY : projects
    RETRIEVAL_RUN ||--o{ EVIDENCE : selects
    SECTION_CHUNK ||--o{ EVIDENCE : references
    ANSWER_RUN ||--o{ ANSWER_CLAIM : contains
    ANSWER_CLAIM }o--o{ EVIDENCE : cites
    EVAL_SUITE ||--o{ EVAL_CASE : contains
    EVAL_RUN ||--o{ EVAL_RESULT : records
    EVAL_CASE ||--o{ EVAL_RESULT : evaluates
~~~

这里的“聚合”用于说明一致性和身份，不要求每个对象都映射为独立数据库表。

## 权威事实与派生数据

| 数据 | 分类 | 恢复方式 |
| --- | --- | --- |
| vault 中的 Markdown 字节 | 权威源事实 | 由用户自己的版本控制与备份负责 |
| VaultRAG 配置 | 项目输入合同 | 从显式配置和 CLI 覆盖重新加载 |
| SourceDocument 与 DocumentRevision 元数据 | 源事实的本地快照 | 重新扫描和散列源文件 |
| SectionChunk | 可重建解析投影 | 用固定解析器与切片器版本重建 |
| 关键词与向量索引 | 可重建检索投影 | 从活动切片重建 |
| RetrievalRun 与 AnswerRun | 运行证据 | 可按保留策略删除，不反向修改源事实 |
| EvalSuite | 版本化测试资产 | 由代码仓库保存 |
| EvalRun 报告 | 派生证据 | 用锁定版本坐标重新运行 |

向量索引不能成为唯一数据源，详见 [[Embedding、向量索引与派生数据边界]]。

## VaultConfig

VaultConfig 是一次运行的有效配置快照，至少表达：

| 字段 | 语义 |
| --- | --- |
| configVersion | 配置 Schema 主版本 |
| vaultRoot | 私有运行时根目录，不进入公开报告 |
| includeRules / excludeRules | 文件发现规则 |
| dataDir | 所有派生数据位置，必须在 vault 外 |
| parserProfile | Frontmatter、Wikilink、Callout 和代码块解析策略 |
| chunkingProfile | 目标大小、最大值、重叠和版本 |
| lexicalProfile | 关键词分析器、字段权重和版本 |
| embeddingProfile | 模型标识、维度、距离函数和归一化策略 |
| generationProfile | 生成模型、上下文预算、Prompt 和 Schema 版本 |
| privacyProfile | 日志、运行记录和内容留存策略 |

启动时生成 config fingerprint。影响切片或索引语义的字段变化必须触发全量重建；只影响输出格式的字段不能伪造新的源版本。

## VaultSnapshot

VaultSnapshot 表示某次扫描观察到的完整源集合：

- snapshotId：随机或时间有序运行标识，不承担内容身份。
- startedAt / completedAt：扫描时间。
- rootFingerprint：根目录的本地散列标识，不能反推出私人绝对路径。
- configFingerprint：本次扫描的有效配置。
- documents：每个相对路径的状态和内容散列。
- status：BUILDING、READY、FAILED 或 CANCELLED。
- summary：新增、修改、删除、未变、跳过和失败计数。

只有 READY 快照可以驱动新索引代次。FAILED 或 CANCELLED 不覆盖上一活动快照。

## SourceDocument

SourceDocument 表示一个逻辑源路径：

| 字段 | 语义 |
| --- | --- |
| sourceId | 由规范化相对路径派生的稳定标识 |
| relativePath | 使用斜杠和 Unicode 规范化后的 vault 相对路径 |
| mediaType | v0.x 固定为 text/markdown |
| lifecycleState | ACTIVE 或 DELETED |
| currentRevisionId | 当前活动内容版本 |
| firstSeenAt / lastSeenAt | 本地观测时间 |

v0.1 不尝试推断重命名。路径变化被建模为旧 SourceDocument 删除和新 SourceDocument 新增。这一规则简单、确定且不会根据内容相似度误合并两篇笔记。

如果未来引入可选 Frontmatter ID，必须先定义冲突、复制和缺失语义，不能直接覆盖路径身份。

## DocumentRevision

DocumentRevision 表示某个源文件的不可变内容版本：

- revisionId 由 sourceId、文件内容 SHA-256 与解析合同版本派生。
- sourceHash 对原始文件字节计算；修改时间和大小只用于诊断，不作为正确性依据。
- title、aliases、tags 和时间字段来自成功解析的 Frontmatter 或标题回退。
- parseStatus 区分 OK、DEGRADED 和 FAILED。
- warnings 保存安全类别与位置，不保存整篇正文。
- observedSnapshotId 指向发现该版本的扫描。

同一 sourceHash 在解析器合同变化后仍可产生不同 revisionId，因为 AST 和切片语义可能改变。

## SectionChunk

SectionChunk 是检索最小内容单元，而不是随意固定字符片段。建议字段：

| 字段 | 语义 |
| --- | --- |
| chunkId | 文档版本、标题路径、序号、内容散列与切片器版本的确定性标识 |
| revisionId | 所属不可变文档版本 |
| headingPath | 从一级标题到当前章节的有序路径 |
| ordinal | 同一章节内的稳定顺序 |
| body | 进入索引的正文，保留代码和必要 Markdown 语义 |
| startLine / endLine | 对源文件的定位范围 |
| tokenEstimate | 用选定估算器计算的上下文预算参考 |
| contentHash | 切片规范化内容散列，用于 Embedding 复用 |
| linkTargets | 当前切片中解析出的 Wikilink 目标元数据 |
| flags | CODE_HEAVY、TABLE、CALLOUT、OVERSIZED 等诊断标记 |

chunkId 不能包含绝对路径。标题重命名或章节重排会生成新切片；旧切片随新代次切换而退出活动索引。

## IndexGeneration

IndexGeneration 是一组原子可查询的派生数据：

| 字段 | 语义 |
| --- | --- |
| generationId | 本次构建标识 |
| snapshotId | 对应完整源快照 |
| parserVersion / chunkerVersion | 解析与切片合同 |
| lexicalVersion | 关键词分析器和字段权重 |
| embeddingModel / dimension | 向量兼容坐标 |
| status | BUILDING、READY、ACTIVE、FAILED、RETIRED |
| activatedAt | 成为唯一活动代次的时间 |

构建顺序必须是“暂存 → 完整校验 → 原子激活”。查询只读取一个 ACTIVE 代次，不能混用新旧关键词、向量或元数据。

当 Embedding 模型或维度变化时，新代次必须重建向量；维度不匹配应在写入前失败。

## RetrievalQuery 与 RetrievalRun

RetrievalQuery 是归一化后的查询合同：

- rawText：用户原始问题。
- normalizedText：只做受控空白和 Unicode 规范化，不丢弃代码符号、数字与否定词。
- mode：LEXICAL、VECTOR 或 HYBRID。
- filters：允许的路径前缀、标签或显式集合。
- topK 与 evidenceBudget：有安全上限。
- generationId：固定到一次活动索引代次。

RetrievalRun 记录：

- retrievalRunId、时间、状态和耗时。
- query fingerprint；默认不持久化原始私人问题。
- 各召回器版本、参数和候选排名。
- 融合、去重、邻接扩展和最终证据选择结果。
- 空结果、超时、取消和部分降级原因。

## Evidence

Evidence 是应用层生成的不可变候选证据：

| 字段 | 语义 |
| --- | --- |
| evidenceId | 当前 RetrievalRun 内的短标识，如 E1、E2 |
| chunkId | 真实活动切片 |
| sourceId / revisionId | 引用身份与版本 |
| relativePath / headingPath | 用户可读定位 |
| startLine / endLine | 可选精确定位 |
| excerpt | 发送给模型的受控内容 |
| rankSignals | 关键词、向量、融合和多样性字段 |

evidenceId 由程序生成并随 Prompt 一起提供。模型只能引用这些标识，不能提交任意文件名作为来源。

## AnswerRun、AnswerClaim 与 Citation

AnswerRun 表示一次完整问答状态机：

| 字段 | 语义 |
| --- | --- |
| answerRunId | 运行标识 |
| retrievalRunId | 唯一证据来源 |
| status | ANSWERED、INSUFFICIENT_EVIDENCE、CONFLICTING_EVIDENCE、MODEL_UNAVAILABLE、INVALID_MODEL_OUTPUT、CANCELLED 或 FAILED |
| modelProfile | 模型与推理参数版本 |
| promptVersion / schemaVersion | 可重放合同 |
| claims | 通过结构与引用校验的主张 |
| warnings | 降级、截断或来源冲突 |
| timing | 检索、组装、模型和校验耗时 |

AnswerClaim 至少包含 claimId、text 和 evidenceIds。Citation 是校验后的多对多映射，不接受下列结果：

- 引用了本次候选集外的 evidenceId。
- ANSWERED 但某个可核查主张没有证据。
- 引用的 chunk 不属于固定 generationId。
- 模型改写了相对路径、行号或源版本。
- 引用内容与主张没有可解释支持关系。

最后一项可由规则、评测器与人工抽样共同判断，但“证据标识是否真实”必须完全确定性校验。

## EvalSuite、EvalCase 与 EvalRun

EvalSuite 是版本化能力合同，不是临时问题列表。每条 EvalCase 建议包含：

- caseId、category、question。
- 固定 demo vault revision 与索引配置。
- expectedSourceIds 或 expectedChunkTraits。
- expectedAnswerStatus。
- requiredClaims、forbiddenClaims 和可选参考答案。
- securityExpectations，例如不得服从资料内指令。
- judgePolicy，区分确定性规则、人工量表和可选模型评审。

EvalRun 必须锁定代码 revision、依赖锁、操作系统、模型、Prompt、Schema、解析器、切片器、索引、数据集和配置 fingerprint。结果见 [[VaultRAG 质量评测与发布门禁]]。

## 状态机

### 索引代次

~~~mermaid
stateDiagram-v2
    [*] --> BUILDING
    BUILDING --> READY: 构建与一致性校验通过
    BUILDING --> FAILED: 解析或索引失败
    BUILDING --> CANCELLED: 用户取消
    READY --> ACTIVE: 原子切换
    ACTIVE --> RETIRED: 新代次激活
    FAILED --> [*]
    CANCELLED --> [*]
    RETIRED --> [*]
~~~

### 回答运行

~~~mermaid
stateDiagram-v2
    [*] --> RETRIEVING
    RETRIEVING --> INSUFFICIENT_EVIDENCE: 证据不足
    RETRIEVING --> ASSEMBLING: 证据满足候选条件
    ASSEMBLING --> GENERATING
    GENERATING --> MODEL_UNAVAILABLE: 调用失败
    GENERATING --> VALIDATING: 收到完整输出
    VALIDATING --> ANSWERED: 结构、主张与引用通过
    VALIDATING --> CONFLICTING_EVIDENCE: 冲突合同命中
    VALIDATING --> INVALID_MODEL_OUTPUT: 校验失败
    RETRIEVING --> CANCELLED
    GENERATING --> CANCELLED
~~~

失败终态不会写入源 vault，也不会把未验证文本显示成最终答案。

## 不变量

实现和测试至少守住以下不变量：

1. 一个查询只读取一个 ACTIVE IndexGeneration。
2. 一个 AnswerRun 只引用其 RetrievalRun 提供的 Evidence。
3. 一个 Evidence 只指向该代次中的活动 SectionChunk。
4. 源文件散列变化后，旧 DocumentRevision 不再是当前版本。
5. 删除源文件后，新活动代次中不存在其可查询切片。
6. Embedding 维度、模型和归一化策略不跨代次混用。
7. FAILED 或 CANCELLED 构建不能替换上一 ACTIVE 代次。
8. 任何运行对象都不能成为修改源 Markdown 的入口。
9. 私人绝对路径和正文默认不进入持久运行日志。
10. “模型说引用存在”不能替代应用校验。

这些不变量应先写成测试，再选择具体数据库或检索库。

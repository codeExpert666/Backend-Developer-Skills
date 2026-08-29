---
title: VaultRAG 总体架构与模块边界
aliases:
  - VaultRAG 架构设计
  - VaultRAG Go 模块边界
tags:
  - 项目实践
  - VaultRAG
  - 架构
  - Go
  - 本地应用
created: 2026-08-29T15:38:49
updated: 2026-08-29T15:38:49
---

本文定义 VaultRAG 的进程形态、模块职责、依赖方向、存储边界和演进路径。项目从 Go 模块化单体与 CLI 开始，不以拆服务或堆叠基础设施证明复杂度。

产品范围见 [[VaultRAG 产品边界与演示场景]]，领域身份见 [[VaultRAG 领域模型与数据契约]]。

> [!note] 状态说明
> 这是待实现架构。具体 Go 包名与第三方库仍需技术验证；图中的职责和依赖规则是实现必须守住的项目合同。

## 架构目标

- 在个人电脑上以一个主要二进制运行，安装、升级和卸载可解释。
- 不向源 vault 写任何状态，派生数据可以整体删除后重建。
- 文件解析、检索、模型和界面通过本地能力合同隔离。
- 关键词搜索不依赖 Ollama；模型故障不会破坏索引。
- 每次查询固定到一个活动索引代次，避免新旧数据混读。
- 测试可以用内存或伪适配器覆盖确定性语义，也能用真实本地依赖做集成验证。
- 后续增加 HTTP 或 UI 时，不把传输 DTO 扩散到领域层。

## 运行时拓扑

~~~mermaid
flowchart TB
    U["用户"] --> CLI["VaultRAG CLI"]
    UI["可选本地 UI"] --> HTTP["回环地址 HTTP Adapter"]
    CLI --> APP["Application Use Cases"]
    HTTP --> APP
    APP --> FS["只读文件系统 Adapter"]
    APP --> META["Manifest / Metadata Store"]
    APP --> MD["Markdown Parser"]
    APP --> LEX["Lexical Index"]
    APP --> VEC["Vector Index"]
    APP --> OLLAMA["Ollama Adapter"]
    OLLAMA --> OD["本机 Ollama Daemon"]
    META --> D["Vault 外部数据目录"]
    LEX --> D
    VEC --> D
    FS --> V["Markdown Vault"]
~~~

只有只读文件系统适配器能够访问 vault。元数据、索引、缓存和运行报告都位于独立数据目录。

## 为什么以 Go 单体开始

Go 是当前候选主语言，原因是：

- 文件扫描、并发 Embedding、取消和 CLI 适合用 Context 与受控 Goroutine 实现。
- 可以发布无需额外语言运行时的跨平台二进制。
- 静态类型有利于固化回答、终态和错误合同。
- 标准库能够覆盖文件、散列、HTTP、JSON、测试和性能分析的主要底座。
- 个人项目可以在一处观察完整链路，降低部署和故障面。

这不是“Go 天生更适合 AI”的结论。模型与检索质量由合同、数据和评测决定，不由语言品牌决定。Python 可以用于离线分析或评测辅助，但不进入首发运行时关键路径。

## 分层与依赖方向

候选代码结构：

~~~text
cmd/vaultrag/
internal/
  domain/
  application/
  ports/
  adapters/
    filesystem/
    markdown/
    metadata/
    lexical/
    vector/
    ollama/
    http/
  platform/
    config/
    logging/
    telemetry/
demo-vault/
evals/
docs/
~~~

目录在代码仓库创建后才能算存在；这里描述职责，而不是要求提前建立空包。

### domain

只保存稳定业务语义：

- SourceDocument、DocumentRevision、SectionChunk 和 IndexGeneration。
- RetrievalQuery、Evidence、AnswerRun 和 EvalCase。
- 状态、错误类别、不变量和纯规则。

domain 不依赖 CLI、HTTP、Ollama DTO、数据库行或具体索引库。

### application

编排用例：

- InitializeProject
- DiagnoseEnvironment
- BuildIndex
- GetStatus
- SearchVault
- AskVault
- RunEvaluation

应用层负责事务式顺序、取消、超时、原子激活和错误映射，不直接解析 Markdown 或发送 HTTP。

### ports

定义最小本地能力合同：

- SourceReader：发现与只读打开源文件。
- MarkdownParser：从字节生成中立文档结构。
- MetadataStore：保存快照、代次和运行元数据。
- LexicalIndex：写入、删除、查询关键词投影。
- VectorIndex：写入、删除、查询向量投影。
- Embedder：文本批量转换为固定维度向量。
- Generator：根据版本化 Prompt 生成结构化回答候选。
- Clock、IDGenerator 和 Reporter：隔离时间、标识和输出。

ports 不暴露某个 SDK 的请求响应类型，原则见 [[模型 SDK、业务适配层与 AI 框架边界]]。

### adapters

适配器负责易变实现：

- 文件系统适配器执行路径规范化、排除规则和只读打开。
- Markdown 适配器处理 Frontmatter、标题、代码块、Wikilink 和行号。
- 元数据适配器处理 Schema、事务和活动代次指针。
- 关键词与向量适配器映射索引库能力。
- Ollama 适配器处理健康检查、模型能力、超时、响应与错误分类。
- CLI / HTTP 适配器只处理参数、传输和展示。

替换适配器不能改变领域终态或把源文件写权限引入应用层。

## 依赖规则

~~~mermaid
flowchart LR
    X["CLI / HTTP"] --> A["Application"]
    A --> D["Domain"]
    A --> P["Ports"]
    Z["Adapters"] --> P
    Z --> D
    P --> D
~~~

禁止：

- domain 导入数据库、Ollama、CLI 或 HTTP 包。
- application 依赖某个索引库的命中对象。
- adapters 绕过应用层直接切换活动代次。
- HTTP Handler 自己拼 Prompt、查索引或校验引用。
- 模型输出直接成为领域对象而不经过解析和校验。

可以用架构测试或 Go 包依赖检查固化这些规则。

## 三条关键用例

### 构建索引

~~~mermaid
sequenceDiagram
    participant U as User
    participant A as BuildIndex
    participant F as SourceReader
    participant P as MarkdownParser
    participant S as Stores
    participant E as Embedder

    U->>A: index
    A->>F: scan read-only files
    F-->>A: source snapshot
    A->>P: parse changed revisions
    P-->>A: chunks and warnings
    A->>S: build staging generation
    A->>E: embed reusable misses
    E-->>A: vectors or classified failure
    A->>S: consistency check
    A->>S: atomically activate
    A-->>U: change summary and generation
~~~

任何步骤失败都保留上一活动代次。详细规则见 [[VaultRAG Markdown 摄取与增量索引设计]]。

### 搜索

1. 读取一个活动 generationId。
2. 规范化问题和过滤器。
3. 并行执行可用召回器。
4. 在应用层融合、去重和控制来源多样性。
5. 返回 Evidence，不调用生成模型。

### 问答

1. 复用 SearchVault 获得固定证据集。
2. 根据上下文预算组装 Prompt。
3. 调用 Generator 获得结构化候选。
4. 执行完整性、Schema、证据标识和主张覆盖校验。
5. 返回回答、拒答、冲突或确定失败终态。

问答不允许跳过 SearchVault 直接把整个 vault 传给模型。

## 元数据与索引存储

项目需要三种能力，不要求由三个服务提供：

| 能力 | 需要的语义 | 候选方向 |
| --- | --- | --- |
| Manifest / Metadata | 事务、唯一约束、代次状态、运行坐标 | 嵌入式 SQLite 或等价本地存储 |
| Lexical Index | 中文与代码词召回、字段权重、排名诊断 | Bleve、SQLite FTS5 或小型自建基线 |
| Vector Index | 固定维度向量、Top-K、删除和重建 | 嵌入式向量库或带向量能力的统一索引 |

实现前用同一 demo vault 验证：

- 许可证与公开项目兼容。
- macOS 与 Linux 目标能稳定构建，是否依赖 CGO 明确。
- 中文分词、精确标识符和短查询表现可测。
- 删除、批量写、崩溃恢复和原子代次可实现。
- 维度不匹配和损坏索引能明确失败。
- 依赖仍处于早期版本时可通过 Adapter 和版本锁隔离。

技术验证完成后写 ADR 锁定选择；项目实践不把候选库写成既成事实。

## 数据目录布局

候选逻辑布局：

~~~text
data-dir/
  metadata/
  generations/
    generation-id/
      lexical/
      vector/
      manifest.json
  cache/
    embeddings/
  runs/
  reports/
  locks/
~~~

要求：

- data-dir 不能位于 vault 内，也不能通过符号链接回到 vault。
- generation 先写临时目录，校验成功后原子发布活动指针。
- 缓存只按内容散列与完整模型坐标复用。
- 清理只能删除明确识别的非活动代次，不能用宽泛路径或未解析变量。
- 运行报告默认保存摘要和散列，不保存全文问题与笔记正文。

## 并发与取消

- 扫描、解析和 Embedding 使用有界 Worker，不按文件数无限创建 Goroutine。
- 所有 I/O 与模型调用接受 Context，取消后等待 Worker 有序退出。
- 同一 data-dir 同时只允许一个写构建；搜索可以继续读取旧活动代次。
- 激活使用短事务或原子指针，不持有跨模型调用的锁。
- 批量 Embedding 受模型能力、上下文和内存预算约束。
- 退出前刷新必要元数据，但不把失败暂存区标记为 READY。

Race Test、泄漏测试和取消集成测试进入 [[VaultRAG 质量评测与发布门禁]]。

## 本地 HTTP 与 UI 边界

HTTP 是 CLI 稳定后增加的适配器：

- 默认只监听回环地址。
- 首先暴露项目自己的 health、status、search 和 ask 合同。
- 若适配第三方聊天界面，只实现和测试所需最小兼容子集，不宣称完整兼容某个开放协议。
- UI 不获得源文件写权限；打开笔记只传递经过编码的相对路径定位。
- 浏览器跨域、远程绑定、认证和流式输出未完成前不开放局域网访问。

Open WebUI 等界面可以作为后续消费者，但不成为核心索引与回答正确性的依赖。

## 演进触发条件

只有出现证据后才考虑拆分或更换基础设施：

| 触发证据 | 可重评方向 |
| --- | --- |
| 单进程索引显著阻塞长期在线查询 | 独立索引 Worker 或后台任务 |
| 嵌入式索引无法承载已定义数据规模 | 外部向量或全文检索服务 |
| 多个真实用户需要隔离与共享 | 认证、授权、租户和服务端部署 |
| 多提供商成为真实维护需求 | 增加供应商 Adapter 和兼容测试 |
| Wikilink 图对固定评测有稳定增益 | 图扩展召回，而非直接引入通用 GraphRAG |

在触发条件前保持模块化单体，以更低维护成本获得更完整的正确性证据。

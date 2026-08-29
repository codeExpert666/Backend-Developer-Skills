---
title: VaultRAG 迭代实施与开源交付路线图
aliases:
  - VaultRAG 实施路线图
  - VaultRAG GitHub 开源计划
tags:
  - 项目实践
  - VaultRAG
  - 实施路线
  - GitHub
  - 开源
created: 2026-08-29T15:38:49
updated: 2026-08-29T15:38:49
---

本文把 VaultRAG 拆成可单独验证、可独立发布的实施阶段，并规定代码仓库、合成数据、文档、CI、Release 和公开表达边界。每个阶段按退出证据推进，不按固定周数赶工。

总体定位见 [[VaultRAG 项目实践总览]]，各版本门禁见 [[VaultRAG 质量评测与发布门禁]]。

> [!important] 当前状态
> 目前只有笔记仓库中的候选设计，没有 VaultRAG 代码仓库、GitHub 项目、许可证决定、Release 或实现证据。路线图中的目录、命令和版本均为待执行计划。

## 实施原则

1. 先建立确定性文件与搜索闭环，再接入生成模型。
2. 每个阶段先写能力合同和失败测试，再完成成功路径。
3. 公开 demo 与私人验收彻底分离。
4. 一个 Go 进程能解决时不拆服务。
5. 每次只引入解决当前阶段问题的依赖。
6. 索引、Prompt 和模型变化都要有重建或回归语义。
7. README 的 implemented / planned 状态由 Release 证据决定。
8. 真实模型、浏览器或跨平台检查未运行时明确披露。

## 阶段依赖

~~~mermaid
flowchart LR
    P0["阶段 0：项目与决策基线"] --> P1["阶段 1：合成语料与领域骨架"]
    P1 --> P2["阶段 2：只读摄取与增量 Manifest"]
    P2 --> P3["阶段 3：关键词搜索 v0.1"]
    P3 --> P4["阶段 4：Embedding 与混合检索"]
    P4 --> P5["阶段 5：带引用回答 v0.2"]
    P5 --> P6["阶段 6：评测、安全与观测 v0.3"]
    P6 --> P7["阶段 7：本地 API 与桌面集成 v0.4"]
    P7 --> P8["阶段 8：稳定发布与维护"]
~~~

阶段 3 已经形成可用产品；不需要等大模型问答完成才发布首版。

## 阶段 0：项目与决策基线

### 目标

创建独立、可公开的代码仓库，并在写业务代码前锁定真实性与依赖选择流程。

### 实施

- 确认 VaultRAG 是否作为最终公开名称，检查 GitHub 名称冲突与商标风险。
- 选择 GitHub 仓库所有者、可见性和默认分支。
- 建立 README 第一屏，明确“本地、只读、证据优先、中文技术 Markdown”。
- 建立 docs/architecture、docs/adr、docs/security 和 docs/reports 的职责。
- 评估 MIT、Apache-2.0 等许可证候选，并完成依赖兼容检查后再决定。
- 创建 CONTRIBUTING、SECURITY、CODE_OF_CONDUCT、Issue / PR 模板候选。
- 配置 Go 版本、模块路径、格式、Lint、测试和最小 CI。
- 建立“不提交私人 vault、派生索引、模型文件、日志和绝对路径”的 ignore 与检查。

### 首批 ADR

| ADR | 要回答的问题 |
| --- | --- |
| 0001 | 为什么 Go 模块化单体与 CLI 先行 |
| 0002 | 元数据存储采用什么，事务和迁移如何实现 |
| 0003 | 关键词索引实现与中文 / 标识符策略 |
| 0004 | 向量索引实现、许可证和跨平台构建 |
| 0005 | 只读根目录、数据目录和符号链接策略 |
| 0006 | 回答 Schema、引用身份和拒答终态 |

ADR 可以先处于 Proposed，技术验证后才标记 Accepted。

### 失败验证

- 提交一个私人绝对路径或 demo 外 Markdown，秘密 / 隐私检查应失败。
- README 引用不存在的命令时，文档 smoke 应失败。
- 未选择许可证时不发布二进制 Release。

### 退出标准

- 仓库可以 fresh clone、构建和运行空测试。
- 公开定位、非目标、数据声明和当前未实现项清楚。
- CI 不依赖私人文件、本地 Ollama 或密钥。
- 依赖技术验证任务和 ADR 有明确所有者与准入标准。

## 阶段 1：合成语料与领域骨架

### 目标

先把项目合同和评测问题固化为代码可消费资产。

### 实施

- 独立编写 demo-vault，覆盖产品文档定义的 Markdown 与 Obsidian 结构。
- 为每个文件标记 synthetic 示例，不复制个人笔记句子。
- 创建 exact、paraphrase、cross-document、no-answer、conflict、mutation 和 injection 用例。
- 实现领域对象、状态枚举、错误类别和版本 fingerprint。
- 定义 SourceReader、MarkdownParser、MetadataStore、LexicalIndex、VectorIndex、Embedder 和 Generator Port。
- 建立 fake Adapter，先测试 BuildIndex、SearchVault 和 AskVault 状态机。
- 固化 config Schema 与未知字段失败策略。

### 失败验证

- 重复 sourceId、非法相对路径和未知终态被拒绝。
- FAILED / CANCELLED generation 不能成为 ACTIVE。
- AnswerRun 引用不存在 Evidence 时失败。
- 测试夹具扫描不依赖文件遍历随机顺序。

### 退出标准

- 领域不变量由 fast test 覆盖。
- demo vault 和 eval Schema 可被验证器完整读取。
- 所有公开内容通过私人路径、秘密和版权人工检查。
- 代码尚未支持真实索引时，README 仍明确标记 planned。

## 阶段 2：只读摄取与增量 Manifest

### 目标

建立不依赖模型的可靠源快照、Markdown 解析、切片和原子代次。

### 实施

- 实现规范路径、排除规则、文件限制和默认不跟随符号链接。
- 计算源字节散列，建立 Snapshot、SourceDocument 和 Revision。
- 通过中立 AST 解析 Frontmatter、标题、代码块、Callout 和 Wikilink。
- 实现标题感知切片、行号、内容散列与确定性 chunkId。
- 选定并实现 MetadataStore Schema、迁移和写锁。
- 实现新增、修改、删除、重命名、解析合同升级和取消。
- 建立 staging generation 与原子活动指针。
- 完成 init、doctor、index 和 status 的最小 CLI。

### 失败验证

- dataDir 位于 vault 内或通过符号链接落入时拒绝。
- 非法 UTF-8、超大文件、坏 Frontmatter 和未闭合围栏有稳定结果。
- 扫描中修改文件、进程取消和存储失败不激活半成品。
- 删除与重命名后 Manifest 不保留活动旧 Revision。
- 完整流程前后源文件内容和目录项不变。

### 退出标准

- 同输入连续构建得到相同 Revision 与 Chunk。
- 完整构建和等价增量构建结果一致。
- 上一活动代次在失败和取消后仍可读取。
- CLI 输出变更摘要、警告、失败类别和 generationId。
- 摄取设计满足 [[VaultRAG Markdown 摄取与增量索引设计]]。

## 阶段 3：关键词搜索与 v0.1

### 目标

交付第一个不依赖 Ollama 的实际可用版本。

### 实施

- 完成关键词库技术验证和 ADR。
- 索引 title、aliases、headingPath、body、code、tags 与 relativePath。
- 固化中文、英文和技术标识符分析策略。
- 实现 search mode lexical、过滤器、Top-K 和 JSON / 人类可读输出。
- 返回 Evidence 的路径、标题、行号、版本和命中诊断。
- 增加 exact、code-context、no-result 和 mutation eval。
- 编写 fresh clone、索引 demo、搜索和清理说明。

### 失败验证

- 配置键、错误码、数字和路径片段不因 tokenizer 丢失。
- 搜索只读取一个活动 generation。
- 删除后的内容无法命中。
- 损坏关键词索引可诊断并重建，不返回随机结果。
- 无结果与索引未就绪使用不同退出码和状态。

### 退出标准

- v0.1 门禁全部通过。
- demo vault 的关键 exact 用例达到确认后的基线。
- Release 制品和校验和可在目标平台运行。
- README 第一屏明确 v0.1 只有确定性搜索，没有 RAG 问答。

## 阶段 4：Embedding 与混合检索

### 目标

在保持精确召回的同时增加中文语义召回。

### 实施

- 完成向量实现技术验证和 ADR，确认许可证、构建与删除语义。
- 实现 Ollama Embedder、批次、超时、取消和能力探针。
- 定义 embeddingProfile、输入模板、缓存键和重建条件。
- 建立向量索引、维度校验、内容散列缓存和全量重建。
- 并行执行关键词与向量召回。
- 实现 RRF、重叠去重、来源多样性、邻接扩展和 Evidence 预算。
- search 支持 lexical、vector 与 hybrid 对照输出。
- 增加 paraphrase、cross-document、降级和模型切换 eval。

### 失败验证

- Embedding 模型不存在、超时、维度变化和非法向量分别失败。
- 向量不可用时只在明确模式下降级为 lexical。
- 新旧向量 profile 不混用。
- 缓存键缺少任一版本坐标时测试失败。
- 混合融合不因不同原始分数尺度失真。

### 退出标准

- 固定 eval 显示 hybrid 相对 lexical 的真实变化，包含退化用例。
- 删除、更新和重建后两个召回通道一致。
- 参数、模型和索引版本可以在报告中定位。
- 没有稳定增益时允许暂停向量发布，而不是为了路线图强行启用。

## 阶段 5：带引用回答与 v0.2

### 目标

把固定 Evidence 转换为经过程序校验的回答、拒答或冲突终态。

### 实施

- 实现 Ollama Generator 本地能力合同。
- 固化 Prompt、Answer Schema 和上下文预算。
- 实现 Evidence 组装、完整响应、JSON、Schema、引用和主张校验。
- 由应用渲染相对路径、headingPath、行号和 revision。
- 实现 ANSWERED、INSUFFICIENT_EVIDENCE、CONFLICTING_EVIDENCE、MODEL_UNAVAILABLE 与 INVALID_MODEL_OUTPUT。
- ask 支持 JSON 与适合终端阅读的 Markdown 输出。
- 使用当前已有本地聊天模型建立主模型与对照模型的独立 eval；精确标签和硬件写入运行记录。
- 增加无答案、冲突、伪引用、非法结构和取消用例。

### 失败验证

- 模型引用 E999、绝对路径或未提供文件时被阻断。
- ANSWERED 中无证据主张不能通过。
- 资料内“忽略规则”不能改变输出合同。
- Ollama 停止后不回退为无引用普通聊天。
- 截断、非法 JSON 和超时不展示部分答案。

### 退出标准

- v0.2 候选门槛经 baseline 确认并通过。
- 每条显示引用均可打开或手工定位到源 Revision。
- 拒答和冲突在 demo 中稳定复现。
- 真实模型报告与确定性测试分开展示。

## 阶段 6：评测、安全、观测与 v0.3

### 目标

把项目从“能演示”提升为“变更后知道是否退化”。

### 实施

- 完成 eval CLI、数据版本、报告 Schema 和前后对比。
- 落地 Recall@K、MRR、引用有效性、主张覆盖、拒答和冲突指标。
- 建立人工抽样量表和可选独立 judge，不让 judge 决定硬安全。
- 增加阶段耗时、计数、状态和版本坐标观测。
- 固化 Prompt、模型、切片和检索变更回归。
- 完成只读、路径、符号链接、日志、注入、清理和供应链安全集。
- 建立性能基线、取消、Race、Fuzz 和恢复演练。
- 编写威胁模型、隐私说明、诊断包和漏洞报告流程。

### 失败验证

- 一个评测器故意给正确 / 错误样例时能分别判定。
- SKIP 或模型不可用不显示为 PASS。
- 默认日志、报告和制品无法找到私人正文与绝对路径。
- 非回环地址、越界路径和宽泛清理目标被拒绝。
- Prompt 或模型变化导致质量退化时门禁失败。

### 退出标准

- v0.3 门禁通过并保存完整版本坐标。
- 当前硬门槛、软指标和已知限制在 README 可见。
- 同一 Release 的 demo eval 可以由他人复跑。
- 安全边界满足 [[VaultRAG 安全、隐私与只读边界]]。

## 阶段 7：本地 API、Obsidian 定位与 v0.4

### 目标

在核心语义稳定后增加本地消费入口，而不重写应用层。

### 实施

- 实现 health、status、search、ask 和 eval report 的本地 HTTP Adapter。
- 默认绑定回环地址，加入请求大小、并发、超时、取消与 CORS 边界。
- 为 Obsidian 文件打开生成经过验证和编码的本地 URI。
- 在不支持标题直达时保留相对路径、headingPath 与行号回退。
- 选择一个第三方 UI 做最小兼容 Spike；只有真实需求与契约测试通过才纳入版本。
- 评估是否需要 SSE 或流式输出；完整验证前只显示临时状态。
- 增加 HTTP 契约、浏览器与桌面 smoke。

### 失败验证

- 0.0.0.0 与非回环监听默认拒绝。
- 模型输出的路径不能驱动桌面打开动作。
- 超大请求、断开连接和取消能释放资源。
- 第三方 UI 不兼容时返回明确错误，不宣称完整协议兼容。
- CLI 与 HTTP 对同一用例产生同一领域终态。

### 退出标准

- v0.4 门禁通过。
- 受支持平台实测 Obsidian 文件定位。
- 第三方 UI 是可选消费者，核心 CLI 无需它也能完整工作。
- HTTP 暴露面、已知限制和关闭方式写清。

## 阶段 8：稳定发布与维护

### 目标

形成长期可维护的开源项目，而不是一次性演示仓库。

### 实施

- 定义语义化版本、支持平台和兼容窗口。
- 自动构建 macOS 与 Linux 候选架构制品，并在真实环境 smoke。
- 生成校验和、SBOM、依赖与许可证报告。
- 固化配置、元数据 Schema、索引重建和升级说明。
- 建立 Changelog、弃用策略、已知问题和安全发布流程。
- 用 Issue 模板收集脱敏 doctor 输出，不要求用户上传 vault。
- 建立小型公开 Roadmap，区分 committed、exploring 和 not planned。
- 对依赖升级执行契约、eval、性能和跨平台回归。

### 重评方向

只有证据支持时才讨论：

- 文件监听与后台自动同步。
- 新模型供应商。
- Graph / Wikilink 扩展召回。
- 更多文档格式。
- 编辑器插件。
- 团队与远程服务。

每个方向先写产品问题、隐私变化、维护成本和退出条件。

### 退出标准

- 最近 Release 可以从全新环境安装、运行 demo、验证和卸载。
- 用户能安全删除派生数据且不会影响 vault。
- Issue、PR、安全报告和 Release 过程可重复。
- 维护范围与项目作者可承受投入一致。

## GitHub 仓库建议结构

候选结构：

~~~text
.
├── cmd/vaultrag/
├── internal/
├── demo-vault/
├── evals/
├── docs/
│   ├── architecture/
│   ├── adr/
│   ├── reports/
│   └── security/
├── scripts/
├── .github/
│   ├── workflows/
│   ├── ISSUE_TEMPLATE/
│   └── pull_request_template.md
├── README.md
├── CONTRIBUTING.md
├── SECURITY.md
├── CODE_OF_CONDUCT.md
├── CHANGELOG.md
├── LICENSE
├── go.mod
└── go.sum
~~~

只有阶段需要时才创建目录和文件，不为架构图提前提交空包。

## README 第一屏

README 应让访问者在一分钟内知道：

- VaultRAG 解决什么问题。
- 为什么是本地、只读和证据优先。
- 当前最新 Release 真正实现了什么。
- 需要哪些本地条件。
- 最短 demo 命令。
- 数据全部合成，个人 vault 不在仓库。
- 关键限制和 Roadmap 链接。

建议使用能力矩阵：

| 能力 | 状态 | 证据 |
| --- | --- | --- |
| Markdown 扫描与增量索引 | planned / implemented | 测试或 Release 链接 |
| 关键词搜索 | planned / implemented | eval 报告 |
| 混合检索 | planned / implemented | 对照报告 |
| 带引用问答 | planned / implemented | answer eval |
| 只读与路径安全 | planned / implemented | 安全测试 |
| 本地 HTTP / Obsidian 定位 | planned / implemented | smoke 报告 |

发布前不允许手工把 planned 改成 implemented 而没有对应门禁证据。

## 开源数据与隐私

公开仓库只包含：

- 独立编写的合成 demo vault。
- 合成问题、期望来源与攻击夹具。
- 脱敏配置和运行报告。
- 不含私人路径的截图或终端录屏。

禁止包含：

- 个人 Obsidian vault 的任何原文、目录、标题或别名。
- 真实公司名称、内部系统、同事、客户和业务数据。
- 本机用户名、绝对路径、主机名、动态地址和模型缓存。
- 真实 Prompt / 回答日志与派生向量。
- 受限版权资料或从其他项目复制的大段内容。

公开表述统一使用：

> 受个人技术笔记场景启发，使用合成示例仓库独立设计与实现。

## CI 与 Release 策略

### Pull Request

- 格式、Lint、单元、架构、Fuzz seed 和文档检查。
- 临时文件与嵌入式存储集成。
- demo vault 确定性端到端。
- 秘密、依赖和许可证检查。
- 不要求 Ollama 或私人 vault。

### 主分支与定期任务

- Race、长时 Fuzz、跨平台构建和性能趋势。
- 允许在受控自托管环境运行 local-live 模型评测，但不接触私人数据。
- 定期依赖升级必须附回归报告。

### Release Candidate

- 所有目标平台构建和真实制品 smoke。
- demo eval、安全、性能和兼容报告。
- Changelog、已知限制、升级与重建说明。
- SBOM、校验和、签名或来源证明候选。
- 人工核对 README 能力矩阵与 CLI help。

发布工作流使用不可变 Tag 和可定位 commit；不能在服务器或 Release 页面临时手工重建不同制品。

## 公开演示与项目表达

候选项目标题：

> VaultRAG——面向中文技术 Markdown 的本地只读证据问答引擎

只有相应版本完成后，简介才可以逐步增加：

- v0.1：增量 Markdown 索引与确定性关键词搜索。
- v0.2：Ollama Embedding、混合检索、主张级引用与拒答。
- v0.3：固定评测、安全失败集、版本化 Prompt 与运行观测。
- v0.4：本地 HTTP、Obsidian 文件定位与跨平台 Release。

不要写：

- “准确率 95%”而不说明数据集、指标和版本。
- “支持任意 Obsidian 仓库”而没有边界夹具。
- “完全离线、绝对安全”而忽略安装网络、依赖和本地进程风险。
- “OpenAI 完全兼容”而只实现少量字段。
- “生产级知识库”而没有真实多用户、容量和运维证据。
- “Agent 自动管理笔记”，因为项目明确只读且不执行工具。

## 演示证据清单

一个成熟 Release 的演示应能打开：

- 当前架构图与能力矩阵。
- demo vault 和合成声明。
- 全量与增量索引结果。
- exact、paraphrase、cross-document、no-answer 与 conflict 用例。
- 一条回答的 RetrievalRun、Evidence、Claim 与 Citation 映射。
- Prompt Injection、路径逃逸和源写入失败测试。
- eval 与性能报告的完整版本坐标。
- Release 制品、校验和、SBOM 和已知限制。

演示只展示当前版本，不播放未来能力的预制假结果。

## 项目复盘模板

每个里程碑结束回答：

1. 这一阶段解决了什么用户问题？
2. 哪些不变量和失败路径新增？
3. 哪个依赖候选被选择，证据与代价是什么？
4. 固定 eval 哪些类别提升、退化或不变？
5. 真实模型和硬件结果适用范围是什么？
6. 是否发现私人数据、路径或日志风险？
7. README 哪些 planned 可以改为 implemented？
8. 哪个复杂能力仍不值得引入，重评触发条件是什么？

复盘结论进入代码仓库报告或 ADR；本笔记仓库只保留长期设计与可定位执行摘要。

## 开工前确认清单

在实际创建 GitHub 项目前确认：

- 最终项目名与仓库名。
- 许可证候选和依赖兼容策略。
- 支持平台与 Go 最低版本候选。
- 元数据、关键词与向量实现的技术验证顺序。
- v0.1 是否只包含关键词搜索。
- demo vault 主题和内容原创检查。
- 私人 vault 的本地验收方式与不落盘规则。
- Release 前是否由用户显式确认公开发布。

这些决定会影响外部状态或维护承诺，因此在代码实施阶段应由用户确认；未确认前可以完成本地 Spike，但不创建公开 Release。

## 最短可行动路径

1. 创建本地私有或未发布的 Go 仓库骨架。
2. 独立写一个十余篇合成 Markdown 的最小 demo vault。
3. 写 SourceDocument、Revision、Chunk 和 Generation 不变量测试。
4. 实现只读扫描、内容散列和确定性标题切片。
5. 实现 Manifest 与新增、修改、删除测试。
6. 选择关键词基线并交付 search。
7. 通过 v0.1 门禁后再接 Ollama Embedding。

这条路径让第一个真正可用成果尽早出现，也为后续 RAG 提供可验证底座。

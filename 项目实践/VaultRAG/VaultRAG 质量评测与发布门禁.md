---
title: VaultRAG 质量评测与发布门禁
aliases:
  - VaultRAG 评测方案
  - VaultRAG 发布门禁
tags:
  - 项目实践
  - VaultRAG
  - 评测
  - 测试
  - 发布门禁
created: 2026-08-29T15:38:49
updated: 2026-08-29T15:38:49
---

本文定义 VaultRAG 从一个改动到可发布版本需要哪些测试、评测、性能、安全和复现证据。通用方法见 [[AI 评测、版本化与发布门禁]]。

> [!important] 当前状态
> 下列门禁是候选质量合同，不是已通过结果。首个基线运行前，不在 README、简历或演示中写固定准确率、延迟或资源数字。

## 证据分层

| 证据层 | 可以证明 | 不能证明 |
| --- | --- | --- |
| 格式与静态检查 | 代码、配置、文档和 Schema 满足基础规则 | 运行行为正确 |
| 单元测试 | 纯规则、状态机、切片和融合在固定输入下正确 | 真实文件系统、索引与 Ollama 行为 |
| 契约测试 | Port、Schema、CLI / HTTP 与 Adapter 理解相同语义 | 第三方服务在所有环境稳定 |
| 集成测试 | 真实临时文件、嵌入式存储和伪 / 真 HTTP 能协作 | 完整用户链路和模型质量 |
| 端到端测试 | 从扫描到搜索、回答和引用的一条链路可重放 | 所有边界和硬件性能 |
| 检索评测 | 固定查询是否召回期望来源与排名 | 最终回答一定忠实 |
| 回答评测 | 固定模型与证据下的状态、主张和引用表现 | 概率模型永不退化 |
| 安全失败测试 | 已建模路径、写入、注入和泄漏场景被阻断 | 不存在未知漏洞 |
| 性能测试 | 指定 revision、数据、模型和硬件下的资源与延迟 | 其他机器或更大规模 |
| 人工演示 | 他人能观察当前版本的成功和失败链路 | 项目已用于真实生产 |

README 中的每个质量数字都应链接或定位到相应证据层。

## Definition of Done

一个能力只有同时满足适用项才算完成：

1. 产品输入、输出、非目标、终态和恢复动作已明确。
2. 领域合同、CLI / HTTP Schema 与实现同步。
3. 正常、边界、失败、取消和不兼容路径有自动化测试。
4. 日志、指标和运行 ID 可以定位阶段，不默认泄露正文。
5. 文件路径、只读、网络、Prompt Injection 和秘密经过安全审查。
6. 影响检索、Prompt 或模型的改动执行固定 eval。
7. 性能敏感改动有同环境基线与对比，不只报告平均值。
8. fresh clone 的构建、demo 和验证命令可运行。
9. CI 在干净环境通过，跳过项明确显示 SKIP。
10. 公开文档只把当前 Release 已通过门禁的能力写成完成。

“能够调用 Ollama”“某个问题回答正确”或“单元测试通过”都不单独等于完成。

## 测试组合

### 领域与应用层

| 类型 | 重点 |
| --- | --- |
| 表驱动单元测试 | 状态机、变化分类、ID、版本兼容、错误映射、融合和预算 |
| 属性测试 | 同输入确定性、排序稳定、集合不变量、路径规范化 |
| Fuzz | Markdown、Frontmatter、Wikilink、路径、JSON 回答和配置 |
| Race Test | 有界 Worker、并发查询、取消、活动代次切换和关闭 |
| 泄漏测试 | 取消和错误后 Goroutine、文件句柄与临时目录是否释放 |

### 文件与 Markdown

- 临时 vault 中的普通文件、隐藏目录、非法编码和大文件。
- 标题、代码块、表格、Callout、Frontmatter 与 Wikilink 夹具。
- 符号链接内链、外链、循环、断链与扫描中替换。
- 内容散列、重复构建、修改、删除、重命名和解析器升级。
- 完整运行前后的文件路径与内容散列不变。

### 存储与索引

- 从空状态建库、Schema 升级和损坏数据诊断。
- 暂存 generation、READY 校验、ACTIVE 原子切换和 RETIRED 清理。
- 构建中断、磁盘不足、重复启动和写锁冲突。
- 关键词字段、中文与标识符召回。
- 向量维度、模型坐标、删除、缓存复用和全量重建。
- 上一活动代次在新构建失败时仍可查询。

### Ollama Adapter

- 健康检查、模型不存在、非法地址和服务不可达。
- Embedding 数量、顺序、维度、NaN / Inf 和批次错误。
- 生成完整、超时、取消、截断、非法 JSON 与未知 Schema。
- 主张无引用、引用不存在、伪造路径和状态矛盾。
- 使用伪 HTTP Server 的确定性契约测试。
- 标记为 local-live 的真实 Ollama 测试，不在普通 CI 假装运行。

### CLI 与 HTTP

- 参数、配置优先级、未知字段、帮助和稳定退出码。
- 人类可读输出与可选 JSON 输出的契约。
- search 与 ask 的不同依赖和失败语义。
- serve 只绑定回环地址，请求大小、超时和取消。
- HTTP Adapter 不绕过应用层或直接访问源文件。

## 确定性测试与真实模型评测分离

测试分为三条轨道：

### fast

- 不需要 Ollama、网络或平台服务。
- 使用合成夹具、内存 Port 或伪 HTTP Server。
- 每个 PR 必跑。
- 失败表示代码或合同确定性错误。

### integration

- 使用真实临时文件和选定嵌入式存储。
- 可以启动本地伪 Ollama 协议服务。
- 每个 PR 或主分支必跑，时间上限明确。

### local-live

- 依赖开发者已安装的 Ollama 和锁定模型。
- 执行真实 Embedding、生成、性能与概率评测。
- 手工或受控发布候选运行。
- 报告不得合并成普通 CI 的绿色状态。

真实模型波动不能成为确定性单元测试偶发失败的借口。

## Eval 数据集

公开 eval 只绑定合成 demo vault。建议分层：

| 类别 | 用例目标 |
| --- | --- |
| exact | 命令、配置键、错误码、路径片段和数字 |
| paraphrase | 中文同义改写与不同标题表述 |
| cross-document | 必须组合两篇或更多来源 |
| code-context | 代码与解释需要同时召回 |
| no-answer | 仓库没有答案，必须拒答 |
| conflicting | 来源明确冲突，必须报告差异 |
| temporal | 旧版与当前说明需要区分 |
| mutation | 新增、修改、删除和重命名后的结果 |
| injection | 资料内指令、伪来源与数据外带诱导 |
| degradation | Embedding、模型或某索引不可用 |

每条用例至少包含 caseId、question、固定 vault revision、expectedStatus、期望来源或切片特征、必需主张、禁止主张和判定策略。

### 私有验收集

可以在个人 vault 上维护不公开的 smoke / eval 集，用于确认真实语料价值。规则：

- 问题、答案、路径和正文不进入公开仓库或云端 CI。
- 只保存聚合结果、失败类别和本地运行坐标。
- 公开项目不能把私人集结果与 demo 集结果混为一个数字。
- 私有集用于发现产品缺口，不成为无法复核的营销证据。

## 检索指标

### Recall@K

期望来源或切片是否至少一个进入前 K。对于多来源问题，分别报告：

- any-source Recall@K。
- all-required-sources Recall@K。
- 按类别的 Recall@K。

### MRR

第一个相关结果出现位置的倒数均值，适合精确问答和单主要来源用例。

### nDCG@K

当用例有多级相关性时，衡量排序质量。标注规则必须先固定，不能为了结果临时调整等级。

### 索引新鲜度

- 新增内容首次可查时间。
- 修改后旧 Revision 残留数。
- 删除后任一召回通道残留数。
- 完整与增量构建结果差异。

旧内容残留是正确性失败，不用平均召回率抵消。

## 回答与引用指标

| 指标 | 定义 |
| --- | --- |
| status accuracy | 最终 ANSWERED、拒答或冲突终态是否正确 |
| citation validity | 引用 evidenceId 是否真实且属于当前运行 |
| claim citation coverage | 可核查主张中至少有引用的比例 |
| citation precision | 引用内容是否实际支持对应主张 |
| unsupported claim rate | 没有证据支持的事实主张比例 |
| required claim coverage | 参考合同中的关键点是否覆盖 |
| refusal precision / recall | 应拒答与不应拒答用例是否正确 |
| conflict detection | 冲突用例是否保留各方来源并返回冲突 |
| instruction resistance | 恶意资料是否改变系统边界或导致外带 |

citation validity 可以确定性判断；citation precision、事实忠实度和帮助程度需要规则、人工量表或独立评审组合。

## 评测器本身的验证

- 先用故意正确和故意错误样例验证评分代码。
- 期望来源使用 sourceId / chunk 特征，不依赖易变显示文本。
- 人工量表定义 0～3 或等价清晰等级，并提供边界示例。
- 可选 LLM judge 只用于软质量，不负责路径、权限和引用真实性。
- Judge 看到相同证据与候选，不访问私人 vault 或互联网。
- 报告人工与模型判定分歧，不把评审模型输出当绝对真值。
- 评分代码、Prompt 和数据集与结果一起版本化。

## 候选发布门槛

以下是首个 baseline 后需要确认或调整的候选，不是当前成绩：

| 维度 | 候选门槛 | 性质 |
| --- | --- | --- |
| 确定性测试 | 适用测试 100% 通过 | 硬门禁 |
| 源内容写入 | 0 个新增、修改、删除或重命名 | 硬门禁 |
| 路径逃逸 | 0 个成功越界读取或清理 | 硬门禁 |
| 陈旧切片 | 删除 / 修改后 0 个旧切片可召回 | 硬门禁 |
| citation validity | 100% | 硬门禁 |
| 注入导致越权或伪造动作 | 0 个 | 硬门禁 |
| Recall@5 | demo eval 总体不低于 90%，关键 exact 用例 100% | 待 baseline 校准 |
| all-required-sources Recall@10 | 跨文档用例不低于 85% | 待 baseline 校准 |
| claim citation coverage | 不低于 95% | 待 baseline 校准 |
| no-answer 正确拒答 | 不低于 95% | 待 baseline 校准 |
| conflict detection | 固定冲突集 100% | 候选硬门禁 |
| unsupported claim rate | 关键用例为 0，总体单独报告 | 候选硬门禁 |

性能不在没有基线时编造固定阈值。首轮只建立可重复报告，再根据目标硬件和 vault 规模确定预算。

## 性能与资源报告

每次性能结果记录：

- 代码 revision、构建参数、Go 与依赖版本。
- 操作系统、架构、CPU、内存、磁盘和电源状态。
- demo / synthetic vault revision、文件数、字节、Chunk 数和变化比例。
- Ollama、模型完整标签、量化、上下文与并发。
- 冷启动、预热、重复次数和异常值处理。
- P50、P95、P99、吞吐、错误率、取消时间和峰值资源。

分阶段场景：

| 场景 | 指标 |
| --- | --- |
| 全量扫描与解析 | 文件 / 秒、字节 / 秒、峰值内存、失败数 |
| 单文件增量 | 变更检测、切片、索引和激活耗时 |
| 批量 Embedding | Chunk / 秒、模型加载、内存和失败分类 |
| 关键词搜索 | P50 / P95、Top-K 和索引大小 |
| 混合搜索 | 查询 Embedding、双召回、融合与总延迟 |
| ask | 检索、组装、首响应、生成、校验和总延迟 |
| 取消 | Worker 与模型请求停止时间、残留资源 |
| 重建与恢复 | 旧代次可用性、空间峰值和激活时间 |

不能把生成数据上的结果写成真实用户规模或生产容量。

## 可观测性

一次 ask 的 Trace 或等价阶段记录应还原：

~~~mermaid
flowchart LR
    Q["Question"] --> R["Retrieve"]
    R --> F["Fuse and Select"]
    F --> P["Prompt Build"]
    P --> M["Model"]
    M --> V["Validate"]
    V --> A["Answer Status"]
~~~

共享字段：

- runId、generationId 和 config fingerprint。
- parser、chunker、retrieval、embedding、Prompt、Schema 与模型版本。
- 候选数、Evidence 数、引用数和状态。
- 各阶段耗时、取消、重试和降级。

标签不包含完整问题、路径、正文或高基数错误文本。

## CI 门禁顺序

~~~mermaid
flowchart LR
    F["格式、Lint 与文档检查"] --> U["单元、Fuzz Seed 与架构测试"]
    U --> I["临时文件与真实存储集成"]
    I --> C["CLI、HTTP、Schema 与 Adapter 契约"]
    C --> S["秘密、依赖、许可证与静态安全"]
    S --> E["demo vault 端到端与确定性 eval"]
    E --> B["跨平台构建"]
    B --> A["制品、SBOM、校验和与报告"]
~~~

- local-live 模型评测在受控环境运行，不在 Fork PR 暴露私人资源。
- 定期 Fuzz、Race、长时稳定和性能任务与 PR 快速门禁分开。
- SKIP、UNAVAILABLE 和 DEGRADED 不显示为 PASS。
- CI 成功不自动证明真实模型 eval 已运行。

## 版本门禁

### v0.1：确定性索引与搜索

- 扫描、解析、Manifest、增量和关键词搜索测试通过。
- 删除、重命名、取消、失败保留旧代次与只读验收通过。
- demo vault 和 exact / mutation eval 可重放。
- fresh clone 可以构建并执行主线。

### v0.2：混合检索与带引用问答

- Embedding 坐标、向量重建和降级合同通过。
- 混合检索 eval 与候选门槛有 baseline 依据。
- 回答 Schema、引用有效性、拒答、冲突和模型失败通过。
- 真实模型报告单独可定位。

### v0.3：质量工程

- eval CLI、报告 Schema、观测和性能基线稳定。
- 注入、路径、日志、清理与供应链门禁通过。
- Prompt、模型和检索变更有回归比较。
- 已知限制和恢复说明完整。

### v0.4：本地 API 与开源交付

- 回环 HTTP、安全限制和最小兼容契约通过。
- Obsidian 文件定位在受支持平台实测。
- 跨平台 Release、校验和、SBOM 与安装说明通过。
- 公开演示与 README 能从 Release 复现。

## 每次 Release 保存的证据

- Tag、完整 commit、构建工作流和干净 Git 状态。
- Go、依赖锁、配置 Schema 和数据迁移版本。
- parser、chunker、lexical、embedding、Prompt 与 answer Schema 版本。
- demo vault 与 eval suite revision。
- 单元、集成、端到端、安全、检索和回答评测摘要。
- 性能环境与结果，未运行项目明确列出。
- SBOM、许可证、漏洞、秘密和制品校验和。
- 已知限制、升级、回滚和派生数据重建说明。

原始私人日志与个人 vault 结果不进入 Release 证据。

## 发布前人工检查

- 从全新临时目录按 README 完成安装和 demo。
- 对照 help 检查 README 中的命令和参数。
- 打开每个引用，核对路径、标题、行号和 Revision。
- 验证无答案、冲突、Ollama 停止和索引失败体验。
- 搜索仓库、Git 历史、制品和报告中的私人路径与秘密。
- 确认当前能力矩阵没有把 roadmap 写成 implemented。
- 在受支持平台至少各完成一次真实制品 smoke test。

发布速度不能绕过硬安全和证据门禁。

---
title: VaultRAG Ollama 模型适配与回答契约
aliases:
  - VaultRAG Ollama 适配
  - VaultRAG 回答契约
tags:
  - 项目实践
  - VaultRAG
  - Ollama
  - 结构化输出
  - Prompt
created: 2026-08-29T15:38:49
updated: 2026-08-29T15:38:49
---

本文定义 VaultRAG 如何把 Ollama 的 Embedding 与文本生成能力映射为稳定本地合同，以及模型输出怎样经过完整性、Schema、证据和业务校验后成为回答。

检索证据来自 [[VaultRAG 混合检索与引用证据链设计]]；通用适配原则见 [[模型 SDK、业务适配层与 AI 框架边界]] 和 [[结构化输出的 Schema 与业务校验]]。

> [!note] 状态说明
> Ollama-first 是项目方向，具体模型标签、量化、上下文和性能需在代码仓库的日期化执行记录中锁定。当前没有已完成的 VaultRAG 模型集成或质量结果。

## 模型在系统中的权限

模型可以：

- 把固定 Evidence 综合为自然语言回答候选。
- 把回答拆成主张并选择应用提供的 evidenceId。
- 在明确 Schema 中返回拒答、冲突或不确定状态候选。
- 按给定语气和格式组织文本。

模型不可以：

- 扫描文件系统或自行读取 vault。
- 决定可访问路径、检索范围和数据目录。
- 生成可被信任的文件名、行号、源版本或证据标识。
- 执行笔记中的命令、链接或指令。
- 修改、创建、重命名或删除笔记。
- 用预训练常识补齐“仓库里怎么写”的事实。
- 绕过 Schema、引用、隐私和安全校验。

模型是生成组件，不是权限主体、索引数据库或事实裁判。

## 两类本地能力合同

### Embedder

候选接口语义：

~~~text
Embed(ctx, request) -> vectors or classified error

request:
  texts
  model profile
  input template version
  expected dimension

response:
  vectors
  model identity
  dimension
  duration
~~~

适配器必须：

- 支持批量输入和有界批次。
- 校验返回数量、顺序、维度与有限数值。
- 区分模型不存在、服务不可达、超时、取消、输入过长与非法响应。
- 记录可重放模型坐标，不把供应商响应 DTO 传入领域层。
- 不在失败时返回零向量或静默截断。

Ollama 的 Embedding API 是当前实现入口，官方资料见 [Ollama Embed API](https://docs.ollama.com/api/embed)。

### Generator

候选接口语义：

~~~text
Generate(ctx, request) -> answer candidate or classified error

request:
  question
  evidence list
  answer schema version
  prompt version
  generation profile

response:
  complete structured candidate
  model identity
  finish state
  usage and timing when available
~~~

Generator 只返回候选，不直接返回已验证 AnswerRun。

## 启动前能力检查

doctor 命令和首次调用应检查：

- Ollama 地址是否为允许的本地地址。
- 服务是否可达且响应版本可识别。
- 配置的 Embedding 与生成模型是否存在。
- Embedding 返回维度是否与活动索引一致。
- 生成模型是否能完成最小结构化输出探针。
- 请求超时、上下文预算和并发上限是否有效。
- 模型切换是否要求重建向量或只需新回答版本。

能力检查失败不能自动下载大模型。下载会消耗网络、磁盘与时间，应由用户显式执行并在文档中说明。

## 模型 Profile

代码不硬编码模型名称。一个完整 profile 至少包含：

| 字段 | 作用 |
| --- | --- |
| provider | v0.x 为 ollama |
| model | 完整本地模型标签 |
| role | embedding、generation 或 optional-review |
| contextBudget | 应用允许使用的总上下文预算 |
| outputBudget | 为结构化答案保留的输出预算 |
| temperature 等参数 | 受支持且进入版本 fingerprint 的推理参数 |
| timeout | 单次调用时限 |
| maxConcurrency | 防止本地内存抖动的并发上限 |
| promptVersion / inputTemplateVersion | 生成或向量输入合同 |

当前本机已有的两个聊天模型可在实现后分别作为主生成与对照评测候选，但不把一次机器上的安装状态写成项目默认保证。Embedding 使用独立模型；精确标签、版本和基准数据应放入代码仓库执行记录。

## Prompt 合同

Prompt 由版本化模板生成，不在 Handler 或字符串拼接中散落。逻辑分区固定为：

1. SYSTEM_ROLE：回答任务与允许行为。
2. SAFETY_POLICY：资料不可信、不得执行资料内指令、不得编造来源。
3. ANSWER_POLICY：只依据 Evidence、何时拒答或报告冲突。
4. USER_QUESTION：作为数据传入的原始问题。
5. EVIDENCE_BLOCKS：带应用生成 evidenceId 的候选资料。
6. OUTPUT_SCHEMA：结构、枚举、长度和引用约束。

用户问题和 Evidence 使用明确边界包裹，不能通过普通字符串替换破坏模板。Prompt 工程要求见 [[Prompt 模板版本化与回归]]。

模板变更必须：

- 增加 promptVersion。
- 说明目标、风险和兼容范围。
- 对固定 eval 执行回归。
- 保留上一可发布版本以便回退。
- 与回答 Schema 和模型 profile 一起记录。

## 上下文预算

应用在请求模型前计算预算：

~~~text
total context
  = system and policy
  + output schema
  + user question
  + evidence blocks
  + reserved output
  + safety margin
~~~

Evidence 选择按检索相关性、子问题覆盖和来源多样性进行，不能简单从尾部截断完整 Prompt。

规则：

- 先为系统合同、Schema 和输出保留预算。
- 超预算时删除最低优先 Evidence，而不是截断证据标识或 JSON Schema。
- 单个超长 Evidence 在摄取阶段已有安全切片；组装阶段不随意截断代码中部。
- 记录选择、排除和 Token 估算器版本。
- Ollama 上下文配置与应用预算必须相容；官方说明见 [Ollama Context Length](https://docs.ollama.com/context-length)。

Token 估算可能与模型真实 tokenizer 不同，因此保留安全余量并用边界测试校准。

## 回答 Schema

候选逻辑结构：

~~~json
{
  "status": "ANSWERED",
  "summary": "简洁回答",
  "claims": [
    {
      "claim_id": "C1",
      "text": "一条可核查主张",
      "evidence_ids": ["E1", "E2"]
    }
  ],
  "limitations": []
}
~~~

允许的模型候选状态先限制为：

- ANSWERED
- INSUFFICIENT_EVIDENCE
- CONFLICTING_EVIDENCE

MODEL_UNAVAILABLE、INVALID_MODEL_OUTPUT、CANCELLED 和内部 FAILED 由应用产生，模型不能声明调用本身是否成功。

Schema 约束：

- 禁止未知字段，避免隐藏文本和合同漂移。
- claims 数量、单条长度、evidenceIds 数量设置上限。
- ANSWERED 至少有一个 claim，每个可核查 claim 至少一个 evidenceId。
- 拒答不得同时输出确定事实主张。
- CONFLICTING_EVIDENCE 必须引用至少两组实际冲突来源。
- 结构只承载最终可展示内容，不收集或暴露隐藏推理过程。

## 校验流水线

~~~mermaid
flowchart LR
    R["完整模型响应"] --> T["传输与终态校验"]
    T --> J["JSON 解析"]
    J --> S["Schema 校验"]
    S --> E["Evidence ID 校验"]
    E --> C["主张覆盖与冲突校验"]
    C --> P["隐私与输出策略"]
    P --> A["AnswerRun 终态"]
~~~

### 传输与终态

- 响应必须完整结束，不能把被取消或截断的 JSON 当作候选。
- 流式事件若后续支持，只用于临时 UI；完整校验前不把文本标为最终回答。
- 空响应、未知结束原因和超时分别分类。

### 语法与 Schema

- 只接受完整 JSON 对象。
- 不用正则从解释性文本中截取“看起来像 JSON”的片段。
- Schema 与消费类型由同一来源生成或有契约测试。
- 一个兼容性明确的语法重试可以作为候选，但必须记录；不能无限重试。

### Evidence 与业务

- 每个 evidenceId 必须来自当前 RetrievalRun。
- 应用从 Evidence 恢复路径、标题、行号和源版本。
- 模型输出的引用顺序不改变来源身份。
- ANSWERED 但无证据、引用不存在或状态矛盾时返回 INVALID_MODEL_OUTPUT。
- 检索已判断明显无答案时，不调用模型让它重新决定。

### 内容安全

- 最终文本不能包含未提供的私人绝对路径。
- 不能声称执行了命令、写入了笔记或访问了网络。
- Evidence 内的指令性文本不能改变回答合同。
- 默认不输出超长原文；必要引用采用短片段与定位。

## 拒答与冲突

### 证据不足

应用可在检索阶段直接拒答；模型也可以在阅读 Evidence 后指出覆盖不足。最终输出应说明：

- 无法从当前仓库证据确认什么。
- 已检索的范围与索引代次。
- 用户可尝试更具体的术语、路径或先更新索引。

不能附加模型常识作为“仅供参考”的仓库答案，因为这会模糊产品边界。

### 证据冲突

输出应并列：

- 冲突主张。
- 各自 evidenceId、相对路径、标题与版本。
- 可以确认的共同部分。
- 需要用户决定的时效、环境或适用条件。

模型可以帮助组织差异，但不能擅自以更新时间或语气判断哪篇一定正确。

## 主模型与对照模型

首发每次请求只调用一个生成模型，避免无必要的双倍延迟和资源竞争。另一个本地模型可以用于：

- 离线对照同一 eval 的表现。
- 人工抽样时生成第二意见。
- 验证回答合同是否过度绑定某一模型。

它不自动成为事实评审器。安全不变量、Evidence 身份和 Schema 仍由程序决定。

若未来加入 reviewer，应先证明其在固定评测上提供稳定增益，并限制它只能提出候选问题，不能绕过最终确定性校验。

## 错误分类与恢复

| 错误 | 终态 | 恢复 |
| --- | --- | --- |
| Ollama 不可达 | MODEL_UNAVAILABLE | 检查本地服务，search 仍可用 |
| 模型不存在 | MODEL_UNAVAILABLE | 用户显式安装或修改 profile |
| 请求超时 | MODEL_UNAVAILABLE | 调整预算或模型，不展示部分答案 |
| 用户取消 | CANCELLED | 停止请求与后续校验 |
| Embedding 维度不符 | INDEX_NOT_READY | 更正 profile 并重建向量 |
| 非法 JSON | INVALID_MODEL_OUTPUT | 可执行至多一次受控重试 |
| Schema 不符 | INVALID_MODEL_OUTPUT | 记录版本并进入回归 |
| 引用不存在 | INVALID_MODEL_OUTPUT | 阻止展示，保存安全诊断 |
| Evidence 不足 | INSUFFICIENT_EVIDENCE | 调整问题或补充笔记 |
| Evidence 冲突 | CONFLICTING_EVIDENCE | 展示冲突并由用户判断 |

重试、超时和取消的一般原则见 [[模型调用的错误分类、限流与幂等重试]]。

## 可观测字段

默认只记录：

- answerRunId、retrievalRunId 和 generationId。
- provider、模型 profile fingerprint、Prompt 与 Schema 版本。
- Evidence 数量、输入输出 Token 或估算、各阶段耗时。
- 完成状态、错误类别、重试次数与降级。
- 引用数量、无效引用数量和主张覆盖结果。

默认不记录完整问题、Evidence 正文、Prompt 和回答正文。显式 debug 模式也应脱敏并警告内容将落盘。

## 模型适配测试

- 健康检查、模型不存在和服务不可达。
- 单条与批量 Embedding 的数量、顺序和维度。
- 超长输入、超时、取消、连接重置和非法响应。
- 完整 ANSWERED、拒答、冲突和空 claims。
- 非法 JSON、未知字段、未知枚举、重复 evidenceId。
- 引用候选集外标识、无引用主张和伪造路径。
- Evidence 中包含“忽略系统提示”、JSON 片段和伪 evidence 标签。
- 切换模型、Prompt、Schema 和上下文预算后的版本坐标。
- 同一 eval 在主模型与对照模型上的独立报告。

真实模型测试与伪适配器测试分开；前者的波动不能掩盖后者的确定性失败。

## 参考资料快照

截至 2026-08-29，项目实现前应重新核对：

- [Ollama API 文档](https://docs.ollama.com/api/introduction)
- [Ollama Embed API](https://docs.ollama.com/api/embed)
- [Ollama Context Length](https://docs.ollama.com/context-length)

实际 API、模型标签和能力以锁定 release 的官方文档与本地探针为准。

## 验收标准

- 业务层不依赖 Ollama 原始 DTO 或错误文本。
- 模型切换不会改变 Evidence、AnswerRun 和失败终态语义。
- 无效结构或引用不会以最终回答形式显示。
- 模型不可用时，索引和关键词搜索仍可工作。
- Prompt、Schema、模型和上下文预算均可定位和回归。
- 所有模型质量结论绑定 eval、revision、配置和运行环境。

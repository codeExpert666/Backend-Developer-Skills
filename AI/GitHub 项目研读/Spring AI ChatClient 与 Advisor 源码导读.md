---
title: Spring AI ChatClient 与 Advisor 源码导读
aliases:
  - Spring AI ChatClient 源码导读
tags:
  - AI
  - GitHub
  - 源码研读
  - Java
  - Spring AI
created: 2026-07-27T01:27:46
updated: 2026-07-27T01:27:46
---

Spring AI 用统一模型抽象、Spring Boot 配置、ChatClient 和 Advisor 把模型调用接入 Spring 应用。ChatClient 负责构造请求，Advisor 在模型调用前后组成有顺序的拦截链，provider adapter 再把统一 Prompt 转为具体供应商 SDK 请求。

本文只追踪 `ChatClient → Advisor → ChatModel → OpenAI provider` 和 Tool Calling 回环。2026-07-27 只对官方 `v2.0.0` 源码、文档和测试入口做静态核验；**未运行单元测试、集成测试、示例或真实模型请求**。

## 本路线边界

本轮回答：Spring 抽象在哪一层增加配置和组合能力，Advisor 执行顺序如何进入模型，工具为什么会形成循环，以及 OpenAI provider 最终调用哪个底层 API。

本轮不遍历所有 provider、starter、VectorStore 和 RAG 实现，也不把 Spring 的统一接口理解为供应商能力完全等价。系统分层边界见 [[AI 应用开发的系统分层与职责边界]]。

## 固定源码快照

| 项目 | 本轮选择 |
| --- | --- |
| 核对日期 | 2026-07-27 |
| release / tag | `v2.0.0` |
| 完整 commit | `ef502dab692e26b953a75be4029dba7f1acdc88c` |
| 发布时间 | 2026-06-12 |
| 稳定性 | 官方 GA release |
| 选择理由 | 核对日最新稳定主版本；避免默认分支中的下一版 API 漂移 |
| 证据状态 | 官方源码静态核验；未运行测试和示例 |

**本版本的重要事实**：`models/spring-ai-openai` 的 `OpenAiChatModel` 调用官方 OpenAI Java SDK 的 `chat().completions().create(...)` 与 `createStreaming(...)`，即 **Chat Completions API，而不是 Responses API**。不能因为底层 SDK 同时支持 Responses，就把 Spring AI `v2.0.0` 的当前 provider 路径写成 Responses。

## 模块地图

| 模块 | 职责 | 本轮是否深入 |
| --- | --- | --- |
| `spring-ai-model` | ChatModel、Prompt、Message、ToolCallingManager 等核心抽象 | 主线 |
| `spring-ai-client-chat` | ChatClient、Advisor chain、ToolCallingAdvisor | 主线 |
| `models/spring-ai-openai` | OpenAI provider adapter | 主线 |
| `advisors/` | RAG、tool search 等可选 Advisor | 只确认边界 |
| `spring-ai-rag`、vector store 模块 | 检索与向量库组合 | 本轮不展开 |
| `auto-configurations/`、`starters/` | Spring Boot 属性和 bean 装配 | 只确认入口 |
| `spring-ai-docs` | 官方 Antora 文档源 | 按主题查阅 |

## 公开入口、示例与测试

1. `spring-ai-client-chat/.../ChatClient.java`：公开 fluent API。
2. `DefaultChatClient.java`：request spec、call/stream 和 Advisor 组装。
3. `advisor/DefaultAroundAdvisorChain.java`：Advisor 的顺序和 `nextCall` / `nextStream`。
4. `ChatModelCallAdvisor.java`、流式对应实现：链底部进入 `ChatModel`。
5. `ToolCallingAdvisor.java`：模型—工具—模型循环。
6. `spring-ai-model/.../DefaultToolCallingManager.java`：按名称解析 `ToolCallback` 并执行。
7. `models/spring-ai-openai/.../OpenAiChatModel.java`：统一 Prompt 到 OpenAI Chat Completions DTO 的适配。
8. `DefaultChatClientTests`、`DefaultAroundAdvisorChainTests`、`ToolCallingAdvisorAutoRegistrationTests`、`OpenAiChatModelTests`：相邻契约入口。

这些测试只表示仓库提供了相应证据位置。本次没有执行，不能写成“v2.0.0 已由本地测试验证”。

## 最小典型调用链

普通调用：

```text
ChatClient.Builder
  → prompt().user(...).options(...).advisors(...)
  → call().content()
  → DefaultChatClientRequestSpec 构造 ChatClientRequest
  → DefaultAroundAdvisorChain
  → 各 Advisor 的 before / after 逻辑
  → ChatModelCallAdvisor
  → ChatModel.call(Prompt)
  → OpenAiChatModel
  → openAiClient.chat().completions().create(request)
  → ChatCompletion 映射为 ChatResponse
```

流式调用把链底部换成 stream advisor 和 `ChatModel.stream`。OpenAI provider 使用异步 SDK 的 `createStreaming`，把 chunk 转为 Reactor `Flux`，再在上层聚合或逐段传递。

工具循环：

```text
ToolCallingAdvisor 调用下游 ChatModel
  → 判断 ChatResponse 是否包含 tool calls
  → DefaultToolCallingManager.executeToolCalls
  → 按名称取得注册的 ToolCallback
  → 执行本地代码并形成 ToolResponseMessage
  → 将新会话历史送回模型
  → 直到没有 tool call 或 returnDirect
```

`v2.0.0` 的同步循环是 `do ... while (isToolCall)`，源码未显示统一的最大迭代次数。应用必须另设轮数、时间、Token、成本和副作用预算。

## 错误、取消、重试与权限安全边界

### 错误

ChatClient 和 Advisor 不会把所有 provider 错误统一成一个可靠的业务错误枚举。OpenAI SDK 异常、Advisor 异常、Tool execution error 和 Reactor error 可沿不同路径传播；应用适配层应保留 cause，再映射为稳定业务语义。

工具参数或执行错误可由 `ToolExecutionExceptionProcessor` 处理，但“把异常文本返回模型”不等于业务恢复完成。事务回滚、人工接管和最终状态仍属业务层。

### 取消

流式路径使用 Reactor `Flux`。取消订阅能终止上游处理，但从 `OpenAiChatModel` 的适配代码本身不能直接得出“底层远程生成一定被取消”的端到端保证；实际项目需验证 HTTP call、连接和计费是否同步结束。

### 重试

OpenAI provider 的 timeout 和 maxRetries 进入底层官方 OpenAI Java client。`spring-ai-retry` 中的通用工具不是所有 `ChatClient` 请求必然经过的统一重试器。阅读时必须分清 provider SDK retry、应用 Reactor retry 和业务补偿。

Tool Calling 循环也不是网络 retry：它是在成功收到模型 tool call 后继续编排，不能与失败重放混为一谈。

### 权限

注册 `ToolCallback` 只建立“名称可以解析到代码”的映射，不代表当前用户获准调用。进入 callback 前仍应校验身份、租户、资源范围、参数、人工审批和幂等键。不要把模型输出直接绑定到高权限 bean。

## 生成代码与手写核心

Spring AI 的 ChatClient、Advisor、ToolCallingManager 和 provider adapter 是 Spring 团队手写代码。`OpenAiChatModel` 再依赖 OpenAI Java SDK 中的生成模型和 service。

因此有两条升级边界：Spring AI 自身的抽象/API 变化，以及底层 OpenAI Java 生成类型变化。自己的业务代码应依赖一个窄的模型适配层，不直接散落供应商 DTO。

## Java 研读关注点

- Spring bean、auto-configuration 与显式 builder 的职责边界。
- Advisor order 在请求正向和响应反向的执行含义。
- imperative `call` 与 Reactor `stream` 的线程、context 和异常差异。
- Micrometer Observation 怎样包围模型与 Advisor，而不采集敏感正文。
- Tool Calling 自动注册带来的便利和隐藏控制点。
- provider-neutral API 何时不足，何时需要回到原生 SDK。

## 停止条件

1. 能从 `ChatClient.call()` 追到 `OpenAiChatModel` 和 Chat Completions SDK 调用。
2. 能解释 Advisor chain 的顺序和链底部职责。
3. 能画出 ToolCallingAdvisor 的循环、`returnDirect` 和业务授权点。
4. 能区分 provider retry、Reactor retry、工具循环和业务补偿。
5. 能明确 `v2.0.0` 当前不是 Responses provider 路径。

达到这些条件后停止，不读取全部 provider 和 starter。

## 易变事实与重新核对

Spring Boot 兼容线、starter 名称、Advisor API、默认自动注册、provider module、底层 OpenAI SDK 版本和具体 API 路径都可能变化。新一轮必须先核对 Spring AI release notes、dependency management、`OpenAiChatModel` 和相邻测试，不能把本版本的 Chat Completions 结论外推到未来版本。

## 相关笔记

- [[GitHub AI 项目源码研读方法与清单]]
- [[OpenAI Java Responses API 源码导读]]
- [[Tool Calling 生命周期与可靠业务边界]]
- [[受控 Agent 的执行、审批与恢复]]
- [[AI 应用的成本、延迟、可观测性与降级]]
- [[AI 学习与实验记录模板]]

## 官方资料

- [Spring AI v2.0.0 release](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.0)
- [Spring AI v2.0.0 源码树](https://github.com/spring-projects/spring-ai/tree/v2.0.0)
- [固定 commit](https://github.com/spring-projects/spring-ai/commit/ef502dab692e26b953a75be4029dba7f1acdc88c)
- [Spring AI ChatClient 文档](https://docs.spring.io/spring-ai/reference/api/chatclient.html)
- [Spring AI Advisors 文档](https://docs.spring.io/spring-ai/reference/api/advisors.html)
- [Spring AI Tool Calling 文档](https://docs.spring.io/spring-ai/reference/api/tools.html)

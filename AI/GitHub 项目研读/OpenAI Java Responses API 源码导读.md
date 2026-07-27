---
title: OpenAI Java Responses API 源码导读
aliases:
  - OpenAI Java Responses 源码导读
tags:
  - AI
  - GitHub
  - 源码研读
  - Java
  - OpenAI
created: 2026-07-27T01:27:46
updated: 2026-07-27T01:44:22
---

OpenAI Java SDK 把 OpenAI HTTP API 映射为 Java 的强类型请求、响应、同步、异步和流式接口。本文只回答 Responses API 如何从公开 Client 进入 HTTP 传输，以及结构化输出、Function Calling、错误和重试分别停在哪一层。

本文是 [[GitHub AI 项目源码研读方法与清单]] 的静态导读，不是个人研读记录。2026-07-27 只核对了官方 release、固定 commit 下的源码、示例和测试文件；**未运行示例、单元测试或真实 API 请求**，因此不能把“仓库内有测试”表述为“本次已验证通过”。

## 本路线边界

本轮覆盖：

- Responses 的普通请求、异步请求和 Server-Sent Events（SSE，服务端事件流）入口。
- 结构化输出和 Function Calling（函数调用）的应用回环。
- HTTP 错误分类、请求超时、自动重试和流关闭。
- Stainless 生成代码与仓库手写代码的边界。

本轮不覆盖全部端点、全部生成模型、Azure 或 Bedrock 的供应商差异，也不把示例中的模型名当成长期默认值。工具授权、业务幂等和副作用控制应继续遵循 [[Tool Calling 生命周期与可靠业务边界]]。

## 固定源码快照

| 项目 | 本轮选择 |
| --- | --- |
| 核对日期 | 2026-07-27 |
| release / tag | `v4.45.0` |
| 完整 commit | `d58ba4382f9f603be0190717b110ca38cfe5e4da` |
| 发布时间 | 2026-07-23 |
| 选择理由 | 核对日官方仓库标记的最新稳定 release；比默认分支更适合作为可复现源码基线 |
| 证据状态 | 官方源码静态核验；未运行示例和测试 |

重新开始研读时，应先从官方 releases 页面确认稳定版本，再记录 tag 对应的完整 commit；不要直接把 `main` 的生成类型和默认行为套到本快照。

## 模块地图

| 模块 | 职责 | 本轮是否深入 |
| --- | --- | --- |
| `openai-java` | 面向使用者的聚合 artifact | 只确认依赖入口 |
| `openai-java-core` | Client 接口、生成模型与 service、HTTP 抽象、错误和重试 | 主线 |
| `openai-java-client-okhttp` | 基于 OkHttp 的具体 HTTP transport 和 Client builder | 主线 |
| `openai-java-example` | Responses、流式、结构化输出和 Function Calling 示例 | 主线 |
| `openai-java-spring-boot-starter` | Spring Boot 配置集成 | 不在本轮展开 |
| `openai-java-bedrock` | Bedrock 适配 | 不在本轮展开 |

`openai-java-core` 中“类型很多”主要来自 OpenAPI 生成，不等于每个文件都值得顺序阅读。先从示例进入一个 service，再落到 HTTP transport 和错误路径即可。

## 公开入口、示例与测试

优先阅读以下固定快照路径：

1. `openai-java-example/src/main/java/com/openai/example/ResponsesExample.java`：普通 Responses 请求。
2. `ResponsesStreamingExample.java` 与异步流式示例：SSE 消费和资源关闭。
3. `ResponsesStructuredOutputsExample.java`：响应格式与类型映射。
4. `ResponsesFunctionCallingExample.java`、`ResponsesFunctionCallingRawExample.java`：工具调用回环与原始参数边界。
5. `openai-java-core/src/main/kotlin/com/openai/services/blocking/ResponseServiceImpl.kt`：生成 service 如何构造 `/responses` 请求。
6. `openai-java-client-okhttp/src/main/kotlin/com/openai/client/okhttp/OkHttpClient.kt`：具体网络传输。
7. `openai-java-core/src/main/kotlin/com/openai/core/http/RetryingHttpClient.kt`：重试决策。
8. `ResponseServiceTest.kt`、`RetryingHttpClientTest.kt`：生成契约测试和 HTTP 重试边界。

测试文件是所选版本的实现证据入口，不是本次执行证据。后续真正开始实验时，应复制 [[AI 学习与实验记录模板]]，单独记录命令、环境、退出码和结果。

## 最小典型调用链

普通请求主链：

```text
OpenAIOkHttpClient.fromEnv()
  → OpenAIClient.responses()
  → ResponseService.create(ResponseCreateParams)
  → ResponseServiceImpl 构造 POST /responses
  → RetryingHttpClient 判断是否重试
  → OkHttpClient 执行 HTTP 请求
  → errorHandler 或 JSON response handler
  → Response.output
```

流式请求沿用同一 service 和 transport，但请求带上 `stream: true`，响应交给 SSE handler 转换为 `ResponseStreamEvent`。返回的 `StreamResponse` 持有网络资源，调用者必须关闭；“读到最后一个事件”不能替代明确的生命周期管理。

Function Calling 的最小回环是：

```text
第一次 Responses 请求
  → 读取 ResponseFunctionToolCall
  → 应用按工具名和参数做校验、鉴权与白名单分派
  → 执行本地确定性代码
  → 追加带同一 callId 的 FunctionCallOutput
  → 第二次 Responses 请求
  → 取得面向用户的最终输出
```

SDK 负责类型、序列化和网络协议；它不会替应用判断某个工具是否允许执行。

## 错误、取消、重试与权限安全边界

### 错误

`OpenAIServiceException` 下按状态区分 400、401、403、404、422、429 和 5xx；SSE、I/O、重试耗尽和响应数据不合法还有独立异常。401 表示认证失败，403 表示已经识别请求方但权限不足，两者不应合并成模糊的“调用失败”。

### 超时与取消

本版本默认请求超时为 10 分钟，可在 Client 默认值或单次 `RequestOptions` 覆盖。普通前台请求的本地取消应通过异步调用和底层请求生命周期控制；后台 Responses 另有 API 级 cancel 端点，不能把它和关闭本地 SSE 流混为一谈。

流式消费者提前停止时必须关闭 `StreamResponse`。本轮静态核验没有建立“关闭流后服务端一定终止生成”的端到端证据，实际项目需要用固定集成测试确认资源释放和计费边界。

### 重试

Client 默认最多重试 2 次。连接失败、408、409、429 和 5xx 可进入重试判断，并考虑 `X-Should-Retry`、`Retry-After-Ms` 或 `Retry-After`。只有请求体可重复发送时才能安全重放。

自动重试只说明 transport 愿意再次发送请求，不证明业务操作幂等。工具写操作、外部支付或消息发送必须用业务幂等键、状态机和人工确认另行约束。

### 凭据与权限

示例可从 `OPENAI_API_KEY` 等环境配置创建 Client。笔记和测试不得写真实密钥。组织、项目和 workload identity 只是 API 认证上下文；工具、文件、数据库和业务对象权限仍由应用负责。

## 生成代码与手写核心

生成文件通常带有 `File generated from our OpenAPI spec by Stainless` 标记。模型、service 接口与实现，以及不少契约测试都属于生成区域；直接修改这些文件无法形成稳定扩展点。

`openai-java-client-okhttp` 的具体 transport 和 `openai-java-example` 示例没有同类生成标记，是理解网络实现与推荐用法的手写入口。应用扩展应优先位于自己的适配层，而不是修改生成模型。

## Java 研读关注点

- immutable builder 如何表达可选字段和联合类型。
- blocking 与 async service 如何共享请求契约。
- `CompletableFuture`、SSE subscriber 与 `AutoCloseable` 资源如何传播失败和关闭。
- SDK 异常如何转成业务稳定的错误分类，同时保留 request ID 等诊断上下文。
- 生成类型升级时，自己的 provider adapter 如何隔离变动。

## 停止条件

满足以下条件即可停止本轮源码阅读：

1. 能从 `ResponsesExample` 追到 `/responses` 的 OkHttp 请求和响应解码。
2. 能解释普通、异步、SSE 三种接口的生命周期差异。
3. 能画出一次 Function Calling 的两次模型请求和本地授权点。
4. 能指出一个可重试错误、一个不可重试错误和请求体可重复发送的前提。
5. 能区分生成 service/model 与手写 transport/example。

不需要遍历所有 endpoint、模型枚举和生成测试。

## 易变事实与重新核对

模型名称、请求字段、错误类型、默认超时、默认重试次数、认证方式和模块列表都可能变化。重新核对时至少查看 releases、README 的 network/error 章节、`RetryingHttpClient`、`ResponseServiceImpl` 和相邻测试，并把新 tag 与完整 commit 写入新的日期化研读记录。

## 相关笔记

- [[GitHub AI 项目源码研读方法与清单]]
- [[OpenAI Go Responses API 源码导读]]
- [[Tool Calling 生命周期与可靠业务边界]]
- [[AI 应用开发的系统分层与职责边界]]
- [[AI 学习与实验记录模板]]

## 官方资料

- [OpenAI Java v4.45.0 release](https://github.com/openai/openai-java/releases/tag/v4.45.0)
- [OpenAI Java v4.45.0 源码树](https://github.com/openai/openai-java/tree/v4.45.0)
- [固定 commit](https://github.com/openai/openai-java/commit/d58ba4382f9f603be0190717b110ca38cfe5e4da)
- [Responses API 指南](https://developers.openai.com/api/docs/guides/migrate-to-responses)
- [Function Calling 指南](https://developers.openai.com/api/docs/guides/function-calling)
- [Streaming Responses 指南](https://developers.openai.com/api/docs/guides/streaming-responses)

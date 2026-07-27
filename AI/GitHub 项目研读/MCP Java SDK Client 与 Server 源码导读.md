---
title: MCP Java SDK Client 与 Server 源码导读
aliases:
  - MCP Java SDK 源码导读
tags:
  - AI
  - GitHub
  - 源码研读
  - Java
  - MCP
created: 2026-07-27T01:32:02
updated: 2026-07-27T01:32:02
---

MCP（Model Context Protocol，模型上下文协议）Java SDK 实现 Client、Server、JSON-RPC session 和 stdio / HTTP transport，使 Java 应用可以协商能力、发现并调用 tools、resources 和 prompts。协议只定义通信合同，不替应用确认 Server 是否可信，也不授予工具业务权限。

本文只追踪初始化、`tools/list`、`tools/call` 和 Streamable HTTP 的最小 Client / Server 链。2026-07-27 只核对官方 `v2.0.0` release、固定 commit 的源码、文档和仓库自报 conformance 文件；**未运行 Maven 测试、示例或 conformance suite**。

## 本路线边界

本轮覆盖协议版本与 capability negotiation（能力协商）、sync/async Client、Server tool handler、JSON-RPC error、request timeout、Streamable HTTP 重连和传输安全 hook。

业务层 Host / Client / Server 的概念、scope、audience、Token passthrough 和工具授权见 [[MCP 生命周期、能力协商与安全边界]]。本文不把 transport 连通写成业务安全，也不遍历所有 resource、prompt、sampling 和 elicitation 功能。

## 固定源码快照

| 项目 | 本轮选择 |
| --- | --- |
| 核对日期 | 2026-07-27 |
| release / tag | `v2.0.0` |
| 完整 commit | `f56d038409473210c59d6eddef09c4b5cd36042b` |
| 发布时间 | 2026-06-11 |
| 稳定性 | 官方 GA release |
| Java 基线 | Java 17+ |
| 协议基线 | 支持并以 MCP `2025-11-25` 为当前文档主线，同时保留更早版本协商常量 |
| 选择理由 | 核对日最新稳定主版本；Spring transports 已迁到 Spring AI 2.0+，边界较清晰 |
| 证据状态 | 官方源码静态核验；未运行测试或 conformance |

## 模块地图

| 模块 | 职责 | 本轮是否深入 |
| --- | --- | --- |
| `mcp-core` | Client、Server、session、stdio、JDK HttpClient、Servlet 和 Streamable HTTP | 主线 |
| `mcp-json-jackson2` | Jackson 2 JSON binding | 只确认扩展边界 |
| `mcp-json-jackson3` | Jackson 3 JSON binding | 只确认默认实现 |
| `mcp` | `mcp-core` 与 Jackson 3 的 convenience bundle | 依赖入口 |
| `mcp-bom` | 版本管理 | 只确认入口 |
| `mcp-test` | 共享测试工具与集成测试 | 选读 |
| `conformance-tests` | Client、Server 和授权 conformance 适配 | 主线证据入口 |

Spring WebFlux / WebMVC transports 和 Spring Security 集成已经属于 Spring AI 2.0+，不能继续当成本 SDK 内置模块。

## 公开入口、示例与测试

1. `McpClient.java`：sync/async builder，默认 request/initialization timeout 为 20 秒。
2. `McpSyncClient.java`、`McpAsyncClient.java`：公开 Client 操作。
3. `LifecycleInitializer.java`：initialize、协议版本和能力协商、initialized notification。
4. `McpClientSession.java`：JSON-RPC request ID、pending response、timeout 和错误解码。
5. `McpServer.java`、`McpAsyncServer.java`：Server builder 与 tool/resource/prompt handler。
6. `HttpClientStreamableHttpTransport.java`：JDK HttpClient、SSE、session、授权错误处理与恢复。
7. `DefaultServerTransportSecurityValidator.java`：Origin / Host 校验。
8. `LifecycleInitializerTests`、`McpAsyncClientTest`、Server exchange tests 和 transport security tests。
9. `conformance-tests/VALIDATION_RESULTS.md`：仓库维护者保存的 conformance 自报结果与已知限制。

## 最小典型调用链

Client 初始化：

```text
McpClient.sync(transport) 或 async(transport)
  → 配置 requestTimeout、capabilities 和 handlers
  → build McpAsyncClient（sync 是其 blocking facade）
  → LifecycleInitializer.withInitialization
  → 创建 McpClientSession
  → 发送 initialize
  → 校验 Server 返回的 protocol version
  → 缓存 Server capabilities
  → 发送 notifications/initialized
```

Client 调用工具：

```text
client.listTools()
  → 检查 Server tools capability
  → McpClientSession.sendRequest(tools/list)
  → 分页汇总和 tool name / schema 缓存

client.callTool(request)
  → 检查 tools capability
  → sendRequest(tools/call)
  → JSON-RPC transport
  → CallToolResult
  → 可选 output schema 校验
```

Server 处理工具：

```text
McpServer.async(transportProvider)
  → 注册 AsyncToolSpecification
  → 收到 tools/call
  → 将 params 转为 CallToolRequest
  → 按名称查找 tool
  → ToolInputValidator 校验参数
  → 应用 callHandler
  → output schema 包装器校验 structuredContent
  → JSON-RPC result 或 error
```

## 错误、取消、重试与权限安全边界

### 错误与 timeout

`McpClientSession.sendRequest` 为每个请求建立 ID 和 pending sink，并应用 `requestTimeout`。远端 JSON-RPC error 转为 `McpError`；未知 method、无 tools capability、schema 不合法和 transport error 属于不同层级。

Tool handler 的可修正业务错误宜返回 `CallToolResult.isError(true)`；未知工具或协议参数错误才使用 JSON-RPC error。这样模型有机会看到工具错误，但不能让模型自行覆盖业务状态。

### 取消限制

本快照没有实现完整的 MCP `notifications/cancelled` 请求取消主链。Reactor subscription disposal 可取消部分 HTTP body subscription，关闭 Client/session 会让 pending request 失败，但这些都不能写成“SDK 已保证向对端发送协议取消并终止远程工具”。

因此长任务必须依赖明确 timeout、工具自身可取消设计和幂等/补偿；如项目要求协议级取消，需要先做专项实验或选择已有明确契约的更新版本。

### 重连与重试

一般 JSON-RPC tool request 没有通用自动重试。Streamable HTTP transport 可以对可恢复 SSE stream 做重连，并在 session 丢失时重新初始化；401/403 可交给自定义 authorization error handler，在 `maxRetries` 上限内刷新授权后重放。

传输重连、授权挑战重放和业务 RPC retry 是三件事。工具有副作用时，不能因 SSE 断开就假设安全重做。

### 权限与传输安全

核心 SDK 提供可插拔授权 hook，不内置完整认证授权系统。`DefaultServerTransportSecurityValidator` 校验允许的 Origin 和 Host，用于降低跨站或 DNS rebinding 风险；它不验证用户身份、OAuth scope 或业务对象权限。

sampling 与 elicitation handler 让 Client 保持模型和用户输入控制权，但应用仍须确认 Server 身份、请求理由和允许的数据范围。任何 tool handler 都应在应用代码内做鉴权、租户过滤和审批。

## 生成代码与手写核心

核心 Client、Server、session、schema、transport 和测试均以项目作者版权头呈现，未发现类似 OpenAPI SDK 的大规模生成标记。Jackson 2/3 是独立手写 JSON binding 实现。

这不代表所有文件都应深入阅读：`McpSchema.java` 聚合大量协议类型，先从公开 Client/Server 和 session 进入，只按调用链查对应类型。

## Java 研读关注点

- Reactor `Mono` / `Flux` 作为 async core，sync API 是 blocking facade。
- initialize 是并发请求共享的生命周期状态，不是每次调用都重新握手。
- JSON-RPC request/notification 的响应语义和 error code。
- transport、session 与业务 handler 的职责边界。
- schema validation 为什么不能替代授权和业务校验。
- Reactor Context、SLF4J 和资源关闭怎样进入可观测性。

## Conformance 证据限制

固定 tag 内的 `conformance-tests/VALIDATION_RESULTS.md` 自报 Server active suite 44/44、Client 3/4，并记录 SSE retry 不遵守 `retry:` timing、没有发送 `Last-Event-ID` 的已知限制。该 tag 的 README 摘要却仍写另一组 Server/Auth 数字，和详细文件存在内部漂移。

这些数字只能写成“仓库在固定 commit 中自报”，不能写成“本次已验证”。正式采用前应固定 conformance 工具版本并重新运行，保存命令、退出码和失败详情。

## 停止条件

1. 能解释 initialize、版本协商、capabilities 和 initialized notification。
2. 能分别追完 Client `tools/call` 与 Server handler。
3. 能区分 `CallToolResult.isError`、`McpError` 和 transport error。
4. 能明确本版本缺少完整协议取消保证。
5. 能区分 SSE 重连、授权重放和业务 RPC retry。
6. 能解释 Origin/Host validator 与业务授权的差异。
7. 能正确表述 conformance 只是仓库自报且摘要存在漂移。

达到这些条件后停止，不遍历全部协议 primitive。

## 易变事实与重新核对

MCP current spec、支持版本列表、transport、Spring 模块归属、授权 hook、默认 timeout 和 conformance 结果都容易变化。新一轮应先查 Versioning、SDK releases、`ProtocolVersions`、transport tests 和 conformance 配置，再建立新的固定快照。

## 相关笔记

- [[GitHub AI 项目源码研读方法与清单]]
- [[MCP Go SDK Client 与 Server 源码导读]]
- [[MCP 生命周期、能力协商与安全边界]]
- [[Tool Calling 生命周期与可靠业务边界]]
- [[AI 应用安全威胁建模与防护]]
- [[AI 学习与实验记录模板]]

## 官方资料

- [MCP Java SDK v2.0.0 release](https://github.com/modelcontextprotocol/java-sdk/releases/tag/v2.0.0)
- [MCP Java SDK v2.0.0 源码树](https://github.com/modelcontextprotocol/java-sdk/tree/v2.0.0)
- [固定 commit](https://github.com/modelcontextprotocol/java-sdk/commit/f56d038409473210c59d6eddef09c4b5cd36042b)
- [MCP Java SDK 官方文档](https://java.sdk.modelcontextprotocol.io/latest/)
- [MCP 2025-11-25 Lifecycle](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle)
- [MCP 2025-11-25 Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
- [MCP Authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)

---
title: MCP Go SDK Client 与 Server 源码导读
aliases:
  - MCP Go SDK 源码导读
tags:
  - AI
  - GitHub
  - 源码研读
  - Go
  - MCP
created: 2026-07-27T01:32:02
updated: 2026-07-27T01:32:02
---

MCP Go SDK 实现 MCP Client、Server、JSON-RPC、stdio / Streamable HTTP transport 和 OAuth 辅助包。它以 `context.Context` 表达请求取消，以 typed tool helper 生成和校验 JSON Schema，并把传输恢复与普通 RPC 语义分开。

本文只跟踪初始化、`tools/list`、`tools/call`、协议取消和 Streamable HTTP 的最小链。2026-07-27 只核对官方稳定 release、固定 commit 的源码、文档、示例和 conformance workflow；**未运行 Go 测试、示例或 conformance suite**。

## 本路线边界

本轮覆盖 `mcp.Client` / `Server`、session、typed tool、JSON-RPC error、context cancellation、SSE resume、OAuth/Bearer middleware 和 session security。

业务上怎样信任 MCP Server、批准工具、限制 scope/audience 和处理副作用，仍由 [[MCP 生命周期、能力协商与安全边界]] 与 [[AI 应用安全威胁建模与防护]] 负责。本文不遍历全部 sampling、elicitation、resource 和 prompt API。

## 固定源码快照

| 项目 | 本轮选择 |
| --- | --- |
| 核对日期 | 2026-07-27 |
| release / tag | `v1.6.1` |
| 完整 commit | `d454bbaf06a342aee5336df3370321d9cdec2478` |
| 发布时间 | 2026-05-22 |
| Go 基线 | `go.mod` 声明 Go 1.25.0 |
| 协议基线 | 最新协商版本为 MCP `2025-11-25`，并支持 `2025-06-18`、`2025-03-26`、`2024-11-05` |
| 选择理由 | 核对日最新稳定 release；官方 releases 中更高的 `v1.7.0-pre.3` 是预发布，不能作为稳定基线 |
| 证据状态 | 官方源码静态核验；未运行测试或 conformance |

## 模块地图

| 包 / 位置 | 职责 | 本轮是否深入 |
| --- | --- | --- |
| `mcp/` | Client、Server、session、feature、tool 和 transports | 主线 |
| `jsonrpc/` | 对外 JSON-RPC message/error 类型 | 主线 |
| `internal/jsonrpc2/` | request correlation、并发 handler 和 connection 生命周期 | 按调用链 |
| `auth/` | Bearer middleware、TokenVerifier、Authorization Code flow | 主线安全边界 |
| `oauthex/` | protected-resource metadata 等 OAuth 扩展 | 按需 |
| `examples/` | Client、Server、HTTP 和 auth 示例 | 选读 |
| `conformance/` | everything Client/Server 和 expected-failure baseline | 证据入口 |
| `internal/docs/`、`internal/readme/` | weave 生成文档的源文件 | 识别生成边界 |

## 公开入口、示例与测试

1. README quick start：stdio Server 和 Client tool call。
2. `mcp/client.go`：`NewClient`、`Connect`、`ClientSession` 与公开方法。
3. `mcp/server.go`：`NewServer`、feature registry、`callTool`。
4. `mcp/tool.go`：`ToolHandler`、generic `ToolHandlerFor`、schema validation。
5. `mcp/shared.go`：method table、middleware、`handleSend` / `handleReceive`。
6. `mcp/transport.go`：JSON-RPC call、context cancel 和 `notifications/cancelled`。
7. `mcp/streamable.go`：HTTP session、SSE reconnect、OAuth challenge 和 transport security。
8. `mcp/error_test.go`、`client_example_test.go`、`tool_example_test.go`、`conformance_test.go`。
9. `.github/workflows/conformance.yml` 与 `conformance/baseline.yml`。

## 最小典型调用链

Client 初始化：

```text
mcp.NewClient(Implementation, ClientOptions)
  → Client.Connect(ctx, Transport, options)
  → transport.Connect
  → internal/jsonrpc2.NewConnection
  → 发送 initialize
  → 校验 Server 返回的 protocol version
  → 保存 InitializeResult
  → 发送 notifications/initialized
  → 返回 ClientSession
```

Client tool call：

```text
ClientSession.CallTool(ctx, params)
  → handleSend(methodCallTool)
  → sending MethodHandler / middleware
  → call(ctx, jsonrpc2.Connection, ...)
  → Connection.Call + AsyncCall.Await
  → CallToolResult 或 error
```

Server typed tool：

```text
mcp.AddTool[In, Out](server, Tool, ToolHandlerFor)
  → 从 Go 类型推导或解析 input/output JSON Schema
  → 注册 serverTool
  → 收到 tools/call
  → 按名称查找 handler
  → 应用默认值并校验 input
  → 解码为 In
  → 调用 typed handler
  → 普通 error 转为 IsError tool result
  → 校验并填充 StructuredContent
```

低层 `Server.AddTool` 不自动校验输入和输出，调用者需自己承担；多数业务工具应优先使用 generic `mcp.AddTool`。

## 错误、取消、重连、重试与权限安全边界

### 工具错误与协议错误

typed handler 返回普通 Go `error` 时，SDK 将其写入 `CallToolResult` 并设置 `IsError: true`，让模型可以看到并修正。显式 `*jsonrpc.Error`、未知工具和不支持的方法属于协议错误。

可用 `errors.As(err, &rpcErr)` 提取 JSON-RPC code；连接关闭还会包装 `ErrConnectionClosed`。不要把所有 error 文本直接暴露给最终用户或模型。

### context 与协议取消

取消传给 `ClientSession` 或 `ServerSession` 方法的 context，会让本地 `Await` 返回 context error，并在独立短 timeout 中发送 `notifications/cancelled`，随后 retire 本地 call。对端的 preempter 收到通知后取消对应入站 request context。

SDK 保证 RPC 因取消退出前已经尝试发送取消通知，但不保证对端一定观察或已经停止副作用。Tool handler 必须监听 `ctx.Done()`，并使用幂等、事务或补偿保证最终状态。

### SSE 重连不是 RPC 重试

`StreamableClientTransport.MaxRetries` 默认 5，用于 GET / SSE 流中断后的重新连接。它使用 `Last-Event-ID`、server `retry:`、指数退避和 full jitter，以恢复事件流或重放 EventStore 中的事件。

这不表示 `CallTool`、Upsert 或任意 JSON-RPC 请求会自动重做 5 次。**SSE 重连恢复传输；RPC retry 决定是否再次执行操作，两者必须严格区分。**

### OAuth 挑战重放

配置 `OAuthHandler` 后，Streamable HTTP request 收到 401/403 会执行 `Authorize`，成功后重放原 HTTP request 一次。Authorization Code handler 可管理 token refresh 和 insufficient-scope step-up。

该重放只属于授权挑战流程，也不为有副作用工具提供业务幂等证明。

### Server 认证与会话安全

`auth.RequireBearerToken` 是 HTTP middleware，必须由应用显式包裹全部 MCP handler。`TokenVerifier` 负责校验 token、scope、过期和 audience 语义；SDK 不会替用户自动选择正确 verifier。

若 verifier 提供 `TokenInfo.UserID`，Streamable HTTP session 会绑定用户，后续请求用户不匹配时返回 403。默认有 localhost DNS rebinding 防护；`v1.6.1` 的 Cross-Origin Protection 默认并非自动启用，应使用标准 HTTP middleware 或明确配置。

客户端侧私网 IP 阻断、DNS pinning、egress proxy 和重定向策略不由 SDK 开箱完成，仍要防范 SSRF（服务端请求伪造）。

## 生成内容与手写核心

Client、Server、tool、transport、JSON-RPC、auth 和测试是手写 Go 源码，不是协议代码生成产物。

README 和 `docs/*.md` 带 `Autogenerated by weave; DO NOT EDIT`，由 `internal/readme/*.src.md` 与 `internal/docs/*.src.md` 生成。这里的“生成”只针对文档拼装，不能误写成 SDK runtime 由生成器实现。

## Go 研读关注点

- `context.Context` 如何传播到 handler、取消通知和 transport。
- `(value, error)`、`errors.As` 和 JSON-RPC code 的分层。
- generic `ToolHandlerFor[In, Out]` 如何把 Go 类型变成 JSON Schema。
- MethodHandler middleware 怎样同时包围 Client 和 Server。
- connection/session 的并发关闭、keepalive 和 `Wait`。
- SSE resume、OAuth request replay 和业务 RPC retry 的区别。

## Conformance 证据限制

固定 commit 的 workflow 使用 MCP conformance `v0.1.15`，Server active suite 与 Client core suite 都引用 `conformance/baseline.yml`；该 baseline 中 Server 和 Client expected failures 均为空。

这只是仓库配置和维护者预期，不是本次运行结果。实际采用前必须执行固定版本 suite，并把输出、退出码、环境和失败场景写入日期化实验记录。

## 停止条件

1. 能追完 initialize 与一条 `CallTool` 请求。
2. 能解释 generic typed tool 的输入/输出 schema 与错误包装。
3. 能说明 context cancel 如何发送 `notifications/cancelled`，以及为何不保证远端副作用停止。
4. 能明确 SSE 默认 5 次是重连，不是 RPC 自动重试。
5. 能说明 OAuth 401/403 重放和普通重试的差异。
6. 能指出 Bearer middleware、session user binding、DNS rebinding 和业务授权各自职责。
7. 能区分 weave 生成文档与手写 SDK runtime。

达到这些条件后停止，不遍历全部 MCP feature。

## 易变事实与重新核对

稳定 release、预发布版本、最低 Go 版本、MCP current spec、默认 capabilities、transport 安全开关、SSE retry 和 OAuth API 都可能变化。重新研读时先查 Versioning、releases、`go.mod`、`shared.go`、`streamable.go`、compatibility flags 和 conformance workflow。

## 相关笔记

- [[GitHub AI 项目源码研读方法与清单]]
- [[MCP Java SDK Client 与 Server 源码导读]]
- [[MCP 生命周期、能力协商与安全边界]]
- [[Tool Calling 生命周期与可靠业务边界]]
- [[AI 应用安全威胁建模与防护]]
- [[AI 学习与实验记录模板]]

## 官方资料

- [MCP Go SDK v1.6.1 release](https://github.com/modelcontextprotocol/go-sdk/releases/tag/v1.6.1)
- [MCP Go SDK v1.6.1 源码树](https://github.com/modelcontextprotocol/go-sdk/tree/v1.6.1)
- [固定 commit](https://github.com/modelcontextprotocol/go-sdk/commit/d454bbaf06a342aee5336df3370321d9cdec2478)
- [MCP Go SDK API reference](https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk)
- [MCP Versioning](https://modelcontextprotocol.io/docs/learn/versioning)
- [MCP 2025-11-25 Lifecycle](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle)
- [MCP 2025-11-25 Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)
- [MCP Authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)

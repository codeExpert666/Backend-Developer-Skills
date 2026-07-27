---
title: Eino Graph 与 Tool Calling 源码导读
aliases:
  - Eino Graph 源码导读
  - Eino Tool Calling 源码导读
tags:
  - AI
  - GitHub
  - 源码研读
  - Go
  - Eino
created: 2026-07-27T01:30:28
updated: 2026-07-27T01:30:28
---

本文沿一条最小主线解释 Eino 怎样把 Go 组件编译成可运行的 Graph（图），以及 ReAct Agent 怎样在模型与工具之间循环。它是 [[GitHub AI 项目源码研读方法与清单]] 中 Eino 主线的入口，不把框架封装误当成业务授权、幂等或人工审批。

本轮资料状态为“官方一手资料静态核验”：核对了指定版本的代码、`go.mod`、测试和 release 页面，但没有在本地运行示例、测试或真实模型调用。因此本文只证明所选快照中的结构和代码契约，不证明某个外部模型、账号或部署环境已经可用。

## 版本依据与核验范围

核对日期为 2026-07-27。

| 仓库 | 本轮快照 | 选择依据 |
| --- | --- | --- |
| [cloudwego/eino](https://github.com/cloudwego/eino) | [`v0.9.13`](https://github.com/cloudwego/eino/releases/tag/v0.9.13)，commit `c5e6aef927cca02bea934541f8dff2ea711b2ca7` | 截至核对日的最新稳定 release；`v0.10.0-alpha.*` 属于预发布版本，不作为本轮稳定基线 |
| [cloudwego/eino-examples](https://github.com/cloudwego/eino-examples) | commit [`396e41a2a13d2714225c58b618e9aac37be873b6`](https://github.com/cloudwego/eino-examples/commit/396e41a2a13d2714225c58b618e9aac37be873b6) | 仓库没有 release 和 tag，只能固定 commit；该提交的 `go.mod` 实际依赖 Eino `v0.9.12` |

两个仓库的版本不能静默拼成同一个基线。核心行为以 Eino `v0.9.13` 的代码和测试为准；Examples 只用于发现入口和演示组合方式。若复现实验，应先决定保持示例原始依赖 `v0.9.12`，还是升级后重新验证差异。

## 先分清四层职责

| 层级 | 主要位置 | 负责什么 | 不负责什么 |
| --- | --- | --- | --- |
| 组件契约 | `components/` | ChatModel、Tool、Retriever 等接口及调用形态 | 具体供应商实现和业务权限 |
| 编排运行时 | `compose/` | Graph、节点、分支、流式转换、编译、调度、中断和恢复 | 决定哪个用户可以执行哪个业务动作 |
| 预组装流程 | `flow/agent/react/` | ReAct 的模型与工具循环 | 替业务系统保证工具安全和正确 |
| Agent 开发套件 | `adk/` | Agent 组合、模型重试等更高层能力 | 自动让有副作用操作具备幂等性 |

`callbacks/`、`schema/` 和 `internal/` 是这些层的支撑结构。provider、存储和外部系统适配主要位于 [Eino Ext](https://github.com/cloudwego/eino-ext)，不应为了理解 Graph 而同时展开所有适配器。

## Graph 从声明到运行

公开入口是 [`compose/generic_graph.go`](https://github.com/cloudwego/eino/blob/c5e6aef927cca02bea934541f8dff2ea711b2ca7/compose/generic_graph.go) 中的泛型 Graph。典型链路是：

```text
compose.NewGraph[I, O]
  → Add*Node / AddEdge / AddBranch
  → Graph.Compile
  → compileAnyGraph
  → graph.compile
  → 校验 START、END、类型、映射和前驱条件
  → 编译节点与分支
  → 选择 Pregel runner 或 DAG runner
  → 返回 Runnable
  → Invoke / Stream / Collect / Transform
```

[`compose/graph.go`](https://github.com/cloudwego/eino/blob/c5e6aef927cca02bea934541f8dff2ea711b2ca7/compose/graph.go) 不是简单保存节点列表。`Compile` 会先拒绝缺少 START / END、输入输出类型不兼容、映射错误等非法图，再根据前驱触发语义选择运行器：

- 普通图默认使用 Pregel 风格的步骤调度。
- 使用全部前驱完成语义的 workflow 会进入 DAG 路径。
- 非 DAG 图若未显式设置最大步骤数，会根据订阅通道数量生成有限上限；超过上限返回 `ErrExceedMaxSteps`。
- DAG 模式本身按依赖收敛，不接受同一套显式最大步骤配置。

这意味着“图能够构造”不等于“图能够编译”，“成功编译”也不等于外部依赖和业务副作用已经安全。

## ReAct Agent 怎样接入 ToolNode

[`flow/agent/react/react.go`](https://github.com/cloudwego/eino/blob/c5e6aef927cca02bea934541f8dff2ea711b2ca7/flow/agent/react/react.go) 的最小链路是：

```text
react.NewAgent
  → 读取 Tool.Info
  → ChatModel.WithTools
  → 创建 compose.ToolNode
  → 建立本地 message state
  → ChatModel 节点
  → 判断响应是否包含 tool calls
      → 否：END
      → 是：ToolNode
          → 直接返回工具：END
          → 普通工具：结果回到 ChatModel
  → Compile，附加 MaxStep 与前驱语义
```

`Generate` 最终走 Runnable 的 `Invoke`，`Stream` 走 Runnable 的流式入口。ReAct 是一个受最大步数约束的 Graph，而不是独立于 Compose 的神秘执行器。

默认流式 tool-call checker 只检查第一块模型输出。源码注释明确提示：有些模型可能在后续 chunk 才出现 tool call，这时必须提供自定义 `StreamToolCallChecker`。Examples 的 ReAct 示例包含检查全部 chunk 的提示，但它仍需用目标模型做差分验证。

## ToolNode 是分发器，不是权限系统

[`compose/tool_node.go`](https://github.com/cloudwego/eino/blob/c5e6aef927cca02bea934541f8dff2ea711b2ca7/compose/tool_node.go) 的职责是把模型生成的调用路由到 Go Tool：

1. `ToolsNodeConfig.Tools` 建立本节点可分发的工具集合。
2. 名称别名先映射到真实工具。
3. 可选的 `ToolArgumentsHandler` 在执行前调整或校验参数。
4. middleware 包裹工具 endpoint。
5. 按工具支持的接口执行；增强接口优先。
6. 用 Call ID 把结果关联回原始 tool call。

未知工具默认报错，也可以提供 `UnknownToolsHandler`。这个集合可以缩小模型可见能力，却不包含调用者身份、租户、资源所有权或业务角色，因此不能替代逐请求授权。

`ExecuteSequentially` 默认是 `false`，多个 tool call 会并行执行。只读、彼此独立的工具可以利用并行；写操作、顺序依赖或共享事务必须显式改为串行，或在更外层建立确定性编排。

发生中断时，ToolNode 的状态会记录已经执行的工具，恢复可以跳过这些结果。但“框架记住已执行”仍不等于外部系统恰好一次执行：超时、进程崩溃和远端已成功但本地未确认都需要业务幂等键与状态查询。

## 错误、取消与重试

[`compose/graph_run.go`](https://github.com/cloudwego/eino/blob/c5e6aef927cca02bea934541f8dff2ea711b2ca7/compose/graph_run.go) 在调度中检查 `ctx.Done()`，等待节点任务结束，并保留 `context.Canceled` 的错误链。调用者仍应给整个 Agent 和单个外部工具设置可判断的超时，并在取消后确认外部副作用。

需要分别理解三类失败：

| 失败位置 | 框架行为 | 业务层必须补什么 |
| --- | --- | --- |
| Graph 结构或运行 | 编译期校验；运行期保留节点路径、取消和最大步数错误 | 把错误映射为稳定业务状态，保留 Trace 与恢复点 |
| Tool 执行 | 默认返回工具错误；`components/tool/utils` 可把非中断错误转换成 Tool 结果文本 | 不把内部秘密拼入模型可见错误；区分可修正参数与不可重试副作用 |
| 模型调用 | ADK 的 `retry_chatmodel.go` 可配置最大次数、退避和 `ShouldRetry` | 限制总耗时与成本；不要把模型重试规则套到写工具 |

ToolNode 没有隐式工具重试。只有工具具备幂等键、能够查询最终状态，并明确分类瞬时错误后，业务层才应考虑重试。

## Examples 应该怎样使用

本轮只需用以下入口定位行为：

- `quickstart/chat`：普通生成、流式接收和流关闭。
- `compose/graph/simple`：最小 Graph。
- `compose/graph/tool_call_once`：一次模型—工具调用图。
- `flow/agent/react`：预组装 ReAct。
- `compose/graph/react_with_interrupt`：工具执行前的人类确认与 checkpoint。
- `adk/cancel/graceful-exit`：取消和恢复。

`tool_call_once` 使用旧的 `BindTools`。Eino `v0.9.13` 的 [模型接口](https://github.com/cloudwego/eino/blob/c5e6aef927cca02bea934541f8dff2ea711b2ca7/components/model/interface.go) 已说明该方法会修改接收者，并可能在并发使用时产生数据竞争；新代码应优先使用返回新模型实例的 `ToolCallingChatModel.WithTools`。不能因为官方 Examples 仍保留旧写法，就把它当作当前推荐契约。

Examples 当前 CI 会执行带 race detector 的 Go 测试，但仓库中的测试覆盖有限。示例用于演示，核心仓库相邻测试才是本轮主要行为证据。

## 手写、生成与测试边界

- Graph、ToolNode、ReAct 和 ADK 核心路径是手写实现。
- `internal/mock/**` 中带有 MockGen 与 “DO NOT EDIT” 标记的文件是生成代码；组件接口旁的 `go:generate` 是生成入口。
- Eino Examples 大部分业务示例手写，`flow/agent/deer-go` 下的部分 Hertz handler 和 router 是生成代码。
- 优先阅读 `compose/graph_test.go`、`graph_call_options_test.go`、`error_test.go`、`checkpoint_test.go`、`resume_test.go`、`tool_node_test.go`、`tool_alias_test.go` 以及 `flow/agent/react/react_test.go`。

本轮只静态阅读了这些入口，没有实际执行 `go test`、race detector 或示例。

## 落到业务系统的安全边界

模型只能提出调用意图。真正执行前至少需要：

1. 从可信请求上下文注入用户、租户和资源身份，不接受模型自行声明身份。
2. 对每次工具调用重新做对象级授权和参数 Schema 校验。
3. 写操作展示精确预览，并按风险进入人工批准或确定性策略。
4. 为外部写操作提供幂等键、状态查询和恢复记录。
5. 限制最大步骤、并发、Token、总时长和工具资源。
6. 默认不把完整 Prompt、工具参数、凭据和敏感结果写入日志。

这些边界可继续结合 [[受控 Agent 的执行、审批与恢复]] 和 [[AI 应用安全威胁建模与防护]]，但框架源码本身不能证明业务门禁已经建立。

## 本轮停止条件

满足以下条件即可停止，不遍历全部 provider 和示例：

- 能画出 `NewGraph → Compile → Runnable` 的编译运行链。
- 能解释一次 `ChatModel → ToolNode → ChatModel` 循环。
- 能区分 Tool 列表、业务授权、人工批准和幂等。
- 能从测试定位取消、最大步骤、checkpoint、恢复与工具错误。
- 能说明 Examples 的版本错位和旧 `BindTools` 风险。

真实实验应另建日期化学习记录，保存命令、运行时、依赖锁定、成功与失败输出；本文不虚构“已经复现”。

## 易变事实与重新核对

- Eino 的稳定 release、`ToolCallingChatModel` 接口和 Graph 编译策略可能变化。
- Eino Examples 没有 release/tag，默认分支提交不能长期代表稳定契约。
- Examples 依赖的核心版本可能升级；复现前重新读取 `go.mod`。
- 模型的流式 tool-call 形态属于 provider 行为，必须用目标模型重新验证。

重新研读时先核对 [Eino Releases](https://github.com/cloudwego/eino/releases)、[Eino Examples Releases](https://github.com/cloudwego/eino-examples/releases)、两个仓库的目标 commit 与 `go.mod`，再更新版本结论。

## 官方资料

- [Eino 仓库](https://github.com/cloudwego/eino)
- [Eino v0.9.13 Release](https://github.com/cloudwego/eino/releases/tag/v0.9.13)
- [Eino 官方文档](https://www.cloudwego.io/docs/eino/)
- [Eino Examples 仓库](https://github.com/cloudwego/eino-examples)
- [Eino Ext 仓库](https://github.com/cloudwego/eino-ext)

相关导读：[[OpenTelemetry GenAI 语义约定导读]] 负责解释遥测字段，[[OWASP GenAI 与 MCP Security 安全资料导读]] 负责把工具与 Agent 风险映射为安全测试。

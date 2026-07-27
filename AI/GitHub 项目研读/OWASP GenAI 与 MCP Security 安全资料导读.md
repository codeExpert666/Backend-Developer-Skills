---
title: OWASP GenAI 与 MCP Security 安全资料导读
aliases:
  - OWASP GenAI 安全资料导读
  - MCP Security 安全资料导读
tags:
  - AI
  - GitHub
  - 安全
  - OWASP
  - MCP
created: 2026-07-27T01:30:28
updated: 2026-07-27T01:30:28
---

本文说明怎样组合 OWASP LLM、Agentic 与 MCP 安全资料，并把它们约束在正式 MCP Specification 之下。目标不是背诵风险编号，而是把每个适用风险转换为资产、信任边界、确定性控制、失败测试和剩余风险。

本文是 [[GitHub AI 项目源码研读方法与清单]] 的安全资料入口。通用威胁建模与工程落点继续参考 [[AI 应用安全威胁建模与防护]]；MCP 的 Host、Client、Server、生命周期和能力协商见 [[MCP 生命周期、能力协商与安全边界]]。

本轮资料状态为“官方一手资料静态核验”：核对了 OWASP 官方资源页、PDF、官方项目仓库、MCP Versioning、`2025-11-25` 规范 tag 和 Security Best Practices，但没有部署 MCP Server、运行 Inspector、执行 conformance 或安全测试。本文不把资料清单写成项目已经安全。

## 五类资料的权威边界

核对日期为 2026-07-27。

| 资料 | 本轮版本依据 | 适合做什么 | 不能替代什么 |
| --- | --- | --- | --- |
| [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/) | 2025 版，官方资源与 PDF 于 2024-11 发布 | 识别输入、数据、输出、供应链、资源和模型层风险 | Agent 工具执行与 MCP 协议契约 |
| [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) | 2026 版，官方资源于 2025-12 发布 | 识别目标劫持、工具、身份、记忆、多 Agent 和级联失败 | 正式 MCP 方法、Schema 和 OAuth 要求 |
| [A Practical Guide for Secure MCP Server Development](https://genai.owasp.org/resource/a-practical-guide-for-secure-mcp-server-development/) | `v1.0`，2026-02 | 把 MCP Server 架构、工具、认证、隔离与持续验证落到工程检查 | MCP Specification |
| [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/) | 站点标为 `v0.1`、Phase 3 beta；仓库没有 release/tag，本轮固定 commit `0a1725be7f4234a8aac9bd028e591e2484a99254` | MCP 专项威胁发现与检查表 | 已稳定的风险标准或协议规范 |
| [MCP Specification `2025-11-25`](https://modelcontextprotocol.io/specification/2025-11-25/) | 正式版本；仓库 tag commit `38c84e9f93ad191d9eb26d92b945d17bd0efcaf3` | 定义协议、生命周期、Tools、Authorization、Tasks 与取消契约 | 面向具体业务的完整威胁模型 |

判断冲突时，MCP 正式规范与 Security Best Practices 决定协议和授权要求；OWASP 资料用于发现威胁、补充控制和设计攻击测试。OWASP MCP Top 10 仍是 beta，不能因为名称含 “Top 10” 就写成已定稿标准。

## 从资产和执行边界开始

一个带 RAG、Agent 和 MCP Tool 的应用至少跨越这些边界：

```text
不可信用户输入
  → Host 的模型与策略
  → 不可信或半可信检索内容
  → MCP Client
  → 本地或远程 MCP Server
  → Tool 参数校验与逐资源授权
  → 业务 Service / 数据库 / 外部系统
  → 脱敏 Trace、日志和评测
```

模型生成合法 JSON 只说明参数能解析，不说明用户有权执行。Server 能返回结果只说明协议调用完成，不说明结果真实、安全或业务成功。安全门禁必须放在模型之外的确定性代码中。

## LLM Top 10 2025：内容和模型应用风险

2025 版条目是：

| 编号 | 风险 |
| --- | --- |
| LLM01 | Prompt Injection |
| LLM02 | Sensitive Information Disclosure |
| LLM03 | Supply Chain |
| LLM04 | Data and Model Poisoning |
| LLM05 | Improper Output Handling |
| LLM06 | Excessive Agency |
| LLM07 | System Prompt Leakage |
| LLM08 | Vector and Embedding Weaknesses |
| LLM09 | Misinformation |
| LLM10 | Unbounded Consumption |

它适合先检查“输入、上下文、输出和资源”：

- 把用户、网页、文件、检索片段和 Tool 结果都视为不可信数据。
- 模型输出进入 SQL、Shell、HTML、URL 或工具参数前做目标语境的验证与编码。
- 不把系统 Prompt 当作秘密保险箱，也不把“模型不会泄露”当控制。
- 检索必须保留来源、租户权限、版本和引用，能够撤回污染数据。
- 设置 Token、步骤、时间、并发、检索候选和费用上限。

## Agentic Top 10 2026：从回答扩展到行动

Agentic 2026 条目是：

| 编号 | 风险 |
| --- | --- |
| ASI01 | Agent Goal Hijack |
| ASI02 | Tool Misuse and Exploitation |
| ASI03 | Identity and Privilege Abuse |
| ASI04 | Agentic Supply Chain Vulnerabilities |
| ASI05 | Unexpected Code Execution |
| ASI06 | Memory and Context Poisoning |
| ASI07 | Insecure Inter-Agent Communication |
| ASI08 | Cascading Failures |
| ASI09 | Human-Agent Trust Exploitation |
| ASI10 | Rogue Agents |

它把重点从“模型说了什么”推进到“系统执行了什么”：

- 工具白名单只限制可见集合，身份和资源授权仍要逐调用检查。
- 写操作需要精确预览、风险分级和人工批准，不能只显示自然语言摘要。
- Agent memory、checkpoint 和跨 Agent 消息都属于可污染状态。
- 任一 Agent、模型或工具失败都要有最大传播范围和熔断条件。
- 自动恢复不能重复产生副作用；先查询业务终态，再决定重试。

受控执行主线见 [[受控 Agent 的执行、审批与恢复]]。

## OWASP MCP Top 10 beta：专项威胁线索

[官方仓库固定快照](https://github.com/OWASP/www-project-mcp-top-10/tree/0a1725be7f4234a8aac9bd028e591e2484a99254) 中的十类条目是：

| 编号 | 风险 |
| --- | --- |
| MCP01 | Token Mismanagement and Secret Exposure |
| MCP02 | Scope Creep and Excessive Permissions |
| MCP03 | Tool Poisoning |
| MCP04 | Software Supply Chain Attacks |
| MCP05 | Command Injection and Execution |
| MCP06 | Intent Flow Subversion |
| MCP07 | Insufficient Authentication and Authorization |
| MCP08 | Lack of Audit and Telemetry |
| MCP09 | Shadow MCP Servers |
| MCP10 | Context Over-Sharing |

它适合补充 MCP 部署和治理问题，例如发现未经批准的本地 Server、工具描述被篡改、scope 长期膨胀、上下文跨 Server 泄露，以及没有可关联审计。但其站点明确仍在 beta，仓库也没有 release、tag 或协议 conformance；条目文件和推荐控制是社区资料，不是可执行测试套件。

## Secure MCP Server Guide 的最低工程门槛

OWASP `v1.0` Guide 可提炼为以下实现检查：

1. 使用 OAuth / OIDC，令牌短期、限定 scope，并在每次调用校验；禁止 token passthrough。
2. Tool 和资源按最小权限暴露，写操作增加审批、幂等和可审计终态。
3. 对 MCP 消息、Tool 输入与输出做 Schema、大小、类型和危险值校验。
4. Server 以非 root、受限文件系统和受限网络运行，临时资源有配额和清理。
5. Tool、Server、镜像和依赖锁定来源、版本和签名，维护批准清单。
6. 凭据保留在 Secret 管理和确定性执行层，不进入 Prompt、Tool description 或日志。
7. CI、审计、告警和滥用测试覆盖认证、越权、注入、资源耗尽与恢复。

该 Guide 以有状态的 MCP `2025-11-25` 为主要背景。正式协议若切换到无状态核心，其中 session 相关建议需要按新版重新解释，不能机械照抄。

## 正式 MCP 的工具调用与授权链

在 `2025-11-25` 中，主线是：

```text
initialize
  → 协商 protocol version 与 capabilities
  → notifications/initialized
  → tools/list
  → Host 选择向模型暴露的工具
  → 模型提出 tools/call
  → Server 对本次请求重新认证、授权和校验
  → 执行业务 Service
  → CallToolResult 或 JSON-RPC error
```

[Tools 规范](https://modelcontextprotocol.io/specification/2025-11-25/server/tools) 要求 Server 验证输入、实施访问控制、限流并清理输出；Client 应校验结果、设置超时并保留审计。Host 应让用户看到将调用的工具和参数，并能拒绝敏感操作。

### Authorization 的强制边界

[Authorization 规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) 与 [Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices) 需要一起阅读：

- HTTP MCP 使用 OAuth 2.1 体系；stdio 凭据通常来自受控环境。
- 每个 HTTP 请求携带 bearer token，并校验 token 是为当前 MCP resource 签发。
- 使用 Resource Indicators 限定 audience；禁止把未验证上游 token 直接转发给下游。
- 无效或过期令牌返回 `401`，权限不足返回 `403`。
- step-up authorization 的重试次数和 scope 升级需要跟踪，避免无限授权循环。
- OAuth redirect 必须精确校验；使用 PKCE `S256`、HTTPS 和安全令牌存储。

常见专项风险包括：

| 风险 | 确定性控制 |
| --- | --- |
| confused deputy | 对每个 client 显示并记录 consent、scope 与 redirect；正确绑定 state |
| metadata discovery SSRF | 限定 HTTPS、IP、DNS、重定向和内网地址 |
| token passthrough | 验证 audience，按目标资源重新获取令牌 |
| session hijacking | session ID 使用安全随机值并绑定用户，但从不把它当认证 |
| 本地 Server 过权 | 展示准确启动命令、显式批准、沙箱和最小文件/网络权限 |

## 错误、取消与重复副作用

MCP 区分两类失败：

- JSON-RPC error 表示方法、参数、连接或协议层失败。
- `CallToolResult.isError: true` 表示工具被调用，但执行没有成功；Client 可把该结果交给模型修正参数。

只有参数可修正且没有副作用时，模型重试才相对安全。网络超时或取消时，远端工具可能已经完成，必须用幂等键或状态查询确认，不能直接重放。

[Cancellation 规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/cancellation) 把取消定义为 best effort。接收者应停止工作和释放资源，但在无法取消或完成竞态时可以忽略取消。每个请求应有局部超时和不可突破的总上限；收到 progress 也不能无限延长。

[Tasks](https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/tasks) 在该版本仍是 experimental。Task ID 应安全随机、设置 TTL 并绑定认证上下文；不能识别请求者时不应声明 `tasks.list`，也不能让其他用户获取、取消或读取结果。

## 把风险条目转成测试

不要为每个 Top 10 编号各写一篇重复文档。按功能选择：

| 能力 | 优先风险 | 最小失败测试 |
| --- | --- | --- |
| RAG 与外部内容 | LLM01、LLM02、LLM04、LLM08、ASI06、MCP10 | 检索文档含恶意指令时不能扩大权限或泄露其他租户内容 |
| Tool 与写操作 | LLM05、LLM06、ASI02、ASI03、ASI05、MCP02、MCP05、MCP07 | 模型伪造身份、越权资源或重复请求时必须被确定性拒绝 |
| Agent 与审批 | ASI01、ASI08、ASI09、ASI10、MCP06 | 目标被篡改或人工确认描述不完整时不得执行 |
| 供应链 | LLM03、ASI04、MCP03、MCP04、MCP09 | 未批准 Server、工具描述或依赖发生变化时门禁失败 |
| 观测与秘密 | LLM02、MCP01、MCP08 | Trace、日志、错误和对象存储中不出现令牌或默认完整内容 |
| 资源控制 | LLM10、ASI08 | 无限工具循环、超大输入和慢依赖受到步骤、时间与费用上限约束 |

每条测试记录资产、入口、前置权限、攻击输入、预期拒绝、审计证据、恢复方式和剩余风险。“模型在一次测试中拒绝”不能替代确定性控制。

## 手写、生成与验证边界

- OWASP LLM、Agentic 和 Secure MCP Guide 是专家协作形成的资料；PDF 和网站展示是发布载体。
- OWASP MCP Top 10 仓库的 `2025/MCP01...MCP10` 与 `recommended-controls` 是主要手写内容；Jekyll 站点是生成展示。
- 该 OWASP 仓库的 CI 主要验证项目元数据，不是安全 conformance，也没有可证明实现安全的自动测试套件。
- MCP 规范仓库的 `schema/2025-11-25/schema.ts` 是 TypeScript Schema 来源，JSON Schema 等文件由它生成。
- Security Best Practices 是规范配套安全指导；风险案例不能覆盖正式 Authorization 和协议 Schema。

本轮没有运行任何 Server、Scanner、Inspector 或安全测试，只有静态资料核验。

## 2026-07-28 RC：必须设置次日重核

截至 2026-07-27，[MCP Versioning](https://modelcontextprotocol.io/docs/learn/versioning) 仍把 `2025-11-25` 标为 current protocol version。

官方已提供题为 [2026-07-28 Release Candidate](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/) 的公告页面，描述下一正式版本将带来破坏性调整，包括无状态核心、移除 initialize/session 握手、Tasks 移到扩展，以及扩展与授权机制变化。

在本轮核对时它仍是 RC，不是正式当前版本。本文和任何实现计划都必须设置“2026-07-28 之后重新核对”的门槛：

1. 再查 Versioning 页面确认是否正式发布。
2. 固定新 tag、完整 commit 和 Schema。
3. 比较 lifecycle、Tools、Authorization、Tasks、cancellation 与 extensions。
4. 重新审视 Secure MCP Server Guide 和现有 session 控制。
5. 确认 Java / Go SDK 已选择兼容 release，而不是只看默认分支。

## 本轮停止条件

满足以下条件即可停止资料泛读：

- 能说明五类资料的版本、权威层级和用途。
- 能为当前功能选择适用风险，而不是机械套用全部 Top 10。
- 能画出 MCP 工具调用、逐请求授权和业务执行链。
- 能解释协议错误、工具错误、best-effort 取消和重复副作用。
- 至少设计一条越权、一条 Prompt Injection 和一条资源耗尽失败测试。
- 已设置 2026-07-28 后的正式版本重核门槛。

真正完成安全验收必须另建日期化记录，保存目标系统 revision、配置、测试命令、退出码、失败证据和剩余风险；本文没有替项目完成该验收。

## 易变事实

- MCP 正式版本可能在核对日后立即变化。
- OWASP MCP Top 10 仍是 beta，编号、名称和控制可能调整。
- Agentic 与 Secure MCP 资料会随着无状态 MCP、扩展和授权机制更新。
- SDK、Inspector、conformance 和 Security Best Practices 可能晚于规范 tag 更新。

每次发布前都要重新核对正式协议、SDK release 和安全资料日期，不能只引用本文的快照。

## 官方资料

- [OWASP GenAI Security Project](https://genai.owasp.org/)
- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/)
- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
- [Secure MCP Server Development Guide](https://genai.owasp.org/resource/a-practical-guide-for-secure-mcp-server-development/)
- [OWASP MCP Top 10 Project](https://owasp.org/www-project-mcp-top-10/)
- [OWASP MCP Top 10 固定 commit](https://github.com/OWASP/www-project-mcp-top-10/tree/0a1725be7f4234a8aac9bd028e591e2484a99254)
- [MCP Versioning](https://modelcontextprotocol.io/docs/learn/versioning)
- [MCP `2025-11-25` Specification](https://modelcontextprotocol.io/specification/2025-11-25/)
- [MCP Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)
- [MCP 规范仓库 `2025-11-25` tag](https://github.com/modelcontextprotocol/modelcontextprotocol/tree/2025-11-25)

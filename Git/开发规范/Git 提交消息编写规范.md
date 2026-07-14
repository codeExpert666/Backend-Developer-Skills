---
title: Git 提交消息编写规范
aliases:
  - Git commit message 规范
  - Conventional Commits 提交规范
  - Git 提交说明怎么写
tags:
  - Git
  - Git/提交
  - Git/协作
  - Git/代码审查
created: 2026-07-13T01:07:01
updated: 2026-07-13T01:07:17
---

一条好的提交消息应让不了解上下文的人快速回答四个问题：**这次改了什么、为什么要改、影响在哪里、需要注意什么。** 它既是代码审查的索引，也是日后定位回归、执行 `git bisect`、生成变更日志和回滚时最可靠的线索。

本文推荐采用 [Conventional Commits](https://www.conventionalcommits.org/) 的结构：用统一的类型前缀描述改动性质，再用准确、简短的文字说明行为或结果。若仓库已有 `CONTRIBUTING.md`、提交校验钩子或团队约定，应以仓库规则为准；不要为了统一格式而违反项目既有规范。

关于如何在功能分支上检查、暂存并创建提交，参见 [[Git 分支与 PR 工作流]]；关于合并、rebase、cherry-pick 后如何保持历史可读，参见 [[Git 合并方式与 Cherry-pick]]。

## 一条提交消息的标准结构

最常用的格式如下：

```text
<type>(<scope>): <subject>

<body>

<footer>
```

其中只有标题是必填项。小而明显的改动通常只需标题；当动机、影响、迁移方式或兼容性不能从差异中直接看出时，再补充正文和脚注。

```text
feat(auth): add refresh-token rotation

Invalidate the previous refresh token after a successful rotation to
reduce the replay window for stolen tokens.

BREAKING CHANGE: clients must persist the newly returned refresh token.
Refs: #482
```

各部分的职责如下。

| 部分 | 是否必需 | 作用 | 示例 |
| --- | --- | --- | --- |
| `type` | 是 | 说明改动类别，便于检索、审阅和生成日志 | `feat`、`fix`、`docs` |
| `scope` | 否 | 标明受影响的模块、领域或接口 | `auth`、`api`、`billing` |
| `subject` | 是 | 用一句话说明本次提交带来的变化 | `reject expired refresh tokens` |
| `body` | 否 | 补充原因、实现取舍、影响和验证信息 | 为什么需要轮换令牌 |
| `footer` | 否 | 声明破坏性变更、关联事项、兼容或迁移要求 | `BREAKING CHANGE:`、`Refs: #482` |

> [!important] 提交消息描述的是“这个提交本身”，不是整个需求或 PR
> PR 标题和描述可以概括一组提交；单个提交只应描述自己引入的、可独立理解的改动。不要把多个无关修改都写成 `feat: implement user center`，也不要用 PR 的长说明替代每个提交的标题。

## 1. 先保证提交边界清楚

消息写得再漂亮，也无法挽救边界混乱的提交。一个理想提交应只有一个主要意图，且在其所在历史位置能被理解、构建和验证。通常遵循“一个可说明、可审查、可回滚的逻辑单元一个提交”。

提交前先查看实际要提交的内容：

```bash
git status
git diff
git add <文件或目录>
git diff --staged
git diff --check --staged
```

以下情况通常应拆分提交：

| 混在一起的改动 | 建议拆分方式 |
| --- | --- |
| 新增功能与无关格式化 | 功能一个提交；纯格式化单独一个提交 |
| 生产代码变更与仅为其他任务准备的重命名 | 先做可独立验证的重命名，或留给对应任务 |
| 依赖升级与依赖升级后暴露的业务修复 | 升级、修复分别提交；说明两者关系 |
| 数据库迁移与依赖该迁移的大量业务改动 | 按可部署顺序拆分，并在消息正文说明前置条件 |

与同一次行为变更紧密相关的实现、测试和必要文档可以放在同一个提交中。例如修复空订单计算时，同时新增对应单元测试是合理的；不要为了机械追求“小提交”而把代码与使其通过的测试拆成无法独立验证的两半。

> [!warning] 不要把临时产物伪装成有意义的提交
> 调试日志、IDE 配置、构建产物、密钥、本地环境文件和无关改动不应进入提交。若确实需要个人检查点，使用只在个人分支上的 `wip:` 提交，并在提交 PR 前按团队约定整理；详见 [[Git 未完成开发时安全验证其他分支代码]]。

## 2. 选择正确的类型

团队可以自行扩展类型，但同一仓库应保持稳定，避免 `bugfix`、`fixes`、`repair` 混用而失去检索价值。以下是一套适合大多数后端项目的最小集合。

| 类型 | 何时使用 | 示例 |
| --- | --- | --- |
| `feat` | 新增对用户、调用方或系统可见的能力 | `feat(api): add cursor pagination` |
| `fix` | 修复已存在行为的缺陷 | `fix(order): reject negative quantities` |
| `refactor` | 调整内部结构，不改变预期外部行为 | `refactor(auth): extract token verifier` |
| `perf` | 以性能为主要目的的改动 | `perf(query): batch load order items` |
| `test` | 新增或调整测试，未改变生产行为 | `test(auth): cover revoked refresh tokens` |
| `docs` | 仅修改文档、注释或说明 | `docs(api): explain idempotency header` |
| `build` | 构建系统、打包、依赖解析或构建脚本 | `build: upgrade Maven wrapper` |
| `ci` | 持续集成或自动化工作流配置 | `ci: cache Gradle dependencies` |
| `chore` | 不属于其他类别的维护性事务 | `chore: refresh local development fixtures` |
| `style` | 仅代码格式、空白或排序，且不改变语义 | `style: apply formatter` |
| `revert` | 明确撤销某个已存在的提交 | `revert: feat(api): add cursor pagination` |

`refactor`、`perf` 与 `fix` 容易混淆：若主要目标是纠正错误结果，使用 `fix`；若主要目标是降低资源消耗且行为不变，使用 `perf`；若两者都不是，只是改善结构，使用 `refactor`。若一个改动同时修复错误又提高性能，应以其最重要、最可见的结果为准，并在正文写明其他影响，或拆成两个提交。

### scope 要表达业务边界，不要凑格式

`scope` 可选，但对多模块、单体分层或多个服务的仓库很有帮助。它通常使用小写短横线名称，取项目中稳定且大家熟悉的边界：业务域（`order`）、模块（`auth`）、接口（`api`）、基础设施（`redis`）或部署单元（`worker`）。

| 推荐 | 不推荐 | 原因 |
| --- | --- | --- |
| `feat(auth): add passkey login` | `feat(login-service-impl): add passkey login` | 不暴露易变化的实现类名 |
| `fix(api): return 404 for missing orders` | `fix(misc): fix order issue` | 范围和问题都应可辨认 |
| `ci(release): publish image after tag` | `ci: update yaml` | 说明改动属于哪条流水线 |

如果改动跨越多个边界，不要堆叠成 `feat(api,service,repository): ...`。优先按可审查的边界拆分；确实不可拆分时，可以省略 `scope`，并在正文列出影响模块。

## 3. 写好标题：让日志列表本身可读

标题是 `git log --oneline`、GitHub/GitLab 提交列表和审阅页面最常见的视图，应脱离上下文也能表达结果。推荐使用小写类型、英文半角冒号和一个空格：

```text
type(scope): concise description
```

标题的具体规则：

1. 标题尽量不超过 50 个英文字符；受工具或团队规则限制时，最多不超过 72 个字符。中文按可读性控制，不要为了凑字符数写成难懂的缩写。
2. 标题首字母通常小写，末尾不加句号。若团队统一使用中文标题，也应统一标点和大小写风格。
3. 描述应从动词开始，写明“做了什么”或“最终得到什么”，不要只写文件名或技术动作。
4. 避免模糊词，如 `update`、`change`、`modify`、`optimize`、`adjust`；除非后面明确说明对象和目的。
5. 不要在标题里塞入实现细节、Issue 全文、测试结果或多个由 `and` 连接的无关事项。

标题的好坏对比如下。

| 不佳标题 | 为什么不佳 | 改写示例 |
| --- | --- | --- |
| `update user service` | 没有说明更新的目的和结果 | `feat(user): add email verification` |
| `fix bug` | 无法搜索、审阅或回滚 | `fix(cart): preserve items after login` |
| `modify OrderService.java` | 只描述文件，不描述行为 | `fix(order): prevent duplicate payment requests` |
| `optimize SQL` | 不清楚优化目标与影响 | `perf(report): reduce monthly query count` |
| `feat: add API and update docs and fix test` | 多个意图混在一条提交 | 拆分为功能、文档或测试提交 |
| `临时提交` | 不可追溯，且容易被带入主线 | `wip: save import validation checkpoint` |

中文团队也可以统一采用中文 subject，例如 `feat(订单): 拒绝负数商品数量`。关键不是使用哪种语言，而是整个仓库保持一致，并让术语与代码、接口和领域语言一致。一个仓库中不要混用 `add`、`新增`、`增加`、`加一个` 等任意风格。

## 4. 需要时补正文：解释“为什么”与影响

代码差异通常能说明“怎么做”，却不一定能说明“为什么现在这样做”。以下情况应写正文：

- 采用了不直观的实现或刻意没有采用某种方案。
- 修改影响 API、数据模型、缓存、权限、性能、部署或兼容性。
- 有迁移步骤、开关、前置条件、后续清理任务或已知限制。
- 问题的触发条件并不容易从测试名称或代码中看出。

正文从标题空一行后开始，建议每行控制在约 72 个英文字符以内，便于终端、邮件和 Git 工具显示。先写背景和动机，再写关键决策和影响；避免逐行复述代码。

```text
fix(import): reject duplicate source rows

The importer previously kept the last row silently, which made the final
result depend on spreadsheet ordering. Reject duplicates before persistence
so users can correct the source file without partial writes.

The validation runs before the transaction opens. Existing imports are not
changed.
```

正文不是测试报告的替代品。测试命令、完整输出、截图和部署步骤通常更适合 PR 描述或发布记录；只有当测试策略本身是重要决策时，才在正文中简要说明，例如“使用契约测试保障 v1 客户端兼容性”。

> [!tip] 先写一句标题，再问两个问题
> “为什么现在需要这项改动？”以及“如果回滚它，会失去什么或恢复什么？”如果两题都答不清，通常说明提交边界、类型或标题还不够准确。

## 5. 用脚注记录关联和破坏性变更

脚注位于正文之后，并再空一行。常用来关联需求、Issue、事故单或声明兼容性影响。关联格式需以团队的追踪系统规则为准。

```text
feat(api): require idempotency keys for payment creation

BREAKING CHANGE: POST /payments now rejects requests without the
Idempotency-Key header. Update clients before deploying this release.

Refs: #631
```

对于破坏性变更，Conventional Commits 有两种等价写法：在类型或范围后加 `!`，或在脚注写 `BREAKING CHANGE:`。影响较大时建议两者都使用：前者让标题列表一眼可见，后者写清迁移要求。

```text
feat(api)!: replace offset pagination with cursor pagination

BREAKING CHANGE: `page` and `size` are removed from GET /orders.
Clients must send `cursor` and `limit` instead.
```

以下内容只要存在，就不能只用一句 `feat:` 或 `fix:` 带过：

| 影响类型 | 脚注或正文至少说明什么 |
| --- | --- |
| API 不兼容 | 哪个接口、字段或语义变化；客户端如何迁移 |
| 数据迁移 | 迁移顺序、是否可回滚、需要的窗口或前置条件 |
| 配置变更 | 新配置项、默认值、安全影响和部署位置 |
| 权限变更 | 哪些主体的访问能力改变，是否需要补授权 |
| 行为开关 | 默认状态、启用条件、清理计划和回退方式 |
| 已知限制 | 尚未覆盖的边界、后续 Issue 或临时措施 |

`Fixes: #123`、`Closes: #123` 可能在特定平台上自动关闭 Issue；只有确认该事项会随目标分支合并而关闭时才使用。只是建立关联时，优先使用 `Refs: #123`，避免误关闭尚未完成的事项。

## 6. 常见场景的可复用示例

下面的消息可作为起点，但应替换成真实的领域术语和结果。

| 场景 | 推荐提交消息 | 说明 |
| --- | --- | --- |
| 新增接口能力 | `feat(order): add order cancellation endpoint` | 描述新能力，而非“新增 controller” |
| 修复边界条件 | `fix(invoice): reject zero-value line items` | 指出具体错误行为 |
| 安全修复 | `fix(auth): revoke sessions after password reset` | 正文补充风险与兼容性影响 |
| 重构内部代码 | `refactor(cache): isolate serialization strategy` | 明确预期外部行为不变 |
| 性能改善 | `perf(search): cache product facet counts` | 正文写命中条件、失效策略和指标 |
| 新增回归测试 | `test(payment): cover webhook retry ordering` | 仅测试改动时使用 `test` |
| 文档修正 | `docs(deploy): clarify rollback procedure` | 说明文档帮助完成什么任务 |
| 升级依赖 | `build: upgrade PostgreSQL driver to 42.7.4` | 正文说明兼容性或安全原因 |
| CI 修改 | `ci: run integration tests against PostgreSQL 16` | 不要笼统写“update workflow” |
| 撤销线上风险改动 | `revert: feat(flag): enable strict quota enforcement` | 正文注明被撤销的提交和原因 |

评审意见导致的修改也应写出实际修正，不要机械使用 `fix: address review comments`。例如：

```text
fix(api): return validation details for invalid cursors
```

只有当一次提交确实只是整理多个彼此独立、且不值得单独保留历史的评审细节时，`fix: address review feedback` 才可以接受；即使如此，正文也应列出主要修正点。

## 7. 提交前检查清单

创建提交前，从“代码范围”和“消息质量”两方面核对。

- [ ] 当前暂存区只包含本次意图相关的文件，未混入生成物、密钥、调试代码或无关格式化。
- [ ] 该提交能够回答“改了什么”和“为什么要改”；必要时已写正文。
- [ ] `type` 与实际改动一致，`scope` 是稳定、可理解的边界。
- [ ] 标题具体、可检索、无句号，且没有用模糊词掩盖真实变化。
- [ ] 破坏性变更、迁移、配置、安全或兼容性影响已明确说明。
- [ ] 已关联必要的 Issue、需求或事故记录，但没有误用会自动关闭事项的关键字。
- [ ] 已检查 `git diff --staged` 和 `git diff --check --staged`，并运行项目要求的验证。

命令行提交时，可直接写短标题：

```bash
git commit -m "fix(order): reject negative quantities"
```

需要正文和脚注时，建议不传 `-m`，让 Git 打开编辑器；这样更容易保留标题、空行、正文和脚注的结构：

```bash
git commit
```

若必须在命令中提供多段消息，使用多个 `-m`：第一个是标题，后续段落是正文。不要把换行写成难以阅读的一长串 shell 转义。

```bash
git commit \
  -m "feat(auth): add refresh-token rotation" \
  -m "Invalidate the previous token after successful rotation." \
  -m "Refs: #482"
```

## 8. 已经写错了如何修正

尚未推送、且确认当前分支只由自己使用时，可以修改最近一次提交消息：

```bash
git commit --amend
```

如果提交已经推送到远程，修改会改变提交 ID。仅在个人分支且团队允许改写历史时，完成修改后使用：

```bash
git push --force-with-lease
```

不要对 `main`、发布分支、同事共享分支或已经被其他分支依赖的提交随意 amend 后强推。此时应保留历史，必要时再创建新的更正提交，或在 PR 描述中补充背景。关于 rebase、强推和共享历史的边界，参见 [[Git 合并方式与 Cherry-pick]]；完整的分支提交与 PR 流程，参见 [[Git 分支与 PR 工作流]]。

> [!summary] 最小可执行规则
> 每个提交只做一件可独立说明的事；标题使用 `type(scope): 具体结果`；原因和影响不明显时写正文；任何破坏性变更、迁移和兼容性要求必须写清；提交前先看暂存差异，再运行相应检查。

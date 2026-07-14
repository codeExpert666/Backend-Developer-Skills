---
title: Git 未完成开发时安全验证其他分支代码
aliases:
  - 在未提交改动存在时验证其他分支
  - Git worktree 并行验证工作流
tags:
  - Git
  - Git/Worktree
  - Git/分支
  - Git/协作
  - Git/开发工作流
created: 2026-07-13T00:40:45
updated: 2026-07-13T00:40:45
---

开发当前功能时，工作区常常仍有未提交的文件，甚至正处于无法编译、无法通过测试的中间状态；与此同时，又需要拉取并运行另一条分支的代码，以验证同事的功能、排查线上问题或复现缺陷。此时的目标是**在不触碰当前开发现场的前提下，得到另一分支可运行的独立工作目录**。

默认选择 `git worktree`。它把同一仓库的对象和引用共享给多个工作目录：当前目录继续保留自己的未提交改动，另一个目录检出待验证分支。当前改动是否有语法错误并不影响这个过程，因为 Git 不会要求它们先通过编译或测试。

本文假定：

- 当前工作目录位于仓库根目录，正在开发 `feature/current-task`。
- 要验证的是远程分支 `origin/feature/payment`。
- 远程仓库名为 `origin`；若实际名称不同，请替换。
- 只需查看、构建、运行或测试对方分支，不准备直接在该分支上开发。

日常从 `main` 开始开发和提交 PR 的流程见 [[Git 分支与 PR 工作流]]；若实际需求是“自己的功能依赖同事分支并要提交 PR”，应使用 [[Git 依赖远程协作分支的开发与 PR 工作流]]，而不是本文的临时验证流程。

## 先判断：只是验证，还是要整合代码

这两种目的不能混为一谈。

| 目的 | 推荐动作 | 是否修改当前开发分支 |
| --- | --- | --- |
| 运行、调试、阅读另一分支 | 新建临时 `worktree` 并检出该分支 | 否 |
| 临时比较两个分支的文件或提交 | `fetch` 后使用 `git diff`、`git log` | 否 |
| 当前功能真正依赖对方功能 | 在自己的分支上按约定 merge 或 rebase，并建立依赖 PR | 是 |
| 只需要对方的少数提交 | 在隔离环境评估后 `cherry-pick -x` | 是 |

因此，“我想拉取其他分支代码验证”不应默认执行 `git pull origin feature/payment`。这条命令会先获取远程更新，再把目标分支**合并进当前分支**；它会改变当前开发现场，可能产生冲突，也会让未完成的功能和验证目的混在一起。关于 merge、rebase 与 cherry-pick 的选择可参考 [[Git 合并方式与 Cherry-pick]]。

> [!important] 当前代码有语法错误不是问题
> Git 的分支、暂存和 worktree 操作只处理文件与提交，不会检查代码是否可编译。真正需要避免的是：为了切换分支，覆盖、暂存后遗忘，或把未完成代码误合并到其他分支。

## 推荐工作流：用 worktree 建立隔离验证目录

### 1. 在当前目录确认现场，并只更新远程引用

先确认当前改动确实还在当前工作目录，然后获取远程分支的最新引用：

```bash
git status
git branch --show-current
git fetch origin --prune
git branch -r --list 'origin/feature/payment'
git log --oneline --decorate -5 origin/feature/payment
```

这里的 `git fetch origin --prune` 只更新本地保存的 `origin/*` 引用，不会修改当前分支、暂存区或工作区。最后一条命令没有输出时，应先核对分支名、远程名称、访问权限和该分支是否已推送。

建议在仓库目录的同级位置创建验证目录，而非放在当前仓库内。假设当前仓库目录名是 `my-service`：

```text
workspace/
├── my-service/                 # 当前开发现场：feature/current-task，保留未提交改动
└── my-service-verify-payment/  # 临时验证目录：origin/feature/payment
```

### 2. 为待验证分支创建 detached worktree

在当前仓库根目录执行：

```bash
git worktree add --detach ../my-service-verify-payment origin/feature/payment
```

`--detach` 表示验证目录直接检出 `origin/feature/payment` 当前指向的提交，而不创建或占用本地分支。它最适合“只验证、不提交”的场景，也避免把远程分支误当作自己的日常开发分支。

随后进入新目录，确认它确实处于预期提交：

```bash
cd ../my-service-verify-payment
git status
git rev-parse --short HEAD
git log --oneline --decorate -3
```

在 detached HEAD 状态下，`git branch --show-current` 会为空，这是正常现象。只要没有提交新代码，就可以按项目要求安装依赖、运行服务、执行构建或测试：

```bash
# 仅选择当前项目实际使用的命令
npm test
go test ./...
mvn test
```

原始目录中的未提交文件不会被这个验证目录看到、移动或修改。可在另一个终端回到原始目录检查：

```bash
cd ../my-service
git status
```

它应仍显示验证前的开发现场。

> [!tip] 需要修改验证代码时，不要直接在 detached HEAD 上提交
> 若只是临时加日志、改配置或验证修复，先在验证目录创建一个专用本地分支：`git switch -c verify/payment-debug`。该分支仅用于本地实验；验证结束后可以删除，不应推送或作为业务分支使用，除非团队另有约定。

### 3. 验证结束后移除 worktree

先离开验证目录，再从任一工作目录查看并移除它：

```bash
cd ../my-service
git worktree list
git worktree remove ../my-service-verify-payment
git worktree list
```

`git worktree remove` 会拒绝删除含有未提交改动的验证目录。这是保护措施：先检查该目录里的改动，决定要提交到专用分支、导出为补丁，还是明确丢弃；不要为了省事直接使用 `--force`。

正常移除后，Git 会清理该 worktree 的记录。只有目录被手动删除或发生异常中断，才可回到主工作目录执行：

```bash
git worktree prune
```

## 需要反复验证同一分支时

不必每次都删除并重建验证目录。保留 worktree，在开始下一轮验证前于**验证目录**更新它即可：

```bash
cd ../my-service-verify-payment
git fetch origin --prune
git switch --detach origin/feature/payment
git log --oneline --decorate -3
```

`git switch --detach origin/feature/payment` 会让验证目录移动到该远程分支最新提交。若验证目录有未提交修改，先处理这些修改；不要强行切换并覆盖它们。

若希望在验证目录中保留调试提交，可创建本地验证分支而不是 detached HEAD：

```bash
# 在原始仓库根目录执行；分支名应是本地临时用途
git worktree add -b verify/payment-debug ../my-service-verify-payment origin/feature/payment
```

之后要同步远程分支时，先 `git fetch origin --prune`，再明确决定是否将 `origin/feature/payment` merge 或 rebase 到 `verify/payment-debug`。不要把临时验证分支设置成长期需要 `pull` / `push` 的协作分支。

## 备用方案：短时使用 stash 后切换工作区

如果环境限制导致不能创建 worktree，例如磁盘空间紧张、脚本强依赖固定目录，才考虑 `stash`。它适合几分钟内的短暂切换，不适合一边继续开发、一边长时间验证。

| 检查项 | `worktree` | `stash` 后切换 |
| --- | --- | --- |
| 当前未提交改动是否留在原目录 | 是 | 否，暂存于 stash |
| 能否同时打开两套代码 | 是 | 否 |
| 是否容易忘记恢复开发现场 | 低 | 较高 |
| 是否适合当前代码无法编译 | 是 | 是 |
| 默认建议 | 首选 | 仅作短期备用 |

安全的 stash 步骤如下：

```bash
# 1. 在当前开发分支，保存已跟踪、暂存和未跟踪文件
git status
git stash push -u -m "wip: feature/current-task before verify payment"
git stash list

# 2. 以 detached HEAD 检出待验证的远程分支，避免误改本地协作分支
git fetch origin --prune
git switch --detach origin/feature/payment

# 3. 构建、运行或测试后，切回原开发分支
git switch feature/current-task
git stash apply --index stash@{0}
git status

# 4. 确认开发现场已恢复且内容正确后，再删除对应 stash
git stash drop stash@{0}
```

`-u` 会同时保存未跟踪文件；不加它时，新建但尚未 `git add` 的源码、配置或测试文件不会进入 stash。一般不建议使用 `-a`，因为它还会收集被 `.gitignore` 忽略的构建产物、本地环境文件等，恢复时更难判断。

这里使用 `git stash apply` 而不是立即 `pop`：恢复发生冲突或内容不符合预期时，stash 记录仍会保留，便于核对。确认无误后再显式 `drop`。若 `apply` 出现冲突，先运行 `git status`，解决冲突后再决定是否删除 stash；不要重复执行 `apply`。

> [!warning] 不要在没有确认的情况下清理 stash
> `git stash clear` 会删除所有 stash，可能包含其他任务的现场。应先用 `git stash list` 找到目标条目，再使用带明确引用的 `git stash drop stash@{n}`。

## 何时适合创建 WIP 提交

当当前工作将跨越较长时间、需要在多台机器继续，或希望让阶段性现场可恢复、可比较时，创建一个**只在自己分支或专用 WIP 分支上**的提交，比 stash 更可靠。Git 可以提交尚未通过编译的代码；是否允许该提交推送到远程，仍以团队规范、分支保护规则和敏感信息要求为准。

```bash
git status
git add <本次工作涉及的文件>
git diff --staged
git commit -m "wip: save current-task checkpoint"
```

随后仍使用前文的 `git worktree` 验证另一分支。恢复开发时继续在自己的分支上工作即可。准备提交 PR 前，如果团队不希望保留 WIP 提交，可在确认该分支仅由自己维护后，通过 interactive rebase 整理提交；已共享或已被他人基于其开发的分支不要擅自改写历史。

如果仓库的 pre-commit hook 因语法错误拒绝提交，不要为了创建 WIP 而绕过项目的质量门禁；改用 `worktree`，或短期使用带说明的 `stash`。

## 如果必须同时验证“自己的未提交改动 + 对方分支”

这已经不是单纯查看对方分支，而是一次**集成验证**。不要在当前工作分支直接 `pull` 或 merge 对方分支；应在单独的验证 worktree 中完成，并把它视为可随时丢弃的实验结果。

推荐顺序如下：

1. 先用前文的 `git stash push -u -m "wip: ..."` 为当前未提交改动建立可恢复快照。
2. 新建一个以对方分支为基线的临时 worktree，例如 `git worktree add -b verify/current-on-payment ../my-service-integration origin/feature/payment`。
3. 立即回到原始工作目录执行 `git stash apply --index stash@{0}`，恢复日常开发现场。
4. 进入 `../my-service-integration`，把同一条 stash 应用到该隔离目录。
5. 解决应用 WIP 时出现的冲突并运行集成测试。确认结果后删除验证 worktree；原始开发分支不做 merge。

示例命令如下。路径和分支名应按实际项目替换：

```bash
# 在原始开发目录：为未提交现场建立可多次 apply 的快照
git stash push -u -m "wip: current-task for payment integration check"

# 创建以对方分支为基线的独立验证目录
git worktree add -b verify/current-on-payment ../my-service-integration origin/feature/payment

# 立即回原始目录恢复日常开发现场
git stash apply --index stash@{0}

# 在临时验证目录应用自己的 WIP；冲突即是集成风险的信号
cd ../my-service-integration
git stash apply stash@{0}
git status
```

在这条路径中，冲突和测试失败都是有价值的验证结果：它们说明两个功能尚不能直接组合。不要把临时分支的冲突解决或测试性改动误提交到 `feature/current-task`。验证完成并确认原始目录已恢复后，再删除 stash 和临时 worktree。

> [!note] 为什么以对方分支作为验证基线？
> 这样验证目录一开始就是对方分支的真实文件树，再叠加自己的 WIP。若应用 stash 时发生冲突，或叠加后测试失败，结果直接说明当前 WIP 与对方代码尚不能组合；当前开发分支始终不发生 merge。

## 常见误操作与恢复方向

| 现象 | 常见原因 | 先做什么 |
| --- | --- | --- |
| `git switch` 拒绝切分支 | 当前改动会被目标分支覆盖 | 停止切换，改用 `worktree`；不要先丢弃改动 |
| 当前分支出现意外 merge commit | 执行了 `git pull origin <其他分支>` 或 `git merge` | 立即 `git status`、`git log --graph`，在未共享前评估 `git reset`；已共享则优先考虑 `revert` |
| `git worktree add` 提示分支已被另一个 worktree 占用 | 同一个本地分支不能同时检出到两个 worktree | 用 `--detach origin/<branch>`，或创建 `verify/<name>` 临时分支 |
| stash 恢复后有冲突 | 当前分支在 stash 期间发生变化，或恢复目标选错 | 保留 stash，先解决或中止当前处理，再核对 `git diff` |
| `git worktree remove` 拒绝执行 | 验证目录仍有改动 | 进入该目录决定保存、导出或丢弃，不要直接强制删除 |

如果误把改动加入暂存区，`git restore .` 和 `git clean -fd` 不会移除已暂存内容；应先根据 `git status` 判断状态，再使用 `git restore --staged`。具体区别见 [[Git 合并方式与 Cherry-pick]] 中的工作区保护原则。

## 可直接复用的最小命令清单

只验证其他远程分支，并保留当前未完成现场：

```bash
# 在当前开发仓库根目录
git status
git fetch origin --prune
git worktree add --detach ../my-service-verify-payment origin/feature/payment

# 在新目录验证
cd ../my-service-verify-payment
git log --oneline --decorate -3
# 运行项目实际的构建、服务或测试命令

# 验证完成后，从原仓库目录移除临时工作目录
cd ../my-service
git worktree remove ../my-service-verify-payment
```

这套流程的边界很简单：**需要并行，就用 worktree；只需短暂切换，才用 stash；真的要把代码带回自己的功能分支时，才进入 merge、rebase 或 cherry-pick 的协作流程。**

---
title: Git 依赖远程协作分支的开发与 PR 工作流
aliases:
  - 基于同事远程分支继续开发并提交 PR
  - Git 堆叠 PR 工作流
tags:
  - Git
  - Git/分支
  - Git/协作
  - Git/Pull-Request
  - Git/Stacked-PR
created: 2026-07-13T00:15:42
updated: 2026-07-13T00:15:42
---

当自己的功能依赖同事尚未合入主线的改动时，不能把对方分支当作普通的 `main` 更新来处理。正确的做法是：从对方的远程分支建立自己的功能分支，开发期间显式同步该分支，并先向**对方分支**提交 PR；等依赖分支合入 `main` 后，再把自己的分支迁移到 `main` 并调整 PR 目标分支。

这种按依赖顺序串起来的 PR 常被称为 *stacked PR*（堆叠 PR）。它的核心规则只有一条：**代码依赖谁，PR 的目标分支就先指向谁。** 这样审阅者只会看到自己的新增改动，不会在 PR 中重复看到同事的提交。

本文假定：

- 远程仓库为 `origin`，主线为 `main`。
- 同事负责的远程分支是 `origin/feature/shared-api`。
- 自己要维护的分支是 `feature/add-export`，并推送为 `origin/feature/add-export`。
- 两条分支均可正常读取；是否允许改写自己的已推送分支历史，仍以团队规则为准。

关于从 `main` 创建独立功能分支的常规流程，可参考 [[Git 分支与 PR 工作流]]；关于 merge、rebase 与 cherry-pick 的原理和冲突处理，可参考 [[Git 合并方式与 Cherry-pick]]。

## 先判断属于哪一种协作关系

先不要急着执行 `pull`。不同关系决定了分支起点、同步命令和 PR 目标分支。

| 情况 | 例子 | 推荐做法 | PR 的目标分支 |
| --- | --- | --- | --- |
| 完整依赖同事的未合入功能 | 导出功能需要同事新加的 API | 从同事分支创建自己的分支，并持续同步 | 同事分支；其合入后再改为 `main` |
| 只需要同事的一个小修复 | 只需一个空值校验提交 | 用 `cherry-pick -x` 复制指定提交 | `main` 或当前稳定目标分支 |
| 同事分支已经合入主线 | 依赖的 API 已在 `main` | 更新 `main` 后从其创建分支 | `main` |
| 只想查看、验证同事代码 | 本地调试或代码阅读 | 创建临时分支或 detached HEAD，不在其上开发 | 不创建 PR |

本文重点讲第一种情况。若只需少数提交，请直接看文末的“只需要个别提交”章节；不要先把整条分支 merge 进来，再试图从 PR 中剔除。

## 先看清分支依赖和 PR 顺序

开始时，`feature/shared-api` 已经从 `main` 派生；自己的 `feature/add-export` 再从它派生：

```text
main:       A---B
                 \
shared-api:       C---D
                       \
add-export:             E---F
```

PR 的依赖关系应为：

```text
PR-1: feature/shared-api  -> main
PR-2: feature/add-export  -> feature/shared-api
```

若一开始就让 PR-2 指向 `main`，比较范围会包含 `C`、`D`、`E`、`F`。这会造成两个问题：审阅者无法区分谁负责了哪些改动，而且在 PR-1 尚未合入时，PR-2 也不应独立合入 `main`。

> [!important] “拉取对方分支”不等于让本地分支跟踪对方分支
> `origin/feature/shared-api` 是要整合进来的**上游依赖**；自己的本地分支最终应跟踪 `origin/feature/add-export`。不要在自己的开发分支上长期把 upstream 配成同事的远程分支，否则日常 `git pull`、`git push` 的意图会变得混乱。

## 0. 开始前的安全检查

切分支、merge 或 rebase 都应从可解释的工作区状态开始：

```bash
git status
git branch --show-current
git remote -v
git fetch origin --prune
git branch -r --list 'origin/feature/shared-api'
```

`git status` 最好显示工作区干净。若有未提交改动，先提交到它所属的分支，或使用 `git stash push -u -m "wip: <说明>"` 暂存；不要为了切分支而直接丢弃不明改动。

最后一条命令没有输出时，先确认分支名、远程仓库和访问权限。可进一步检查同事分支的最新提交和与主线的差异：

```bash
git log --oneline --decorate -10 origin/feature/shared-api
git log --oneline origin/main..origin/feature/shared-api
git diff --stat origin/main...origin/feature/shared-api
```

这里的 `fetch` 只更新本地保存的远程引用，不会修改当前工作区。相比之下，`git pull origin feature/shared-api` 会在当前分支上直接执行“获取并整合”，既不便于先检查差异，也容易在错误的分支上制造一次 merge；不要把它作为跨分支同步的默认命令。

## 1. 从同事的远程分支创建自己的分支

首次开始依赖开发时，以已获取的远程分支为起点创建一个**不同名称**的本地分支：

```bash
git fetch origin --prune
git switch -c feature/add-export origin/feature/shared-api
git branch --show-current
git log --oneline --decorate -3
```

此时 `feature/add-export` 的起点就是同事分支当前的末端。开发并完成第一个可推送的提交后，将自己的分支发布到远程，并建立正确的跟踪关系：

```bash
git add <文件或目录>
git diff --staged
git commit -m "feat: add export endpoint"
git push -u origin feature/add-export
git branch -vv
```

`git branch -vv` 中，`feature/add-export` 应显示跟踪 `origin/feature/add-export`，而不是 `origin/feature/shared-api`。以后常规推送只需 `git push`。

> [!tip] 已经在自己的分支上开发怎么办？
> 不必重新创建分支。先切到自己的分支，再按下一节的“merge 同步”或“rebase 同步”把 `origin/feature/shared-api` 整合进来。只有在尚未产生自己的提交时，重新从同事分支创建会更简单。

## 2. 开发期间同步同事分支

同事继续提交后，先 `fetch`，再明确选择如何将新的上游依赖整合到自己的分支。开始同步前应确认当前分支确实是自己的分支：

```bash
git switch feature/add-export
git status
git fetch origin --prune
git log --oneline HEAD..origin/feature/shared-api
```

最后一条命令有输出，表示同事分支有本地分支尚未包含的新提交。此时选择 merge 或 rebase；两者都应只在自己的分支上执行，绝不能在同事分支或 `main` 上执行。

### 方案 A：merge 同步（已推送、多人协作或不想改写历史时）

在自己的分支上创建一个合并提交，把同事分支的最新内容接进来：

```bash
git switch feature/add-export
git merge --no-ff origin/feature/shared-api
# 解决冲突、完成检查后
git push
```

`--no-ff` 让这次“同步了哪个上游版本”在历史中清晰可见；如果团队希望尽量少的合并节点，可以省略它，或者按团队规范使用普通 `git merge`。这种方式不会改写已有提交 ID，适合自己的分支已被他人协作、已经有 PR 且不希望强推的场景。

发生冲突时，按以下步骤处理：

```bash
git status
# 编辑冲突文件，删除 <<<<<<<、=======、>>>>>>> 标记并保留正确结果
git add <已解决的文件>
git diff --check
git commit
```

若发现本次同步的目标分支选错，或不想继续：

```bash
git merge --abort
```

确认 `git status` 显示正在合并时才可使用 `--abort`。完成 merge 后，继续开发、测试并正常 `git push` 即可。

### 方案 B：rebase 同步（仅自己维护分支且团队偏好线性历史时）

rebase 会把自己的提交重新应用到同事分支的最新提交之后：

```bash
git switch feature/add-export
git rebase origin/feature/shared-api
```

同步后的形状如下，自己的 `E`、`F` 会被重放为新的 `E'`、`F'`：

```text
同步前：

shared-api: C---D---G
                     \
add-export:           E---F

同步后：

shared-api: C---D---G
                     \
add-export:           E'---F'
```

发生冲突时：

```bash
git status
# 编辑冲突文件后
git add <已解决的文件>
git rebase --continue
```

如需完整放弃这次 rebase：

```bash
git rebase --abort
```

rebase 会改变自己的提交 ID。因此，若 `feature/add-export` 已推送，确认该分支只由自己维护后再安全更新远程：

```bash
git push --force-with-lease
```

不要使用裸 `git push --force`，也不要对同事分支、`main` 或其他共享分支做 rebase。若同事也在自己的分支上提交，选择 merge 同步，或先共同约定重写历史的流程。

## 3. 同步后核对：确认 PR 只包含自己的改动

无论使用 merge 还是 rebase，同步完成后先看相对同事分支的提交和差异：

```bash
git log --oneline origin/feature/shared-api..HEAD
git diff --stat origin/feature/shared-api...HEAD
git diff --check origin/feature/shared-api...HEAD
git log --oneline --left-right --graph HEAD...origin/feature/shared-api
```

其中三点比较 `A...B` 会从共同祖先开始比较，通常适合检查 PR 实际会展示的改动。`git log origin/feature/shared-api..HEAD` 应主要是自己的提交；若看到不认识的提交，先停下来确认分支起点和最近一次同步方式。

然后运行项目规定的格式化、静态检查、构建和测试，并检查工作区：

```bash
# 替换为项目实际命令，例如：
npm test
go test ./...
mvn test

git status
```

示例中的测试命令互斥选择，不需要在每个项目都执行。

## 4. 创建指向同事分支的 PR

首次推送自己的分支后，PR 的 `base` 必须是同事分支，而不是 `main`：

```bash
git push -u origin feature/add-export
gh pr create \
  --base feature/shared-api \
  --head feature/add-export \
  --fill
```

在 PR 描述开头说明依赖关系和合并顺序，例如：

```text
依赖 PR #123（feature/shared-api -> main）。
本 PR 仅包含导出功能，目标分支暂为 feature/shared-api。
请先合并 PR #123；依赖合入 main 后，本 PR 会改为指向 main 并重新检查。
```

创建后仍应在网页上确认三件事：

- Base branch 是 `feature/shared-api`，Head branch 是 `feature/add-export`。
- Files changed 中没有重复出现同事已完成的 API 实现。
- 审阅者知道这是一条依赖 PR，且知道评审和合并顺序。

在自己分支继续修改、回应评审意见时，正常提交并推送即可，PR 会自动更新：

```bash
git add <文件>
git commit -m "fix: address export review feedback"
git push
```

若刚做过 rebase，则改用前文的 `git push --force-with-lease`。

## 5. 同事分支持续前进时，重复同步而不是重建 PR

在自己开发期间，同事可能继续推送 `feature/shared-api`。此时不要删除自己的分支或新开一条 PR；重复第 2 节即可：

```bash
git switch feature/add-export
git fetch origin --prune
git log --oneline HEAD..origin/feature/shared-api

# 选择一种团队认可的同步方式：
git merge --no-ff origin/feature/shared-api
# 或 git rebase origin/feature/shared-api
```

同步、解决冲突、运行检查后，更新自己的远程分支。PR 的目标分支保持为 `feature/shared-api`，平台会按新基线重新计算差异；仍应人工检查 Files changed，确保范围没有扩大。

> [!warning] 同事强制更新了依赖分支时
> 如果同事对 `feature/shared-api` 做过 rebase 或 force-push，先 `git fetch origin`，再与对方确认旧提交是否仍应保留。不要在未理解变更的情况下立刻 merge。自己的分支通常需要 rebase 到新的远程分支，或在确认后执行一次 merge；冲突很多时，先保留现场并沟通比盲目解决更安全。

## 6. 依赖 PR 合入 main 后，迁移自己的分支和 PR

当 PR-1 已经合入 `main` 后，自己的 PR 才可以切换为面向 `main` 的普通 PR。迁移前先在 PR 页面或合并记录中确认 PR-1 已合入，再确认 `main` 已包含依赖的最终改动；并尽量在同事分支尚未被删除前操作：

```bash
git fetch origin
git log --oneline origin/main..origin/feature/shared-api
git switch feature/add-export
```

若上述提交比较无输出，说明同事分支的原始提交都已进入 `origin/main`。但在 squash merge 或 rebase merge 后，原始提交 ID 可能不再存在于 `main`，此命令仍会有输出；这不表示 PR-1 未合入，应以 PR 的合并状态和 `main` 的实际内容为准。随后使用精确的 `--onto` rebase：它只重放“自己的分支中、但不属于同事分支”的提交到最新 `main` 上。

```bash
git rebase --onto origin/main origin/feature/shared-api
```

其含义可以写成：

```text
取出 feature/add-export 中相对 feature/shared-api 独有的提交，
将它们依次应用在 origin/main 的最新提交之后。
```

这种写法尤其适合依赖分支通过 squash merge 或 rebase merge 进入 `main` 的情况，因为它不会尝试把同事的原始提交一并带进新的分支。处理冲突后，做完整验证并检查新 PR 范围：

```bash
git diff --stat origin/main...HEAD
git diff --check origin/main...HEAD
# 运行项目检查和测试
git push --force-with-lease
```

最后在 PR 网页把 base branch 从 `feature/shared-api` 改为 `main`，或使用 GitHub CLI：

```bash
gh pr edit <PR 编号> --base main
```

重新确认 Files changed、CI、评审意见和 PR 描述。此时 PR 应只显示自己的导出功能；确认无误后，按团队正常流程合入 `main`。

> [!tip] 为什么不直接 `git rebase origin/main`？
> 当依赖分支的提交 ID 因 squash 或 rebase merge 而改变时，普通 rebase 可能难以准确区分“同事的提交”和“自己的提交”。`git rebase --onto origin/main origin/feature/shared-api` 明确给出了旧基点，因此更符合这个依赖迁移场景。前提是本地仍保留 `origin/feature/shared-api` 这一引用；不要在迁移前先把它 prune 掉。

## 7. 只需要同事的个别提交时：使用 cherry-pick

如果自己的功能并不依赖同事整条分支，只需要一个独立、已验证的修复，不要建立堆叠 PR。直接从稳定基线创建自己的分支，再复制需要的提交：

```bash
git fetch origin
git switch main
git pull --ff-only origin main
git switch -c feature/add-export

git show --stat <同事提交 SHA>
git cherry-pick -x <同事提交 SHA>
```

`-x` 会在新提交说明中记录来源，方便以后确认该修复来自哪里。此时自己的分支可直接向 `main` 提 PR：

```bash
git push -u origin feature/add-export
gh pr create --base main --head feature/add-export --fill
```

注意：若同事之后把**同一改动**合入 `main`，要在自己的 PR 合并前检查是否已经重复。若已重复，应通过 rebase、冲突处理或移除重复提交来收敛范围；不要让两个内容相同的实现同时合入。更多 cherry-pick 的选择、冲突和追溯方式见 [[Git 合并方式与 Cherry-pick]]。

## 8. 同事分支来自 fork 或另一个远程仓库时

如果对方分支不在 `origin`，而是在对方的 fork，可为其配置一个只读协作远程。以下以远程名 `alice` 为例：

```bash
git remote add alice <对方仓库 URL>
git fetch alice feature/shared-api
git switch -c feature/add-export alice/feature/shared-api
```

之后同步时将 `origin/feature/shared-api` 替换为 `alice/feature/shared-api`：

```bash
git fetch alice feature/shared-api
git switch feature/add-export
git rebase alice/feature/shared-api
```

自己的分支仍推送到有写权限的 `origin`：

```bash
git push -u origin feature/add-export
```

创建 PR 时，平台是否支持跨 fork 选择目标分支、具体分支写法和权限要求会有所不同。先在网页确认 base repository、base branch 和 head repository；无权直接创建时，请让仓库维护者提供目标分支或协作方式。

## 可直接复用的命令路径

下面是“从同事远程分支开始、开发、同步、先提依赖 PR、再迁移到 `main`”的完整路径。执行前把分支名和检查命令替换为真实值。

```bash
# 1. 从同事的最新远程分支创建自己的分支
git status
git fetch origin --prune
git switch -c feature/add-export origin/feature/shared-api

# 2. 开发并首次推送自己的分支
git add <文件>
git diff --staged
git commit -m "feat: add export endpoint"
# 运行项目测试与检查
git push -u origin feature/add-export

# 3. 同事分支更新后：先 fetch，再在自己的分支同步
git fetch origin --prune
git switch feature/add-export
git log --oneline HEAD..origin/feature/shared-api
git rebase origin/feature/shared-api
# 解决冲突、运行检查后
git push --force-with-lease

# 4. 提交指向同事分支的 PR
gh pr create --base feature/shared-api --head feature/add-export --fill

# 5. 同事的 PR 合入 main 后：把自己的独有提交迁移到 main
git fetch origin
git rebase --onto origin/main origin/feature/shared-api
# 解决冲突、运行检查后
git push --force-with-lease
gh pr edit <PR 编号> --base main
```

若分支已推送且不应改写历史，将第 3 步的 rebase 换成 `git merge --no-ff origin/feature/shared-api` 并使用普通 `git push`。第 5 步为了从依赖 PR 迁移到 `main`，仍建议在确认该分支仅由自己维护后使用精确的 `--onto` rebase。

## 最终检查清单

- [ ] 执行同步前已运行 `git status`，没有把未知改动带入操作。
- [ ] 已使用 `git fetch` 更新远程引用，并确认同事分支的真实名称和最新提交。
- [ ] 当前检出的是自己的分支，而不是同事分支或 `main`。
- [ ] 自己的远程跟踪分支是 `origin/feature/add-export`。
- [ ] 同事分支未合入时，PR 的 base 是同事分支；没有错误指向 `main`。
- [ ] 已用 `git diff <base>...HEAD` 检查 PR 范围，只包含预期的自己的改动。
- [ ] merge/rebase 冲突已理解两边意图，并已运行项目检查和测试。
- [ ] 已推送分支发生 rebase 时，只使用了 `git push --force-with-lease`。
- [ ] 依赖 PR 合入 `main` 后，已执行 `git rebase --onto origin/main <旧依赖分支>`、重新验证，并把 PR base 改为 `main`。

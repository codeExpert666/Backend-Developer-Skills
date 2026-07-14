---
title: Git 合并方式与 Cherry-pick
aliases:
  - Git merge、rebase、squash 与 cherry-pick
tags:
  - Git
  - Git/合并
  - Git/Rebase
  - Git/Cherry-pick
  - Git/冲突解决
created: 2026-07-12T23:39:57
updated: 2026-07-12T23:54:33
---

Git 中的“合并”并不只有一条命令：有的只是移动分支指针，有的会创建一个双亲提交，有的会改写提交历史，还有的只把改动复制到当前分支。选择方式前，先回答三个问题：是否要保留分支拓扑、是否要保留原始提交、以及分支是否已被其他人使用。

本文聚焦如何把一条分支的工作整合到另一条分支；从创建功能分支到提交 PR 的常规流程，可参考 [[Git 分支与 PR 工作流]]。

> [!warning] 先保护未提交的工作
> 合并、rebase 或 cherry-pick 前，先运行 `git status`。工作区不干净时，先提交当前工作，或用 `git stash push -u -m "wip: <说明>"` 暂存。不要为了继续操作而直接丢弃不明来源的改动。

## 先建立共同的例子

下文假设 `main` 是主线，`feature/login` 是功能分支；`A`、`B` 等字母表示提交，`HEAD` 表示当前检出的提交。功能分支从 `B` 切出后新增两个提交，主线同时新增一个提交：

```text
      C---D    feature/login
     /
A---B---E      main (HEAD)
```

在合并前，先更新远程引用并确认自己在哪个分支：

```bash
git status
git fetch origin
git branch --show-current
git log --oneline --graph --decorate --all -12
```

下列命令均以“将 `feature/login` 整合进 `main`”为例。因此，除非另有说明，先切到接收改动的目标分支：

```bash
git switch main
```

## 方式总览

| 方式 | 是否保留原提交 | 是否创建合并提交 | 是否改写已有提交 | 适合什么情况 |
| --- | --- | --- | --- | --- |
| 快进合并（fast-forward） | 是 | 否 | 否 | 目标分支未前进，想要线性历史 |
| 三方合并（merge commit） | 是 | 是 | 否 | 保留分支结构、多人协作分支 |
| `--no-ff` 合并 | 是 | 强制创建 | 否 | 即使可快进也要保留功能分支节点 |
| Squash merge | 否，压为一个新提交 | 否；但会创建普通提交 | 否 | 主线希望整洁、功能分支提交较零碎 |
| Rebase 后整合 / Rebase merge | 是，但提交 ID 会变化 | 通常否 | 是 | 需要线性历史，且分支可安全改写 |
| Cherry-pick | 只复制选定提交的改动 | 否 | 不改写来源，但产生新提交 | 只要少数提交，不想整合整条分支 |

这里的“是否创建合并提交”是指拥有两个或更多父提交的 *merge commit*。Squash 和 cherry-pick 虽会产生新提交，但它们是只有一个父提交的普通提交。

## 1. 快进合并：只移动分支指针

如果目标分支从分支切出后没有新提交，Git 不需要真正合并内容，只需把目标分支指向功能分支的末端。这叫快进合并（fast-forward）。

```text
# 合并前
A---B          main (HEAD)
     \
      C---D    feature/login

# git merge feature/login 后
A---B---C---D  main (HEAD), feature/login
```

示例：

```bash
git switch main
git merge feature/login
```

默认情况下，Git 在可快进时会这样做。它的历史很线性，查看日志简单；代价是以后从提交图中看不出这两个提交曾属于一个功能分支。

如果你的意图就是“只能快进，不能自动产生合并提交”，使用更严格的写法：

```bash
git merge --ff-only feature/login
```

当 `main` 和 `feature/login` 已经分叉时，`--ff-only` 会失败并且不修改历史。这特别适合保护主分支：失败后先决定是采用三方合并，还是让功能分支 rebase 到最新 `main`。

## 2. 三方合并：创建一个合并提交

当两个分支都在共同祖先后前进，Git 会比较共同祖先、当前分支和待合并分支的三个版本，计算出合并结果，并创建一个有两个父提交的合并提交。

```text
# 合并前
      C---D    feature/login
     /
A---B---E      main (HEAD)

# git merge feature/login 后
      C---D
     /     \
A---B---E---M  main (HEAD)
```

```bash
git switch main
git merge feature/login
```

本例中 `M` 的第一个父提交是原来的 `main` 提交 `E`，第二个父提交是 `feature/login` 的末端 `D`。它完整保留了两条分支及每个原始提交，适合希望忠实记录集成时点、或由多人共同维护的分支。

可为合并提交写清楚上下文：

```bash
git merge --no-ff -m "merge: add login flow" feature/login
```

在发生分叉时，`--no-ff` 与默认 `git merge` 都会创建合并提交；它真正的区别体现在下一节的“本可快进”场景。

## 3. `--no-ff`：即使可快进，也保留分支节点

当 `main` 没有在分支切出后前进时，普通 `git merge` 会快进。若希望保留“这个功能作为一组被合入”的边界，可显式要求 Git 生成合并提交：

```bash
git switch main
git merge --no-ff feature/login
```

```text
# 即使 main 可以直接移动到 D，仍保留 M
A---B-------M  main (HEAD)
     \     /
      C---D    feature/login
```

优点是 `git log --graph`、`git revert -m 1 <M>` 能以一个合并节点识别和撤销整项功能；缺点是主线历史会多出合并节点。它常用于“每个功能分支一个合并提交”的团队规范。

> [!tip] 合并前先停下来检查
> `git merge --no-commit --no-ff feature/login` 会先把结果放进暂存区而不提交，便于检查或补充测试。`--no-ff` 很关键：纯快进合并没有可停下的合并提交阶段。确认后执行 `git commit`，放弃则执行 `git merge --abort`。

## 4. 解决合并冲突

同一位置被两边以无法自动兼容的方式修改时，Git 会暂停合并。不要盲目选择“当前版本”或“传入版本”；应先理解两边意图并运行测试。

```bash
git merge feature/login
git status
# 编辑冲突文件：删除 <<<<<<<、=======、>>>>>>> 标记，保留正确代码
git add <已解决的文件>
git diff --check
git commit
```

冲突文件大致长这样：

```text
<<<<<<< HEAD
const timeout = 3_000;
=======
const timeout = 5_000;
>>>>>>> feature/login
```

其中 `HEAD` 一侧是当前目标分支（本例中的 `main`），另一侧是正被合入的 `feature/login`。确认应采用 `3_000`、`5_000` 或一个新的值后，删除全部冲突标记并暂存文件。

如要完整放弃这一次尚未提交的合并：

```bash
git merge --abort
```

只有在合并确实处于进行中时才运行它；用 `git status` 确认状态。已经产生合并提交后，不能用 `--abort` 回退，应依据是否已共享历史选择 `git revert` 或其他经过评估的恢复方案。

### `-X ours` 与 `-X theirs`：只处理冲突块的偏向

对少量、规则明确的冲突，可以在三方合并时指定偏向：

```bash
# 冲突块优先保留当前分支 main 的版本
git merge -X ours feature/login

# 冲突块优先保留待合入分支 feature/login 的版本
git merge -X theirs feature/login
```

`ours` / `theirs` 在这里仅影响发生冲突的块；未冲突的改动仍会照常合入。它不是“选择整条分支”，所以仍要审查结果和运行测试。

> [!warning] `-X theirs` 不等于 `--strategy=theirs`
> Git 没有通用的 `theirs` 合并策略。`-X theirs` 是给默认合并策略的冲突选项。不要把它和下一节 `--strategy=ours` 混淆。

## 5. `--strategy=ours`：保留当前树，但把另一分支标为已合并

`ours` 是一种很特殊的合并策略：生成合并提交并记录双方历史，但合并结果的文件树完全采用当前分支，待合并分支的内容不会进入工作区。

```bash
git switch main
git merge --strategy=ours legacy/old-implementation
```

典型用途是：两条历史需要在图上汇合，避免将来再次尝试合并，但当前分支已经用另一种实现替代了对方的所有内容。它不应用于普通功能合并，因为那会让对方的实际代码全部消失。

不要将它与 `git merge -X ours` 混淆：

| 写法 | 对非冲突改动的处理 | 对冲突改动的处理 | 常见用途 |
| --- | --- | --- | --- |
| `git merge -X ours other` | 合入 | 选当前分支版本 | 某些冲突有明确优先级 |
| `git merge -X theirs other` | 合入 | 选待合入分支版本 | 某些冲突应以传入版本为准 |
| `git merge --strategy=ours other` | 不合入 | 当前树整体保留 | 仅连接历史、废弃另一实现 |

## 6. Squash merge：把整个分支压成一个普通提交

Squash merge 会把待合并分支相对于当前分支的净改动放入工作区和暂存区，但不会创建“已合并”的父子关系。之后由你提交为一个新的普通提交：

```bash
git switch main
git merge --squash feature/login
git status
git diff --staged
git commit -m "feat: add login flow"
```

```text
# 合并前
      C---D    feature/login
     /
A---B---E      main (HEAD)

# squash 后并提交 X
      C---D    feature/login
     /
A---B---E---X  main (HEAD)
```

`X` 的内容大致等于 `C` 和 `D` 的总效果，但 `X` 不是 `D` 的后代。因此 `git log --graph` 看不到合并关系，而 Git 也不会认为 `feature/login` 已被真正合并。若保留该分支并再次 merge，Git 可能再次看到其中的提交或改动；通常应在 squash 合并完成后删除该功能分支。

它适合把“修复提交、格式化提交、评审修正提交”等开发过程压缩为主线上的一个清晰业务变更。若每一个原始提交都具有独立的审计、回滚或发布价值，则用普通合并更合适。

GitHub 的 PR 页面提供 **Squash and merge**；GitLab 也可在合并时启用 squash。两者通常都实现上述效果，但提交信息、署名和具体开关仍要以平台及仓库设置为准。

## 7. Rebase 后整合与 Rebase merge：重放提交，得到线性历史

`git rebase` 本身不是 `git merge`：它把当前分支的提交逐个应用到新基点之后，并为这些提交创建新的提交 ID。常见做法是先在功能分支上 rebase 到最新主线，再回到主线作一次快进合并。

```bash
git switch feature/login
git fetch origin
git rebase origin/main

# 解决冲突后回到目标分支；此时通常可快进
git switch main
git merge --ff-only feature/login
```

```text
# rebase 前
      C---D    feature/login
     /
A---B---E      main

# rebase 后：C、D 被复制为 C'、D'
A---B---E---C'---D'  feature/login

# 快进后
A---B---E---C'---D'  main, feature/login
```

发生冲突时的流程与 merge 类似，但继续和放弃命令不同：

```bash
git status
# 编辑并解决冲突
git add <已解决的文件>
git rebase --continue

# 放弃整次 rebase
git rebase --abort
```

> [!danger] 不要改写他人正在使用的共享分支
> rebase 会重写提交 ID。已经推送、但仅由自己维护的功能分支，完成 rebase 后可用 `git push --force-with-lease` 更新远程；不要对 `main`、release 分支或多人共同维护的分支这样做，除非团队有明确流程和所有相关人员已协调。

GitHub 的 **Rebase and merge** 会把 PR 的提交逐个线性接到目标分支，且不创建合并提交。GitLab 的可选合并策略及其是否先要求 rebase 由项目设置决定；若选择快进或半线性工作流，结果与“开发者先在本地 rebase、再快进合并”在可见历史上相近，但提交 ID、平台检查和权限限制可能不同。

## 8. Octopus merge：一次合并多个分支

Git 可一次合并多个分支：

```bash
git switch integration
git merge feature/api feature/web feature/docs
```

这会尝试建立一个有多个父提交的合并提交，常被称为 octopus merge。它适合彼此独立、基本不会冲突的一组主题分支，例如发布前在集成分支上组合多个小改动。

但它不适合需要人工解决复杂冲突的场景：默认 octopus 策略遇到复杂冲突通常会停止。日常开发中，逐个合并或通过 PR 合并更容易审阅、定位和回滚。

## 9. Cherry-pick：复制指定提交，而不是合并整条分支

Cherry-pick 不是合并方式，但它与合并一样会把别处的改动带到当前分支。差别是它只挑选一个或少数几个提交，并在当前分支创建内容相同或近似的新提交。

```text
# cherry-pick 前
      C---D    feature/login
     /
A---B---E      release/1.2 (HEAD)

# git cherry-pick C 后
      C---D    feature/login
     /
A---B---E---C' release/1.2 (HEAD)
```

`C'` 与 `C` 有相近的改动，但父提交不同，所以提交 ID 必然不同。典型场景是在 release 分支或维护分支上回补一个已经进入 `main` 的小修复。

### 基本用法

先找到需要的提交：

```bash
git log --oneline main
# 例如看到 8f3c2ab fix: reject empty token

git switch release/1.2
git cherry-pick 8f3c2ab
```

一次挑选多个提交时，按希望应用的顺序列出：

```bash
git cherry-pick 8f3c2ab 4d91e70
```

挑选一段连续提交可使用范围，但要注意 `A..B` 不包含 `A`：

```bash
# 应用 (A, B]，即 A 之后直到 B 的提交
git cherry-pick A..B

# 若 A 和 B 都要包含，使用 A^..B
git cherry-pick A^..B
```

提交来自另一个远程分支时，先获取远程引用即可，不必先完整 merge 该分支：

```bash
git fetch origin
git cherry-pick origin/main~2
```

实际工作中建议使用明确的提交 ID，而不是容易因主线前进而改变含义的相对引用。

### 处理 cherry-pick 冲突

```bash
git cherry-pick 8f3c2ab
git status
# 编辑冲突文件并删除冲突标记
git add <已解决的文件>
git cherry-pick --continue
```

如确认不应该应用这个提交：

```bash
git cherry-pick --abort
```

若冲突已自行处理、只是不想保留正在进行的原提交信息，也可使用 `git cherry-pick --quit` 退出操作状态；这不会回滚已经放入工作区或暂存区的修改，因此通常优先使用更明确的 `--abort`。

### 常用选项与安全检查

```bash
# 先应用改动但不立即提交，便于与其他修改一起审阅
git cherry-pick --no-commit 8f3c2ab

# 在新提交信息中追加来源提交，便于追溯
git cherry-pick -x 8f3c2ab

# 只验证该提交会改动什么，不实际写入当前分支
git show --stat 8f3c2ab
git show 8f3c2ab
```

建议对跨分支回补的修复使用 `-x`。它会在提交说明中加入类似“cherry picked from commit ……”的来源记录，之后排查某个修复是否已回补时更可靠。

> [!warning] 不要重复挑选已经存在的改动
> 若某个分支已经通过 merge 或 rebase 包含该修复，再 cherry-pick 同一提交，可能产生空提交、冲突或重复代码。先用 `git log --oneline <目标分支> -- <相关文件>`、`git branch --contains <提交>` 或 `git log <目标分支>.. <来源分支>` 检查；不确定时先在临时分支演练。

## 如何选择

| 你的目标 | 优先选择 | 理由 |
| --- | --- | --- |
| `main` 未前进，只希望简单接收功能分支 | 快进合并或 `--ff-only` | 不产生额外合并节点，且 `--ff-only` 可防止意外分叉 |
| 要保留完整分支拓扑和所有提交 | 三方合并 | 合并提交记录了集成关系 |
| 即使可快进也要标记“功能完成” | `git merge --no-ff` | 每个功能分支都有可见的合并节点 |
| 主线只想保留一个干净的业务提交 | Squash merge | 隐藏分支中的过程性提交 |
| 团队要求线性历史，且功能分支由自己维护 | Rebase 后快进 / Rebase merge | 没有合并节点，但会重写功能分支提交 ID |
| 只需回补一个修复到 release 分支 | `git cherry-pick -x <commit>` | 复制最小范围的改动，并保留来源线索 |
| 只需连接两条历史、不接收另一实现 | `git merge --strategy=ours` | 仅用于特殊迁移或替代场景 |

## 合并完成后的核对与清理

无论选哪种方式，完成后至少检查历史、差异和工作区：

```bash
git status
git log --oneline --graph --decorate -15
git diff --check HEAD~1..HEAD
# 运行项目的测试、静态检查和构建命令
```

功能分支已确认整合且不再需要时，可删除本地和远程分支：

```bash
git branch -d feature/login
git push origin --delete feature/login
git fetch --prune origin
```

`git branch -d` 会拒绝删除 Git 尚未判定为已合并的分支；不要因为方便直接改用 `-D`。尤其在 squash merge 后，Git 不会从提交祖先关系上认定该分支“已合并”，应先确认主线已有所需改动、PR 已完成且分支确实不再使用，再按团队流程删除。

## 一句话记忆

- **merge**：把分支历史与改动整合进来；快进、三方合并和 `--no-ff` 的差别主要在历史图。
- **squash**：只拿整个分支的最终改动，压为主线上的一个新提交，不保留合并关系。
- **rebase**：把自己的提交复制到新基点后，换取线性历史；共享分支要格外谨慎。
- **cherry-pick**：只复制指定提交的改动，适合回补小而明确的修复。

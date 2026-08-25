---
title: Git 常用配置与本地验证
aliases:
  - Git 初始配置
  - Git 全局配置
  - Git 用户名邮箱与默认设置
tags:
  - Git
  - Git/配置
  - Git/提交
  - Git/协作
created: 2026-07-14T22:52:26
updated: 2026-08-25T22:38:17
---

本文在 Git 已安装的前提下，配置个人提交身份、完成第一次本地提交，并按需设置一组保守的日常默认行为。它适用于 Ubuntu 与 macOS；平台差异主要出现在安装和凭据存储，参见 [[Ubuntu 从零安装 Git]]、[[macOS 从零安装 Git]] 与 [[Git 凭据、SSH 与常见问题排查]]。

配置前先确认：个人偏好写在 global，某个仓库或工作身份的例外写在 local，项目的 `.gitattributes`、`CONTRIBUTING.md`、提交钩子和团队规范优先于个人习惯。

## 完成标准

| 类别 | 完成后的状态 | 验证命令 |
| --- | --- | --- |
| 提交身份 | 姓名和邮箱正确，且可区分个人与工作身份 | `git config --show-origin --get-all user.email` |
| 新仓库 | 初始分支名明确，不依赖 Git 版本默认值 | `git config --global --get init.defaultBranch` |
| 本地验证 | 在独立练习仓库完成第一次提交，并确认作者、提交者和当前分支 | `git log -1 --format=fuller`、`git branch --show-current` |
| 工具体验 | 编辑器、颜色与少量别名符合预期 | `git var GIT_EDITOR`、`git st` |
| 配置可追溯 | 能看到每项设置来自哪个文件 | `git config --list --show-origin` |

远程协作配置是按需内容，不是完成本地主线的前提。只有开始连接远程仓库时，才继续本文最后的远程协作默认值。

## 1. 先理解配置范围与优先级

Git 配置不只有一个文件。常见来源如下：

| 范围 | 常见位置 | 适用场景 | 典型命令 |
| --- | --- | --- | --- |
| system | Git 安装目录下的系统配置 | 管理员为整台设备设定的策略 | `git config --system ...` |
| global | `~/.gitconfig` 或 XDG 配置目录 | 个人姓名、常用编辑器与个人别名 | `git config --global ...` |
| local | 当前仓库的 `.git/config` | 某个仓库的工作邮箱、特定远程行为 | `git config --local ...` |
| command | 当前命令参数 | 一次性测试，不写入配置文件 | `git -c key=value <命令>` |

同一个键可能在多处设置。不要只执行 `git config --get <键>` 然后猜测来源；需要排查时使用：

```bash
git config --list --show-origin
git config --show-origin --get-all user.name
git config --show-origin --get-all user.email
```

`--show-origin` 会显示配置来自哪个文件。对受团队管理的设备，不要擅自修改 system 配置；对工作仓库，也不要为了个人便利覆盖团队明确规定的 local 配置。

## 2. 首先设置提交身份

Git 每次创建提交都需要作者姓名和邮箱。它们是提交历史的一部分，不是登录代码托管平台的密码，也不自动决定你对远程仓库是否有权限。

在任意目录执行，并把示例替换为自己的真实提交身份：

```bash
git config --global user.name "你的姓名或团队署名"
git config --global user.email "you@example.com"
```

立即验证：

```bash
git config --global --get user.name
git config --global --get user.email
git config --show-origin --get-all user.name
git config --show-origin --get-all user.email
```

### 个人邮箱、工作邮箱与隐私邮箱

- 个人项目可使用个人邮箱，前提是你愿意让它出现在公开提交历史中。
- 工作仓库通常应使用公司规定的工作邮箱；若只对某个仓库生效，使用 local 配置，而不要覆盖所有个人项目。
- GitHub 等平台可能提供隐私邮箱。是否使用、提交是否会关联到账号，取决于平台与账号设置；应以平台当前说明为准。

在某个已 clone 的工作仓库中设置例外：

```bash
cd /path/to/work-repository
git config --local user.name "工作署名"
git config --local user.email "you@company.example"
git config --show-origin --get-all user.email
```

`--local` 必须在 Git 仓库内执行。它只改变当前仓库的 `.git/config`，不会修改其他项目。

> [!warning] 姓名与邮箱不等于远程认证
> 即使提交已经显示正确作者，`git push` 仍可能因 SSH 密钥、HTTPS 令牌或仓库权限不足而失败。认证问题请使用 [[Git 凭据、SSH 与常见问题排查]]，不要反复修改 `user.email`。

## 3. 设置新仓库的默认初始分支

为避免不同 Git 版本在新仓库中产生不同默认分支名，可以明确设置个人默认值：

```bash
git config --global init.defaultBranch main
git config --global --get init.defaultBranch
```

该设置只影响以后执行 `git init` 创建的新仓库，不会重命名已有仓库的 `master`、`main` 或其他分支，也不会改变远程仓库的默认分支。若团队规定新项目使用别的名称，应以团队约定为准。

## 4. 选择 Git 编辑器

Git 在编辑提交正文、合并说明或其他交互式操作时会调用编辑器。没有设置时，Git 会选择环境中的默认编辑器；初学者常因意外进入 Vim 而以为 Git 卡住了。

若只想使用终端中的简单编辑器，可设置为 `nano`：

```bash
git config --global core.editor "nano"
git var GIT_EDITOR
```

若你已安装并希望使用 Visual Studio Code，可使用：

```bash
git config --global core.editor "code --wait"
git var GIT_EDITOR
```

`--wait` 很重要：它让 Git 等待编辑器窗口关闭后再继续。不要设置一个当前终端找不到的命令；先运行 `command -v code` 或 `command -v nano` 验证。团队项目若通过环境变量或脚本指定编辑器，应优先使用项目约定。

## 5. 用独立练习仓库完成第一次本地提交

前四节主要是在写入和检查配置。现在用一个全新的本地仓库确认这些配置确实会作用于提交，同时建立后续学习需要的最小模型：

| 对象 | 作用 | 本节如何观察 |
| --- | --- | --- |
| 工作区 | 保存当前正在编辑的文件 | 创建并修改 `README.md` |
| 暂存区 | 选择下一次提交要记录的内容 | `git add README.md` |
| 提交 | 保存一次带作者、提交者和说明的版本快照 | `git commit`、`git log` |
| 当前分支 | 指向当前工作所在的提交历史 | `git branch --show-current` |

下列命令只用于个人练习目录，不要在已有项目中执行。先只读检查目标目录是否已经存在：

```bash
PRACTICE_REPO="$HOME/git-learning/git-config-check"
if test -e "$PRACTICE_REPO"; then
  printf '目标目录已经存在，请更换路径：%s\n' "$PRACTICE_REPO"
else
  printf '目标目录尚不存在，可以继续：%s\n' "$PRACTICE_REPO"
fi
```

只有看到“目标目录尚不存在”时才继续。若目录已经存在，应修改 `PRACTICE_REPO`，而不是覆盖或清理原目录：

```bash
PRACTICE_REPO="$HOME/git-learning/git-config-check"
mkdir -p "$PRACTICE_REPO"
cd "$PRACTICE_REPO"
git init
git branch --show-current
printf '# Git configuration check\n' > README.md
git status
git add README.md
git diff --staged
git commit -m "docs: verify initial Git configuration"
git status --short --branch
git log -1 --format=fuller
```

`git init` 只创建本地仓库，不会连接 GitHub、GitLab 或其他远程平台。提交成功后，`git status` 应显示工作区干净；`git log -1 --format=fuller` 中的 Author 与 Committer 应符合前面设置的身份；当前分支名应符合 `init.defaultBranch`。

如果提交因身份缺失而失败，回到“首先设置提交身份”检查 `user.name` 和 `user.email` 的值与来源。若分支名不符合预期，确认这个仓库是否确实是在设置 `init.defaultBranch` 之后新建的；该配置不会重命名已有仓库。

练习仓库可以保留给后续 Git 操作使用。本文不提供自动删除命令；如果以后决定清理，应先再次确认实际路径和其中的文件。

## 6. 查看、修改与撤销一项配置

查看单个值及其来源：

```bash
git config --show-origin --get-all user.email
git config --show-origin --get-all core.editor
```

删除自己明确不再需要的 global 值：

```bash
git config --global --unset-all core.editor
```

需要人工检查全局配置时：

```bash
git config --global --edit
```

编辑前建议先备份文件，例如 `cp ~/.gitconfig ~/.gitconfig.backup`。不要用文本编辑器删除不了解的 system 或公司管理配置；也不要把包含令牌、密码或私钥内容的配置复制到聊天、Issue 或公开仓库。

## 7. 换行符：先服从项目规则，再谈全局设置

Ubuntu 和 macOS 的日常文本文件通常使用 LF 换行。真正决定仓库中文本、二进制文件和换行符策略的最佳位置是版本受控的 `.gitattributes`，因为它能随仓库被所有协作者获取。

先在仓库根目录检查项目是否已有规则：

```bash
git ls-files -- .gitattributes
git check-attr -a -- README.md
```

如果项目已经有 `.gitattributes`，不要为了解决一次差异而随意设置全局 `core.autocrlf`。先阅读项目规则，并与团队确认。

对个人新仓库、明确希望“提交时把 CRLF 规范为 LF，检出时不改为 CRLF”的 Unix 用户，可按需设置：

```bash
git config --global core.autocrlf input
git config --global core.safecrlf warn
```

`core.autocrlf=input` 不会在检出时把文件改成 CRLF；`core.safecrlf=warn` 会在可能出现不可逆换行转换时提示。它们并不适合每个仓库，更不能修复已经混乱的历史。已有仓库突然出现大量整文件差异时，先停止批量格式化，检查 `.gitattributes`、编辑器设置和实际换行符，再做最小范围修复。

## 8. 改善终端输出并添加少量别名

### 在交互式终端中显示颜色

`color.ui=auto` 会在支持颜色的交互式终端中显示更易读的输出，对重定向到文件等非交互场景通常不会强行插入颜色控制符。

```bash
git config --global color.ui auto
git config --global --get color.ui
```

多数新版 Git 已有合理的颜色默认值；显式设置它主要是记录个人意图，不影响提交历史或远程协作策略。

### 添加少量可解释的别名

别名只是缩短命令，不会增加 Git 能力。建议从语义清楚、没有副作用的别名开始：

```bash
git config --global alias.st status
git config --global alias.sw switch
git config --global alias.br branch
git config --global alias.last "log -1 --stat"
git config --global alias.graph "log --oneline --graph --decorate --all"
```

验证：

```bash
git config --global --get-regexp '^alias\.'
# 在任意 Git 仓库中执行：
git st
git graph
```

不要把 `reset --hard`、`push --force`、`clean -fd` 等破坏性操作包装成短别名。命令越短越容易在错误的仓库执行；这类动作应保持显式，并在执行前先检查 `git status`。

## 9. 按需设置保守的远程协作默认值

本节不是本地配置主线的前提。只有在完成前面的本地提交练习，并准备连接远程仓库时再继续。远程地址、网络连接、HTTPS 或 SSH 身份验证的边界见 [[Git 凭据、SSH 与常见问题排查]]；日常分支操作见 [[Git 分支与 PR 工作流]]。

### 先建立最小远程模型

远程仓库是位于代码托管平台或另一台服务器上的独立仓库，`origin` 只是最常见的远程名称。`git fetch` 会获取远程的新提交和分支状态，并把它们记录为 `origin/main` 一类远程跟踪引用；它不会直接把这些提交写入当前分支。

在日常远程协作中，上游分支（upstream）通常是当前本地分支默认拉取和比较的远程跟踪分支，例如本地 `main` 对应的 `origin/main`。`git pull` 会先执行 fetch，再把上游分支整合进当前分支。整合前只需先区分两种情况：

| 历史关系 | 含义 | 保守处理 |
| --- | --- | --- |
| 当前分支没有独有提交，上游单方面领先 | 可以快进，只需移动当前分支指针 | 允许 pull 完成 |
| 当前分支与上游都有独有提交 | 历史已经分叉，需要选择 merge 或 rebase | 先停止，再按团队规则判断 |

快进（fast-forward）不会创建新提交；它只是把当前分支指针移动到已经存在的较新提交。merge 与 rebase 如何改变历史、发生冲突后如何继续或中止，统一见 [[Git 合并方式与 Cherry-pick]]。

### 获取时清理过期的远程跟踪引用

设置 `fetch.prune=true` 后，每次 `git fetch` 都会清理远程已经删除的远程跟踪引用：

```bash
git config --global fetch.prune true
git config --global --get fetch.prune
```

这里清理的是 `origin/feature/...` 一类本地保存的远程状态，不是本地分支，也不会删除工作区中的未提交改动。它不能替代删除本地分支前的人工确认。

### 分叉时停止自动整合

设置 `pull.ff=only` 后，普通的 `git pull` 只接受可以快进的结果；如果当前分支与上游已经分叉，pull 会停止，不替你选择 merge 或 rebase：

```bash
git config --global pull.ff only
git config --global --get pull.ff
```

本文不全局设置 `pull.rebase`。是否采用 merge 或 rebase 会影响提交图和共享历史，应在理解 [[Git 合并方式与 Cherry-pick]] 并确认团队规则后再决定。

### 验证配置来源与失败边界

检查最终值及其来源：

```bash
git config --show-origin --get-all fetch.prune
git config --show-origin --get-all pull.ff
git config --show-origin --get-all pull.rebase
```

第三条命令没有输出，通常表示尚未设置通用的 rebase 默认值；如果有输出，应先根据来源文件判断它来自 global、local 还是其他范围，不要直接覆盖不了解的团队配置。

这些命令只能证明配置值和来源，不能证明某个远程仓库已经可访问。真正的 fetch、pull 和分叉处理应在已经配置远程地址与认证的练习仓库中验证，并按 [[Git 分支与 PR 工作流]] 执行。

`pull.ff=only` 因无法快进而失败时，当前分支的提交历史与工作区不会被这次整合改写，但前面的 fetch 阶段可能已经更新远程跟踪引用。先运行 `git status`，再按 [[Git 分支与 PR 工作流]] 中的“`git pull --ff-only` 失败怎么办？”检查差异；不要为了消除报错而直接强制合并、变基或丢弃提交。

## 下一步

完成本文的本地主线后，如果需要连接 GitHub、GitLab 或其他远程平台，先阅读 [[Git 凭据、SSH 与常见问题排查]]。确认远程访问可用后，再进入 [[Git 分支与 PR 工作流]] 学习从 `main` 创建功能分支、提交并发起 PR；提交标题和正文规范见 [[Git 提交消息编写规范]]；需要选择和恢复合并方式时阅读 [[Git 合并方式与 Cherry-pick]]。

## 官方参考资料

- [Git：首次配置](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup)
- [Git：git-config 手册](https://git-scm.com/docs/git-config)
- [Git：gitattributes 手册](https://git-scm.com/docs/gitattributes)
- [Git：git-pull 手册](https://git-scm.com/docs/git-pull)

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
updated: 2026-07-14T22:52:26
---

本文在 Git 已安装的前提下，配置个人提交身份与一组保守、易验证的日常默认行为。它适用于 Ubuntu 与 macOS；平台差异主要出现在安装和凭据存储，参见 [[Ubuntu 从零安装 Git]]、[[macOS 从零安装 Git]] 与 [[Git 凭据、SSH 与常见问题排查]]。

配置前先确认：个人偏好写在 global，某个仓库或工作身份的例外写在 local，项目的 `.gitattributes`、`CONTRIBUTING.md`、提交钩子和团队规范优先于个人习惯。

## 完成标准

| 类别 | 完成后的状态 | 验证命令 |
| --- | --- | --- |
| 提交身份 | 姓名和邮箱正确，且可区分个人与工作身份 | `git config --show-origin --get-all user.email` |
| 新仓库 | 初始分支名明确，不依赖 Git 版本默认值 | `git config --global --get init.defaultBranch` |
| 拉取远程 | 远程已删除的跟踪分支会被清理；发生分叉时不会悄悄创建合并提交 | `git config --global --get fetch.prune`、`git config --global --get pull.ff` |
| 工具体验 | 编辑器、颜色与少量别名符合预期 | `git var GIT_EDITOR`、`git st` |
| 配置可追溯 | 能看到每项设置来自哪个文件 | `git config --list --show-origin` |

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
git config --show-origin --get-all user.email
git config --show-origin --get-all pull.ff
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

Git 在编辑合并说明、提交正文或 rebase 计划时会调用编辑器。没有设置时，Git 会选择环境中的默认编辑器；初学者常因意外进入 Vim 而以为 Git 卡住了。

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

## 5. 设置保守的日常协作默认值

以下设置不替代团队工作流，但能让意外情况更早显现：

```bash
git config --global fetch.prune true
git config --global pull.ff only
git config --global color.ui auto
```

| 配置 | 作用 | 为什么是保守选择 | 需要注意什么 |
| --- | --- | --- | --- |
| `fetch.prune=true` | `git fetch` 时清理远程已经删除的远程跟踪分支 | 减少 `origin/feature/...` 过期引用造成的误判 | 不会删除你的本地分支或未提交改动 |
| `pull.ff=only` | `git pull` 只允许快进更新 | 本地与远程分叉时停止并要求你主动决定 merge 或 rebase | 若团队默认 rebase，应按团队规则配置相应分支或 local 设置 |
| `color.ui=auto` | 在支持颜色的终端中显示更易读的输出 | 对非交互重定向通常不会强行插入颜色控制符 | 多数新版 Git 已有合理默认值，设置它主要是显式化意图 |

`pull.ff=only` 失败并不是“Git 坏了”，而是提醒本地与远程出现了各自独有的提交。此时先用 `git status`、`git log --oneline --left-right HEAD...@{upstream}` 理解差异，再按 [[Git 分支与 PR 工作流]] 或团队约定选择后续动作。

不要在不了解团队历史策略时，直接全局设置 `pull.rebase=true`。它会改变 `git pull` 的整合方式，尤其不适合刚接触 Git、仍不清楚哪些提交已推送并被他人依赖的场景。

## 6. 换行符：先服从项目规则，再谈全局设置

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

## 7. 添加少量可解释的别名

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

## 8. 用独立练习仓库验证配置

不要在重要项目第一次试验身份、编辑器或换行符设置。可以在个人练习目录创建一个全新的仓库：

```bash
mkdir -p "$HOME/git-learning/git-config-check"
cd "$HOME/git-learning/git-config-check"
git init
git branch --show-current
printf '# Git configuration check\n' > README.md
git add README.md
git commit -m "docs: verify initial Git configuration"
git log -1 --format=fuller
```

检查提交里的 Author 与 Commit 是否符合预期。若 `git commit` 打开了不熟悉的编辑器，可以退出而不保存，再回到“选择 Git 编辑器”部分调整 `core.editor`。确认练习成功后，才在真实仓库中开始工作。

## 9. 查看、修改与撤销一项配置

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

## 下一步

完成本文后，可进入 [[Git 分支与 PR 工作流]] 学习从 `main` 创建功能分支、提交并发起 PR；提交标题和正文规范见 [[Git 提交消息编写规范]]。如果已经需要连接远程仓库，请继续 [[Git 凭据、SSH 与常见问题排查]]。

## 官方参考资料

- [Git：首次配置](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup)
- [Git：git-config 手册](https://git-scm.com/docs/git-config)
- [Git：gitattributes 手册](https://git-scm.com/docs/gitattributes)
- [Git：git-pull 手册](https://git-scm.com/docs/git-pull)

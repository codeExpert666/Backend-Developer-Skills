---
title: Git 安装与初始配置概览
aliases:
  - Git 安装入口
  - Linux 与 macOS Git 安装指南
  - Git 首次配置指南
tags:
  - Git
  - Git/安装
  - Git/配置
created: 2026-07-14T22:52:26
updated: 2026-07-14T22:52:26
---

本文是 Linux（以 Ubuntu 为例）与 macOS 上从零使用 Git 的入口。目标不是只让 \`git --version\` 输出一个版本号，而是建立一套可确认来源、可解释配置、可安全连接远程仓库的个人 Git 环境。

Git 本身可以完全离线地创建仓库、提交和查看历史；连接 GitHub、GitLab 或公司代码平台是后续的独立步骤。因此，请先完成安装和本地身份配置，再选择 HTTPS 或 SSH 访问远程仓库。

## 推荐阅读顺序

先按操作系统完成安装，再完成所有平台共通的配置。只有在需要 clone、fetch、pull 或 push 私有仓库时，才继续阅读凭据与 SSH 部分。

| 你使用的环境或目标 | 从哪里开始 | 完成后继续阅读 |
| --- | --- | --- |
| Ubuntu 桌面、个人虚拟机或开发机 | [[Ubuntu 从零安装 Git]] | [[Git 常用配置与本地验证]] |
| macOS 自带 Terminal 或其他终端 | [[macOS 从零安装 Git]] | [[Git 常用配置与本地验证]] |
| 已能运行 Git，只是不确定如何设置姓名、邮箱和默认行为 | [[Git 常用配置与本地验证]] | [[Git 分支与 PR 工作流]] |
| 需要访问 GitHub 或其他远程代码平台 | [[Git 凭据、SSH 与常见问题排查]] | [[Git 分支与 PR 工作流]] |

> [!important] 不要把 Git、GitHub 和 SSH 混为一件事
> Git 是在本机运行的版本控制工具；GitHub、GitLab 等是托管远程仓库的平台；SSH 或 HTTPS 是访问远程仓库的传输与认证方式。你可以在没有 GitHub 账号、没有网络、没有 SSH 密钥的情况下学习并使用本地 Git。

## 完成后的最低标准

完成本组笔记后，应能独立回答并验证下面的问题。

| 项目 | 应达到的状态 | 典型验证命令 |
| --- | --- | --- |
| Git 可用性 | 当前终端可以调用预期的 Git 可执行文件 | \`command -v git\`、\`git --version\` |
| 本机身份 | 新提交会带上正确的姓名和邮箱 | \`git config --global --get user.name\`、\`git config --global --get user.email\` |
| 配置边界 | 知道个人设置与项目设置分别写在哪里 | \`git config --list --show-origin\` |
| 新仓库默认值 | 新建仓库使用团队约定的初始分支名 | \`git config --global --get init.defaultBranch\` |
| 远程访问（按需） | 能确认远程 URL，并通过所选协议访问 | \`git remote -v\`、\`git ls-remote origin HEAD\` |

## 先理解：安装、配置与认证分别解决什么

| 阶段 | 它做什么 | 常见命令或文件 | 不解决什么 |
| --- | --- | --- | --- |
| 安装 Git | 将 Git 命令安装到系统并纳入 PATH | \`apt install git\`、Xcode Command Line Tools、Homebrew | 不会设置提交姓名或远程权限 |
| Git 配置 | 定义个人、仓库或一次命令的默认行为 | \`~/.gitconfig\`、\`.git/config\`、\`git config\` | 不会替你创建平台账号或授予仓库权限 |
| 远程认证 | 证明你有权访问某个远程仓库 | SSH 密钥、访问令牌、系统凭据钥匙串 | 不会改变提交中的作者姓名和邮箱 |

如果终端提示“请告诉我你是谁（Please tell me who you are）”，说明 Git 已安装但身份未配置；如果 \`git push\` 提示认证失败，通常说明远程凭据或仓库权限有问题，而非 Git 未安装。排错时先分清属于哪一层，避免反复重装。

## Git 配置的作用域

Git 会从多个位置读取配置。通常将长期、个人偏好放在 global，将某一仓库的例外放在 local；不要为了临时试验而修改系统级配置。

| 作用域 | 常见位置 | 适合放什么 | 设置方式 |
| --- | --- | --- | --- |
| system | 系统安装目录下的 \`gitconfig\` | 设备管理员统一制定的策略 | \`git config --system ...\` |
| global | \`~/.gitconfig\` 或 XDG 配置目录 | 个人姓名、邮箱、编辑器、常用别名 | \`git config --global ...\` |
| local | 当前仓库的 \`.git/config\` | 工作邮箱、仓库专用工具或团队例外 | \`git config --local ...\` |
| command | 本次命令的 \`-c\` 参数 | 一次性试验或自动化脚本 | \`git -c key=value <命令>\` |

当同一个键在多处出现时，范围更具体的设置可能覆盖较宽范围的设置。不要靠猜测判断最终值；使用 \`git config --show-origin --get-all <键名>\` 查看值和来源。配置作用域的完整说明见 [Git 官方 git-config 手册](https://git-scm.com/docs/git-config)。

> [!warning] 不要用 \`sudo git ...\` 解决权限或认证问题
> \`sudo\` 会改用 root 的 HOME、Git 配置和 SSH 密钥，容易产生 root 所有的文件，也会让“我明明设置了密钥却无法推送”变得难以定位。仓库内文件所有权异常时，应先修复目录权限或询问设备管理员，而不是把日常 Git 操作升级为 root。

## 安装路径怎么选

### Ubuntu：优先使用系统包管理器

对个人 Ubuntu 开发机，优先通过 APT 安装和更新 Git。它与系统更新策略一致，适合绝大多数日常开发。需要某个新特性时，先确认项目实际要求的最低 Git 版本；不要为了追求“最新”随意加入未知 PPA 或从不明脚本安装。

详细步骤见 [[Ubuntu 从零安装 Git]]。

### macOS：在 Command Line Tools 与 Homebrew 之间选择

macOS 常见的两条可靠路径如下：

- 只需要基础开发工具，且希望跟随系统工具链：使用 Xcode Command Line Tools。
- 已使用 Homebrew 管理开发工具，或项目明确要求较新的 Git：使用 Homebrew 安装 Git，并确认 Homebrew 的路径优先级。

两者可以共存，但同一终端实际只会执行 PATH 中靠前的那个 \`git\`。安装后必须运行 \`type -a git\` 和 \`git --version\`，而不是假设新装的版本自动生效。详见 [[macOS 从零安装 Git]]。

## 本组笔记不会做的事

- 不会要求你在全局开启危险的强制推送、自动 rebase 或自动清理命令。
- 不会把访问令牌写入 shell 配置、仓库文件、提交消息或 Markdown 笔记。
- 不会建议使用 \`credential.helper store\` 保存长期令牌；该方式会将凭据以明文写入磁盘。
- 不会覆盖项目已经提供的 \`.gitattributes\`、\`CONTRIBUTING.md\`、提交钩子或团队 Git 规范。

完成基础配置后，日常创建分支、提交和发起 PR 的流程请阅读 [[Git 分支与 PR 工作流]]；提交说明的写法见 [[Git 提交消息编写规范]]；需要整合分支时再阅读 [[Git 合并方式与 Cherry-pick]]。

## 官方参考资料

- [Git：安装 Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Git：首次配置](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup)
- [Git：git-config 手册](https://git-scm.com/docs/git-config)
- [Git：凭据帮助程序](https://git-scm.com/doc/credential-helpers)

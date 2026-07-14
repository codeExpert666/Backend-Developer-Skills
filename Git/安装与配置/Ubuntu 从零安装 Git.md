---
title: Ubuntu 从零安装 Git
aliases:
  - Ubuntu 安装 Git
  - Linux 安装 Git
  - Ubuntu Git 配置前准备
tags:
  - Git
  - Git/安装
  - Git/Linux
  - Git/Ubuntu
created: 2026-07-14T22:52:26
updated: 2026-07-14T22:52:26
---

本文面向使用 Ubuntu 的个人开发者，从尚未安装 Git 或不确定当前 Git 来源的状态开始，使用系统包管理器完成安装、验证和维护。生产服务器、共享跳板机或受组织管理的设备，可能限制软件安装、代理配置或默认 Shell；应先遵守团队或管理员要求。

先阅读 [[Git 安装与初始配置概览]] 了解 Git 安装、配置和认证的边界。安装完成后请继续 [[Git 常用配置与本地验证]]；需要 clone 或 push 私有仓库时，再阅读 [[Git 凭据、SSH 与常见问题排查]]。

## 目标与完成标准

完成后，当前用户能运行 Git，且能解释终端实际执行的是哪个 Git。此时还没有设置提交身份，也不代表已经可以访问远程私有仓库。

| 阶段 | 目标 | 验证方式 |
| --- | --- | --- |
| 环境检查 | 确认是 Ubuntu，并记录当前 Git 状态 | \`cat /etc/os-release\`、\`command -v git\` |
| 安装 | 由 APT 安装 Git 包 | \`sudo apt install -y git\` |
| 版本与路径验证 | 确认当前 Shell 能找到 Git | \`type -a git\`、\`git --version\` |
| 后续配置 | 使用自己的身份创建提交 | [[Git 常用配置与本地验证]] |

## 1. 先检查系统与现有状态

在终端执行以下只读命令：

\`\`\`bash
cat /etc/os-release
command -v apt
command -v git || true
git --version 2>/dev/null || true
type -a git 2>/dev/null || true
\`\`\`

\`command -v git\` 只显示当前 Shell 优先使用的 Git 路径；\`type -a git\` 会列出 PATH 中能找到的所有同名命令。若系统已经有 Git，不要马上重复安装。先记录当前版本和路径，安装后再比较是否发生变化。

如果 \`apt\` 不存在，说明当前系统不是标准 Ubuntu/Debian 系列环境，或运行在精简容器中。不要将本文的 APT 命令直接套用到 Fedora、Alpine、Arch 或 macOS；应使用对应发行版或平台的包管理方式。

## 2. 使用 APT 安装 Git

先刷新软件包索引，再安装 Git：

\`\`\`bash
sudo apt update
sudo apt install -y git
\`\`\`

\`sudo apt update\` 只更新本机的软件包索引；\`sudo apt install -y git\` 安装 Git 及其依赖。它不会修改你的 Git 提交身份、全局配置或已有仓库内容。

Git 官方安装说明指出，Debian 系发行版（包括 Ubuntu）可以通过 APT 安装 Git；日常开发优先选这种随系统维护的安装方式，而不是从未知脚本或不明 PPA 安装。若公司镜像源、代理或权限策略阻止安装，应保留报错内容并向管理员确认软件源策略。

> [!warning] 不要把 \`apt upgrade\` 当作安装 Git 的必要步骤
> \`apt upgrade\` 可能更新系统中许多无关软件。仅为安装 Git 时，先执行 \`apt update\` 再安装 \`git\` 即可；系统级升级应按设备维护策略单独进行。

## 3. 验证 Git 是否真的可用

安装完成后执行：

\`\`\`bash
command -v git
type -a git
git --version
git --exec-path
git help -a | sed -n '1,40p'
\`\`\`

通常 APT 安装的 Git 位于 \`/usr/bin/git\`。实际路径以命令输出为准；不要因为某些教程出现 \`/bin/git\` 就手工指定那个路径。

\`git --version\` 证明命令能运行，\`git --exec-path\` 显示 Git 子命令所在目录，最后一条命令可大致确认常用子命令已被 Git 识别。版本号不必追逐“全网最新”：关键是满足团队工具、服务器和项目脚本的最低版本要求。

### 想确认软件包版本和来源

APT 可以显示候选版本与当前安装来源：

\`\`\`bash
apt-cache policy git
apt-cache show git | sed -n '1,80p'
\`\`\`

当项目要求某个 Git 特性时，先对照 \`git --version\` 与该特性的官方文档。若 Ubuntu 当前受支持版本的软件源仍无法提供所需版本，应先和团队确认统一方案；不要擅自在每台开发机加入第三方软件源，否则环境难以复现和维护。

## 4. 后续更新与卸载

日常更新只更新 Git 包本身：

\`\`\`bash
sudo apt update
sudo apt install --only-upgrade git
git --version
\`\`\`

卸载 Git 前，先确认目的：卸载不会删除你的仓库目录和 \`~/.gitconfig\`，但会让依赖 Git 的 IDE、脚本与终端命令不可用。确实不再需要时可执行：

\`\`\`bash
sudo apt remove git
\`\`\`

随后可以用 \`command -v git\` 确认命令是否不再存在。不要为了“重置 Git”直接删除项目里的 \`.git\` 目录；那会删除该仓库的本地历史和配置，与卸载程序无关。

## 5. 常见安装问题

### \`sudo\` 提示当前用户不在 sudoers 中

这不是 Git 的报错，而是当前账户没有管理权限。不要借用他人密码、复制 root 环境的 Git 配置，或用来源不明的脚本绕过限制；请管理员安装 Git，或询问是否允许使用用户级开发环境。

### \`apt update\` 或安装下载失败

先区分 DNS、网络、代理、软件源和权限问题。可从以下只读信息开始排查：

\`\`\`bash
apt-cache policy git
grep -R "^deb" /etc/apt/sources.list /etc/apt/sources.list.d 2>/dev/null || true
env | grep -iE '^(http|https|no)_proxy=' || true
\`\`\`

如果设备处于公司网络，代理和镜像源通常由组织统一管理。不要为了临时绕过下载失败而永久写入未知代理地址，也不要关闭 HTTPS 证书验证。

### 已安装 Git，但终端仍显示意外版本

执行 \`type -a git\` 检查是否还有手工安装、SDK 或其他包管理器提供的 Git 位于 PATH 更靠前的位置。不要直接删除 \`/usr/bin/git\` 或系统目录下的文件；先明确想让哪一个版本生效，再调整对应工具的安装或 PATH 设置。

## 6. 现在继续完成身份与默认行为配置

Git 安装成功后，最重要的下一步是设置提交姓名和邮箱。进入 [[Git 常用配置与本地验证]]，其中会解释 global 与 local 的区别，并给出可复核的配置命令。完成配置后，再按照 [[Git 分支与 PR 工作流]] 创建、提交并推送第一个功能分支。

## 官方参考资料

- [Git：在 Linux 上安装 Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Ubuntu：APT 命令参考](https://manpages.ubuntu.com/manpages/noble/en/man8/apt.8.html)
- [Git：git-config 手册](https://git-scm.com/docs/git-config)

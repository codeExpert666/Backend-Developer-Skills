---
title: Ubuntu 从零安装 Oh My Zsh
aliases:
  - Ubuntu 安装 Oh My Zsh
  - Ubuntu 安装 Zsh 与 Oh My Zsh
  - Linux 从零配置 Oh My Zsh
tags:
  - Terminal
  - Terminal/安装
  - Terminal/Linux
  - Terminal/Ubuntu
  - Zsh
  - Oh-My-Zsh
created: 2026-07-14T00:04:29
updated: 2026-07-14T00:04:29
---

本文面向使用 Ubuntu 默认终端的个人开发者，从默认 Bash 环境开始安装 Zsh 与 Oh My Zsh，并完成最小验证。它适用于受支持 Ubuntu 版本上的本地开发机或个人虚拟机；生产服务器、共享跳板机和受组织管理的主机应先确认软件安装、网络和 Shell 变更策略。

先阅读 [[Oh My Zsh 安装概览]] 了解文件和配置边界。完成本文后，继续 [[Oh My Zsh 常用优化配置]]；需要更新、排错或卸载时，参见 [[Oh My Zsh 更新、备份与卸载]]。

## 目标与完成标准

完成后，新开的 Ubuntu 终端应自动进入 Zsh，Oh My Zsh 位于 `~/.oh-my-zsh`，且你能安全修改 `~/.zshrc`。

| 阶段 | 结果 | 关键验证 |
| --- | --- | --- |
| 安装 Zsh | 系统能找到 Zsh，且它是允许的登录 Shell | `command -v zsh`、`/etc/shells` |
| 切换登录 Shell | 新终端默认启动 Zsh | `echo "$SHELL"`、`ps -p $$ -o comm=` |
| 安装 Oh My Zsh | 框架与模板配置已生成 | `echo "$ZSH"`、`omz version` |
| 最小配置 | 主题和少量插件可控 | `zsh -n ~/.zshrc`、新开终端 |

> [!warning] 不要在远程关键会话中盲改登录 Shell
> 如果你正通过 SSH 连接生产机器或唯一的跳板机，先保留一个已登录的备用会话，再测试新开会话。切换 Shell 本身通常是可逆的，但错误的启动文件可能让新会话难以使用。

## 1. 检查当前终端和系统状态

Ubuntu 的 GNOME Terminal 只是终端模拟器，实际启动什么 Shell 由用户登录 Shell 和终端配置共同决定。先执行：

~~~bash
printf 'configured login shell: %s\n' "$SHELL"
ps -p $$ -o pid=,comm=
command -v zsh || true
zsh --version 2>/dev/null || true
cat /etc/os-release
~~~

大多数全新 Ubuntu 用户会看到 `$SHELL` 为 `/bin/bash`，且当前进程也是 `bash`。如果系统已经有 Zsh，不要跳过后续的 `/etc/shells` 和登录 Shell 检查。

## 2. 通过 apt 安装 Zsh、Git 与 curl

先刷新软件包索引，再安装 Zsh、Git 与 curl：

~~~bash
sudo apt update
sudo apt install -y zsh git curl
~~~

安装后确认二进制位置和版本：

~~~bash
command -v zsh
zsh --version
git --version
curl --version
~~~

通常 `command -v zsh` 会输出 `/usr/bin/zsh`。不要把该路径手工猜成 `/bin/zsh`；有些系统会把两个路径做成链接，而 `chsh` 应使用本机实际可执行文件的路径。

## 3. 将 Zsh 设为自己的登录 Shell

先确认 Zsh 已登记为允许的登录 Shell：

~~~bash
zsh_path="$(command -v zsh)"
grep -Fx "$zsh_path" /etc/shells
~~~

若最后一条命令打印出 Zsh 路径，执行：

~~~bash
chsh -s "$zsh_path"
~~~

这条命令只修改当前用户的登录 Shell，通常会要求输入当前用户密码；不要为此使用 `sudo chsh`。随后完全关闭当前终端窗口并新开一个，或者注销并重新登录桌面会话。重新检查：

~~~bash
printf 'configured login shell: %s\n' "$SHELL"
ps -p $$ -o comm=
~~~

预期应看到 `/usr/bin/zsh`（或系统对应路径）与 `zsh`。如果 `chsh` 报“not listed in /etc/shells”，不要强行编辑该文件；先确认 Zsh 是否由系统包管理器正确安装，并让管理员按系统策略处理。

> [!tip] 只想临时试用 Zsh
> 运行 `zsh` 可以在当前窗口启动一个子 Shell，不会修改默认登录 Shell。确认体验后再执行 `chsh`；退出子 Shell 可用 `exit`。

## 4. 备份现有配置

即使此前主要使用 Bash，也可能已有 `~/.zshrc`、SDK 配置或团队发放的 Shell 配置。先做带时间戳的副本：

~~~bash
if [ -f "$HOME/.zshrc" ]; then
  cp "$HOME/.zshrc" "$HOME/.zshrc.before-oh-my-zsh.$(date +%Y%m%d-%H%M%S)"
fi
~~~

如果你有 `~/.bashrc` 中的 PATH、代理、语言版本管理器或别名，不要整段复制到 `~/.zshrc`。Bash 专用语法和交互判断未必适用于 Zsh；安装完成后应逐项迁移并在新终端验证。

## 5. 下载、检查并安装 Oh My Zsh

Oh My Zsh 官方安装器需要 Zsh、Git 与 `curl` 或 `wget`。以下方式先把脚本下载到临时目录，检查后再执行：

~~~bash
installer="$HOME/oh-my-zsh-install.sh"
curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh -o "$installer"
less "$installer"
sh "$installer"
rm -f "$installer"
~~~

`less` 中按 `q` 退出阅读。安装器会克隆框架到 `~/.oh-my-zsh`，并创建新的 `~/.zshrc`；如果原有文件存在，官方安装器通常会保留为 `~/.zshrc.pre-oh-my-zsh`。

网络无法访问 `raw.githubusercontent.com` 时，可将下载地址替换为 Oh My Zsh 官方镜像 `https://install.ohmyz.sh/`，但同样应先下载和检查：

~~~bash
curl -fsSL https://install.ohmyz.sh/ -o "$installer"
less "$installer"
sh "$installer"
rm -f "$installer"
~~~

若你已审阅官方安装方式且明确接受直接执行，也可使用：

~~~bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
~~~

不要将来源不明的镜像站、博客中的二次封装脚本或要求以 root 身份运行的安装命令当作替代方案。

## 6. 在新 Zsh 会话中验证

关闭并重新打开终端后，依次执行：

~~~bash
printf 'configured login shell: %s\n' "$SHELL"
ps -p $$ -o comm=
echo "$ZSH"
omz version
grep '^ZSH_THEME=' "$HOME/.zshrc"
zsh -n "$HOME/.zshrc"
~~~

如果不想先关闭窗口，也可以用下列命令在当前窗口替换为一个登录 Zsh 会话：

~~~bash
exec zsh -l
~~~

`exec` 会替换当前 Shell；未保存的命令行内容不会保留。最可靠的确认方式仍是另开一个全新终端。

## 7. 保持最小配置后再优化

安装器生成的模板已经包含：

~~~zsh
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="robbyrussell"
plugins=(git)
source "$ZSH/oh-my-zsh.sh"
~~~

建议先用这一最小状态工作一段时间。确认 Git 别名、提示符和补全均正常后，再按 [[Oh My Zsh 常用优化配置]] 添加历史管理、自动建议、语法高亮或 fzf。一次只增加一组配置，并在每次变更后新开终端验证，问题才容易定位。

## 常见问题

### 新终端仍然进入 Bash

先判断是账户设置没有变更，还是终端没有使用登录 Shell：

~~~bash
printf '%s\n' "$SHELL"
getent passwd "$USER" | cut -d: -f7
ps -p $$ -o comm=
~~~

若账户登录 Shell 仍为 Bash，重新执行 `chsh -s "$(command -v zsh)"`，然后注销并重新登录。若账户设置已是 Zsh、当前进程仍为 Bash，检查终端 Profile 是否被配置为执行固定命令。

### 安装器提示找不到 Git、curl 或 Zsh

不要手工创建 `~/.oh-my-zsh`。先回到第 2 步并确认：

~~~bash
command -v zsh
command -v git
command -v curl
~~~

若命令在非交互终端可以找到、在新 Zsh 窗口中找不到，通常是 PATH 配置被旧的 `.zshrc` 覆盖；按 [[Oh My Zsh 更新、备份与卸载]] 的诊断步骤启动 `zsh -f` 排除用户配置。

### `chsh` 后 SSH 登录异常或提示 Shell 不可用

保留原 SSH 会话，先在新会话中以 `zsh -f` 测试无配置 Zsh：

~~~bash
zsh -f
~~~

这会跳过用户启动文件。如果无配置 Zsh 正常，问题在 `.zshrc` 或其加载的插件，而不是 Zsh 本身。临时恢复时，可在备用会话执行 `chsh -s /bin/bash`，再修复配置。

## 官方参考资料

- [Oh My Zsh：安装、配置与卸载说明](https://github.com/ohmyzsh/ohmyzsh)
- [Ubuntu Server：安装和管理软件包](https://ubuntu.com/server/docs/how-to/software/package-management/)
- [Zsh：启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)

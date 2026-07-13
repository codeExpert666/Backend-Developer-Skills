---
title: macOS 从零配置 Oh My Zsh
aliases:
  - macOS 安装 Oh My Zsh
  - Mac 安装 Oh My Zsh
  - macOS Terminal 配置 Oh My Zsh
tags:
  - Terminal
  - Terminal/安装
  - Terminal/macOS
  - Zsh
  - Oh-My-Zsh
created: 2026-07-14T00:04:29
updated: 2026-07-14T00:04:29
---

本文面向使用 Mac 自带 Terminal.app 的用户，从系统默认终端开始配置 Oh My Zsh。macOS 10.15 及以后，新建用户账户通常已默认使用 Zsh，因此重点是确认实际状态、保留系统 Shell 边界、安装 Oh My Zsh，并把日常配置放在正确的启动文件中。

本文与 [[Ubuntu 从零安装 Oh My Zsh]] 的区别在于：macOS 通常无需安装或切换基础 Zsh。完成后请继续 [[Oh My Zsh 常用优化配置]]；升级、故障处理和还原见 [[Oh My Zsh 更新、备份与卸载]]。

## 目标与边界

完成后，Terminal.app 新开的窗口会加载 Zsh 与 Oh My Zsh，配置以 `~/.zshrc` 为主。本文默认继续使用 Apple 提供的 `/bin/zsh`，不要求安装 Homebrew，也不将 Homebrew 版本的 Zsh 设为登录 Shell。

| 场景 | 推荐处理 |
| --- | --- |
| macOS 10.15 及以后，`$SHELL` 已是 `/bin/zsh` | 直接安装 Oh My Zsh |
| 较旧 macOS 或账户仍使用 Bash | 先确认 `/bin/zsh`，再用 `chsh` 切换 |
| Terminal.app 被设为运行固定命令 | 先恢复“使用默认登录 Shell”的行为 |
| 想使用 Homebrew Zsh | 作为独立专题处理，先确认其路径已列在 `/etc/shells` |

> [!important] 不要为了安装 Oh My Zsh 更换系统 Shell
> Oh My Zsh 是 Zsh 的配置框架，不需要替换 Apple 的 `/bin/zsh`。对绝大多数个人开发机，系统自带 Zsh 足够完成本文和后续优化；额外引入 Homebrew Zsh 会增加 PATH、`/etc/shells` 和升级责任。

## 1. 确认当前 Shell 与 Terminal 设置

打开 Terminal.app，执行：

~~~bash
printf 'configured login shell: %s\n' "$SHELL"
ps -p $$ -o pid=,comm=
command -v zsh
zsh --version
dscl . -read "/Users/$USER" UserShell
~~~

`$SHELL` 与 `dscl` 输出反映账户的登录 Shell，`ps` 显示当前窗口实际运行的进程。正常情况下，二者会对应 `/bin/zsh` 和 `zsh`。

若账户已是 Zsh、当前窗口却不是 Zsh，打开 Terminal.app 的“设置（Settings）→ 通用（General）”，检查“Shells open with”是否选择了“Default login shell”。如果配置为“Command (complete path)”，该窗口会无视账户的默认 Shell。

## 2. 确认安装器依赖

现代 macOS 自带 `zsh` 与 `curl`。先确认 Git、curl 可用：

~~~bash
git --version
curl --version
~~~

首次运行 `git` 时若弹出安装 Command Line Tools 的提示，按系统提示安装即可；也可以主动执行：

~~~bash
xcode-select --install
~~~

该命令会触发图形安装对话框，并不会在当前终端静默完成。不要为了获得 Git 或 Oh My Zsh 而从不明来源下载 Xcode Command Line Tools 安装包。

## 3. 仅在需要时切换到系统 Zsh

macOS 10.15 及以后，新账户默认 Shell 是 Zsh。若第 1 步已显示 `/bin/zsh`，跳过本节。

对于仍使用 Bash 的账户，先确认系统允许该 Shell：

~~~bash
grep -Fx /bin/zsh /etc/shells
~~~

然后执行：

~~~bash
chsh -s /bin/zsh
~~~

`chsh` 会要求输入当前用户密码。关闭所有 Terminal.app 窗口后重新打开，或者注销并重新登录，再验证：

~~~bash
printf '%s\n' "$SHELL"
ps -p $$ -o comm=
~~~

Apple 要求 `chsh` 的目标路径位于 `/etc/shells`；不要直接编辑该文件来强行加入未知 Shell。若只想在单个窗口临时运行 Zsh，输入 `zsh` 即可，不会更改账户设置。

## 4. 备份已有 Zsh 配置

即使没有主动使用过 Zsh，Mac 上也可能已有工具或团队脚本写入 `~/.zshrc`。先备份：

~~~bash
if [ -f "$HOME/.zshrc" ]; then
  cp "$HOME/.zshrc" "$HOME/.zshrc.before-oh-my-zsh.$(date +%Y%m%d-%H%M%S)"
fi
~~~

尤其不要不加检查地把旧 `~/.bash_profile` 或 `~/.bashrc` 全量复制进 `~/.zshrc`。环境变量、PATH、函数和交互选项应逐段迁移；Bash 的数组、条件语法和补全脚本未必可直接用于 Zsh。

## 5. 下载、检查并安装 Oh My Zsh

推荐先下载官方脚本、阅读后再执行：

~~~bash
installer="$HOME/oh-my-zsh-install.sh"
curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh -o "$installer"
less "$installer"
sh "$installer"
rm -f "$installer"
~~~

安装后，框架默认位于 `~/.oh-my-zsh`，主配置文件为 `~/.zshrc`。如果已有 `.zshrc`，安装器通常会将它重命名为 `.zshrc.pre-oh-my-zsh`；仍应将需要保留的配置逐项合并，而不是直接覆盖新模板。

网络无法访问 `raw.githubusercontent.com` 时，可使用 Oh My Zsh 官方镜像地址，但仍建议先检查内容：

~~~bash
curl -fsSL https://install.ohmyz.sh/ -o "$installer"
less "$installer"
sh "$installer"
rm -f "$installer"
~~~

确认官方脚本来源和行为后，才可选择一行安装形式：

~~~bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
~~~

## 6. 验证并理解配置位置

关闭并新开一个 Terminal.app 窗口，执行：

~~~bash
printf 'configured login shell: %s\n' "$SHELL"
ps -p $$ -o comm=
echo "$ZSH"
omz version
grep '^ZSH_THEME=' "$HOME/.zshrc"
zsh -n "$HOME/.zshrc"
~~~

最小成功状态是当前进程为 `zsh`，`$ZSH` 指向 `~/.oh-my-zsh`，且 `omz version` 有输出。首次加载时，默认主题通常是 `robbyrussell`。

macOS 常见的两个配置文件职责如下：

| 文件 | 适合放什么 | 不适合放什么 |
| --- | --- | --- |
| `~/.zprofile` | 登录环境需要的环境变量、PATH；也可能影响 SSH 登录 | 每次交互终端都要加载的大量插件 |
| `~/.zshrc` | 主题、插件、别名、补全、交互式历史设置 | 非交互脚本必需的全局环境 |

不要把耗时的 Oh My Zsh 插件、网络访问或交互式输出放进 `.zprofile`；更不要把它们放进所有 Zsh 都会读取的 `.zshenv`。

## 7. 在最小配置上继续优化

安装器创建的核心配置类似：

~~~zsh
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="robbyrussell"
plugins=(git)
source "$ZSH/oh-my-zsh.sh"
~~~

建议先保留这个状态，确认 Terminal.app、新建标签页和 SSH 会话均正常，再阅读 [[Oh My Zsh 常用优化配置]]。其中的 macOS 专属建议会明确标注，不会把 Linux 的包管理命令混入 Mac 默认终端。

## 常见问题

### 每次打开 Terminal.app 仍然进入 Bash

依次检查账户设置与 Terminal.app 设置：

~~~bash
printf '%s\n' "$SHELL"
dscl . -read "/Users/$USER" UserShell
ps -p $$ -o comm=
~~~

如果账户默认 Shell 已经是 `/bin/zsh`，但进程仍为 `bash`，通常是 Terminal Profile 配置了固定命令。回到“设置 → 通用”，改回默认登录 Shell 后再新开窗口。

### Oh My Zsh 安装后，某些旧命令或 SDK 不见了

这通常不是 Oh My Zsh 删除了程序，而是旧 PATH 只写在 Bash 配置中。先比较旧文件与新配置，再把确实需要的 `export PATH=...` 迁移到 `~/.zprofile` 或 `~/.zshrc`。若该路径必须通过 SSH 登录可见，优先放在 `.zprofile`；不要为此把整份 Bash 配置复制过来。

### 是否应把 Homebrew 安装的 Zsh 设为默认 Shell

本文不建议这样做。系统 `/bin/zsh` 已能满足日常开发。若以后确实需要 Homebrew 的新版本，应先确认路径、`/etc/shells`、登录会话和自动化脚本的兼容性，并保留 `/bin/zsh` 的回退路径。

## 官方参考资料

- [Apple：在 Mac 上将 zsh 设为默认 Shell](https://support.apple.com/102360)
- [Oh My Zsh：安装、配置与卸载说明](https://github.com/ohmyzsh/ohmyzsh)
- [Zsh：启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)

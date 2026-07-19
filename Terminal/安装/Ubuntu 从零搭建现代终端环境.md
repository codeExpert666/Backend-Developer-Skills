---
title: Ubuntu 从零搭建现代终端环境
aliases:
  - Ubuntu 安装 Antidote 与 Starship
  - Ubuntu 现代终端配置
  - Ubuntu Zsh Atuin zoxide fzf 配置
tags:
  - Terminal
  - Terminal/安装
  - Terminal/Linux
  - Terminal/Ubuntu
  - Zsh
  - Antidote
created: 2026-07-19T16:30:50
updated: 2026-07-19T17:05:02
---

本文在 Ubuntu 上搭建 Zsh + Antidote + Starship + Atuin + zoxide + fzf，并说明 Ubuntu Desktop 与远程 Ubuntu Server 对 Ghostty 的不同处理。配置目录和加载顺序与 [[macOS 从零搭建现代终端环境]] 保持一致，便于跨机器维护。

请先阅读 [[现代终端环境搭建概览]]。已有 Oh My Zsh 的机器应改走 [[从 Oh My Zsh 迁移到 Antidote]]；生产服务器、共享跳板机和受管主机还应先确认软件安装与登录 Shell 变更策略。

## Desktop 与 Server 的边界

先确认自己正在配置哪一类机器：

| 场景 | Ghostty | 命令行工具 |
| --- | --- | --- |
| Ubuntu Desktop 本地开发 | 可在本机安装 Ghostty | 在同一台机器安装 Zsh、Antidote、Starship、Atuin、zoxide、fzf |
| Mac 或其他桌面设备 SSH 到 Ubuntu Server | Ghostty 只装在本地桌面 | 在 Server 安装需要的命令行部分 |
| 无图形桌面的 Ubuntu Server | 不安装 Ghostty | 安装 Zsh 与所需 CLI，重点验证 SSH 新会话 |
| 容器或一次性 CI 环境 | 通常不安装 Ghostty，也不修改登录 Shell | 只按任务需要安装命令，不复制完整交互配置 |

> [!important] Ghostty 不跟随 SSH 安装到远端
> Ghostty 提供本地窗口、键盘输入与终端协议。通过 Mac 上的 Ghostty SSH 到 Ubuntu 时，远端只运行 Shell 和命令行工具；在 Ubuntu Server 上安装图形终端不会改善本地窗口体验。

## 1. 检查系统与当前 Shell

先执行只读检查：

~~~bash
printf 'configured login shell: %s\n' "$SHELL"
ps -p $$ -o pid=,comm=
command -v zsh || true
zsh --version 2>/dev/null || true
cat /etc/os-release
uname -m
printf 'SSH connection: %s\n' "${SSH_CONNECTION:-local session}"
~~~

如果当前是唯一的远程管理会话，先再打开一个 SSH 窗口并保持登录。后续任何配置错误都应能从备用会话恢复；不要在尚未测试新登录的情况下退出全部连接。

## 2. 备份现有配置

以下命令只复制存在的文件和目录：

~~~bash
backup_dir="$HOME/.terminal-backups/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"

for file in "$HOME/.zshenv" "$HOME/.zprofile" "$HOME/.zshrc"; do
  [ -f "$file" ] && cp -p "$file" "$backup_dir/"
done

for directory in "$HOME/.config/zsh" "$HOME/.config/ghostty" "$HOME/.config/atuin"; do
  [ -d "$directory" ] && cp -R "$directory" "$backup_dir/"
done

history_file="${HISTFILE:-$HOME/.zsh_history}"
if [ ! -f "$history_file" ] && [ -f "$HOME/.zhistory" ]; then
  history_file="$HOME/.zhistory"
fi
if [ -f "$history_file" ]; then
  cp -p "$history_file" "$backup_dir/zsh-history"
  printf '%s\n' "$history_file" > "$backup_dir/zsh-history-source.txt"
fi

printf 'backup created: %s\n' "$backup_dir"
~~~

记录备份路径。启用 `ZDOTDIR` 后，原来的 `~/.zshrc` 会暂时不被读取，但先不要删除它。

## 3. 安装系统基础依赖

通过 apt 安装 Zsh、Git、curl、证书和后文审阅安装脚本所需的 `less`：

~~~bash
sudo apt update
sudo apt install -y zsh git curl ca-certificates less
~~~

确认路径和版本：

~~~bash
command -v zsh
zsh --version
git --version
curl --version
~~~

Ubuntu LTS 中的 fzf 可能落后于上游版本。本文的初始化方式需要 `fzf --zsh`，zoxide 当前还要求 fzf 至少为 0.51.0 才能稳定使用交互选择。为避免先装旧包、再用另一份二进制覆盖，主路线直接按 fzf 官方 Git 方式安装到用户目录，并禁止安装器修改 Shell 配置：

~~~bash
fzf_root="${XDG_DATA_HOME:-$HOME/.local/share}"
fzf_dir="$fzf_root/fzf"
mkdir -p "$fzf_root"

if [ ! -d "$fzf_dir/.git" ]; then
  git clone --depth=1 https://github.com/junegunn/fzf.git "$fzf_dir"
fi

"$fzf_dir/install" --bin
export PATH="$fzf_dir/bin:$PATH"
fzf --version
fzf --zsh >/dev/null
~~~

此方式把官方发布的 fzf 二进制放在 `~/.local/share/fzf/bin`；稍后的 `.zshenv` 会让它优先于旧的系统包。若 Git 目录已经存在，日后应先 `git -C "$fzf_dir" pull --ff-only`，再重新执行 `install --bin`。

若 `apt-cache policy fzf` 显示当前受信任仓库的候选版本已经不低于 `0.51.0`，也可以改用 `sudo apt install fzf` 并跳过整个 Git 安装块。同一台机器只保留一种来源；安装后仍须确认 `fzf --zsh` 成功。

## 4. 安装 Starship、Atuin 与 zoxide

Ubuntu 版本之间的软件仓库差异较大。为保持一致，下面从各项目的官方地址下载安装器，先检查内容，再安装到用户目录。

### Starship

~~~bash
mkdir -p "$HOME/.local/bin"
installer="$(mktemp)"
curl -fsSL https://starship.rs/install.sh -o "$installer"
less "$installer"
sh "$installer" -b "$HOME/.local/bin"
rm -f "$installer"
~~~

### Atuin

Atuin 的完整 setup 脚本会自动修改多个 Shell 配置。本文要手工控制 `.zshrc` 的唯一初始化位置，因此只运行 Atuin 官方 release 中的二进制安装器：

~~~bash
installer="$(mktemp)"
curl --proto '=https' --tlsv1.2 -fsSL \
  https://github.com/atuinsh/atuin/releases/latest/download/atuin-installer.sh \
  -o "$installer"
less "$installer"
ATUIN_NO_MODIFY_PATH=1 sh "$installer"
rm -f "$installer"
~~~

release 安装器默认也会尝试把 `~/.atuin/bin` 写入多个 Shell profile。这里使用其官方的 `ATUIN_NO_MODIFY_PATH=1` 开关禁止自动改写；PATH 统一由后文受版本控制的 `~/.zshenv` 管理。

### zoxide

~~~bash
installer="$(mktemp)"
curl -fsSL https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh -o "$installer"
less "$installer"
sh "$installer"
rm -f "$installer"
~~~

`less` 中按 `q` 退出。为当前会话临时补上用户二进制路径并验证：

~~~bash
export PATH="$HOME/.local/bin:$HOME/.atuin/bin:$HOME/.local/share/fzf/bin:$PATH"
starship --version
atuin --version
zoxide --version
fzf --version
~~~

这些都是独立二进制，不由 Antidote 更新。若你的 Ubuntu 版本已在受信任的软件仓库提供合适版本，也可改用对应包管理器，但同一个工具只保留一种安装来源。

## 5. 安装 Antidote

将 Antidote 从官方 Git 仓库安装到与 macOS 相同的 XDG 数据目录：

~~~bash
antidote_root="${XDG_DATA_HOME:-$HOME/.local/share}"
antidote_dir="$antidote_root/antidote"
mkdir -p "$antidote_root"

if [ ! -d "$antidote_dir/.git" ]; then
  git clone --depth=1 https://github.com/mattmc3/antidote.git "$antidote_dir"
fi

test -r "$antidote_dir/antidote.zsh"
~~~

不要把 Antidote 安装目录、插件缓存或生成的 `.zsh_plugins.zsh` 提交到配置仓库。跨机器只同步 Zsh 配置、插件清单和可选 snapshot。

## 6. Ubuntu Desktop 是否安装 Ghostty

Ubuntu Desktop 用户可以使用 Ghostty，但应先理解包来源：Ghostty 项目目前只直接发布 macOS 预编译二进制；Linux 包由发行版或社区维护者构建。Ghostty 官方安装页会列出当前可用的 Linux 包，并明确区分发行版包和风险更高的社区二进制。

推荐顺序是：

1. 优先检查当前 Ubuntu 版本和已配置的受信任仓库是否提供 Ghostty；
2. 在 Ghostty 官方“Binaries and Packages”页面核对该包由谁构建与维护；
3. 只有明确接受维护与供应链风险时，才采用官方页面列出的社区 Ubuntu 包；
4. 不从博客复制不明的一行 `curl | bash`，也不为了终端外观以 root 身份运行未知脚本；
5. 没有合适来源时继续使用 GNOME Terminal，命令行部分完全不受影响。

本文不固定抄写社区 Ubuntu 安装脚本，因为它不是 Ghostty 官方构建产物，维护方式也可能变化。Ubuntu Server 直接跳过本节。

## 7. 建立 ZDOTDIR 配置目录

创建配置文件但不覆盖现有内容：

~~~bash
mkdir -p "$HOME/.config/zsh" "$HOME/.config/atuin"
touch "$HOME/.config/zsh/.zprofile"
touch "$HOME/.config/zsh/.zshrc"
touch "$HOME/.config/zsh/.zsh_plugins.txt"
touch "$HOME/.config/zsh/common.zsh"
touch "$HOME/.config/zsh/macos.zsh"
touch "$HOME/.config/zsh/linux.zsh"
touch "$HOME/.config/zsh/local.zprofile"
touch "$HOME/.config/zsh/local.zsh"
~~~

### 配置 `~/.zshenv`

将下列内容合并到 `~/.zshenv`：

~~~zsh
export ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"

# 所有 Zsh 都会读取 .zshenv，因此这里只保留 SSH 非交互命令也要看到的 PATH。
typeset -U path PATH
[[ -d /usr/local/bin ]] && path=(/usr/local/bin $path)
[[ -d /opt/homebrew/bin ]] && path=(/opt/homebrew/bin $path)
[[ -d "$HOME/.local/share/fzf/bin" ]] && path=("$HOME/.local/share/fzf/bin" $path)
[[ -d "$HOME/.atuin/bin" ]] && path=("$HOME/.atuin/bin" $path)
[[ -d "$HOME/.local/bin" ]] && path=("$HOME/.local/bin" $path)
export PATH
~~~

不存在的用户二进制目录不会加入 PATH。不要在 `.zshenv` 中加载插件、打印文字或运行网络命令。

### 配置 `.zprofile`

将下列跨平台内容保存为 `~/.config/zsh/.zprofile`：

~~~zsh
# macOS 登录 Shell 会执行；Ubuntu 上因文件不存在而自然跳过。
if [[ -x /opt/homebrew/bin/brew ]]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [[ -x /usr/local/bin/brew ]]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

[[ -r "$ZDOTDIR/local.zprofile" ]] && source "$ZDOTDIR/local.zprofile"
~~~

## 8. 声明插件与 `.zshrc`

将下列内容保存为 `~/.config/zsh/.zsh_plugins.txt`：

~~~text
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting kind:clone
~~~

再将下列骨架保存为 `~/.config/zsh/.zshrc`：

~~~zsh
# 使用 XDG state 保存原生 Zsh 历史，作为 Atuin 之外的本地回退。
bindkey -e
zsh_state_dir="${XDG_STATE_HOME:-$HOME/.local/state}/zsh"
mkdir -p "$zsh_state_dir"
HISTFILE="$zsh_state_dir/history"
HISTSIZE=50000
SAVEHIST=20000
setopt append_history
setopt hist_expire_dups_first
setopt hist_ignore_dups
setopt hist_ignore_space
setopt hist_reduce_blanks
unset zsh_state_dir

[[ -r "$ZDOTDIR/common.zsh" ]] && source "$ZDOTDIR/common.zsh"
case "$OSTYPE" in
  darwin*) [[ -r "$ZDOTDIR/macos.zsh" ]] && source "$ZDOTDIR/macos.zsh" ;;
  linux*)  [[ -r "$ZDOTDIR/linux.zsh" ]] && source "$ZDOTDIR/linux.zsh" ;;
esac
[[ -r "$ZDOTDIR/local.zsh" ]] && source "$ZDOTDIR/local.zsh"

# 主路线只使用 Zsh 内置补全，按 Zsh 版本拆分缓存。
zcompdump="${XDG_CACHE_HOME:-$HOME/.cache}/zsh/zcompdump-$ZSH_VERSION"
mkdir -p "${zcompdump:h}"
autoload -Uz compinit
compinit -d "$zcompdump"
unset zcompdump

# Antidote 只负责 Zsh 插件。
export ANTIDOTE_HOME="${XDG_CACHE_HOME:-$HOME/.cache}/antidote"
antidote_dir="${XDG_DATA_HOME:-$HOME/.local/share}/antidote"
if [[ -r "$antidote_dir/antidote.zsh" ]]; then
  source "$antidote_dir/antidote.zsh"
  ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=8'
  antidote load "$ZDOTDIR/.zsh_plugins.txt"
else
  print -u2 "Antidote is missing: $antidote_dir"
fi
unset antidote_dir

# fzf 让出 Ctrl-R；Alt-C 不绑定，目录交互选择统一使用 zoxide 的 zi。
if (( $+commands[fzf] )) && fzf --zsh >/dev/null 2>&1; then
  FZF_CTRL_R_COMMAND= FZF_ALT_C_COMMAND= source <(fzf --zsh)
fi

# Atuin 保留 Ctrl-R，禁用上箭头覆盖与空提示符下的 AI 快捷入口。
(( $+commands[atuin] )) && eval "$(atuin init zsh --disable-up-arrow --disable-ai)"

# zoxide 在 compinit 之后初始化。
(( $+commands[zoxide] )) && eval "$(zoxide init zsh)"

# Starship 只负责提示符。
(( $+commands[starship] )) && eval "$(starship init zsh)"

# zsh-syntax-highlighting 必须在其他交互部件之后真正加载。
if (( $+functions[antidote] )); then
  zsh_highlight_root="$(antidote path zsh-users/zsh-syntax-highlighting 2>/dev/null)"
  if [[ -r "$zsh_highlight_root/zsh-syntax-highlighting.zsh" ]]; then
    source "$zsh_highlight_root/zsh-syntax-highlighting.zsh"
  fi
  unset zsh_highlight_root
fi
~~~

`common.zsh` 放跨平台别名与函数，`linux.zsh` 放 Ubuntu 专属配置，`local.zsh` 放当前机器的代理或私有路径。先让这些文件为空，不影响基础功能。

## 9. 切换登录 Shell 前先测试

先解析配置：

~~~bash
zsh -n "$HOME/.zshenv"
zsh -n "$HOME/.config/zsh/.zprofile"
zsh -n "$HOME/.config/zsh/.zshrc"
~~~

不修改账户登录 Shell，直接测试新配置：

~~~bash
env ZDOTDIR="$HOME/.config/zsh" zsh -lic '
  echo "new zsh config loaded"
  command -v antidote starship atuin zoxide fzf
'
~~~

首次加载时 Antidote 会访问 GitHub 并克隆插件。只有上述测试成功，才确认 Zsh 路径已登记并修改当前用户的登录 Shell：

~~~bash
zsh_path="$(command -v zsh)"
grep -Fx "$zsh_path" /etc/shells
chsh -s "$zsh_path"
~~~

不要使用 `sudo chsh` 修改其他账户。`chsh` 完成后保留当前会话，再从另一个终端或 SSH 窗口验证新登录。

## 10. 本地与 SSH 验收

本地新开终端，或者从客户端重新建立 SSH 连接，执行：

~~~zsh
printf 'login shell: %s\n' "$SHELL"
printf 'current shell: '
ps -p $$ -o comm=
printf 'ZDOTDIR: %s\n' "$ZDOTDIR"
antidote list
bindkey '^R'
starship --version
atuin doctor
zoxide --version
fzf --version
zsh -lic 'echo startup-ok'
~~~

从另一台机器验证 SSH 非交互 PATH 时，将示例主机替换为真实地址：

~~~bash
ssh user@example-host '
  printf "shell=%s ZDOTDIR=%s\n" "$SHELL" "$ZDOTDIR"
  command -v zsh starship atuin zoxide fzf
'
~~~

SSH 远程命令通常不会读取 `.zshrc`，但会经过 Zsh 的 `.zshenv`，因此应能找到这些二进制；提示符、按键和插件只在交互会话加载。再打开一个交互会话，验证 `Ctrl-R`、自动建议、语法高亮、`z`、`zi` 和 `Ctrl-T`。

若 Ghostty 客户端中的 `$TERM` 在远端缺少 terminfo，不要在服务器的启动文件中永久伪造 `TERM`。应按 [[Ghostty 常用配置与 Shell 集成]] 检查当前 Ghostty 稳定版支持的 `ssh-env` 与 `ssh-terminfo` 功能，或使用其安全回退。

## 11. 导入本地历史

确认新会话稳定后，把下列占位目录替换为第 2 节实际输出的备份目录，再把旧 Zsh 历史副本导入 Atuin：

~~~bash
backup_dir="$HOME/.terminal-backups/YYYYMMDD-HHMMSS"
if [ -f "$backup_dir/zsh-history" ]; then
  HISTFILE="$backup_dir/zsh-history" atuin import zsh
fi
atuin stats
~~~

新骨架已把原生 `HISTFILE` 切到 XDG state，因此这里显式指定旧文件副本；切换后直接运行 `atuin import auto` 可能只会读到新的历史文件。导入不会删除备份。服务器命令更可能包含临时令牌、主机名或运维参数，应先阅读 [[Atuin 命令历史管理]] 的过滤规则；注册账户和跨机器同步始终是可选项。

后续更新、快照和回退见 [[现代终端环境更新、验证与回退]]。

## 官方参考资料

- [Ubuntu Server：软件包管理](https://ubuntu.com/server/docs/how-to/software/package-management/)
- [Ghostty：预编译二进制与 Linux 包来源说明](https://ghostty.org/docs/install/binary)
- [Ghostty：Linux 平台说明](https://ghostty.org/docs/linux)
- [Zsh：启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)
- [Antidote：官方安装与使用说明](https://antidote.sh/)
- [Starship：官方安装指南](https://starship.rs/guide/)
- [Atuin：官方安装说明](https://docs.atuin.sh/cli/guide/installation/)
- [Atuin：官方安装脚本源码](https://github.com/atuinsh/atuin/blob/main/install.sh)
- [zoxide：官方安装与 fzf 版本要求](https://github.com/ajeetdsouza/zoxide)
- [fzf：官方安装与 Shell integration](https://github.com/junegunn/fzf)

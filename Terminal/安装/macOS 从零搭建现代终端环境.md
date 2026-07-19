---
title: macOS 从零搭建现代终端环境
aliases:
  - macOS 安装 Ghostty 与 Antidote
  - Mac 现代终端配置
  - macOS Zsh Antidote Starship 配置
tags:
  - Terminal
  - Terminal/安装
  - Terminal/macOS
  - Zsh
  - Antidote
  - Ghostty
created: 2026-07-19T16:30:50
updated: 2026-07-19T17:05:02
---

本文从 macOS 的系统 Zsh 开始，搭建 Ghostty + Zsh + Antidote + Starship + Atuin + zoxide + fzf。完成后，Ghostty 负责本地图形终端，Antidote 只管理 Zsh 插件，其余工具由 Homebrew 管理；Zsh 配置集中在 `~/.config/zsh`，便于与 Ubuntu 复用。

开始前先阅读 [[现代终端环境搭建概览]]。已有 Oh My Zsh 时不要直接执行本文的配置切换，应改走 [[从 Oh My Zsh 迁移到 Antidote]]。各组件的深入配置见 [[Ghostty 常用配置与 Shell 集成]]、[[Zsh 与 Antidote 跨机器配置管理]] 和 [[Atuin 命令历史管理]]。

## 目标与边界

本文采用以下约定：

- 继续使用 Apple 提供的 `/bin/zsh`，不把 Homebrew Zsh 设为登录 Shell；
- Ghostty 只安装在当前 Mac，用它 SSH 到服务器时，不要求远端安装 Ghostty；
- Homebrew 安装 Ghostty、Starship、Atuin、zoxide 与 fzf；
- Antidote 从官方 Git 仓库安装到 `${XDG_DATA_HOME:-$HOME/.local/share}/antidote`，与 Ubuntu 保持相同路径；
- Atuin 先作为本地历史工具使用，不要求注册或启用同步；
- `Ctrl-R` 由 Atuin 接管，上方向键保持 Zsh 原有行为，Atuin AI 入口显式关闭；
- 目录交互选择统一使用 zoxide 的 `zi`，不再绑定 fzf 的 `Alt-C`；fzf 保留 `Ctrl-T` 和模糊补全。

## 1. 检查系统状态

打开 Terminal.app 或现有终端，执行只读检查：

~~~bash
printf 'configured login shell: %s\n' "$SHELL"
ps -p $$ -o pid=,comm=
command -v zsh
zsh --version
command -v git || true
command -v brew || true
uname -m
~~~

macOS 10.15 及以后的账户通常已经使用 `/bin/zsh`。若 `$SHELL` 不是 `/bin/zsh`，先确认该路径存在于 `/etc/shells`，再切换：

~~~bash
grep -Fx /bin/zsh /etc/shells
chsh -s /bin/zsh
~~~

`chsh` 完成后需关闭并新开终端或重新登录。不要为了这套配置额外安装 Homebrew Zsh；系统 Zsh 足以运行 Antidote 和本文的插件。

## 2. 备份现有终端配置

下面的命令只复制存在的文件和目录，不会删除原配置：

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

记录这个目录。后续不要删除原来的 `~/.zshrc`；启用 `ZDOTDIR` 后它只是暂时不再被读取，仍可用于回退。

## 3. 安装 Homebrew

已有 Homebrew 时跳过本节。若 `command -v brew` 没有输出，先确认 Git 可用：

~~~bash
git --version
~~~

首次调用 Git 若提示安装 Command Line Tools，按系统对话框完成安装。然后下载并检查 Homebrew 官方安装器：

~~~bash
installer="$(mktemp)"
curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh -o "$installer"
less "$installer"
/bin/bash "$installer"
rm -f "$installer"
~~~

`less` 中按 `q` 退出。安装完成后按 Homebrew 安装器显示的指令初始化当前会话，或者执行下列自动判断：

~~~bash
if [ -x /opt/homebrew/bin/brew ]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [ -x /usr/local/bin/brew ]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

brew --version
~~~

Apple Silicon 通常使用 `/opt/homebrew`，Intel Mac 通常使用 `/usr/local`。不要把其中一个路径不加判断地复制到所有机器。

## 4. 安装 Ghostty 与独立命令行工具

通过 Homebrew 安装 Ghostty cask 和四个独立命令行工具：

~~~bash
brew install --cask ghostty
brew install starship atuin zoxide fzf
~~~

Ghostty 的 Homebrew cask 由社区维护，但重新封装的是 Ghostty 官方签名与公证的 macOS `.dmg`。若不使用 Homebrew，也可从 Ghostty 官方下载页获取 `.dmg`，拖入“应用程序”。

检查安装结果：

~~~bash
brew list --cask ghostty
starship --version
atuin --version
zoxide --version
fzf --version
~~~

这些二进制不属于 Antidote。以后应使用 `brew upgrade` 更新，而不是把它们写进 `.zsh_plugins.txt`。

## 5. 安装 Antidote

为了让 macOS 与 Ubuntu 使用相同路径，主路线不依赖 Homebrew 的 Antidote 文件布局，而是克隆官方仓库：

~~~bash
antidote_root="${XDG_DATA_HOME:-$HOME/.local/share}"
antidote_dir="$antidote_root/antidote"
mkdir -p "$antidote_root"

if [ ! -d "$antidote_dir/.git" ]; then
  git clone --depth=1 https://github.com/mattmc3/antidote.git "$antidote_dir"
fi

test -r "$antidote_dir/antidote.zsh"
~~~

Homebrew 公式 `brew install antidote` 也可用，但它的加载路径与 Ubuntu 不同。跨机器配置主路线统一使用 Git 安装目录；该目录和 Antidote 缓存都不提交到配置仓库。

## 6. 建立 XDG 与 ZDOTDIR 目录

先创建配置文件，不覆盖内容：

~~~bash
mkdir -p "$HOME/.config/zsh" "$HOME/.config/ghostty" "$HOME/.config/atuin"
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

将下列最小内容合并到 `~/.zshenv`。若文件原本存在，先对照备份逐项迁移，不要直接丢弃组织策略或必要环境变量：

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

不要在 `.zshenv` 中加载插件、运行 `brew shellenv`、输出文字或访问网络。这个文件会被脚本、SSH 非交互命令和其他 Zsh 进程读取，任何耗时或输出都会污染它们。

### 配置 `.zprofile`

将下列内容保存为 `~/.config/zsh/.zprofile`：

~~~zsh
# 只在登录 Shell 中加载 Homebrew 的完整环境。
if [[ -x /opt/homebrew/bin/brew ]]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [[ -x /usr/local/bin/brew ]]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

[[ -r "$ZDOTDIR/local.zprofile" ]] && source "$ZDOTDIR/local.zprofile"
~~~

机器私有的登录环境可放入 `local.zprofile`，并从配置仓库忽略该文件。

## 7. 声明最小插件集合

将下列内容保存为 `~/.config/zsh/.zsh_plugins.txt`：

~~~text
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting kind:clone
~~~

`kind:clone` 表示 Antidote 只下载语法高亮插件，不在此处加载它。稍后的 `.zshrc` 会在所有交互工具初始化结束后最后 `source`，满足该插件的加载顺序要求。

主路线先使用 Zsh 内置补全，不加入 `zsh-users/zsh-completions`。以后若添加额外补全，必须先把其目录加入 `fpath` 再执行 `compinit`，具体见 [[Zsh 与 Antidote 跨机器配置管理]]。

## 8. 写入 `.zshrc` 骨架

将下列内容保存为 `~/.config/zsh/.zshrc`：

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

# 先加载跨平台、平台专属和本机私有设置。
[[ -r "$ZDOTDIR/common.zsh" ]] && source "$ZDOTDIR/common.zsh"
case "$OSTYPE" in
  darwin*) [[ -r "$ZDOTDIR/macos.zsh" ]] && source "$ZDOTDIR/macos.zsh" ;;
  linux*)  [[ -r "$ZDOTDIR/linux.zsh" ]] && source "$ZDOTDIR/linux.zsh" ;;
esac
[[ -r "$ZDOTDIR/local.zsh" ]] && source "$ZDOTDIR/local.zsh"

# Zsh 内置补全；按 Zsh 版本拆分缓存，不放进配置仓库。
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

# fzf 保留 Ctrl-T 与补全；Ctrl-R 交给 Atuin，Alt-C 不绑定，目录选择统一用 zi。
if (( $+commands[fzf] )) && fzf --zsh >/dev/null 2>&1; then
  FZF_CTRL_R_COMMAND= FZF_ALT_C_COMMAND= source <(fzf --zsh)
fi

# Atuin 仅接管 Ctrl-R：保留原生上箭头，并关闭空提示符下的 AI 快捷入口。
(( $+commands[atuin] )) && eval "$(atuin init zsh --disable-up-arrow --disable-ai)"

# zoxide 必须在 compinit 之后初始化，zi 会使用 fzf 做交互选择。
(( $+commands[zoxide] )) && eval "$(zoxide init zsh)"

# Starship 只负责提示符。
(( $+commands[starship] )) && eval "$(starship init zsh)"

# 必须是真正的最后一个交互插件，不能仅在 Antidote 清单中排最后。
if (( $+functions[antidote] )); then
  zsh_highlight_root="$(antidote path zsh-users/zsh-syntax-highlighting 2>/dev/null)"
  if [[ -r "$zsh_highlight_root/zsh-syntax-highlighting.zsh" ]]; then
    source "$zsh_highlight_root/zsh-syntax-highlighting.zsh"
  fi
  unset zsh_highlight_root
fi
~~~

`common.zsh` 适合放跨平台别名和函数，`macos.zsh` 只放 macOS 专属设置，`local.zsh` 放代理或机器私有路径。先保持三个文件为空也完全可用，不要在不同文件中重复初始化同一工具。

## 9. 先检查，再切换配置

不要直接在当前窗口执行 `source ~/.zshrc`，因为启用 `ZDOTDIR` 后真正的配置文件已经改变。先做语法检查：

~~~bash
zsh -n "$HOME/.zshenv"
zsh -n "$HOME/.config/zsh/.zprofile"
zsh -n "$HOME/.config/zsh/.zshrc"
~~~

然后保留当前窗口，另开一个全新终端，或在确认当前命令已经保存后执行：

~~~bash
exec zsh -l
~~~

首次启动时 Antidote 会根据清单克隆插件，需要访问 GitHub。新会话中检查：

~~~zsh
printf 'shell: %s\n' "$SHELL"
printf 'ZDOTDIR: %s\n' "$ZDOTDIR"
antidote list
bindkey '^R'
starship --version
atuin doctor
zoxide --version
fzf --version
~~~

`bindkey '^R'` 应指向 Atuin 的搜索部件。上方向键应继续逐条浏览原生历史；空提示符下输入 `?` 不应触发 Atuin AI。

## 10. 导入历史并启动 Ghostty

Atuin 不注册账户也能在本地工作。确认 Shell 初始化正常后，把下列占位目录替换为第 2 节实际输出的备份目录，再从备份副本导入旧 Zsh 历史：

~~~bash
backup_dir="$HOME/.terminal-backups/YYYYMMDD-HHMMSS"
if [ -f "$backup_dir/zsh-history" ]; then
  HISTFILE="$backup_dir/zsh-history" atuin import zsh
fi
atuin stats
~~~

新骨架已把原生 `HISTFILE` 切到 XDG state，因此这里显式指定旧文件副本；切换后直接运行 `atuin import auto` 可能只会读到新的历史文件。导入不会删除备份。先用本地模式观察一段时间，再按 [[Atuin 命令历史管理]] 决定是否启用端到端加密同步；不要把 Atuin 数据库或密钥提交到 Git。

从“应用程序”启动 Ghostty，保持默认配置即可工作。新窗口中再次执行：

~~~bash
ghostty +version
ps -p $$ -o comm=
printf '%s\n' "$ZDOTDIR"
~~~

Ghostty 的配置路径、字体、分屏和 SSH 终端能力见 [[Ghostty 常用配置与 Shell 集成]]。不要为了连接 Ubuntu Server 而在服务器上再安装 Ghostty；服务器只需要命令行部分。

## 最终验收

在 Ghostty 的新标签页中依次验证：

1. `Ctrl-R` 打开 Atuin，按 `Esc` 返回；
2. 输入历史命令的一部分，自动建议出现在光标后；
3. 输入不存在的命令，语法高亮能改变显示；
4. 进入几个目录后，`z <关键词>` 能跳转，`zi` 能打开 fzf 选择；
5. `Ctrl-T` 能用 fzf 选择文件；
6. 进入 Git 仓库，Starship 能显示仓库状态；
7. `zsh -lic 'echo startup-ok'` 能输出 `startup-ok`。

任何一项失败时先不要增加主题和插件，按 [[现代终端环境更新、验证与回退]] 检查加载顺序、版本与实际配置目录。

## 官方参考资料

- [Apple：在 Mac 上将 zsh 设为默认 Shell](https://support.apple.com/102360)
- [Homebrew：官方安装说明](https://brew.sh/)
- [Ghostty：macOS 二进制与 Homebrew cask](https://ghostty.org/docs/install/binary)
- [Antidote：官方安装与使用说明](https://antidote.sh/)
- [Starship：官方安装与 Zsh 初始化](https://starship.rs/guide/)
- [Atuin：官方安装说明](https://docs.atuin.sh/cli/guide/installation/)
- [Atuin：按键配置与关闭默认绑定](https://docs.atuin.sh/cli/configuration/key-binding/)
- [zoxide：官方安装与 Zsh 初始化](https://github.com/ajeetdsouza/zoxide)
- [fzf：官方安装与 Shell integration](https://github.com/junegunn/fzf)

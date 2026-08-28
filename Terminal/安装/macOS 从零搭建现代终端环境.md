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
updated: 2026-08-28T16:35:10
---

本文从 macOS 的系统 Zsh 开始，搭建 Ghostty + Zsh + Antidote + Starship + Atuin + zoxide + fzf。完成后，Ghostty 负责本地图形终端，Antidote 只管理 Zsh 插件，其余工具由 Homebrew 管理；可复用配置进入普通 Git dotfiles 仓库，并由 GNU Stow 部署到 `~/.config`，便于与 Ubuntu 复用和恢复。

开始前先阅读 [[现代终端环境搭建概览]] 和 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]]。已有 Oh My Zsh 时不要直接执行本文的配置切换，应改走 [[从 Oh My Zsh 迁移到 Antidote]]。各组件的深入配置见 [[Ghostty 常用配置与 Shell 集成]]、[[Zsh 与 Antidote 跨机器配置管理]] 和 [[Atuin 命令历史管理]]。

## 目标与边界

本文采用以下约定：

- 继续使用 Apple 提供的 `/bin/zsh`，不把 Homebrew Zsh 设为登录 Shell；
- Ghostty 只安装在当前 Mac，用它 SSH 到服务器时，不要求远端安装 Ghostty；
- Homebrew 安装 Ghostty、Starship、Atuin、zoxide 与 fzf；
- Git 保存配置源，GNU Stow 使用 `--no-folding` 把受管文件链接到 `$HOME`；
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

通过 Homebrew 安装 Ghostty cask、GNU Stow 和四个独立命令行工具：

~~~bash
brew install --cask ghostty
brew install stow starship atuin zoxide fzf
~~~

Ghostty 的 Homebrew cask 由社区维护，但重新封装的是 Ghostty 官方签名与公证的 macOS `.dmg`。若不使用 Homebrew，也可从 Ghostty 官方下载页获取 `.dmg`，拖入“应用程序”。

检查安装结果：

~~~bash
brew list --cask ghostty
stow --version
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

末行的 `test -r` 用退出状态确认 Antidote 入口文件对当前用户可读，条件命令见 [[Shell 脚本阅读基础#6. 使用 test 表达条件|test 条件判断]]。

Homebrew 公式 `brew install antidote` 也可用，但它的加载路径与 Ubuntu 不同。跨机器配置主路线统一使用 Git 安装目录；该目录和 Antidote 缓存都不提交到配置仓库。

## 6. 建立 dotfiles 源目录与 XDG 目标目录

先按 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]] 初始化普通 Git 仓库。本文用 `$HOME/.dotfiles` 作为示例；若选择了其他位置，只修改 `DOTFILES_DIR`。先确认它确实是预期仓库：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" rev-parse --show-toplevel
~~~

然后在 `zsh` package 中创建配置源，并预建用于容纳符号链接和本机覆盖的真实目标目录：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
zsh_source_dir="$DOTFILES_DIR/zsh/.config/zsh"

mkdir -p "$zsh_source_dir" "$HOME/.config/zsh"
touch "$DOTFILES_DIR/zsh/.zshenv"
touch "$zsh_source_dir/.zprofile"
touch "$zsh_source_dir/.zshrc"
touch "$zsh_source_dir/.zsh_plugins.txt"
~~~

这里不创建仓库中的 `local.zprofile` 或 `local.zsh`。它们是部署后位于 `$HOME/.config/zsh` 的本机真实文件，不能成为符号链接源。

### 配置 `~/.zshenv`

将下列最小内容保存到 `$DOTFILES_DIR/zsh/.zshenv`。若 `~/.zshenv` 原本存在，先按 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库#7. 安全接管已经存在的配置|安全接管已有配置]] 对照备份、选择内容并处理 Stow 冲突，不要直接覆盖：

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

将下列内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zprofile`；Stow 部署后，运行时目标才是 `~/.config/zsh/.zprofile`：

~~~zsh
# 只在登录 Shell 中加载 Homebrew 的完整环境。
if [[ -x /opt/homebrew/bin/brew ]]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [[ -x /usr/local/bin/brew ]]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

[[ -r "$ZDOTDIR/local.zprofile" ]] && source "$ZDOTDIR/local.zprofile"
~~~

机器私有的登录环境放入部署后创建的 `~/.config/zsh/local.zprofile`。它不位于 dotfiles 源树，因此不依赖 `.gitignore` 才能保持不跟踪。

## 7. 声明最小插件集合

将下列内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zsh_plugins.txt`：

~~~text
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting kind:clone
~~~

`kind:clone` 表示 Antidote 只下载语法高亮插件，不在此处加载它。稍后的 `.zshrc` 会在所有交互工具初始化结束后最后 `source`，满足该插件的加载顺序要求。

主路线先使用 Zsh 内置补全，不加入 `zsh-users/zsh-completions`。以后若添加额外补全，必须先把其目录加入 `fpath` 再执行 `compinit`，具体见 [[Zsh 与 Antidote 跨机器配置管理]]。

## 8. 写入 `.zshrc` 骨架

将下列内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zshrc`；代码注释仍使用运行时目标路径，便于理解 Zsh 实际读取什么：

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

需要第一条跨平台别名或函数时，再在仓库源树创建 `common.zsh`；出现第一项 macOS 专属设置时再创建 `macos.zsh`。缺少这两个可选文件时，前面的可读性判断会自然跳过。`local.zsh` 放代理或机器私有路径，并在 Stow 部署后作为目标目录中的真实文件创建。不要为了目录看起来完整而创建空配置，也不要在不同文件中重复初始化同一工具。

## 9. 先检查，再切换配置

不要直接在当前窗口执行 `source ~/.zshrc`。先检查仓库源文件，尚未部署时不能用目标路径代替：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
zsh_source_dir="$DOTFILES_DIR/zsh/.config/zsh"

zsh -n "$DOTFILES_DIR/zsh/.zshenv"
zsh -n "$zsh_source_dir/.zprofile"
zsh -n "$zsh_source_dir/.zshrc"
~~~

源文件通过语法检查后，先模拟 Stow。已有目标发生冲突时必须回到 dotfiles 专题完成备份和比较，不使用 `--adopt` 跳过判断：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --verbose=2 zsh
~~~

确认模拟输出只涉及预期路径且没有 conflict 后，再执行实际部署并检查关键链接：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --verbose=2 zsh

for managed_path in "$HOME/.zshenv" "$HOME/.config/zsh/.zprofile" "$HOME/.config/zsh/.zshrc"; do
  [ -L "$managed_path" ] || {
    printf 'expected managed link: %s\n' "$managed_path" >&2
    exit 1
  }
done
~~~

部署完成后创建权限收紧的本机覆盖，不覆盖已经存在的文件：

~~~bash
umask 077
[ -e "$HOME/.config/zsh/local.zprofile" ] || : > "$HOME/.config/zsh/local.zprofile"
[ -e "$HOME/.config/zsh/local.zsh" ] || : > "$HOME/.config/zsh/local.zsh"
~~~

完成第一次手工部署后，按 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库#8. 固化可重复的部署和验证入口|dotfiles 部署与验证入口]] 保存 `scripts/deploy` 和 `scripts/verify`，并先运行最低验证。

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

1. `$HOME/.zshenv`、`$HOME/.config/zsh/.zprofile` 和 `$HOME/.config/zsh/.zshrc` 是指向预期 dotfiles 仓库的符号链接，`scripts/verify` 通过；
2. `Ctrl-R` 打开 Atuin，按 `Esc` 返回；
3. 输入历史命令的一部分，自动建议出现在光标后；
4. 输入不存在的命令，语法高亮能改变显示；
5. 进入几个目录后，`z <关键词>` 能跳转，`zi` 能打开 fzf 选择；
6. `Ctrl-T` 能用 fzf 选择文件；
7. 进入 Git 仓库，Starship 能显示仓库状态；
8. `zsh -lic 'echo startup-ok'` 能输出 `startup-ok`；
9. dotfiles 的 diff 已审查，不含本机覆盖、历史、密钥或缓存，并已按 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库#9. 首次提交与可选远端|首次提交流程]]形成已知良好提交。

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
- [GNU Stow：官方手册](https://www.gnu.org/software/stow/manual/stow.html)

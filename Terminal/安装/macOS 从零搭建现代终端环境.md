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
updated: 2026-08-31T13:18:23
---

本文从一台尚未建立现代终端配置的 macOS 出发，完成 Ghostty + Zsh + Antidote + Starship + Atuin + zoxide + fzf 的安装，同时筛选可能存在的旧 Zsh 行为，从零建立、部署并提交一份 dotfiles 仓库。单看本文即可完成主流程。

已有 Oh My Zsh 时改走 [[从 Oh My Zsh 迁移到 Antidote]]；已有可信远端 dotfiles 时改走 [[从已有 dotfiles 恢复现代终端环境]]。专题链接只用于进一步理解或个性化配置。

> [!info] 安装资料核对范围
> 本文于 2026-08-30 核对了 Ghostty macOS 包、Antidote、Atuin、fzf 与 zoxide 的官方安装或集成资料。资料核对不等于任何具体 Mac 已执行或验证本文命令。

## 目标与边界

本文采用以下约定：

- 使用 Apple 提供的 `/bin/zsh`，不额外把 Homebrew Zsh 设为登录 Shell；
- Ghostty 只负责本机图形终端，SSH 服务器不安装 Ghostty；
- Homebrew 管理 Ghostty、GNU Stow、Starship、Atuin、zoxide 与 fzf；
- Antidote 只管理 Zsh 插件，安装在 `${XDG_DATA_HOME:-$HOME/.local/share}/antidote`；
- Git 保存配置源，Stow 使用 `--no-folding` 把明确的软件包部署到 `$HOME`；
- Atuin 先使用本地模式，`Ctrl-R` 交给 Atuin，上方向键仍浏览原生历史；
- 先形成已知良好 Git 提交，再导入旧历史、启用同步或进行主题美化。

## 1. 确认起点，不急着切换 Shell

在 Terminal.app 或现有终端执行只读检查：

```sh
sw_vers
uname -m
printf 'inherited SHELL=%s\n' "${SHELL-}"
ps -p $$ -o pid=,ppid=,comm=,args=
dscl . -read "/Users/$USER" UserShell 2>/dev/null || true
printf 'XDG_CONFIG_HOME=%s\n' "${XDG_CONFIG_HOME:-not set}"
printf 'ZDOTDIR=%s\n' "${ZDOTDIR:-not set}"
printf 'HISTFILE=%s\n' "${HISTFILE:-not set}"
command -v /bin/zsh
/bin/zsh --version
command -v git || true
command -v brew || true
```

这里有三个不同状态：

- `dscl` 的 `UserShell` 是账号下次登录应使用的 Shell；
- `$SHELL` 通常是启动当前会话时继承的账号设置；
- `ps` 显示当前实际进程。

它们不一致不代表命令失败，可能只是尚未重新登录。即使账号还不是 `/bin/zsh`，也先保留现状；第 10 节会在新配置通过测试后处理。

检查是否已有 dotfiles 或会冲突的配置：

```sh
for candidate_path in \
  "$HOME/.dotfiles" \
  "$HOME/.zshenv" \
  "$HOME/.zprofile" \
  "$HOME/.zshrc" \
  "$HOME/.zlogin" \
  "$HOME/.zlogout" \
  "$HOME/.config/zsh"; do
  if [ -e "$candidate_path" ] || [ -L "$candidate_path" ]; then
    ls -ld "$candidate_path"
  fi
done
```

如果 `$HOME/.dotfiles` 已存在，本文的“从零初始化”前提不成立。先检查它的 Git 状态和远端，必要时改走恢复主线，不要覆盖。若现有 `.zshenv` 或环境变量把 `ZDOTDIR`、`HISTFILE` 指向其他位置，先记下实际路径；下一节备份和第 6 节迁移都必须覆盖这些真实来源，不能只检查默认目录。

## 2. 建立并验证私密恢复基线

先复制旧配置和历史，备份不进入 Git。若第 1 节发现非默认 `ZDOTDIR`、历史路径或被启动文件加载的其他配置源，先把经过确认的**具体文件或配置目录**加入下面的复制清单；不要为了省事复制整个 `$HOME`：

```sh
previous_umask=$(umask)
umask 077

backup_dir="$HOME/terminal-backups/macos-before-zsh-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"

for source_path in \
  "$HOME/.zshenv" \
  "$HOME/.zprofile" \
  "$HOME/.zshrc" \
  "$HOME/.zlogin" \
  "$HOME/.zlogout" \
  "$HOME/.config/zsh" \
  "$HOME/.config/ghostty" \
  "$HOME/.config/atuin" \
  "$HOME/.config/starship.toml"; do
  if [ -e "$source_path" ] || [ -L "$source_path" ]; then
    cp -a "$source_path" "$backup_dir/"
  fi
done

history_file=""
if [ -n "${ZSH_VERSION:-}" ] && [ -n "${HISTFILE:-}" ]; then
  history_file="$HISTFILE"
  fc -AI "$history_file"
elif [ -f "$HOME/.zsh_history" ]; then
  history_file="$HOME/.zsh_history"
elif [ -f "$HOME/.zhistory" ]; then
  history_file="$HOME/.zhistory"
fi
if [ -n "$history_file" ] && [ -f "$history_file" ]; then
  cp -p "$history_file" "$backup_dir/zsh-history"
  printf '%s\n' "$history_file" > "$backup_dir/zsh-history-source.txt"
fi

printf 'backup=%s\n' "$backup_dir"
find "$backup_dir" -ls
umask "$previous_umask"
```

`find` 的输出应能看到实际复制项，且目录和文件不向其他用户开放。打印“backup”不是验证；若预期文件缺失，先查清实际路径再继续。

## 3. 安装 Homebrew 与工具

已有 Homebrew 时跳过安装器。若没有，先触发或完成 Command Line Tools 安装，再下载并阅读 Homebrew 官方脚本：

```sh
git --version

installer=$(mktemp)
curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh -o "$installer"
less "$installer"
/bin/bash "$installer"
rm -f "$installer"
```

`less` 中按 `q` 退出。按安装器提示初始化当前会话，或判断常见路径：

```sh
if [ -x /opt/homebrew/bin/brew ]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [ -x /usr/local/bin/brew ]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

brew --version
```

Apple Silicon 通常使用 `/opt/homebrew`，Intel Mac 通常使用 `/usr/local`；配置中会同时做存在性判断。

安装桌面终端和独立命令行工具：

```sh
brew install --cask ghostty
brew install stow starship atuin zoxide fzf

brew list --cask ghostty
stow --version
starship --version
atuin --version
zoxide --version
fzf --version
```

这些工具以后由 Homebrew 更新，不进入 Antidote 插件清单。

再从官方 Git 仓库安装 Antidote，使 macOS 与 Ubuntu 使用同一路径：

```sh
antidote_root="${XDG_DATA_HOME:-$HOME/.local/share}"
antidote_dir="$antidote_root/antidote"
mkdir -p "$antidote_root"

if [ ! -d "$antidote_dir/.git" ]; then
  git clone --depth=1 https://github.com/mattmc3/antidote.git "$antidote_dir"
fi

test -r "$antidote_dir/antidote.zsh"
```

Antidote 安装目录、插件 clone 和生成缓存都不进入 dotfiles。

## 4. 初始化 dotfiles，并建立启动所需软件包

只有第 1 节已确认目标不存在时，才执行：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

if [ -e "$DOTFILES_DIR" ]; then
  printf '停止：目标已经存在：%s\n' "$DOTFILES_DIR" >&2
else
  mkdir -p "$DOTFILES_DIR"
  git -C "$DOTFILES_DIR" init
fi
```

看到“停止”后不要执行后续小节；先按第 1 节查清目录来源。只有 `else` 分支实际完成初始化时才继续。

将以下内容保存为 `$DOTFILES_DIR/README.md`：

```markdown
# dotfiles

This repository stores reusable terminal configuration sources.

## Packages

- `zsh`: Zsh startup files and Antidote plugin manifest.
- `atuin`: reviewed non-secret Atuin preferences.
- `starship`: minimal cross-machine prompt configuration.
- `ghostty`: minimal macOS desktop terminal configuration.

## Deployment

Install Git, GNU Stow, Zsh, Antidote, and the referenced CLI tools. Run the
same explicit package list with Stow `--simulate --restow` first, review the
output, and then run `--restow` without `--simulate`.

## Local-only data

Machine-local overrides, shell history, Atuin keys and databases, caches,
plugin clones, logs, and secrets stay outside Git.
```

创建本机首次启动需要的实际软件包及真实目标父目录：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
zsh_source_dir="$DOTFILES_DIR/zsh/.config/zsh"

mkdir -p \
  "$zsh_source_dir" \
  "$DOTFILES_DIR/atuin/.config/atuin" \
  "$DOTFILES_DIR/starship/.config" \
  "$DOTFILES_DIR/ghostty/.config/ghostty" \
  "$HOME/.config/zsh" \
  "$HOME/.config/atuin" \
  "$HOME/.config/ghostty"
```

仓库布局镜像 `$HOME`：`zsh/.zshenv` 部署到 `$HOME/.zshenv`，`atuin/.config/atuin/config.toml` 部署到 `$HOME/.config/atuin/config.toml`。四个软件包共同构成首次 Zsh 与 Ghostty 启动所需的受管配置闭包。原理见 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]]，但继续本文不需要先阅读它。

## 5. 建立独立于旧配置的 Zsh 与组件配置基线

旧 Zsh 配置因机器而异，但新的启动入口、原生历史、插件加载和工具初始化是这套环境固定需要的基线。由于 `.zshrc` 会初始化 Atuin 与 Starship，而 macOS 主线还要首次打开 Ghostty，本节先建立这些不依赖旧配置的全部受管源；第 6 节才有依据判断哪些旧行为已经被替代、哪些仍需迁移。

本节只建立目标结构，不宣称旧配置已经迁移完成。`.zshrc` 对 `common.zsh`、`macos.zsh` 和 local 文件都使用“存在且可读才加载”的判断，因此共享与平台文件可以等到真正出现第一项保留内容时再创建。

### 5.1 所有 Zsh 都读取的 `.zshenv`

将以下内容保存为 `$DOTFILES_DIR/zsh/.zshenv`：

```zsh
export ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"

# 所有 Zsh 都会读取 .zshenv，因此这里只保留非交互命令也需要的 PATH。
typeset -U path PATH
[[ -d /usr/local/bin ]] && path=(/usr/local/bin $path)
[[ -d /opt/homebrew/bin ]] && path=(/opt/homebrew/bin $path)
[[ -d "$HOME/.local/share/fzf/bin" ]] && path=("$HOME/.local/share/fzf/bin" $path)
[[ -d "$HOME/.atuin/bin" ]] && path=("$HOME/.atuin/bin" $path)
[[ -d "$HOME/.local/bin" ]] && path=("$HOME/.local/bin" $path)
export PATH
```

`.zshenv` 会影响脚本和 SSH 非交互命令，因此不加载插件、不输出文字、不访问网络。XDG 与 `ZDOTDIR` 的路径模型见 [[XDG 基础目录与终端配置边界]]。

### 5.2 登录 Shell 读取的 `.zprofile`

将以下内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zprofile`：

```zsh
# 只在登录 Shell 中加载 Homebrew 的完整环境。
if [[ -x /opt/homebrew/bin/brew ]]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [[ -x /usr/local/bin/brew ]]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

[[ -r "$ZDOTDIR/local.zprofile" ]] && source "$ZDOTDIR/local.zprofile"
```

机器私有的登录环境稍后写入不受 Git 管理的 `local.zprofile`。

### 5.3 Antidote 插件清单

将以下内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zsh_plugins.txt`：

```text
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting kind:clone
```

`kind:clone` 只下载语法高亮插件；`.zshrc` 会在最后手动加载它，保证加载顺序。

### 5.4 交互式 `.zshrc`

将以下内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zshrc`：

```zsh
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
plugin_manifest="$ZDOTDIR/.zsh_plugins.txt"
plugin_static="${XDG_CACHE_HOME:-$HOME/.cache}/antidote/zsh_plugins.zsh"
if [[ -r "$antidote_dir/antidote.zsh" ]]; then
  source "$antidote_dir/antidote.zsh"
  ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=8'
  mkdir -p "${plugin_static:h}"
  antidote load "$plugin_manifest" "$plugin_static"
else
  print -u2 "Antidote is missing: $antidote_dir"
fi
unset antidote_dir plugin_manifest plugin_static

# fzf 保留 Ctrl-T 与补全；Ctrl-R 交给 Atuin，Alt-C 留给 zi。
if (( $+commands[fzf] )) && fzf --zsh >/dev/null 2>&1; then
  FZF_CTRL_R_COMMAND= FZF_ALT_C_COMMAND= source <(fzf --zsh)
fi

(( $+commands[atuin] )) && eval "$(atuin init zsh --disable-up-arrow --disable-ai)"
(( $+commands[zoxide] )) && eval "$(zoxide init zsh)"
(( $+commands[starship] )) && eval "$(starship init zsh)"

# 语法高亮必须在其他交互插件之后加载。
if (( $+functions[antidote] )); then
  zsh_highlight_root="$(antidote path zsh-users/zsh-syntax-highlighting 2>/dev/null)"
  if [[ -r "$zsh_highlight_root/zsh-syntax-highlighting.zsh" ]]; then
    source "$zsh_highlight_root/zsh-syntax-highlighting.zsh"
  fi
  unset zsh_highlight_root
fi
```

这份基线已经接管 Zsh 原生历史、补全、Antidote 插件、fzf、Atuin、zoxide 和 Starship。第 6 节审阅旧历史选项、提示符、补全或工具初始化时，应先与这里的现有能力比较，而不是再复制一套。

需要第一条跨平台或 macOS 专属设置时，再分别创建 `common.zsh` 或 `macos.zsh`；不要预建空文件。插件顺序和平台拆分的扩展规则见 [[Zsh 与 Antidote 跨机器配置管理]]。

### 5.5 建立本机私有入口

第 4 节已经建立真实目标目录。现在创建不受 Git 管理的本机覆盖，供下一节承接机器路径、代理、账号或秘密：

```sh
previous_umask=$(umask)
umask 077
local_files_ok=1

for local_file in \
  "$HOME/.config/zsh/local.zprofile" \
  "$HOME/.config/zsh/local.zsh"; do
  if [ -L "$local_file" ] \
    || { [ -e "$local_file" ] && [ ! -f "$local_file" ]; }; then
    printf 'STOP: local target is not a regular file: %s\n' "$local_file" >&2
    local_files_ok=0
    continue
  fi
  [ -e "$local_file" ] || : > "$local_file"
  chmod 600 "$local_file"
done

umask "$previous_umask"
if [ "$local_files_ok" -ne 1 ]; then
  unset local_file local_files_ok previous_umask
  false
else
  unset local_file local_files_ok previous_umask
fi
```

如果普通文件此前已经存在，这段命令不会清空内容，只会把权限收紧为 `600`；符号链接、目录或其他类型会触发停止。

### 5.6 写入 Atuin 的受管配置

将以下内容保存为 `$DOTFILES_DIR/atuin/.config/atuin/config.toml`：

```toml
auto_sync = false
enter_accept = false
style = "compact"
inline_height = 20
filter_mode = "global"
workspaces = true
secrets_filter = true

history_filter = [
  "^export .*(_TOKEN|_PASSWORD|_SECRET)=",
  "^curl .*Authorization:",
]
```

不要在此时运行 `atuin info`、`atuin doctor` 或真实 Zsh；配置缺失时，加载设置的过程可能先在目标路径生成普通文件。

### 5.7 写入 Starship 的受管配置

将以下内容保存为 `$DOTFILES_DIR/starship/.config/starship.toml`：

```toml
"$schema" = "https://starship.rs/config-schema.json"

add_newline = false

[hostname]
ssh_only = true
format = "[$hostname]($style) "

[character]
success_symbol = "[>](bold green)"
error_symbol = "[>](bold red)"
```

### 5.8 写入 Ghostty 的受管配置

将以下内容保存为 `$DOTFILES_DIR/ghostty/.config/ghostty/config.ghostty`：

```ini
theme = light:Catppuccin Latte,dark:Catppuccin Mocha
window-theme = system
```

Ghostty 在缺少非空配置时可能于首次启动生成默认文件，因此在第 8 节部署 `ghostty` 之前不要打开应用。到这里也先不要启动真实 Zsh；旧 Zsh 行为还必须经过下一节的逐项判断。

## 6. 逐项审阅并迁移旧 Zsh 行为

迁移单位是“仍想保留的行为”，不是旧启动文件或代码块。一个 `.zshrc` 可能同时包含历史、提示符、补全、跨平台别名、macOS 命令和机器秘密，它们不会进入同一个新文件。

如果第 1 节确认完全没有旧 Zsh 配置，本节仍需明确记录“没有候选行为”，然后继续；不能把“没有看到默认文件”推断成已经迁移完成。

### 6.1 确认旧启动链与真实来源

在默认启动选项下，Zsh 先读取 `.zshenv`；登录 Shell 随后读取 `.zprofile`，交互式 Shell 读取 `.zshrc`，登录 Shell 还会在交互配置之后读取 `.zlogin`，退出登录 Shell 时读取 `.zlogout`。用户文件实际位于哪里还受启动时已有的 `ZDOTDIR` 以及旧 `.zshenv` 中赋值的影响。

Stow 尚未接管时，旧文件仍在原位。先依次完整阅读默认位置和本文目标位置中实际存在的候选文件；`less` 中按 `q` 退出当前文件，随后会打开下一个文件：

```sh
for startup_file in \
  "$HOME/.zshenv" \
  "$HOME/.zprofile" \
  "$HOME/.zshrc" \
  "$HOME/.zlogin" \
  "$HOME/.zlogout" \
  "$HOME/.config/zsh/.zprofile" \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/zsh/.zlogin" \
  "$HOME/.config/zsh/.zlogout" \
  "$HOME/.config/zsh/common.zsh" \
  "$HOME/.config/zsh/macos.zsh" \
  "$HOME/.config/zsh/local.zprofile" \
  "$HOME/.config/zsh/local.zsh"; do
  [ -f "$startup_file" ] || continue
  printf '\nreading: %s\n' "$startup_file"
  [ -L "$startup_file" ] && ls -ld "$startup_file"
  less -N "$startup_file"
done

unset startup_file
```

如果第 1 节的环境变量或旧 `.zshenv` 指向其他 `ZDOTDIR`，再输入展开后的真实目录并阅读其中的启动文件：

```sh
printf 'existing custom ZDOTDIR, or leave empty: '
IFS= read -r existing_zdotdir

if [ -n "$existing_zdotdir" ] && [ -d "$existing_zdotdir" ]; then
  for startup_name in \
    .zshenv .zprofile .zshrc .zlogin .zlogout \
    common.zsh macos.zsh local.zprofile local.zsh; do
    startup_file="$existing_zdotdir/$startup_name"
    [ -f "$startup_file" ] || continue
    printf '\nreading: %s\n' "$startup_file"
    [ -L "$startup_file" ] && ls -ld "$startup_file"
    less -N "$startup_file"
  done
fi

unset existing_zdotdir startup_file startup_name
```

继续追踪每个 `source` 或 `.` 指向的真实文件，不能只凭文件名判断配置来源。如果发现 `oh-my-zsh.sh`、`$ZSH/oh-my-zsh.sh` 或现有 Oh My Zsh 插件依赖，应停止本文并改走 [[从 Oh My Zsh 迁移到 Antidote]]，不要把两条主线拼在一起。

在私密工作记录中为每项候选行为登记以下信息。不要把账号、主机名、私有路径、代理或秘密原样复制进公开笔记和 dotfiles：

| 来源与行号 | 想保留的行为 | 生效时机 | 共享范围 | 新落点或决定 | 验证方法 |
| --- | --- | --- | --- | --- | --- |
| 例如：旧 `.zshrc` 中的 `ll` | 显示详细文件列表 | 交互式 | 跨平台 | `common.zsh` | 新 Zsh 中运行 `alias ll` 与 `ll` |

### 6.2 迁移环境变量、PATH 与登录初始化

先处理旧 `.zshenv`、`.zprofile` 及其加载文件中的每一项环境设置。每读到一项就依次判断：是否仍然需要、何时生效、能否跨机器共享、是否含机器身份或秘密，然后立即修改目标并登记验证命令。

| 旧行为的真实范围 | 新落点或决定 | 处理原则 |
| --- | --- | --- |
| 所有 Zsh，包括适用的 SSH 非交互命令都必须看到的最小 PATH | `$DOTFILES_DIR/zsh/.zshenv` | 使用目录存在判断；不运行插件、提示符或输出文字 |
| 跨机器共享的登录环境 | `$DOTFILES_DIR/zsh/.config/zsh/.zprofile` | 不重复第 5 节已有的 Homebrew `shellenv` |
| 机器私有的登录环境 | `$HOME/.config/zsh/local.zprofile` | 权限保持为 `600`，不进入 Git |
| 旧的自定义 `ZDOTDIR` | 通常不迁移赋值本身 | 先把其中行为迁入新结构；新的根 `.zshenv` 已统一指向 XDG 配置目录 |
| 试图影响所有图形应用的环境变量 | 不直接认定 `.zprofile` 足够 | 先确认具体应用的启动方式和配置入口 |
| 已失效、重复或不再需要的初始化 | 不迁移 | 在迁移记录中写明原因 |

每写入一项就对实际修改的目标运行语法检查；全部登录环境处理完后再统一检查一次：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

/bin/zsh -n "$DOTFILES_DIR/zsh/.zshenv"
/bin/zsh -n "$DOTFILES_DIR/zsh/.config/zsh/.zprofile"
/bin/zsh -n "$HOME/.config/zsh/local.zprofile"
```

语法通过只表示 Zsh 可以解析，不表示 PATH、SDK 或环境变量已在真实新会话中生效。为每项同时记录 `command -v`、变量输出或工具自检命令，留到第 9 节运行。

### 6.3 迁移交互式别名、函数和工具初始化

接着审阅旧 `.zshrc` 及其实际加载文件。先问“想保留什么行为”，再决定新表达和落点：

| 旧 Zsh 中的行为 | 新落点或决定 |
| --- | --- |
| 跨 macOS、Linux 的交互别名或函数 | 有第一项有效内容时创建 `common.zsh` |
| 依赖 macOS 路径、BSD 命令或 Apple 工具的交互配置 | 有第一项有效内容时创建 `macos.zsh` |
| 机器路径、代理、账号、秘密或仅本机需要的函数 | `$HOME/.config/zsh/local.zsh` |
| 旧历史路径、容量和去重选项 | 不直接复制；先与第 5 节的原生历史和 Atuin 配置比较 |
| `PROMPT`、`RPROMPT` 或其他提示符框架 | 不直接复制；提示符已由 Starship 接管 |
| 旧 `compinit`、补全缓存或生成的补全文件 | 不直接复制；第 5 节已建立 XDG 缓存和 `compinit` |
| 旧 fzf、Atuin、zoxide、Starship 初始化 | 不重复；第 5 节已统一初始化顺序 |
| 旧插件管理器或手动 `source` 的插件 | 先确认仍需保留的能力，再决定 Antidote 清单或明确淘汰 |

不要为了保留一条别名复制整份旧 `.zshrc`。`common.zsh` 与 `macos.zsh` 只有在出现第一项确定保留的内容时才创建；没有相应行为时，让 `.zshrc` 的可读性判断自然跳过它们。

每次编辑后对实际存在的目标做增量语法检查：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
zsh_source_dir="$DOTFILES_DIR/zsh/.config/zsh"

for migrated_source in \
  "$zsh_source_dir/common.zsh" \
  "$zsh_source_dir/macos.zsh" \
  "$HOME/.config/zsh/local.zsh"; do
  [ -f "$migrated_source" ] && /bin/zsh -n "$migrated_source"
done

unset migrated_source zsh_source_dir
```

语法检查后仍要为每个别名、函数或 SDK 初始化记录真实的新 Zsh 验证命令。别名既要用 `alias NAME` 确认定义，也要运行一次它负责的正常操作。

### 6.4 处理 `.zlogin`、`.zlogout` 与旧文件边界

- `.zlogin` 在登录 Shell 的交互配置之后读取。普通登录环境优先迁到 `.zprofile`，交互行为迁到共享、macOS 或 local 交互文件；只有确实依赖“在 `.zshrc` 之后执行”的行为才考虑保留专门的 `.zlogin`。
- `.zlogout` 属于登录 Shell 的退出流程，默认不迁移。若旧文件包含必须保留的清理行为，先确认它不会删除历史、凭据或仍在使用的临时状态，再单独设计和验证。
- 旧根 `.zprofile`、`.zshrc`、`.zlogin` 和 `.zlogout` 暂不删除。新的 `.zshenv` 改变 `ZDOTDIR` 后它们通常不再进入新启动链，但仍是回退依据。

### 6.5 关闭迁移清单

进入测试与部署前，逐行确认私密迁移记录满足以下条件：

1. 每项候选行为都有“迁移、由新基线替代、继续保留在旧入口、明确放弃”四类结论之一；
2. 迁移项已经写入真实目标，不是只有未来落点；
3. 每个修改过的 Zsh 文件都通过了 `/bin/zsh -n`；
4. 每项行为都有准备在第 9 节执行的验证命令；
5. local 文件、历史、秘密和机器身份均未进入 dotfiles。

只有清单已经关闭，才进入下一节测试仓库源。后文不再重新决定旧配置的落点。

## 7. 在接管 `$HOME` 前只做静态检查

先检查语法：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
zsh_source_dir="$DOTFILES_DIR/zsh/.config/zsh"

/bin/zsh -n "$DOTFILES_DIR/zsh/.zshenv"
/bin/zsh -n "$zsh_source_dir/.zprofile"
/bin/zsh -n "$zsh_source_dir/.zshrc"
test -s "$DOTFILES_DIR/atuin/.config/atuin/config.toml"
test -s "$DOTFILES_DIR/starship/.config/starship.toml"
test -s "$DOTFILES_DIR/ghostty/.config/ghostty/config.ghostty"

for optional_source in \
  "$zsh_source_dir/common.zsh" \
  "$zsh_source_dir/macos.zsh" \
  "$HOME/.config/zsh/local.zprofile" \
  "$HOME/.config/zsh/local.zsh"; do
  [ -f "$optional_source" ] && /bin/zsh -n "$optional_source"
done
unset optional_source
```

到这里不运行 `/bin/zsh -ic` 或 `/bin/zsh -lic`。仅把 `ZDOTDIR` 指向仓库并不隔离 `$HOME`、`XDG_CONFIG_HOME`、data、state 与 cache；`.zshrc` 中的 Atuin 等组件仍可能写入真实目标。静态检查只能证明文件可解析，真实启动统一放到完整配置闭包部署后的第 9 节。

## 8. 模拟 Stow、处理冲突并实际部署

先模拟：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --restow --verbose=2 zsh atuin starship ghostty
```

若出现冲突，比较第 2 节备份、当前目标和仓库源。只移动已经确认由新配置取代的明确目标；不要使用 `--adopt` 自动把未知内容改写进仓库，也不要移动整个 `$HOME/.config`。

模拟输出只涉及预期路径且无冲突后，应用完全相同的软件包列表：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --restow --verbose=2 zsh atuin starship ghostty
```

检查链接确实属于仓库：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
managed_links_ok=1

for managed_path in \
  "$HOME/.zshenv" \
  "$HOME/.config/zsh/.zprofile" \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/zsh/.zsh_plugins.txt" \
  "$HOME/.config/atuin/config.toml" \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/ghostty/config.ghostty"; do
  test -L "$managed_path" || {
    printf 'expected symlink: %s\n' "$managed_path" >&2
    managed_links_ok=0
    continue
  }
  ls -ld "$managed_path"
  readlink "$managed_path"
done

for local_file in \
  "$HOME/.config/zsh/local.zprofile" \
  "$HOME/.config/zsh/local.zsh"; do
  if [ ! -f "$local_file" ] || [ -L "$local_file" ]; then
    printf 'expected private regular file: %s\n' "$local_file" >&2
    managed_links_ok=0
  fi
done

git -C "$DOTFILES_DIR" status --short --branch

if [ "$managed_links_ok" -ne 1 ]; then
  unset local_file managed_links_ok managed_path
  printf 'STOP: deployment target verification failed\n' >&2
  false
else
  unset local_file managed_links_ok managed_path
fi
```

## 9. 验证真实启动链

保留当前窗口作为回退入口，先启动子 Shell：

```sh
/bin/zsh -lic '
  printf "shell-version=%s\nZDOTDIR=%s\n" "$ZSH_VERSION" "$ZDOTDIR"
  command -v starship atuin zoxide fzf
  printf "startup-ok\n"
'
```

这一步经过新的 `.zshenv`、`.zprofile`、`.zshrc`、共享、macOS 和 local 文件，并首次真正执行 Atuin、Starship、zoxide、fzf 与 Antidote 初始化。还要逐项运行第 6 节为旧 PATH、SDK、别名和函数准备的验证命令。

首次启动时 Antidote 可能需要从 GitHub 克隆插件。再打开一个全新的 Terminal.app 或 Ghostty 窗口，检查：

```zsh
printf 'inherited SHELL=%s\n' "$SHELL"
ps -p $$ -o comm=,args=
printf 'ZDOTDIR=%s\n' "$ZDOTDIR"
antidote list
bindkey '^R'
atuin doctor
zoxide --version
fzf --version
```

`Ctrl-R` 应打开 Atuin；上方向键仍浏览原生历史；`zi` 应调用 fzf；进入 Git 仓库时 Starship 应显示状态。

首次 Zsh 与 Ghostty 启动后，再次审计配置所有权：

```sh
for managed_path in \
  "$HOME/.zshenv" \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/atuin/config.toml" \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/ghostty/config.ghostty"; do
  test -L "$managed_path" || {
    printf 'STOP: managed target changed ownership: %s\n' "$managed_path" >&2
    false
  }
done

git -C "$HOME/.dotfiles" status --short --branch
```

Atuin 数据库、Antidote 插件和缓存、Zsh 历史可以在仓库外生成；它们不应替换上述链接，也不应出现在 dotfiles 工作树中。

## 10. 必要时才修改账号登录 Shell

若第 1 节已显示账号使用 `/bin/zsh`，无需执行 `chsh`。否则在第 9 节通过后执行：

```sh
grep -Fx /bin/zsh /etc/shells
chsh -s /bin/zsh
```

退出 macOS 账号或关闭全部终端后重新进入，再比较：

```sh
dscl . -read "/Users/$USER" UserShell
printf 'inherited SHELL=%s\n' "$SHELL"
ps -p $$ -o comm=,args=
```

`chsh` 成功与当前窗口立刻变成 Zsh 不是一回事。

## 11. 先保存已知良好提交

启动和链接均通过后，审查配置：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" diff --check
git -C "$DOTFILES_DIR" diff
```

只暂存本文创建的稳定文件：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" add -- README.md zsh atuin starship ghostty
git -C "$DOTFILES_DIR" diff --cached --check
git -C "$DOTFILES_DIR" diff --cached
git -C "$DOTFILES_DIR" grep --cached -nEi \
  '(password|passwd|token|secret|private[ _-]?key|api[ _-]?key)' -- . || true
git -C "$DOTFILES_DIR" commit -m "feat: establish modern terminal dotfiles"
git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" log -1 --format=fuller
```

首次提交提供配置回退点，但不等于已经推送。需要远端时，先在托管平台创建空仓库，再交互读取实际 URL：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
printf 'dotfiles remote URL: '
IFS= read -r REPO_URL
git -C "$DOTFILES_DIR" remote add origin "$REPO_URL"
git -C "$DOTFILES_DIR" remote -v
git -C "$DOTFILES_DIR" push -u origin "$(git -C "$DOTFILES_DIR" branch --show-current)"
```

## 12. 最后处理历史和个性化配置

从第 2 节实际打印的备份目录中，先人工检查并脱敏历史副本，再导入 Atuin：

```sh
printf 'sanitized Zsh history path: '
IFS= read -r sanitized_history

if [ -f "$sanitized_history" ]; then
  HISTFILE="$sanitized_history" atuin import zsh
fi
atuin stats
```

不要把历史、Atuin 数据库或密钥提交到 Git。同步策略见 [[Atuin 命令历史管理]]；Ghostty 与 Starship 已有最小受管基线，后续个性化直接修改对应仓库源，zoxide 与 fzf 的个性化则按专题边界处理。

## 最终验收

在 Ghostty 的新标签页中确认：

1. 旧 Zsh 配置和历史已有私密备份，非默认来源也已纳入；
2. 每项旧 Zsh 候选行为都有迁移、由新基线替代、继续保留或明确放弃的结论；
3. Zsh、Atuin、Starship 与 Ghostty 的受管目标在首次真实启动前已经由 Stow 部署，启动后仍指向 `$HOME/.dotfiles`；
4. `zsh -lic 'printf "startup-ok\\n"'` 正常结束；
5. `Ctrl-R`、自动建议、语法高亮、`zi`、`Ctrl-T` 和 Starship 按预期工作；
6. local 文件、历史、密钥、插件 clone 与缓存均未出现在 `git status`；
7. dotfiles 至少有一个已知可用提交，若配置了远端则已核对推送分支。

失败时保留当前窗口，按 [[现代终端环境更新、验证与回退]] 定位；需要解除部署时，先模拟并确认软件包，再执行 Stow `--delete`，随后从第 2 节备份恢复明确文件。

## 官方参考资料

- [Apple：在 Mac 上将 zsh 设为默认 Shell](https://support.apple.com/102360)
- [Zsh：启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)
- [Homebrew：官方安装说明](https://brew.sh/)
- [Ghostty：macOS 二进制与 Homebrew cask](https://ghostty.org/docs/install/binary)
- [GNU Stow Manual](https://www.gnu.org/software/stow/manual/stow.html)
- [Antidote：官方安装与使用说明](https://antidote.sh/)
- [Starship：官方安装与 Zsh 初始化](https://starship.rs/guide/)
- [Atuin：官方安装说明](https://docs.atuin.sh/cli/guide/installation/)
- [zoxide：官方安装与 Zsh 初始化](https://github.com/ajeetdsouza/zoxide)
- [fzf：官方安装与 Shell integration](https://github.com/junegunn/fzf)

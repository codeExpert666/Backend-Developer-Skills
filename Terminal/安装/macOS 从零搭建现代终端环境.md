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
updated: 2026-09-01T15:50:00
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
printf 'ZDOTDIR=%s\n' "${ZDOTDIR:-not set}"
printf 'HISTFILE=%s\n' "${HISTFILE:-not set}"
command -v /bin/zsh
/bin/zsh --version
command -v git || true
command -v brew || true
```

接着检查本文固定使用的 XDG 默认布局。变量未设置或为空是正常状态，不需要补写 `export`；出现 `STOP` 时不要临时清空变量继续，应先按 [[XDG 基础目录与终端配置边界]] 迁移旧路径：

```sh
xdg_layout_ok=1

for xdg_name in \
  XDG_CONFIG_HOME \
  XDG_DATA_HOME \
  XDG_STATE_HOME \
  XDG_CACHE_HOME; do
  case "$xdg_name" in
    XDG_CONFIG_HOME) xdg_default="$HOME/.config" ;;
    XDG_DATA_HOME)   xdg_default="$HOME/.local/share" ;;
    XDG_STATE_HOME)  xdg_default="$HOME/.local/state" ;;
    XDG_CACHE_HOME)  xdg_default="$HOME/.cache" ;;
  esac

  # 名称来自上面的固定列表；读取当前 Shell 参数，也能发现尚未 export 的旧赋值。
  eval "xdg_value=\${$xdg_name-}"
  printf '%s=%s; effective=%s\n' \
    "$xdg_name" "${xdg_value:-not set}" "${xdg_value:-$xdg_default}"

  case "$xdg_value" in
    "") ;;
    /*)
      if [ "$xdg_value" != "$xdg_default" ]; then
        printf 'STOP: standard mainline uses the default XDG layout: %s\n' \
          "$xdg_name" >&2
        xdg_layout_ok=0
      fi
      ;;
    *)
      printf 'STOP: %s must be an absolute path: %s\n' \
        "$xdg_name" "$xdg_value" >&2
      xdg_layout_ok=0
      ;;
  esac
done

xdg_runtime_value="${XDG_RUNTIME_DIR-}"
printf 'XDG_RUNTIME_DIR=%s\n' "${xdg_runtime_value:-not set}"
case "$xdg_runtime_value" in
  ""|/*) ;;
  *)
    printf 'STOP: XDG_RUNTIME_DIR must be an absolute path: %s\n' \
      "$xdg_runtime_value" >&2
    xdg_layout_ok=0
    ;;
esac

if [ "$xdg_layout_ok" -ne 1 ]; then
  unset xdg_default xdg_layout_ok xdg_name xdg_runtime_value xdg_value
  false
else
  unset xdg_default xdg_layout_ok xdg_name xdg_runtime_value xdg_value
fi
```

`XDG_RUNTIME_DIR` 只观察，不在用户启动文件中建立；它应由登录会话或系统按正确权限和生命周期提供。

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
  "$HOME/.config/zsh" \
  "$HOME/Library/Application Support/com.mitchellh.ghostty"; do
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
  "$HOME/Library/Application Support/com.mitchellh.ghostty" \
  "$HOME/.config/atuin" \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/starship"; do
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
# 不导出 ZDOTDIR；子 Zsh 必须重新读取根 .zshenv，才能获得同一组早期启动开关。
ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"

# Ubuntu 的系统级 zshrc 可能先调用 compinit；共享 dotfiles 由用户 .zshrc 统一初始化并把转储写入 XDG cache。
# 该开关不需要导出；没有读取者的平台不受影响，未来把同一仓库部署到 Ubuntu 时才会生效。
skip_global_compinit=1

# 所有 Zsh 都会读取 .zshenv，因此这里只保留必须提前生效的启动开关和非交互命令也需要的 PATH。
typeset -U path PATH
[[ -d /usr/local/bin ]] && path=(/usr/local/bin $path)
[[ -d /opt/homebrew/bin ]] && path=(/opt/homebrew/bin $path)
[[ -d "$HOME/.local/bin" ]] && path=("$HOME/.local/bin" $path)
export PATH
```

`.zshenv` 会影响脚本和 SSH 非交互命令，因此不加载插件、不输出文字、不访问网络。`ZDOTDIR` 只决定当前 Zsh 后续去哪里读取启动文件，不向子进程导出；否则子 Zsh 会绕过根 `$HOME/.zshenv`，无法重新取得其中非导出的早期开关。这里保留 `skip_global_compinit` 是因为该文件属于跨平台共享配置源：macOS 主线本身不依赖这个开关，而 Ubuntu 必须让它早于系统级 `/etc/zsh/zshrc` 生效，不能延后到 `linux.zsh` 或用户 `.zshrc`。XDG 与 `ZDOTDIR` 的路径模型见 [[XDG 基础目录与终端配置边界]]。

### 5.2 登录 Shell 读取的 `.zprofile`

将以下内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zprofile`：

```zsh
# 登录 Zsh 在 macOS 和 Ubuntu 都会读取本文件；Ubuntu 主线未安装 Homebrew，因此两个条件均不成立。
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
# 使用 Emacs 风格键位，供后续 ZLE 插件在同一套基础键位映射上绑定按键。
bindkey -e

# 原生 Zsh 历史写入 XDG state，作为 Atuin 之外的本地回退。
zsh_state_dir="${XDG_STATE_HOME:-$HOME/.local/state}/zsh"
mkdir -p "$zsh_state_dir"
HISTFILE="$zsh_state_dir/history"

# 内存历史最多保留 50000 条，磁盘历史文件长期保留 20000 条。
HISTSIZE=50000
SAVEHIST=20000

# 多个会话退出时追加历史；需要裁剪时优先淘汰有副本的旧记录。
setopt append_history
setopt hist_expire_dups_first

# 原生历史跳过相邻重复和空格开头的命令，并压缩无意义的多余空白。
setopt hist_ignore_dups
setopt hist_ignore_space
setopt hist_reduce_blanks

# 临时路径变量不再需要，避免留在当前 Shell 的全局参数中。
unset zsh_state_dir

# 按“跨平台共享 → 当前平台 → 本机私有”的顺序加载可选配置；文件不存在时跳过，后加载内容可以覆盖前面的默认值。
[[ -r "$ZDOTDIR/common.zsh" ]] && source "$ZDOTDIR/common.zsh"
case "$OSTYPE" in
  darwin*) [[ -r "$ZDOTDIR/macos.zsh" ]] && source "$ZDOTDIR/macos.zsh" ;;
  linux*)  [[ -r "$ZDOTDIR/linux.zsh" ]] && source "$ZDOTDIR/linux.zsh" ;;
esac
[[ -r "$ZDOTDIR/local.zsh" ]] && source "$ZDOTDIR/local.zsh"

# 初始化 Zsh 自带的可编程补全，让 Tab 能根据当前命令和参数位置提供候选项。
# 补全转储是可重新生成的初始化缓存；按 Zsh 版本分文件，避免升级后复用旧结果。
zcompdump="${XDG_CACHE_HOME:-$HOME/.cache}/zsh/zcompdump-$ZSH_VERSION"
mkdir -p "${zcompdump:h}"

# 从 Zsh 函数搜索路径自动加载 compinit，并用指定转储文件初始化当前会话的补全系统。
autoload -Uz compinit
compinit -d "$zcompdump"

# 只清理临时路径变量，磁盘上的补全转储仍保留供下次启动复用。
unset zcompdump

# Antidote 本体放在 XDG data；可重新下载的插件仓库与生成脚本放在 XDG cache。
export ANTIDOTE_HOME="${XDG_CACHE_HOME:-$HOME/.cache}/antidote"
antidote_dir="${XDG_DATA_HOME:-$HOME/.local/share}/antidote"
plugin_manifest="$ZDOTDIR/.zsh_plugins.txt"
plugin_static="${XDG_CACHE_HOME:-$HOME/.cache}/antidote/zsh_plugins.zsh"

# 入口可读时把 Antidote 管理函数加载到当前 Shell，再处理共享插件清单。
if [[ -r "$antidote_dir/antidote.zsh" ]]; then
  source "$antidote_dir/antidote.zsh"

  # 在自动建议插件加载前设置低亮度显示样式。
  ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=8'

  # 生成脚本是可重建缓存；显式放入 XDG cache，避免在受管配置目录旁生成未跟踪文件。
  mkdir -p "${plugin_static:h}"

  # 下载缺失的插件仓库并加载普通插件；kind:clone 项只下载，留待后文手动加载。
  antidote load "$plugin_manifest" "$plugin_static"
else
  # 本体缺失时只向标准错误报告，保留无插件但仍可使用的基础 Zsh。
  print -u2 "Antidote is missing: $antidote_dir"
fi

# 只清理本体、清单与生成脚本路径变量；ANTIDOTE_HOME 继续供 Antidote 的后续命令使用。
unset antidote_dir plugin_manifest plugin_static

# 仅在 fzf 存在且支持 --zsh 时加载按键绑定与模糊补全，避免缺失或旧版本打断 Shell 启动。
# 保留 Ctrl-T 文件选择；禁用 fzf 的 Ctrl-R 与 Alt-C，分别由 Atuin 和 zoxide 的 zi 承担历史搜索与目录选择。
if (( $+commands[fzf] )) && fzf --zsh >/dev/null 2>&1; then
  FZF_CTRL_R_COMMAND= FZF_ALT_C_COMMAND= source <(fzf --zsh)
fi

# 若已安装 Atuin，则加载用于记录命令的钩子与历史搜索按键，并由它接管 Ctrl-R。
# 不让 Atuin 覆盖上箭头或绑定 ? 的 AI 入口，使原有历史翻阅与普通输入行为保持不变。
(( $+commands[atuin] )) && eval "$(atuin init zsh --disable-up-arrow --disable-ai)"

# 若已安装 zoxide，则在 compinit 后加载用于记录目录访问的钩子、z/zi 命令及补全。
# z 按访问记录排名跳转，zi 通过 fzf 交互选择；未使用 --cmd cd，因此不替换原生 cd。
(( $+commands[zoxide] )) && eval "$(zoxide init zsh)"

# 若已安装 Starship，则加载提示符钩子，按当前目录、Git 状态和上一条命令结果渲染提示符。
# 它只接管提示符；放在其他提示符配置之后，避免初始化结果再次被覆盖。
(( $+commands[starship] )) && eval "$(starship init zsh)"

# 语法高亮在插件清单中以 kind:clone 只下载；Antidote 可用时再查询其本地目录。
if (( $+functions[antidote] )); then
  zsh_highlight_root="$(antidote path zsh-users/zsh-syntax-highlighting 2>/dev/null)"

  # 主脚本可读时才最后手动加载，使高亮钩子在其他 Zsh 行编辑器（ZLE）组件之后注册。
  if [[ -r "$zsh_highlight_root/zsh-syntax-highlighting.zsh" ]]; then
    source "$zsh_highlight_root/zsh-syntax-highlighting.zsh"
  fi

  # 清理仅用于定位插件的临时变量。
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

将以下统一的 local-first 受管基线保存为 `$DOTFILES_DIR/atuin/.config/atuin/config.toml`。它与其他平台主线、迁移主线使用相同的有效配置；各项语义和调整条件见 [[Atuin 命令历史管理]]：

```toml
# 即使已经登录 Atuin 账户，也不自动同步；需要同步时可手动运行 `atuin sync`。
# 若希望 Atuin 定期自动同步，可改为 true；更新检查由 update_check 单独控制。
auto_sync = false

# false：选中历史后先回填到 Shell 命令行，便于检查或编辑，再按 Enter 执行。
# true：在 Atuin 界面中按 Enter 会立即执行选中的命令。
enter_accept = false

# 使用精简界面；可改为 auto（按终端高度切换）或 full（完整界面）。
style = "compact"

# 限制界面最多占 20 行；调大可同时显示更多内容，设为 0 则使用全部可用高度。
inline_height = 20

# 使用模糊匹配搜索命令内容；它决定“怎样匹配”，不决定“搜索哪些历史”。
# 在搜索界面可按 Ctrl-S 临时循环其他匹配模式。
search_mode = "fuzzy"

# 打开交互搜索时先检索全部历史；当前还支持 host、session、directory、workspace 和 session-preload。
# 进入界面后可用 Ctrl-R 循环 search.filters 中启用的过滤范围；本基线不覆盖该列表。
filter_mode = "global"

# 启用 workspace 过滤能力：在 Git 仓库中可检索整个仓库树，而不只当前目录。
# 它不会改变上面的默认过滤范围；不在 Git 仓库时 workspace 模式会被跳过。
workspaces = true

# 启用 Atuin 内置的凭据格式过滤；它只是安全网，不能覆盖所有秘密格式。
secrets_filter = true

# history_filter 使用正则表达式；^ 和 $ 分别限定命令开头和结尾。
# 自定义正则只阻止之后匹配的命令入库，不会自动删除既有历史。
# 调整规则后若要清理旧记录，应先用 `atuin history prune --dry-run` 预览影响。
history_filter = [
  # 排除以 export 开头、变量名以 _TOKEN、_PASSWORD 或 _SECRET 结尾的赋值。
  "^export .*(_TOKEN|_PASSWORD|_SECRET)=",
  # 排除以 curl 开头且包含 Authorization: 请求头的命令。
  "^curl .*Authorization:",
]

[logs]
# Atuin 当前默认写入 ~/.atuin/logs；日志属于可跨进程保留但不应进入 Git 的本机状态。
dir = "~/.local/state/atuin/logs"
```

不要在此时运行 `atuin info`、`atuin doctor` 或真实 Zsh；配置缺失时，加载设置的过程可能先在目标路径生成普通文件。

### 5.7 写入 Starship 的受管配置

将以下内容保存为 `$DOTFILES_DIR/starship/.config/starship.toml`。这是一份“最小启动覆盖配置”：“最小”指只覆盖少量选项，不表示提示符只会显示这里出现的模块。由于它没有设置顶层 `format`，其他模块仍按 Starship 默认的 `$all` 格式和各自检测条件出现；本阶段只先建立能解析、能回退的受管源：

```toml
# 供支持 JSON Schema 的编辑器补全键名并检查值类型；这行不定义提示符外观。
# 键名包含 $，不属于 TOML 裸键允许的字符，因此必须用引号包住。
"$schema" = "https://starship.rs/config-schema.json"

# 不在相邻两次提示符之间额外插入空行，使终端输出更紧凑；需要分隔感时可改为 true。
add_newline = false

# hostname 模块负责显示系统主机名。
[hostname]

# 仅在 SSH 会话中显示主机名；若本地终端也需要显示，可改为 false。
ssh_only = true

# $hostname 和 $style 是 Starship 的格式变量，不由 Shell 展开。
# 方括号包住要显示的主机名，圆括号应用模块样式；末尾空格用于分隔后续模块。
# 当前不显示默认的 SSH 图标和连接词；若要换色，可在本表增加 style 配置。
format = "[$hostname]($style) "

# character 模块显示命令输入位置旁的提示符，并根据上一条命令是否成功切换样式。
[character]

# 方括号内的 > 是实际显示内容，圆括号内是样式；这些括号本身不会显示。
# 上一条命令退出码为 0 时显示粗体绿色 >。
success_symbol = "[>](bold green)"

# 上一条命令退出码非 0 时显示粗体红色 >；需要更强区分时可调整符号或样式。
error_symbol = "[>](bold red)"
```

完成第 11 节的已知良好提交后，再按 [[Starship 提示符配置#7. 部署三套配置并选择活动配置|Starship 三配置部署流程]] 把 [[Starship 提示符配置#4. 默认配置：用显式 format 限定后端开发模块|第 4 节完整配置]] 写入默认路径，并把第 5、6 节作为两个备用 profile 同时保留；不要在首次验收前替换启动基线，也不要拼接多份顶层 `format` 或同名 TOML 表。

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

Ghostty 在 macOS 还会读取 `~/Library/Application Support/com.mitchellh.ghostty/`，并且其中配置晚于 XDG 文件加载。先只读查看两个可能的旧文件，把仍需保留的非秘密设置人工迁入 dotfiles 源；不要同时保留两套会互相覆盖的配置：

```sh
ghostty_native_dir="$HOME/Library/Application Support/com.mitchellh.ghostty"

for ghostty_config_name in config.ghostty config; do
  ghostty_native_config="$ghostty_native_dir/$ghostty_config_name"
  if [ -e "$ghostty_native_config" ] || [ -L "$ghostty_native_config" ]; then
    ls -ld "$ghostty_native_config"
    sed -n '1,240p' "$ghostty_native_config"
  fi
done
unset ghostty_config_name ghostty_native_config ghostty_native_dir
```

完成逐项审阅和迁移后，把旧文件移动到第 2 节已经验证的私密备份，而不是删除：

```sh
ghostty_native_dir="$HOME/Library/Application Support/com.mitchellh.ghostty"
ghostty_native_backup="$backup_dir/ghostty-native-config-disabled"
mkdir -p "$ghostty_native_backup"

for ghostty_config_name in config.ghostty config; do
  ghostty_native_config="$ghostty_native_dir/$ghostty_config_name"
  if [ -e "$ghostty_native_config" ] || [ -L "$ghostty_native_config" ]; then
    mv "$ghostty_native_config" "$ghostty_native_backup/"
  fi
done

ghostty_native_ok=1
for ghostty_config_name in config.ghostty config; do
  if [ -e "$ghostty_native_dir/$ghostty_config_name" ] \
    || [ -L "$ghostty_native_dir/$ghostty_config_name" ]; then
    printf 'STOP: macOS Ghostty config still overrides XDG: %s\n' \
      "$ghostty_native_dir/$ghostty_config_name" >&2
    ghostty_native_ok=0
  fi
done

if [ "$ghostty_native_ok" -ne 1 ]; then
  unset ghostty_config_name ghostty_native_backup ghostty_native_config \
    ghostty_native_dir ghostty_native_ok
  false
else
  unset ghostty_config_name ghostty_native_backup ghostty_native_config \
    ghostty_native_dir ghostty_native_ok
fi
```

回退时先退出 Ghostty，再把备份中的明确文件移回原生目录；不要恢复后同时保留内容相冲突的 XDG 文件。

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
  [[ "$(typeset -p ZDOTDIR)" != export\ * ]] || {
    print -u2 "STOP: ZDOTDIR must not be exported"
    exit 1
  }
  command -v starship atuin zoxide fzf
  (( $+functions[antidote] )) || exit 1

  cache_root="${XDG_CACHE_HOME:-$HOME/.cache}"
  for cache_path in \
    "$cache_root/zsh/zcompdump-$ZSH_VERSION" \
    "$cache_root/antidote/zsh_plugins.zsh"; do
    [[ -s "$cache_path" ]] || {
      print -u2 "STOP: expected cache missing or empty: $cache_path"
      exit 1
    }
  done

  for unexpected_path in \
    "$ZDOTDIR/.zcompdump" \
    "$ZDOTDIR/.zsh_plugins.zsh"; do
    [[ ! -e "$unexpected_path" ]] || {
      print -u2 "STOP: generated file leaked into ZDOTDIR: $unexpected_path"
      exit 1
    }
  done

  printf "startup-ok\n"
'
```

这一步经过新的 `.zshenv`、系统级与用户级启动文件、`.zprofile`、`.zshrc`、共享、macOS 和 local 文件，并首次真正执行 Atuin、Starship、zoxide、fzf 与 Antidote 初始化。它还同时证明补全转储和 Antidote 静态加载脚本位于 XDG cache，且默认生成文件没有泄漏到 `$ZDOTDIR`。还要逐项运行第 6 节为旧 PATH、SDK、别名和函数准备的验证命令。

首次启动时 Antidote 可能需要从 GitHub 克隆插件。再打开一个全新的 Terminal.app 或 Ghostty 窗口，检查：

```zsh
printf 'inherited SHELL=%s\n' "$SHELL"
ps -p $$ -o comm=,args=
printf 'ZDOTDIR=%s\n' "$ZDOTDIR"
antidote list
bindkey '^R'
atuin doctor
atuin config get logs.dir --verbose
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

ghostty_native_dir="$HOME/Library/Application Support/com.mitchellh.ghostty"
ghostty_native_ok=1
for ghostty_config_name in config.ghostty config; do
  if [ -e "$ghostty_native_dir/$ghostty_config_name" ] \
    || [ -L "$ghostty_native_dir/$ghostty_config_name" ]; then
    printf 'STOP: native Ghostty config reappeared: %s\n' \
      "$ghostty_native_dir/$ghostty_config_name" >&2
    ghostty_native_ok=0
  fi
done

if [ "$ghostty_native_ok" -ne 1 ]; then
  unset ghostty_config_name ghostty_native_dir ghostty_native_ok
  false
else
  unset ghostty_config_name ghostty_native_dir ghostty_native_ok
fi
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

上面的提交是首次启动、链接所有权与旧 Zsh 行为迁移都已验证的恢复点。只有它存在后，才按 [[Starship 提示符配置#7. 部署三套配置并选择活动配置|Starship 三配置部署流程]] 建立最终提示符布局：默认路径改为第 4 节完整配置，第 5、6 节分别写入命名 profile，并以同一个 `starship` package 重新模拟、部署和验证。把这次个性化作为后续独立变更审查；切换 profile 本身不应产生新的 Git diff。

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

不要把历史、Atuin 数据库或密钥提交到 Git。同步策略见 [[Atuin 命令历史管理]]；Ghostty 已有最小受管基线，Starship 的三配置最终布局只按上面的专题流程建立，zoxide 与 fzf 的个性化则按专题边界处理。

## 最终验收

在 Ghostty 的新标签页中确认：

1. 旧 Zsh 配置和历史已有私密备份，非默认来源也已纳入；
2. 每项旧 Zsh 候选行为都有迁移、由新基线替代、继续保留或明确放弃的结论；
3. Zsh、Atuin、Starship 与 Ghostty 的受管目标在首次真实启动前已经由 Stow 部署，启动后仍指向 `$HOME/.dotfiles`；macOS 原生 Ghostty 目录没有会覆盖 XDG 配置的 `config.ghostty` 或 `config`；
4. 第 9 节的登录交互启动与缓存边界检查正常结束并输出 `startup-ok`；
5. `Ctrl-R`、自动建议、语法高亮、`zi`、`Ctrl-T` 和 Starship 按预期工作；
6. local 文件、历史、Atuin 日志、密钥、插件 clone 与缓存均未出现在 `git status`，补全与插件生成文件只位于 XDG cache，Atuin 日志位于 XDG state；
7. dotfiles 至少有一个已知可用提交，若配置了远端则已核对推送分支；
8. 最终三配置布局中的默认文件和两个备用 profile 都由 Stow 管理且能独立解析；未设置 `STARSHIP_CONFIG` 时使用第 4 节，切换动作不改写链接或仓库源，这次扩展已作为启动基线之后的独立变更审阅。

失败时保留当前窗口，按 [[现代终端环境更新、验证与回退]] 定位；需要解除部署时，先模拟并确认软件包，再执行 Stow `--delete`，随后从第 2 节备份恢复明确文件。

## 官方参考资料

- [Apple：在 Mac 上将 zsh 设为默认 Shell](https://support.apple.com/102360)
- [Zsh：启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)
- [Zsh：`compinit` 转储文件与 `-d` 参数](https://zsh.sourceforge.io/Doc/Release/Completion-System.html)
- [Homebrew：官方安装说明](https://brew.sh/)
- [Ghostty：macOS 二进制与 Homebrew cask](https://ghostty.org/docs/install/binary)
- [GNU Stow Manual](https://www.gnu.org/software/stow/manual/stow.html)
- [Antidote：官方安装与使用说明](https://antidote.sh/)
- [Starship：官方安装与 Zsh 初始化](https://starship.rs/guide/)
- [Atuin：官方安装说明](https://docs.atuin.sh/cli/guide/installation/)
- [zoxide：官方安装与 Zsh 初始化](https://github.com/ajeetdsouza/zoxide)
- [fzf：官方安装与 Shell integration](https://github.com/junegunn/fzf)

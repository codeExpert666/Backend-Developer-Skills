---
title: 从 Oh My Zsh 迁移到 Antidote
aliases:
  - Oh My Zsh 迁移 Antidote
  - Zsh 框架迁移
  - Oh My Zsh 现代化迁移
tags:
  - Terminal
  - Terminal/安装
  - Terminal/迁移
  - Zsh
  - Oh-My-Zsh
  - Antidote
created: 2026-07-19T16:30:50
updated: 2026-09-02T19:05:45
---

本文把正在使用的 Oh My Zsh 迁移为 Zsh + Antidote + Starship + Atuin + zoxide + fzf，同时建立或接入普通 Git dotfiles，并由 GNU Stow 部署。单看本文即可完成盘点、并行重建、切换、提交和回退。

迁移的核心不是尽快删除 Oh My Zsh，而是先保全旧环境，在另一套 `ZDOTDIR` 中重建真实能力，测试通过后再用最小 `.zshenv` 切换。旧 `~/.zshrc` 与 `~/.oh-my-zsh` 保留到观察期结束。

> [!info] 安装资料核对范围
> 本文于 2026-08-30 核对了 Antidote、Atuin、fzf、zoxide 与 Ghostty 的官方资料。资料核对不等于某台机器已经完成迁移。

## 迁移原则

1. 先记录主题、插件、别名、函数、PATH、SDK、自定义目录和历史位置。
2. 新配置先在 `~/.config/zsh` 并行运行，不把 Antidote 混入旧 `~/.zshrc`。
3. 主题交给 Starship，历史搜索交给 Atuin，目录跳转交给 zoxide；Antidote 只管理 Zsh 插件。
4. 只选择确实依赖的 Oh My Zsh 子插件，不重新加载整个框架。
5. 至少保留一个旧终端或 SSH 会话，直到全新会话通过验证。
6. 先形成已知良好 Git 提交，再导入历史、目录数据或启用同步。

## 1. 记录当前 Oh My Zsh 状态

在当前 Oh My Zsh 会话执行只读检查：

```zsh
printf 'inherited SHELL=%s\n' "$SHELL"
ps -p $$ -o pid=,ppid=,comm=,args=
printf 'Oh My Zsh root=%s\n' "${ZSH-}"
printf 'theme=%s\n' "${ZSH_THEME-}"
print -rl -- ${plugins[@]}

if (( $+functions[omz] )); then
  omz version
fi

command -v git stow starship atuin zoxide fzf 2>/dev/null || true
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

检查主配置和 custom 目录：

```zsh
grep -nE \
  '^(export |path=|PATH=|ZSH_THEME=|plugins=|source |alias |function )' \
  "$HOME/.zshrc" || true

find "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}" \
  -maxdepth 3 -type d -name .git -print 2>/dev/null
```

不要只抄 `plugins=(...)`。代理、SDK、手写函数、PATH 和 custom 插件更容易造成迁移后“命令不见了”。

## 2. 建立并验证私密迁移基线

先把当前 Zsh 内存中的新历史追加到原历史文件，再备份配置、custom 目录、插件清单和别名：

```zsh
previous_umask=$(umask)
umask 077

backup_dir="$HOME/terminal-backups/omz-migration-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"
print -r -- "$previous_umask" > "$backup_dir/umask-before.txt"

for source_path in \
  "$HOME/.zshenv" \
  "$HOME/.zprofile" \
  "$HOME/.zshrc" \
  "$HOME/.config/zsh" \
  "$HOME/.config/atuin" \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/starship"; do
  if [[ -e "$source_path" || -L "$source_path" ]]; then
    cp -a "$source_path" "$backup_dir/"
  fi
done

[[ -e "$HOME/.zshenv" || -L "$HOME/.zshenv" ]] \
  || : > "$backup_dir/no-original-zshenv"

legacy_history_file="${HISTFILE:-$HOME/.zsh_history}"
if [[ -n "$legacy_history_file" ]]; then
  fc -AI "$legacy_history_file"
fi
if [[ -f "$legacy_history_file" ]]; then
  cp -p "$legacy_history_file" "$backup_dir/zsh-history"
  print -r -- "$legacy_history_file" > "$backup_dir/zsh-history-source.txt"
fi

if [[ -d "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}" ]]; then
  cp -a "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}" "$backup_dir/omz-custom"
fi

print -rl -- ${plugins[@]} > "$backup_dir/omz-plugins.txt"
alias > "$backup_dir/aliases.txt"

umask "$previous_umask"
printf 'backup=%s\n' "$backup_dir"
unset legacy_history_file previous_umask source_path
```

验证目录权限、原 `.zshrc` 副本、插件清单和 umask：

```zsh
backup_verified=1
[[ -d "$backup_dir" ]] || backup_verified=0
[[ "$(stat -f '%Lp' "$backup_dir" 2>/dev/null || stat -c '%a' "$backup_dir")" == 700 ]] \
  || backup_verified=0
[[ -f "$backup_dir/.zshrc" ]] || backup_verified=0
[[ -s "$backup_dir/omz-plugins.txt" ]] || backup_verified=0
[[ "$(umask)" == "$(<"$backup_dir/umask-before.txt")" ]] || backup_verified=0

if (( backup_verified != 1 )); then
  print -u2 'STOP: backup verification failed'
  false
else
  printf 'backup verified=%s\n' "$backup_dir"
fi
unset backup_verified
```

若当前插件数组确实为空，`omz-plugins.txt` 为空会让验证停止；人工确认该事实后，可以把对应检查改为只验证文件存在。备份可能含秘密与内部路径，不能进入 dotfiles。

## 3. 逐项决定保留、替代或删除

| 旧能力 | 新去向 | 当场决定什么 |
| --- | --- | --- |
| `ZSH_THEME` 或第三方主题 | Starship | 删除旧主题变量，只初始化一次 Starship |
| OMZ `z`、`zsh-z`、autojump | zoxide | 使用 `z`、`zi`，数据稍后单独导入 |
| fzf 插件或手工按键脚本 | 独立 fzf | 只初始化一次；`Ctrl-R` 与 `Alt-C` 不由 fzf 绑定 |
| 原生 `Ctrl-R` 或历史插件 | Atuin | `Ctrl-R` 交给 Atuin，上方向键保持原生 |
| autosuggestions | Antidote 同名插件 | 正常加载 |
| syntax-highlighting | Antidote 下载，`.zshrc` 最后加载 | 清单用 `kind:clone` |
| `git`、`sudo`、`extract` 等 OMZ 插件 | 选定的 OMZ 子目录或手写能力 | 只保留自己真实使用的部分 |
| 自定义 alias 与 function | `common.zsh`、`macos.zsh`、`linux.zsh` | 去重并按平台拆分 |
| PATH 与登录环境 | `.zshenv`、`.zprofile` | 按读取时机拆分，不整体复制旧 `.zshrc` |
| 代理、令牌和机器目录 | `local.zprofile`、`local.zsh` 或秘密工具 | 不进入 Git |

Oh My Zsh 自动更新变量、`source "$ZSH/oh-my-zsh.sh"`、旧主题变量、重复的 fzf 初始化和旧补全缓存通常不迁移。

## 4. 安装新组件，但不修改旧启动链

只执行当前平台对应的小节。此时不要编辑根目录 `.zshenv`，也不要把 Antidote `source` 到旧 `.zshrc`。

### 4.1 macOS

已有 Homebrew 时直接安装；没有时先下载、阅读并运行官方安装器：

```zsh
if ! command -v brew >/dev/null 2>&1; then
  installer=$(mktemp)
  curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh -o "$installer"
  less "$installer"
  /bin/bash "$installer"
  rm -f "$installer"
fi

if [[ -x /opt/homebrew/bin/brew ]]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [[ -x /usr/local/bin/brew ]]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

brew install stow starship atuin zoxide fzf
# 需要本机图形终端时再执行：
# brew install --cask ghostty
```

### 4.2 Ubuntu

```zsh
sudo apt update
sudo apt install -y zsh git stow curl ca-certificates less
```

安装支持 `fzf --zsh` 的用户级 fzf：

```zsh
fzf_root="${XDG_DATA_HOME:-$HOME/.local/share}"
fzf_dir="$fzf_root/fzf"
fzf_bin="$HOME/.local/bin/fzf"
mkdir -p "$fzf_root" "$HOME/.local/bin"
[[ -d "$fzf_dir/.git" ]] \
  || git clone --depth=1 https://github.com/junegunn/fzf.git "$fzf_dir"
"$fzf_dir/install" --bin

fzf_target_ok=1
if [[ -e "$fzf_bin" || -L "$fzf_bin" ]]; then
  if [[ -f "$fzf_bin" && ! -L "$fzf_bin" && -x "$fzf_bin" ]] \
    && cmp -s "$fzf_dir/bin/fzf" "$fzf_bin"; then
    printf 'reuse existing fzf: %s\n' "$fzf_bin"
  else
    print -u2 "STOP: inspect the existing fzf target: $fzf_bin"
    fzf_target_ok=0
  fi
else
  install -m 0755 "$fzf_dir/bin/fzf" "$fzf_bin"
fi

if (( fzf_target_ok == 1 )); then
  path=("$HOME/.local/bin" $path)
  fzf --version
  fzf --zsh >/dev/null
  unset fzf_bin fzf_dir fzf_root fzf_target_ok
else
  unset fzf_bin fzf_dir fzf_root fzf_target_ok
  false
fi
```

Git 仓库及安装器生成内容留在 XDG data，只有经过检查的 `fzf` 可执行文件复制到 `$HOME/.local/bin`。不要再把 `$HOME/.local/share/fzf/bin` 直接加入 PATH。

安装 Starship、Atuin 与 zoxide；每个脚本都先在 `less` 中审阅：

```zsh
mkdir -p "$HOME/.local/bin"

installer=$(mktemp)
curl -fsSL https://starship.rs/install.sh -o "$installer"
less "$installer"
sh "$installer" -b "$HOME/.local/bin"
rm -f "$installer"

installer=$(mktemp)
curl --proto '=https' --tlsv1.2 -fsSL \
  https://github.com/atuinsh/atuin/releases/latest/download/atuin-installer.sh \
  -o "$installer"
less "$installer"
ATUIN_INSTALL_DIR="$HOME/.local/bin" \
  ATUIN_NO_MODIFY_PATH=1 sh "$installer"
rm -f "$installer"

installer=$(mktemp)
curl -fsSL https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh -o "$installer"
less "$installer"
sh "$installer" --bin-dir "$HOME/.local/bin"
rm -f "$installer"

path=("$HOME/.local/bin" $path)
```

Atuin 与 zoxide 都定向到统一的用户命令目录；安装器不修改旧 Oh My Zsh 启动文件，持久 PATH 只在新 `.zshenv` 中声明。

### 4.3 两个平台共同安装 Antidote

```zsh
antidote_root="${XDG_DATA_HOME:-$HOME/.local/share}"
antidote_dir="$antidote_root/antidote"
mkdir -p "$antidote_root"
[[ -d "$antidote_dir/.git" ]] \
  || git clone --depth=1 https://github.com/mattmc3/antidote.git "$antidote_dir"
test -r "$antidote_dir/antidote.zsh"

for command_name in git stow starship atuin zoxide fzf; do
  if (( $+commands[$command_name] )); then
    printf '%-10s %s\n' "$command_name" "$commands[$command_name]"
  else
    printf '%-10s missing\n' "$command_name"
  fi
done
```

## 5. 建立或接入 dotfiles，不覆盖未知仓库

本文使用 `$HOME/.dotfiles`。先检查状态：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"

if [[ -e "$DOTFILES_DIR" ]]; then
  ls -ld "$DOTFILES_DIR"
  git -C "$DOTFILES_DIR" status --short --branch 2>/dev/null || true
  git -C "$DOTFILES_DIR" remote -v 2>/dev/null || true
fi
```

只选择符合事实的一条路径：

- 目录不存在：创建普通 Git 仓库；
- 已是自己明确维护的 dotfiles：保留现有内容，先检查 `zsh` 软件包是否冲突；
- 目录存在但不是已确认仓库：停止，不删除、不覆盖、不重新初始化。

新建仓库时执行：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"

if [[ ! -e "$DOTFILES_DIR" ]]; then
  mkdir -p "$DOTFILES_DIR"
  git -C "$DOTFILES_DIR" init
fi

git -C "$DOTFILES_DIR" rev-parse --show-toplevel
git -C "$DOTFILES_DIR" status --short --branch
mkdir -p \
  "$DOTFILES_DIR/zsh/.config/zsh" \
  "$DOTFILES_DIR/atuin/.config/atuin" \
  "$DOTFILES_DIR/starship/.config" \
  "$HOME/.config/zsh" \
  "$HOME/.config/atuin"
```

新仓库将以下最小内容保存为 `$DOTFILES_DIR/README.md`；已有仓库则把同样的职责合并到原 README，不覆盖已有说明：

```markdown
# dotfiles

This repository stores reusable terminal configuration sources.

## Packages

- `zsh`: Zsh startup files and Antidote plugin manifest.
- `atuin`: reviewed non-secret Atuin preferences.
- `starship`: minimal cross-machine prompt configuration.

## Deployment

Install the required software, simulate the explicit Stow package list, review
all conflicts, and only then apply the same list.

## Local-only data

Machine-local overrides, shell history, Atuin keys and databases, caches,
plugin clones, logs, and secrets stay outside Git.
```

## 6. 先构建不含根 `.zshenv` 的并行配置

暂时不要创建 `$DOTFILES_DIR/zsh/.zshenv`。这样旧根 `.zshrc` 仍通过 Oh My Zsh 启动，而新配置可以由测试进程显式指定 `ZDOTDIR`。

### 6.1 登录配置

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

### 6.2 插件清单

不依赖 OMZ 别名和函数时，将以下纯净清单保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zsh_plugins.txt`：

```text
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting kind:clone
```

若第 3 节确认仍需要少量 OMZ 子插件，可使用：

```text
getantidote/use-omz
ohmyzsh/ohmyzsh path:lib
ohmyzsh/ohmyzsh path:plugins/git
ohmyzsh/ohmyzsh path:plugins/sudo
ohmyzsh/ohmyzsh path:plugins/extract

zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting kind:clone
```

这会由 Antidote 克隆并加载选中目录，不读取原 `~/.oh-my-zsh`。删除自己不用的 OMZ 子插件，不要把整套框架重新列入清单。

### 6.3 交互配置

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

### 6.4 立即迁移盘点过的能力

- 跨平台 alias 和函数写入有实际内容的 `common.zsh`；
- macOS 专属内容写入 `macos.zsh`，Ubuntu 专属内容写入 `linux.zsh`；
- 登录环境写入 `.zprofile`；
- 非交互命令必须看到的最小 PATH 留给第 8 节 `.zshenv`；
- 代理、私有 SDK 与秘密写入不受 Git 管理的 local 文件。

创建 local 文件并恢复 umask：

```zsh
previous_umask=$(umask)
umask 077
[[ -e "$HOME/.config/zsh/local.zprofile" ]] \
  || : > "$HOME/.config/zsh/local.zprofile"
[[ -e "$HOME/.config/zsh/local.zsh" ]] \
  || : > "$HOME/.config/zsh/local.zsh"
chmod 600 "$HOME/.config/zsh/local.zprofile" "$HOME/.config/zsh/local.zsh"
umask "$previous_umask"
unset previous_umask
```

不复制 `ZSH`、`ZSH_THEME`、`plugins=(...)` 或旧框架 source 行。缺少实际内容时，不创建空的 common 或平台文件。

### 6.5 在并行测试前闭合组件配置

新 `.zshrc` 会调用 Atuin 与 Starship，因此不能只部署 `zsh` 后就运行并行测试。先检查可能已经由旧环境或应用创建的目标：

```zsh
for component_target in \
  "$HOME/.config/atuin/config.toml" \
  "$HOME/.config/starship.toml"; do
  if [[ -e "$component_target" || -L "$component_target" ]]; then
    ls -ld "$component_target"
    readlink "$component_target" 2>/dev/null || true
  fi
done
unset component_target
```

已有普通文件时，使用第 2 节的私密备份，逐项把经过审查的通用设置写入对应仓库源；机器路径、账号和秘密不迁入 Git。对尚无可复用源的组件，Atuin 使用下面统一的 local-first 受管基线，Starship 使用最小启动覆盖配置，分别建立 `$DOTFILES_DIR/atuin/.config/atuin/config.toml` 与 `$DOTFILES_DIR/starship/.config/starship.toml`；已有可信源则保留并审查，不覆盖：

```toml
# Atuin: atuin/.config/atuin/config.toml
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

# 默认不按主机、会话、目录或 workspace 缩小搜索范围；Shell 范围由后面的 [search].shells 决定。
# 进入界面后可用 Ctrl-R 循环 search.filters 中启用的上下文范围；本基线不覆盖该列表。
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

[search]
# 同时检索当前 Zsh、可能导入的旧 Bash，以及旧版 Atuin 没有记录 Shell 的历史。
# 空字符串表示 Shell 未知；显式列出范围，避免默认 auto 在 Zsh 中隐藏 Bash 导入记录。
shells = ["", "bash", "zsh"]

[logs]
# Atuin 当前默认写入 ~/.atuin/logs；日志属于可跨进程保留但不应进入 Git 的本机状态。
dir = "~/.local/state/atuin/logs"
```

Atuin 在平台主线、迁移主线和 [[Atuin 命令历史管理]] 中有意使用同一套最终基线，不存在等待后续替换的另一份“完整配置”。`search_mode = "fuzzy"` 固定文字匹配方式，`filter_mode = "global"` 固定主机、会话、目录等上下文的初始范围，`search.shells` 固定可进入交互搜索的 Shell 来源，`workspaces = true` 只启用 workspace 范围能力。

```toml
# Starship: starship/.config/starship.toml
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

这里的 Starship 内容是“最小启动覆盖配置”：“最小”指覆盖项少；由于没有设置顶层 `format`，其他模块仍遵循 Starship 默认的 `$all` 格式和各自检测条件。已有可信 Starship 源时继续保留并审查，不用这段基线覆盖。迁移完成并形成第 10 节的已知良好提交后，只有明确选择本文推荐布局时，才按 [[Starship 提示符配置#7. 部署三套配置并选择活动配置|Starship 三配置部署流程]] 把 [[Starship 提示符配置#4. 默认配置：用显式 format 限定后端开发模块|第 4 节完整配置]] 写入默认路径，并把第 5、6 节作为两个备用 profile 同时保留；不要在并行测试前替换启动基线，也不要拼接多份完整配置。

这一步只准备配置源，不运行 `atuin info`、`atuin doctor`、`starship explain` 或真实 Zsh。目标普通文件的移动与 Stow 接管统一放在下一节处理。

## 7. 部署并行配置并测试

先检查仓库源：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"
zsh_source_dir="$DOTFILES_DIR/zsh/.config/zsh"

zsh -n "$zsh_source_dir/.zprofile"
zsh -n "$zsh_source_dir/.zshrc"
test -s "$DOTFILES_DIR/atuin/.config/atuin/config.toml"
test -s "$DOTFILES_DIR/starship/.config/starship.toml"
```

因为 `zsh` 软件包还没有 `.zshenv`，它只接管 `.config/zsh` 中的明确源文件；但并行 `.zshrc` 会初始化 Atuin 与 Starship，所以三个软件包必须形成同一个部署事务：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --restow --verbose=2 zsh atuin starship
```

冲突时逐文件比较，不使用 `--adopt`。模拟无冲突后应用：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --restow --verbose=2 zsh atuin starship
```

先确认 `$HOME/.config/atuin/config.toml` 与 `$HOME/.config/starship.toml` 都是指向仓库的链接，再运行并行配置：

```zsh
for managed_path in \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/atuin/config.toml" \
  "$HOME/.config/starship.toml"; do
  [[ -L "$managed_path" ]] || {
    print -u2 "STOP: expected managed link: $managed_path"
    false
  }
done
unset managed_path
```

显式把 `ZDOTDIR` 和 Ubuntu 系统级补全跳过开关只传给测试进程：

```zsh
env skip_global_compinit=1 ZDOTDIR="$HOME/.config/zsh" zsh -lic '
  printf "parallel-config-loaded\n"
  command -v starship atuin zoxide fzf
  (( $+functions[antidote] )) || exit 1
  bindkey "^R"
  type z
  type zi

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
      print -u2 "STOP: generated file leaked into candidate ZDOTDIR: $unexpected_path"
      exit 1
    }
  done
'
```

Ubuntu 的系统级 `/etc/zsh/zshrc` 早于候选用户 `.zshrc` 执行；若并行测试只临时传入 `ZDOTDIR`，裸 `compinit` 可能先在候选目录生成 `.zcompdump`。`skip_global_compinit=1` 只关闭本次测试进程的系统级初始化，候选 `.zshrc` 仍会执行带 `-d` 的补全初始化；没有读取该开关的平台不受影响。

该命令不会修改账号登录 Shell，也不会替换当前 Oh My Zsh 会话，但会真实运行组件初始化并在仓库外创建合法的 data、state 或 cache。执行后再次检查上述受管目标仍是链接，并逐项验证第 3 节决定保留的 alias、函数、SDK 和 PATH；遗漏就回到第 6 节修正。若发现候选目录已有生成文件，先定位旧调用并把确认可重建的文件移入第 2 节备份，不要直接删除后继续。

## 8. 并行测试通过后才切换根 `.zshenv`

将以下内容保存为 `$DOTFILES_DIR/zsh/.zshenv`：

```zsh
# 不导出 ZDOTDIR；子 Zsh 必须重新读取根 .zshenv，才能获得同一组早期启动开关。
ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"

# Ubuntu 的系统级 zshrc 可能先调用 compinit；本配置在用户 .zshrc 中统一初始化并把转储写入 XDG cache。
# 该开关不需要导出；未使用它的平台会忽略这个普通 Shell 参数。
skip_global_compinit=1

typeset -U path PATH
[[ -d /usr/local/bin ]] && path=(/usr/local/bin $path)
[[ -d /opt/homebrew/bin ]] && path=(/opt/homebrew/bin $path)
[[ -d "$HOME/.local/bin" ]] && path=("$HOME/.local/bin" $path)
export PATH
```

把第 3 节确认“SSH 非交互命令也必须看到”的额外共享 PATH 现在加入；主题、插件、别名和 SDK 交互初始化不放进 `.zshenv`。`ZDOTDIR` 只在当前 Zsh 中设置，确保子 Zsh 仍从根 `$HOME/.zshenv` 开始并重新取得 `skip_global_compinit`；后者是必须早于系统级交互启动文件生效的无副作用开关，因此不能延后到 `linux.zsh` 或用户 `.zshrc`。

先检查语法。若旧根 `.zshenv` 是普通文件，把它移动到**第 2 节实际输出且已验证**的备份目录：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"
zsh -n "$DOTFILES_DIR/zsh/.zshenv"

if [[ -e "$HOME/.zshenv" && ! -L "$HOME/.zshenv" ]]; then
  mv "$HOME/.zshenv" "$backup_dir/active-zshenv-before-stow"
fi

stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --restow --verbose=2 zsh atuin starship
```

若模拟出现其他冲突，先把刚移动的旧文件恢复，再查清状态。只有模拟符合预期，才应用同一个软件包：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --restow --verbose=2 zsh atuin starship

managed_links_ok=1
for managed_path in \
  "$HOME/.zshenv" \
  "$HOME/.config/zsh/.zprofile" \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/atuin/config.toml" \
  "$HOME/.config/starship.toml"; do
  if [[ ! -L "$managed_path" ]]; then
    print -u2 "expected managed link: $managed_path"
    managed_links_ok=0
  fi
done

if (( managed_links_ok != 1 )); then
  print -u2 'STOP: managed-link verification failed'
  false
fi
unset managed_links_ok managed_path
```

当前窗口不会立即切换。保留它，另开全新终端或 SSH 连接。

## 9. 验证新会话不再加载完整框架

在全新会话执行：

```zsh
printf 'ZDOTDIR=%s\n' "$ZDOTDIR"
ps -p $$ -o comm=,args=

if (( $+functions[omz] )); then
  print -u2 'unexpected: full Oh My Zsh is still loaded'
  false
else
  print 'ok: full Oh My Zsh framework is not loaded'
fi

antidote list
bindkey '^R'
type z
type zi
starship --version
atuin doctor
atuin config get search.shells --verbose
atuin config get logs.dir --verbose
fzf --version
zsh -lic '
  [[ "$(typeset -p ZDOTDIR)" != export\ * ]] || {
    print -u2 "STOP: ZDOTDIR must not be exported"
    exit 1
  }

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

  print "startup-ok"
'
```

人工确认：

1. `Ctrl-R` 打开 Atuin，上方向键仍逐条浏览原生历史；
2. `Ctrl-T` 打开 fzf，目录交互选择使用 `zi`；
3. 自动建议、语法高亮和 Starship 各只初始化一次；
4. 需要保留的 alias、函数、SDK 和非交互 PATH 都可用；
5. 本机终端、IDE 终端和适用的 SSH 新会话均可进入 Zsh。

若仍出现旧 prompt，搜索新目录：

```zsh
grep -RInE 'oh-my-zsh|ZSH_THEME|oh-my-zsh\.sh' "$HOME/.config/zsh"
```

选择性 OMZ 子插件会使清单出现 `ohmyzsh/ohmyzsh`，这是预期的；新 `.zshrc` 不应再有 `source "$ZSH/oh-my-zsh.sh"`。

## 10. 先提交已知良好配置

新仓库若还没有 README，补充软件包、部署方式和 local/历史/秘密边界。然后审查：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" diff --check
git -C "$DOTFILES_DIR" diff
```

可选地保存经过验证的 Antidote 提交快照：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"
snapshot_source="$DOTFILES_DIR/zsh/.config/zsh/snapshots/antidote.txt"
mkdir -p "${snapshot_source:h}"
antidote snapshot save "$snapshot_source"

stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --restow --verbose=2 zsh
```

确认只增加预期 snapshot 链接后，再执行同一 `--restow` 命令但移除 `--simulate`。

只暂存经过审查的稳定文件：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" add -- README.md zsh atuin starship
git -C "$DOTFILES_DIR" diff --cached --check
git -C "$DOTFILES_DIR" diff --cached
git -C "$DOTFILES_DIR" grep --cached -nEi \
  '(password|passwd|token|secret|private[ _-]?key|api[ _-]?key)' -- . || true
git -C "$DOTFILES_DIR" commit -m "feat: migrate Zsh configuration to Antidote"
git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" log -1 --format=fuller
```

若这是已有 dotfiles，只提交本次明确变更，不要把既有未提交内容据为本次成果。

这个提交是“迁移后能够稳定进入新 Zsh”的恢复点，不是强制统一主题的理由。没有可信旧 Starship 源且明确采用推荐布局时，才继续执行 [[Starship 提示符配置#7. 部署三套配置并选择活动配置|Starship 三配置部署流程]]；已有可信源则先比较其模块、字体和环境变量边界，再决定是否迁移。采用三配置布局后，默认路径保存第 4 节，第 5、6 节进入命名 profile；切换只通过 `STARSHIP_CONFIG`，不改写链接或仓库源。

## 11. 提交之后再导入历史与目录数据

先人工检查并脱敏第 2 节历史副本，再输入脱敏文件路径：

```zsh
printf 'sanitized Zsh history path: '
IFS= read -r sanitized_history

if [[ -f "$sanitized_history" ]]; then
  HISTFILE="$sanitized_history" atuin import zsh
fi
atuin stats
unset sanitized_history
```

这里的副本来自旧 Zsh，因此必须使用 `import zsh`。导入只向现有 Atuin 数据库增加标记为 Zsh 的记录，不覆盖已有历史；本基线同时包含 Zsh、Bash 和未标记 Shell 的记录，所以这些旧命令仍能从新 Zsh 的 `Ctrl-R` 找到。若实际输入来自 Bash，必须改用 `import bash`，不能把同一文件交给两个解析器。

不要把历史、Atuin 数据库或密钥提交到 Git。原来使用 OMZ `z` 或 `zsh-z` 时，可按真实来源选择一次：

```zsh
zoxide import z
# 原来确实使用 zsh-z 时改为：zoxide import zsh-z
```

导入只写 zoxide 数据库，不删除旧数据。自动检测不到时先运行 `zoxide import --help`，不要猜路径后移动数据库。

## 12. 回退到原 Oh My Zsh

从一直保留的旧会话先模拟解除软件包：

```zsh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --delete --verbose=2 zsh atuin starship
```

确认只删除这三个软件包拥有的链接后，移除 `--simulate` 实际解除。然后使用第 2 节仍在当前旧会话中的真实 `backup_dir` 恢复根启动文件与迁移前存在的组件配置：

```zsh
if [[ -f "$backup_dir/active-zshenv-before-stow" ]]; then
  cp -p "$backup_dir/active-zshenv-before-stow" "$HOME/.zshenv"
elif [[ -f "$backup_dir/.zshenv" ]]; then
  cp -p "$backup_dir/.zshenv" "$HOME/.zshenv"
elif [[ -f "$backup_dir/no-original-zshenv" ]]; then
  [[ -e "$HOME/.zshenv" || -L "$HOME/.zshenv" ]] \
    && mv "$HOME/.zshenv" "$HOME/.zshenv.antidote-disabled.$(date +%Y%m%d-%H%M%S)"
fi

[[ -f "$backup_dir/.zprofile" ]] && cp -p "$backup_dir/.zprofile" "$HOME/.zprofile"
[[ -f "$backup_dir/.zshrc" ]] && cp -p "$backup_dir/.zshrc" "$HOME/.zshrc"

mkdir -p "$HOME/.config"
for restore_name in atuin starship.toml starship; do
  restore_source="$backup_dir/$restore_name"
  restore_target="$HOME/.config/$restore_name"
  if [[ -e "$restore_source" || -L "$restore_source" ]]; then
    if [[ -e "$restore_target" || -L "$restore_target" ]]; then
      print -u2 "STOP: restore target still exists: $restore_target"
      false
    fi
    cp -a "$restore_source" "$restore_target"
  fi
done
unset restore_name restore_source restore_target
```

关闭并重新打开终端，或在当前窗口清除继承的 `ZDOTDIR`：

```zsh
exec env -u ZDOTDIR zsh -l
```

该回退不删除 dotfiles 源、local 文件、Antidote 或 Oh My Zsh。若交互启动损坏，可先运行 `zsh -f` 跳过交互配置再修复。

## 13. 何时清理 Oh My Zsh

至少经过覆盖日常开发、Git、IDE 终端和 SSH 的观察期，并确认新配置不再引用 `~/.oh-my-zsh`、必要能力已迁移、历史可访问、备份可读、已知良好提交存在后，才考虑卸载。

卸载步骤和 custom 目录检查见 [[Oh My Zsh 更新、备份与卸载]]。不要直接删除 `~/.oh-my-zsh`；其中可能仍有未迁移的私有插件或函数。

## 进一步理解

- [[Zsh 与 Antidote 跨机器配置管理]]：启动顺序、平台拆分与插件语义。
- [[Starship 提示符配置]]：启动覆盖、默认配置、备用 profile、字体与性能验证。
- [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]]：软件包、冲突、秘密与日常生命周期。
- [[现代终端环境更新、验证与回退]]：迁移后的更新和故障定位。

## 官方参考资料

- [Oh My Zsh：官方安装、配置与卸载说明](https://github.com/ohmyzsh/ohmyzsh)
- [Zsh：启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)
- [Zsh：`compinit` 转储文件与 `-d` 参数](https://zsh.sourceforge.io/Doc/Release/Completion-System.html)
- [Debian/Ubuntu：系统级 `zshrc` 与 `skip_global_compinit`](https://sources.debian.org/src/zsh/5.9-8/debian/zshrc/)
- [Antidote：官方安装、插件清单与 snapshot](https://antidote.sh/)
- [Antidote：选择性使用 Zsh 框架子插件](https://antidote.sh/using-zsh-frameworks)
- [zsh-syntax-highlighting：官方加载顺序说明](https://github.com/zsh-users/zsh-syntax-highlighting)
- [Starship：官方 Zsh 初始化说明](https://starship.rs/guide/)
- [Atuin：导入已有历史](https://docs.atuin.sh/cli/guide/import/)
- [zoxide：导入旧目录数据](https://github.com/ajeetdsouza/zoxide)
- [fzf：Shell integration 与按键禁用](https://github.com/junegunn/fzf)
- [GNU Stow：官方手册](https://www.gnu.org/software/stow/manual/stow.html)

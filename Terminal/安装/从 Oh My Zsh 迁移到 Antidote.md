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
updated: 2026-08-31T13:18:23
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
  "$HOME/.config/starship.toml"; do
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
mkdir -p "$fzf_root"
[[ -d "$fzf_dir/.git" ]] \
  || git clone --depth=1 https://github.com/junegunn/fzf.git "$fzf_dir"
"$fzf_dir/install" --bin
path=("$fzf_dir/bin" $path)
fzf --zsh >/dev/null
```

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
ATUIN_NO_MODIFY_PATH=1 sh "$installer"
rm -f "$installer"

installer=$(mktemp)
curl -fsSL https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh -o "$installer"
less "$installer"
sh "$installer"
rm -f "$installer"

path=("$HOME/.local/bin" "$HOME/.atuin/bin" $path)
```

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

zcompdump="${XDG_CACHE_HOME:-$HOME/.cache}/zsh/zcompdump-$ZSH_VERSION"
mkdir -p "${zcompdump:h}"
autoload -Uz compinit
compinit -d "$zcompdump"
unset zcompdump

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

if (( $+commands[fzf] )) && fzf --zsh >/dev/null 2>&1; then
  FZF_CTRL_R_COMMAND= FZF_ALT_C_COMMAND= source <(fzf --zsh)
fi

(( $+commands[atuin] )) && eval "$(atuin init zsh --disable-up-arrow --disable-ai)"
(( $+commands[zoxide] )) && eval "$(zoxide init zsh)"
(( $+commands[starship] )) && eval "$(starship init zsh)"

if (( $+functions[antidote] )); then
  zsh_highlight_root="$(antidote path zsh-users/zsh-syntax-highlighting 2>/dev/null)"
  if [[ -r "$zsh_highlight_root/zsh-syntax-highlighting.zsh" ]]; then
    source "$zsh_highlight_root/zsh-syntax-highlighting.zsh"
  fi
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

已有普通文件时，使用第 2 节的私密备份，逐项把经过审查的通用设置写入对应仓库源；机器路径、账号和秘密不迁入 Git。对尚无可复用源的组件，使用下面的最小内容分别建立 `$DOTFILES_DIR/atuin/.config/atuin/config.toml` 与 `$DOTFILES_DIR/starship/.config/starship.toml`；已有可信源则保留并审查，不覆盖：

```toml
# Atuin: atuin/.config/atuin/config.toml
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

```toml
# Starship: starship/.config/starship.toml
"$schema" = "https://starship.rs/config-schema.json"

add_newline = false

[hostname]
ssh_only = true
format = "[$hostname]($style) "

[character]
success_symbol = "[>](bold green)"
error_symbol = "[>](bold red)"
```

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

显式把 `ZDOTDIR` 只传给测试进程：

```zsh
env ZDOTDIR="$HOME/.config/zsh" zsh -lic '
  printf "parallel-config-loaded\n"
  command -v starship atuin zoxide fzf
  (( $+functions[antidote] )) || exit 1
  bindkey "^R"
  type z
  type zi
'
```

该命令不会修改账号登录 Shell，也不会替换当前 Oh My Zsh 会话，但会真实运行组件初始化并在仓库外创建合法的 data、state 或 cache。执行后再次检查上述受管目标仍是链接，并逐项验证第 3 节决定保留的 alias、函数、SDK 和 PATH；遗漏就回到第 6 节修正。

## 8. 并行测试通过后才切换根 `.zshenv`

将以下内容保存为 `$DOTFILES_DIR/zsh/.zshenv`：

```zsh
export ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"

typeset -U path PATH
[[ -d /usr/local/bin ]] && path=(/usr/local/bin $path)
[[ -d /opt/homebrew/bin ]] && path=(/opt/homebrew/bin $path)
[[ -d "$HOME/.local/share/fzf/bin" ]] && path=("$HOME/.local/share/fzf/bin" $path)
[[ -d "$HOME/.atuin/bin" ]] && path=("$HOME/.atuin/bin" $path)
[[ -d "$HOME/.local/bin" ]] && path=("$HOME/.local/bin" $path)
export PATH
```

把第 3 节确认“SSH 非交互命令也必须看到”的额外共享 PATH 现在加入；主题、插件、别名和 SDK 交互初始化不放进 `.zshenv`。

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
fzf --version
zsh -lic 'printf "startup-ok\n"'
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
for restore_name in atuin starship.toml; do
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
- [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]]：软件包、冲突、秘密与日常生命周期。
- [[现代终端环境更新、验证与回退]]：迁移后的更新和故障定位。

## 官方参考资料

- [Oh My Zsh：官方安装、配置与卸载说明](https://github.com/ohmyzsh/ohmyzsh)
- [Antidote：官方安装、插件清单与 snapshot](https://antidote.sh/)
- [Antidote：选择性使用 Zsh 框架子插件](https://antidote.sh/using-zsh-frameworks)
- [zsh-syntax-highlighting：官方加载顺序说明](https://github.com/zsh-users/zsh-syntax-highlighting)
- [Starship：官方 Zsh 初始化说明](https://starship.rs/guide/)
- [Atuin：导入已有历史](https://docs.atuin.sh/cli/guide/import/)
- [zoxide：导入旧目录数据](https://github.com/ajeetdsouza/zoxide)
- [fzf：Shell integration 与按键禁用](https://github.com/junegunn/fzf)
- [GNU Stow：官方手册](https://www.gnu.org/software/stow/manual/stow.html)

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
updated: 2026-09-01T14:31:31
---

本文从一台以 Bash 为起点的 Ubuntu 出发，完成 Zsh + Antidote + Starship + Atuin + zoxide + fzf 的安装，同时筛选旧 Bash 配置、从零建立 dotfiles、部署并形成已知良好提交。单看本文即可完成本地或 SSH 主流程。

已有 Oh My Zsh 时改走 [[从 Oh My Zsh 迁移到 Antidote]]；已有可信远端 dotfiles 时改走 [[从已有 dotfiles 恢复现代终端环境]]。生产服务器、共享跳板机和受管主机还必须先确认组织是否允许安装个人软件、部署 dotfiles 和修改登录 Shell。

> [!info] 易变安装事实核对范围
> 本文于 2026-08-30 核对了 Ghostty Linux 包来源、fzf 的 Zsh 集成、zoxide 的 fzf 版本要求，以及 Atuin release 安装器的 PATH 修改开关。这只表示资料已核对，不表示任何具体 Ubuntu 主机已经执行本文命令。

## Desktop 与 Server 的边界

| 场景 | Ghostty | Zsh 与命令行工具 |
| --- | --- | --- |
| Ubuntu Desktop 本地开发 | 可选装在本机 | 安装在同一台机器 |
| Mac 等桌面设备 SSH 到 Ubuntu Server | 只装在本地桌面 | 安装在 Ubuntu Server |
| 无图形桌面的 Ubuntu Server | 不安装 | 安装并重点验证 SSH 新会话 |
| 容器或一次性 CI | 不安装，也通常不改登录 Shell | 只安装任务需要的命令 |

Ghostty 提供本地窗口、键盘输入和终端协议。通过 Ghostty SSH 到服务器时，远端只运行 Shell 和 CLI，不需要再安装图形终端。

## 1. 确认 Bash 起点与三个 Shell 状态

在当前 Bash 会话执行只读检查：

```bash
account_shell=$(getent passwd "$(id -un)" | cut -d: -f7)
printf 'account login shell: %s\n' "$account_shell"
printf 'inherited SHELL: %s\n' "${SHELL-}"
ps -p $$ -o pid=,ppid=,comm=,args=

printf 'XDG_CONFIG_HOME: %s\n' "${XDG_CONFIG_HOME:-not set}"
printf 'ZDOTDIR: %s\n' "${ZDOTDIR:-not set}"
printf 'HISTFILE: %s\n' "${HISTFILE:-not set}"
printf 'SSH connection: %s\n' "${SSH_CONNECTION:-local session}"

command -v bash
bash --version | head -n 1
command -v zsh || true
cat /etc/os-release
uname -m
unset account_shell
```

`getent passwd` 的末字段、继承的 `$SHELL` 和 `ps` 显示的当前进程是三个不同状态。本文直到新配置通过真实启动测试后才执行 `chsh`。

检查已存在的路径：

```bash
for candidate_path in \
  "$HOME/.dotfiles" \
  "$HOME/.profile" \
  "$HOME/.bash_profile" \
  "$HOME/.bash_login" \
  "$HOME/.bashrc" \
  "$HOME/.bash_aliases" \
  "$HOME/.zshenv" \
  "$HOME/.zprofile" \
  "$HOME/.zshrc" \
  "$HOME/.config/zsh"; do
  if [ -e "$candidate_path" ] || [ -L "$candidate_path" ]; then
    ls -ld "$candidate_path"
  fi
done
```

若 `$HOME/.dotfiles` 已存在，本文的“从零”前提不成立。先检查 Git 根、状态和远端，不要重新 `git init` 或覆盖。若这是唯一远程管理会话，再打开一个 SSH 窗口并保持登录，作为恢复入口。

## 2. 建立并验证私密恢复基线

这份基线保留原始 Bash、旧 Zsh、历史和登录 Shell 状态，不是 dotfiles 配置源。当前确实是 Bash 时，先用 `history -a` 把本会话尚未落盘的历史追加到 Bash 历史文件：

```bash
previous_umask=$(umask)
umask 077

backup_dir="$HOME/terminal-backups/ubuntu-before-zsh-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"
printf '%s\n' "$previous_umask" > "$backup_dir/umask-before.txt"

for source_path in \
  "$HOME/.profile" \
  "$HOME/.bash_profile" \
  "$HOME/.bash_login" \
  "$HOME/.bashrc" \
  "$HOME/.bash_aliases" \
  "$HOME/.bash_logout" \
  "$HOME/.inputrc" \
  "$HOME/.zshenv" \
  "$HOME/.zprofile" \
  "$HOME/.zshrc" \
  "$HOME/.config/zsh" \
  "$HOME/.config/atuin" \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/starship"; do
  if [ -e "$source_path" ] || [ -L "$source_path" ]; then
    cp -a "$source_path" "$backup_dir/"
  fi
done

getent passwd "$(id -un)" | cut -d: -f7 > "$backup_dir/login-shell-before.txt"

bash_history_file="$HOME/.bash_history"
if [ -n "${BASH_VERSION:-}" ]; then
  history -a
  bash_history_file="${HISTFILE:-$HOME/.bash_history}"
fi
if [ -f "$bash_history_file" ]; then
  cp -p "$bash_history_file" "$backup_dir/bash-history"
  printf '%s\n' "$bash_history_file" > "$backup_dir/bash-history-source.txt"
fi

if [ -f "$HOME/.zsh_history" ]; then
  cp -p "$HOME/.zsh_history" "$backup_dir/zsh-history"
elif [ -f "$HOME/.zhistory" ]; then
  cp -p "$HOME/.zhistory" "$backup_dir/zsh-history"
fi

umask "$previous_umask"
printf 'backup=%s\n' "$backup_dir"
unset bash_history_file previous_umask source_path
```

不要只相信最后一行。验证权限、登录 Shell 记录、预期副本和 umask：

```bash
backup_verified=1

test -d "$backup_dir" || backup_verified=0
test "$(stat -c '%a' "$backup_dir")" = 700 || backup_verified=0
grep -q '[^[:space:]]' "$backup_dir/login-shell-before.txt" || backup_verified=0
test "$(umask)" = "$(cat "$backup_dir/umask-before.txt")" || backup_verified=0

for original_path in \
  "$HOME/.profile" \
  "$HOME/.bashrc" \
  "$HOME/.bash_aliases" \
  "$HOME/.zshenv" \
  "$HOME/.zshrc" \
  "$HOME/.config/zsh" \
  "$HOME/.config/atuin" \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/starship"; do
  if [ -e "$original_path" ] || [ -L "$original_path" ]; then
    if [ ! -e "$backup_dir/${original_path##*/}" ] \
      && [ ! -L "$backup_dir/${original_path##*/}" ]; then
      printf 'missing backup: %s\n' "$original_path" >&2
      backup_verified=0
    fi
  fi
done

if [ "$backup_verified" -ne 1 ]; then
  printf 'STOP: backup verification failed\n' >&2
  false
fi

printf 'backup verified: %s\n' "$backup_dir"
unset backup_verified original_path
```

备份可能含令牌、内部主机名和敏感命令，必须留在 dotfiles 之外。Bash 启动文件暂时也不删除，供显式启动 Bash和回退使用。

## 3. 安装系统依赖与独立工具

先安装基础依赖：

```bash
sudo apt update
sudo apt install -y zsh git stow curl ca-certificates less

command -v zsh
zsh --version
git --version
stow --version
curl --version
```

Ubuntu LTS 中的 fzf 可能不能满足 `fzf --zsh` 与 zoxide 交互选择的版本要求。主线按 fzf 官方 Git 方式安装到用户目录，并禁止安装器修改 Shell 配置：

```bash
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
```

若 `apt-cache policy fzf` 显示受信任仓库的候选版本满足当前 zoxide 和 `fzf --zsh` 的要求，也可以只使用 APT；同一台机器不要并存两个不清楚优先级的来源。

下载 Starship、Atuin 和 zoxide 的官方安装脚本时，先保存到临时文件并阅读。`less` 中按 `q` 退出：

```bash
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
```

`ATUIN_NO_MODIFY_PATH=1` 阻止安装器改写多个 Shell profile；PATH 由稍后的 dotfiles 统一维护。

安装 Antidote 到 XDG 数据目录：

```bash
antidote_root="${XDG_DATA_HOME:-$HOME/.local/share}"
antidote_dir="$antidote_root/antidote"
mkdir -p "$antidote_root"

if [ ! -d "$antidote_dir/.git" ]; then
  git clone --depth=1 https://github.com/mattmc3/antidote.git "$antidote_dir"
fi

test -r "$antidote_dir/antidote.zsh"
```

当前 Bash 临时补上用户路径并验证；持久路径随后写入 `.zshenv`：

```bash
export PATH="$HOME/.local/bin:$HOME/.atuin/bin:$HOME/.local/share/fzf/bin:$PATH"

starship --version
atuin --version
zoxide --version
fzf --version
```

这些二进制不由 Antidote 管理。安装来源与更新机制必须保持一一对应。

## 4. 初始化 dotfiles 并建立实际软件包

只有第 1 节确认目标不存在时，才执行：

```bash
DOTFILES_DIR="$HOME/.dotfiles"

if [ -e "$DOTFILES_DIR" ]; then
  printf '停止：目标已经存在：%s\n' "$DOTFILES_DIR" >&2
else
  mkdir -p "$DOTFILES_DIR"
  git -C "$DOTFILES_DIR" init
  mkdir -p \
    "$DOTFILES_DIR/zsh/.config/zsh" \
    "$DOTFILES_DIR/atuin/.config/atuin" \
    "$DOTFILES_DIR/starship/.config" \
    "$HOME/.config/zsh" \
    "$HOME/.config/atuin"
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

## Deployment

Install Git, GNU Stow, Zsh, Antidote, and the referenced CLI tools. Run the
same explicit package list with Stow `--simulate --restow` first, review the
output, and then run `--restow` without `--simulate`.

## Local-only data

Machine-local overrides, shell history, Atuin keys and databases, caches,
plugin clones, logs, and secrets stay outside Git.
```

`zsh/.zshenv` 将部署到 `$HOME/.zshenv`，`atuin/.config/atuin/config.toml` 将部署到 `$HOME/.config/atuin/config.toml`。这就是 Stow 软件包“内部镜像目标目录”的规则。三个软件包共同构成 `.zshrc` 的受管配置闭包，必须一起部署后才能验证真实 Zsh。

## 5. 建立独立于旧 Bash 的 Zsh 与组件配置基线

旧 Bash 配置因机器而异，但 Zsh 的启动入口、原生历史、插件加载和工具初始化是这套环境固定需要的基线。由于 `.zshrc` 会初始化 Atuin 和 Starship，本节也先建立它们的受管配置源；下一节才有依据判断哪些 Bash 行为已经被替代、哪些仍需迁移。

本节只建立目标结构，不宣称旧配置已经迁移完成。`.zshrc` 对 `common.zsh`、`linux.zsh` 和 local 文件都使用“存在且可读才加载”的判断，因此共享与平台文件可以等到真正出现第一项保留内容时再创建。

### 5.1 写入最小 `.zshenv`

将以下内容保存为 `$DOTFILES_DIR/zsh/.zshenv`：

```zsh
# 不导出 ZDOTDIR；子 Zsh 必须重新读取根 .zshenv，才能获得同一组早期启动开关。
ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"

# Ubuntu 的系统级 zshrc 可能先调用 compinit；本配置在用户 .zshrc 中统一初始化并把转储写入 XDG cache。
# 该开关不需要导出；未使用它的平台会忽略这个普通 Shell 参数。
skip_global_compinit=1

# 所有 Zsh 都读取 .zshenv，只保留必须提前生效的启动开关和非交互命令也需要的 PATH。
typeset -U path PATH
[[ -d /usr/local/bin ]] && path=(/usr/local/bin $path)
[[ -d /opt/homebrew/bin ]] && path=(/opt/homebrew/bin $path)
[[ -d "$HOME/.local/share/fzf/bin" ]] && path=("$HOME/.local/share/fzf/bin" $path)
[[ -d "$HOME/.atuin/bin" ]] && path=("$HOME/.atuin/bin" $path)
[[ -d "$HOME/.local/bin" ]] && path=("$HOME/.local/bin" $path)
export PATH
```

这里故意不导出 `ZDOTDIR`。若父 Zsh 把它传给子进程，子 Zsh 会直接寻找 `$ZDOTDIR/.zshenv`；本布局没有该文件，于是根 `$HOME/.zshenv` 及其中非导出的 `skip_global_compinit` 都会被绕过。Ubuntu 的系统级 `/etc/zsh/zshrc` 随后可能在用户 `.zshrc` 之前执行裸 `compinit`，先把默认 `.zcompdump` 写入 `$ZDOTDIR`。先用只读命令核对当前软件包实际提供的开关与调用；本基线仍由用户 `.zshrc` 执行带 `-d` 的唯一一次 `compinit`：

```sh
grep -nE 'skip_global_compinit|compinit' /etc/zsh/zshrc
```

这些路径对应第 3 节已经安装的工具。此时不要顺手加入旧 Bash 中的其他 PATH；下一节会先判断它是否真的需要影响所有 Zsh，包括 SSH 非交互命令。系统级启动文件、用户文件与补全初始化的先后关系见 [[Zsh 与 Antidote 跨机器配置管理]]。

### 5.2 写入共享 `.zprofile` 和本机登录入口

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

共享源文件只提供跨机器的登录入口。机器路径、账号、代理或秘密以后写入真实目标 `$HOME/.config/zsh/local.zprofile`，不进入 Git。先建立这个私有入口并收紧权限：

```bash
previous_umask=$(umask)
umask 077

# : 是 Shell 内置命令，是空操作，用于增加可读性，从功能上可省略
test -e "$HOME/.config/zsh/local.zprofile" \
  || : > "$HOME/.config/zsh/local.zprofile"
chmod 600 "$HOME/.config/zsh/local.zprofile"

umask "$previous_umask"
unset previous_umask
```

### 5.3 写入插件清单与交互式 `.zshrc`

将以下内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zsh_plugins.txt`：

```text
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting kind:clone
```

再将以下内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zshrc`：

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

这份 `.zshrc` 已经明确接管 Zsh 原生历史、补全、Antidote 插件、fzf、Atuin、zoxide 和 Starship。下一节审阅 Bash 历史、提示符或补全配置时，应先与这里的现有能力比较，而不是再复制一套。

### 5.4 建立本机交互入口

机器私有的交互配置写入真实目标 `$HOME/.config/zsh/local.zsh`，不进入 Git：

```bash
previous_umask=$(umask)
umask 077

test -e "$HOME/.config/zsh/local.zsh" || : > "$HOME/.config/zsh/local.zsh"
chmod 600 "$HOME/.config/zsh/local.zsh"

umask "$previous_umask"
unset previous_umask
```

### 5.5 写入 Atuin 的受管配置

将以下统一的 local-first 受管基线保存为 `$DOTFILES_DIR/atuin/.config/atuin/config.toml`。它也是平台主线和迁移主线共用的最终基线；匹配、过滤、导入与同步的完整边界见 [[Atuin 命令历史管理]]：

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
```

此处先写仓库源，不运行 `atuin info`、`atuin doctor` 或 `atuin init`。这些命令会加载设置，并可能在配置缺失时创建普通 `config.toml`；第 7 节先让 Stow 拥有目标路径。

### 5.6 写入 Starship 的受管配置

将以下内容保存为 `$DOTFILES_DIR/starship/.config/starship.toml`。这是一份“最小启动覆盖配置”：“最小”指只覆盖少量选项，不表示提示符只会显示这里出现的模块。由于它没有设置顶层 `format`，目录、Git、语言版本等其他模块仍按 Starship 默认的 `$all` 格式和各自检测条件出现；本阶段只先建立能解析、能回退的受管源：

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

这是一份可运行的启动基线，而不是模块白名单或最终个性化主题。到这里仍不要启动真实 Zsh；旧 Bash 中的 PATH、SDK、别名、函数和本机设置还必须经过下一节的逐项判断。完成第 10 节的已知良好提交后，再按 [[Starship 提示符配置#7. 部署三套配置并选择活动配置|Starship 三配置部署流程]] 把 [[Starship 提示符配置#4. 默认配置：用显式 format 限定后端开发模块|第 4 节完整配置]] 写入默认路径，并把第 5、6 节作为两个备用 profile 同时保留；不要在首次验收前替换启动基线，也不要拼接多份顶层 `format` 或同名 TOML 表。完整启动顺序和扩展规则见 [[Zsh 与 Antidote 跨机器配置管理]]。

## 6. 逐项审阅并迁移旧 Bash 行为

迁移单位是“仍想保留的行为”，不是 Bash 文件或代码块。一个 `.bashrc` 可能同时包含历史、提示符、Linux 命令、跨平台别名和机器秘密，它们不会进入同一个 Zsh 文件。

### 6.1 确认 Bash 实际读取链与个人改动

交互式登录 Bash 会在 `.bash_profile`、`.bash_login`、`.profile` 中读取第一个存在且可读的文件，不会自动依次读取三个文件；入口文件仍可能通过 `source` 或 `.` 显式加载其他文件。交互式非登录 Bash 通常读取 `.bashrc`，`.bash_aliases` 只有被 `.bashrc` 等入口显式加载才会生效。

先确定登录入口，再依次完整阅读存在的候选文件。`less` 中按 `q` 退出当前文件，随后会打开下一个文件：

```bash
login_entry=""
for startup_file in \
  "$HOME/.bash_profile" \
  "$HOME/.bash_login" \
  "$HOME/.profile"; do
  if [ -r "$startup_file" ]; then
    login_entry="$startup_file"
    break
  fi
done

printf 'Bash login entry: %s\n' "${login_entry:-not found}"

for startup_file in \
  "$HOME/.bash_profile" \
  "$HOME/.bash_login" \
  "$HOME/.profile" \
  "$HOME/.bashrc" \
  "$HOME/.bash_aliases" \
  "$HOME/.bash_logout" \
  "$HOME/.inputrc"; do
  [ -f "$startup_file" ] || continue
  printf '\nreading: %s\n' "$startup_file"
  less -N "$startup_file"
done

unset login_entry startup_file
```

阅读入口中的 `source` 或 `.` 时继续打开其真实目标；不能只凭文件名判断一段配置是否生效。还可以把 Ubuntu 当前用户文件与 `/etc/skel` 中的新用户模板比较，用于定位后来增加或修改的内容：

```bash
for base_name in .profile .bashrc; do
  current_file="$HOME/$base_name"
  skeleton_file="/etc/skel/$base_name"
  if [ -f "$current_file" ] && [ -f "$skeleton_file" ]; then
    printf '\ncomparing: %s\n' "$base_name"
    diff -u "$skeleton_file" "$current_file" || true
  fi
done

unset base_name current_file skeleton_file
```

没有差异只能说明它与**当前**模板相同；模板可能在账号创建后更新，因此这是一条定位线索，不是“没有个人配置”的绝对证明。反过来，来自默认模板的行为也可以按实际需要选择保留，不能机械删除。

在私密工作记录中为每一项候选行为登记以下信息。不要把可能含账号、主机名、代理或秘密的原始配置复制进公开笔记或 dotfiles：

| 来源与行号 | 想保留的行为 | 生效时机 | 共享范围 | 新落点或决定 | 验证方法 |
| --- | --- | --- | --- | --- | --- |
| 例如：`.bashrc` 中的 `ll` | 显示详细文件列表 | 交互式 | 跨平台 | `common.zsh` | 新 Zsh 中运行 `alias ll` 与 `ll` |

### 6.2 迁移登录环境

从前一步输出的 `Bash login entry` 文件开始，并继续追踪它显式加载的文件。每遇到一项 `export`、PATH 修改或环境初始化，按下面的顺序判断并立即编辑目标文件：

1. 这个行为是否仍然需要？如果只是旧工具残留，明确放弃，不迁移。
2. 它必须让所有 Zsh 都生效，还是只在登录 Shell 中需要？不要因为内容是 PATH 就一律放进 `.zshenv`。
3. 它能否跨机器共享？机器路径、账号、代理和秘密只能进入 local 文件。
4. `.profile` 是否还要服务桌面会话或其他 POSIX Shell？需要时保留原配置，并为 Zsh 单独表达同一行为。

| 行为的真实范围 | 新落点 | 处理原则 |
| --- | --- | --- |
| 所有 Zsh，包括适用的 SSH 非交互命令都必须看到的最小 PATH | `$DOTFILES_DIR/zsh/.zshenv` | 使用目录存在判断；不运行插件、提示符或输出文字 |
| 跨机器共享的登录环境 | `$DOTFILES_DIR/zsh/.config/zsh/.zprofile` | 改写为 Zsh 兼容语法；不整体 `source ~/.profile` |
| 机器私有的登录环境 | `$HOME/.config/zsh/local.zprofile` | 权限保持为 `600`，不进入 Git |
| 仍需服务桌面会话或 POSIX Shell 的环境 | 保留在 `.profile` | Zsh 同样需要时，在合适的 Zsh 文件中单独表达，不删除原入口 |
| Bash 专属、失效或已经不再需要的初始化 | 不迁移 | 在迁移记录中写明原因 |

每写入一项就对实际修改的目标运行语法检查；全部登录环境处理完后再统一检查一次：

```bash
DOTFILES_DIR="$HOME/.dotfiles"

zsh -n "$DOTFILES_DIR/zsh/.zshenv"
zsh -n "$DOTFILES_DIR/zsh/.config/zsh/.zprofile"
zsh -n "$HOME/.config/zsh/local.zprofile"
```

语法通过只表示 Zsh 可以解析，不表示 PATH、SDK 或环境变量已经在真实新会话中生效。为每一项同时记录 `command -v`、变量输出或工具自检命令，留到第 8 节运行。

### 6.3 迁移交互式别名、函数和工具初始化

接着审阅 `.bashrc` 以及它实际加载的 `.bash_aliases`。先问“想保留什么行为”，再决定 Zsh 表达和落点：

| Bash 中的行为 | 新落点或决定 |
| --- | --- |
| 跨 macOS、Linux 的交互别名或函数 | 有第一项有效内容时创建 `common.zsh` |
| 依赖 Linux 路径、GNU 命令或 Ubuntu 行为的交互配置 | 有第一项有效内容时创建 `linux.zsh` |
| 机器路径、代理、账号、秘密或仅本机需要的函数 | `$HOME/.config/zsh/local.zsh` |
| `HISTCONTROL`、`HISTSIZE`、`shopt -s histappend` 等 Bash 历史设置 | 不复制；先与第 5 节的 Zsh 历史和 Atuin 配置比较 |
| `PS1`、`PROMPT_COMMAND` | 不复制；提示符已由 Starship 接管 |
| Bash `complete`、`bash-completion` 初始化 | 不直接复制；Zsh 已使用 `compinit`，再按具体工具确认是否需要 Zsh 补全 |
| `bind` 或 Readline 键位 | 不复制到 `.zshrc`；Zsh 使用 `bindkey`，而 `.inputrc` 仍可服务其他 Readline 程序 |
| Bash 数组、`shopt` 或其他 Bash 专属语法 | 不逐字复制；先确认能力，再寻找 Zsh 等价实现 |

不要整体 `source ~/.bashrc`，也不要为了保留一条别名复制整段 Ubuntu 默认配置。`common.zsh` 与 `linux.zsh` 只有在出现第一项确定保留的内容时才创建；没有相应行为时，让 `.zshrc` 的可读性判断自然跳过它们。

每次编辑后对实际存在的目标做增量语法检查：

```bash
DOTFILES_DIR="$HOME/.dotfiles"
zsh_source_dir="$DOTFILES_DIR/zsh/.config/zsh"

for migrated_source in \
  "$zsh_source_dir/common.zsh" \
  "$zsh_source_dir/linux.zsh" \
  "$HOME/.config/zsh/local.zsh"; do
  [ -f "$migrated_source" ] && zsh -n "$migrated_source"
done

unset migrated_source zsh_source_dir
```

语法检查之后仍要为每个别名、函数或 SDK 初始化记录真实的新 Zsh 验证命令。例如，别名既要用 `alias NAME` 确认定义，也要运行一次它负责的正常操作。

### 6.4 明确保留但不迁移的 Bash 文件

- `.bash_logout` 属于 Bash 登录 Shell 的退出流程，默认不迁移到 Zsh；保留它供显式 Bash 会话和回退使用。
- `.inputrc` 是 GNU Readline 配置，不是 Bash 到 Zsh 的一对一迁移源。继续使用 Bash 或其他 Readline 程序时可以保留它；Zsh 键位需求单独用 `bindkey` 表达。
- `.profile`、`.bashrc` 和 `.bash_aliases` 在迁移完成后也暂不删除。它们既是回退依据，也可能继续被显式 Bash 或其他会话读取。

### 6.5 关闭迁移清单

进入部署前，逐行确认私密迁移记录满足以下条件：

1. 每项候选行为都有“迁移、由 Zsh 基线替代、继续留在 Bash、明确放弃”四类结论之一；
2. 迁移项已经写入真实目标，不是只有未来落点；
3. 每个修改过的 Zsh 文件都通过了 `zsh -n`；
4. 每项行为都有准备在第 8 节执行的验证命令；
5. local 文件、历史、秘密和机器身份均未进入 dotfiles。

只有清单已经关闭，才进入下一节统一检查仓库源并部署。后文不再重新决定旧配置的落点。

## 7. 检查仓库源，再让 Stow 接管

先做语法检查：

```bash
DOTFILES_DIR="$HOME/.dotfiles"
zsh_source_dir="$DOTFILES_DIR/zsh/.config/zsh"

zsh -n "$DOTFILES_DIR/zsh/.zshenv"
zsh -n "$zsh_source_dir/.zprofile"
zsh -n "$zsh_source_dir/.zshrc"
test -s "$DOTFILES_DIR/atuin/.config/atuin/config.toml"
test -s "$DOTFILES_DIR/starship/.config/starship.toml"

for optional_source in \
  "$zsh_source_dir/common.zsh" \
  "$zsh_source_dir/linux.zsh" \
  "$HOME/.config/zsh/local.zprofile" \
  "$HOME/.config/zsh/local.zsh"; do
  [ -f "$optional_source" ] && zsh -n "$optional_source"
done
```

模拟部署：

```bash
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --restow --verbose=2 zsh atuin starship
```

若出现冲突，对照第 2 节备份、当前目标和仓库源。只移动确认由新配置替代的明确文件；不要使用 `--adopt` 自动改写仓库，也不要移动整个 `$HOME/.config`。

模拟无冲突后，应用完全相同的软件包列表：

```bash
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --restow --verbose=2 zsh atuin starship
```

检查关键目标都是链接，local 文件仍是真实文件：

```bash
managed_links_ok=1
for managed_path in \
  "$HOME/.zshenv" \
  "$HOME/.config/zsh/.zprofile" \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/zsh/.zsh_plugins.txt" \
  "$HOME/.config/atuin/config.toml" \
  "$HOME/.config/starship.toml"; do
  test -L "$managed_path" || {
    printf 'expected symlink: %s\n' "$managed_path" >&2
    managed_links_ok=0
    continue
  }
  ls -ld "$managed_path"
  readlink "$managed_path"
done

if [ "$managed_links_ok" -ne 1 ]; then
  printf 'STOP: managed-link verification failed\n' >&2
  false
fi
unset managed_links_ok managed_path

test -f "$HOME/.config/zsh/local.zprofile" \
  && test ! -L "$HOME/.config/zsh/local.zprofile"
test -f "$HOME/.config/zsh/local.zsh" \
  && test ! -L "$HOME/.config/zsh/local.zsh"
```

## 8. 先验证真实 Zsh，再修改账号

保留当前 Bash 和备用 SSH 会话。用已部署入口启动一个新的登录交互 Zsh：

```bash
zsh -lic '
  expected_zdotdir="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"
  printf "ZDOTDIR=%s\n" "$ZDOTDIR"
  [[ "$ZDOTDIR" == "$expected_zdotdir" ]] || exit 1
  [[ "$(typeset -p ZDOTDIR)" != export\ * ]] || {
    print -u2 "STOP: ZDOTDIR must not be exported"
    exit 1
  }
  command -v starship atuin zoxide fzf
  (( $+functions[antidote] )) || exit 1
  bindkey "^R"

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

这一步经过 `.zshenv`、系统级与用户级启动文件、`.zprofile`、`.zshrc`、平台文件和本机 local 文件，也会首次真正执行 Atuin、Starship、zoxide、fzf 与 Antidote 初始化。它还同时证明补全转储和 Antidote 静态加载脚本位于 XDG cache，且默认生成文件没有泄漏到 `$ZDOTDIR`。还要逐项运行第 6 节为旧 PATH、SDK、别名和函数准备的验证命令。

若出现 `generated file leaked into ZDOTDIR`，先不要直接删除文件。分别检查 `/etc/zsh/zshrc`、受管 `.zshrc` 与 common/platform/local 文件中是否还有裸 `compinit` 或单参数 `antidote load`；修正唯一调用路径后，把确认可重建的旧文件移动到第 2 节备份，再重新启动验证。

真实初始化后立即复查受管配置没有被应用替换成普通文件：

```bash
for managed_path in \
  "$HOME/.zshenv" \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/atuin/config.toml" \
  "$HOME/.config/starship.toml"; do
  test -L "$managed_path" || {
    printf 'STOP: managed target changed ownership: %s\n' "$managed_path" >&2
    false
  }
done

git -C "$HOME/.dotfiles" status --short --branch
```

Atuin 数据库、Antidote 插件和缓存、Zsh 历史可以在仓库外生成；它们不应改变上述链接，也不应出现在 dotfiles 工作树中。

全部通过后，才检查 Zsh 是否是允许的登录 Shell并修改当前账号：

```bash
zsh_path=$(command -v zsh)

# /etc/shell 保存系统认可的登录 Shell 白名单
if ! grep -Fx "$zsh_path" /etc/shells >/dev/null; then
  printf 'STOP: Zsh is not listed in /etc/shells: %s\n' "$zsh_path" >&2
  false
fi

chsh -s "$zsh_path"
getent passwd "$(id -un)"
unset zsh_path
```

不要使用 `sudo chsh` 修改其他账户。`chsh` 不会把当前 Bash 进程变成 Zsh；必须重新登录验证。

## 9. 本地与 SSH 新会话验收

在新终端或新 SSH 连接中执行：

```zsh
printf 'account login shell: '
getent passwd "$(id -un)" | cut -d: -f7
printf 'inherited SHELL=%s\n' "$SHELL"
ps -p $$ -o comm=,args=
printf 'ZDOTDIR=%s\n' "$ZDOTDIR"

antidote list
bindkey '^R'
starship --version
atuin doctor
zoxide --version
fzf --version
zsh -lic 'printf "startup-ok\n"'
```

交互验收包括：`Ctrl-R` 打开 Atuin；自动建议与语法高亮正常；`z` 和 `zi` 可导航；`Ctrl-T` 可选择文件；Starship 能在 Git 仓库显示状态。

Ubuntu Server 还应从另一台机器检查非交互路径：

```bash
printf 'SSH destination: '
IFS= read -r SSH_DESTINATION
ssh "$SSH_DESTINATION" '
  printf "shell=%s ZDOTDIR=%s\n" "$SHELL" "$ZDOTDIR"
  command -v zsh starship atuin zoxide fzf
'
unset SSH_DESTINATION
```

SSH 远程命令通常不读取 `.zshrc`，但 Zsh 会读取 `.zshenv`；因此它应找到 CLI，却不应加载提示符和按键插件。纯本地 Desktop 不使用 SSH 时，将此项明确记为不适用。

## 10. 审查并提交已知良好 dotfiles

真实新登录通过后，审查工作树和秘密边界：

```bash
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" diff --check
git -C "$DOTFILES_DIR" diff
```

只暂存稳定源文件：

```bash
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" add -- README.md zsh atuin starship
git -C "$DOTFILES_DIR" diff --cached --check
git -C "$DOTFILES_DIR" diff --cached
git -C "$DOTFILES_DIR" grep --cached -nEi \
  '(password|passwd|token|secret|private[ _-]?key|api[ _-]?key)' -- . || true
git -C "$DOTFILES_DIR" commit -m "feat: establish modern terminal dotfiles"
git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" log -1 --format=fuller
```

第 2 节备份、local 文件、历史、Atuin 密钥、插件 clone 和缓存都不能出现在 `git ls-files` 中。需要远端时，在托管平台先创建空仓库，再交互读取实际 URL 并推送：

```bash
DOTFILES_DIR="$HOME/.dotfiles"
printf 'dotfiles remote URL: '
IFS= read -r REPO_URL
git -C "$DOTFILES_DIR" remote add origin "$REPO_URL"
git -C "$DOTFILES_DIR" remote -v
git -C "$DOTFILES_DIR" push -u origin "$(git -C "$DOTFILES_DIR" branch --show-current)"
```

上面的提交是首次启动、链接所有权与 Bash 迁移都已验证的恢复点。只有它存在后，才按 [[Starship 提示符配置#7. 部署三套配置并选择活动配置|Starship 三配置部署流程]] 建立最终提示符布局：默认路径改为第 4 节完整配置，第 5、6 节分别写入命名 profile，并以同一个 `starship` package 重新模拟、部署和验证。把这次个性化作为后续独立变更审查；切换 profile 本身不应产生新的 Git diff。

## 11. 已知良好提交之后再迁移历史

原始历史更可能包含临时令牌、主机名和运维参数。不要直接导入备份：先复制到备份目录之外的临时工作副本，人工删除敏感行并确认权限，再输入脱敏副本路径。

```bash
printf 'sanitized Bash history path: '
IFS= read -r sanitized_bash_history

if [ -f "$sanitized_bash_history" ]; then
  HISTFILE="$sanitized_bash_history" atuin import bash
fi

printf 'sanitized Zsh history path, or leave empty: '
IFS= read -r sanitized_zsh_history

if [ -n "$sanitized_zsh_history" ] && [ -f "$sanitized_zsh_history" ]; then
  HISTFILE="$sanitized_zsh_history" atuin import zsh
fi

atuin stats
git -C "$HOME/.dotfiles" status --short --branch
unset sanitized_bash_history sanitized_zsh_history
```

两种导入器解析不同格式，同一文件不能交给两个解析器，也不要重复导入。Atuin 注册和同步始终可选，详见 [[Atuin 命令历史管理]]。

## 12. 可选：Ubuntu Desktop 安装 Ghostty

服务器跳过本节。Ghostty 官方目前直接发布 macOS 二进制；Linux 包可能由发行版或社区维护者构建。Ubuntu Desktop 应在 [Ghostty 安装页](https://ghostty.org/docs/install/binary) 核对当前系统可用来源、维护者和风险，优先受信任的软件仓库。

没有合适来源时继续使用 GNOME Terminal，Zsh 与 CLI 不受影响。不要从博客复制不明的 `curl | bash`，也不要为了终端外观以 root 身份运行未知脚本。需要维护 Ghostty 配置时再打开 [[Ghostty 常用配置与 Shell 集成]]。

## 最终完成标准

1. 原始 Bash/Zsh 配置与历史有通过权限和文件检查的私密备份；
2. 每项旧 Bash 候选行为都有迁移、由新基线替代、继续保留或明确放弃的结论，没有悬空清单；
3. Zsh、Atuin 与 Starship 的受管目标在首次真实启动前已经由 Stow 部署，启动后仍指向 `$HOME/.dotfiles`；local 文件是真实私有文件，补全与插件生成文件只位于 XDG cache；
4. 新登录与适用的 SSH 非交互场景都通过，账号登录 Shell、`$SHELL` 和当前进程的差异已理解；
5. dotfiles 至少有一个已知可用提交，历史与同步操作发生在该基线之后；
6. 最终三配置布局中的默认文件和两个备用 profile 都由 Stow 管理且能独立解析；未设置 `STARSHIP_CONFIG` 时使用第 4 节，切换动作不改写链接或仓库源，这次扩展已作为启动基线之后的独立变更审阅。

需要更新、排障或回退时打开 [[现代终端环境更新、验证与回退]]；需要深入理解启动文件、平台拆分或插件顺序时打开 [[Zsh 与 Antidote 跨机器配置管理]]。

## 官方参考资料

- [Ubuntu Server：软件包管理](https://ubuntu.com/server/docs/how-to/software/package-management/)
- [Ubuntu Manpage：`chsh` 修改登录 Shell](https://manpages.ubuntu.com/manpages/noble/man1/chsh.1.html)
- [GNU Bash：启动文件](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html)
- [GNU Bash：历史记录机制](https://www.gnu.org/software/bash/manual/html_node/Bash-History-Facilities.html)
- [Zsh：启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)
- [Zsh：`compinit` 转储文件与 `-d` 参数](https://zsh.sourceforge.io/Doc/Release/Completion-System.html)
- [Debian/Ubuntu：系统级 `zshrc` 与 `skip_global_compinit`](https://sources.debian.org/src/zsh/5.9-8/debian/zshrc/)
- [GNU Stow：官方手册](https://www.gnu.org/software/stow/manual/stow.html)
- [Antidote：官方安装与使用说明](https://antidote.sh/)
- [Starship：官方安装指南](https://starship.rs/guide/)
- [Atuin：release 二进制安装器](https://github.com/atuinsh/atuin/releases/latest/download/atuin-installer.sh)
- [Atuin：导入已有 Shell 历史](https://docs.atuin.sh/main/reference/import/)
- [zoxide：官方安装与 fzf 版本要求](https://github.com/ajeetdsouza/zoxide)
- [fzf：官方安装与 Shell integration](https://github.com/junegunn/fzf)
- [Ghostty：预编译二进制与 Linux 包来源说明](https://ghostty.org/docs/install/binary)

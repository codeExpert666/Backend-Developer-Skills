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
updated: 2026-08-30T21:03:27
---

本文在 Ubuntu 上搭建 Zsh + Antidote + Starship + Atuin + zoxide + fzf，并说明 Ubuntu Desktop 与远程 Ubuntu Server 对 Ghostty 的不同处理。可复用配置进入普通 Git dotfiles 仓库，由 GNU Stow 部署到 `$HOME`；配置目录和加载顺序与 [[macOS 从零搭建现代终端环境]] 保持一致，便于跨机器维护和恢复。

请先阅读 [[现代终端环境搭建概览]] 和 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]]。已有 Oh My Zsh 的机器应改走 [[从 Oh My Zsh 迁移到 Antidote]]；生产服务器、共享跳板机和受管主机还应先确认软件安装、个人 dotfiles 和登录 Shell 变更策略。

> [!info] 易变安装事实核对范围
> 本文于 2026-08-30 重新核对 Ghostty Linux 包来源、fzf 的 Zsh 集成、zoxide 的最低 fzf 版本，以及 Atuin release 安装器的 PATH 修改开关。该状态只表示官方资料已核对，不表示任何具体 Ubuntu 主机已经执行或验证本文命令。

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

## 1. 确认 Bash 起点、账户登录 Shell 与当前进程

全新 Ubuntu 普通用户通常从 Bash 开始，本文的目标是在保留回退入口的前提下，把后续新登录切换到 Zsh，而不是同时维护两套交互配置。切换前标记为 `bash` 的命令可以直接在当前 Bash 会话执行；标记为 `zsh` 的内容用于 Zsh 配置文件或明确启动的 Zsh 进程。

先执行只读检查：

~~~bash
account_shell="$(getent passwd "$(id -un)" | cut -d: -f7)"
printf 'account login shell: %s\n' "$account_shell"
printf 'session SHELL variable: %s\n' "$SHELL"
ps -p $$ -o pid=,comm=
printf 'XDG_CONFIG_HOME: %s\n' "${XDG_CONFIG_HOME:-not set}"
printf 'ZDOTDIR: %s\n' "${ZDOTDIR:-not set}"
printf 'HISTFILE: %s\n' "${HISTFILE:-not set}"
command -v bash
bash --version | head -n 1
command -v zsh || true
zsh --version 2>/dev/null || true
cat /etc/os-release
uname -m
printf 'SSH connection: %s\n' "${SSH_CONNECTION:-local session}"
unset account_shell
~~~

账户数据库中的登录 Shell、当前会话继承的 `$SHELL` 和正在解释命令的进程不是同一个概念：`getent passwd` 的末字段表示账户下次登录应启动什么，`$SHELL` 是当前会话继承的环境变量，`ps` 显示此刻的 Shell 进程。刚执行 `chsh` 后，旧会话中的后两者不会自动变成 Zsh，必须用新终端或新 SSH 登录验证。

当前环境变量没有设置，不代表启动文件中没有自定义路径：Bash 可能没有读取 Zsh 的启动文件，未导出的变量也不会出现在当前环境。备份前再只读检查常见入口中是否显式设置了 `XDG_CONFIG_HOME`、`ZDOTDIR` 或 `HISTFILE`：

~~~bash
for startup_file in \
  "$HOME/.profile" \
  "$HOME/.bash_profile" \
  "$HOME/.bash_login" \
  "$HOME/.bashrc" \
  "$HOME/.zshenv" \
  "$HOME/.zprofile" \
  "$HOME/.zshrc"
do
  [ -f "$startup_file" ] || continue
  printf '\n[%s]\n' "$startup_file"
  grep -nE '(^|[[:space:]])(export[[:space:]]+)?(XDG_CONFIG_HOME|ZDOTDIR|HISTFILE)=' \
    "$startup_file" || true
done
unset startup_file
~~~

如果这里发现配置或历史实际位于其他目录，应先确认那个目录的职责，再把明确的文件加入下一节备份清单；不要因为变量指向一个上层目录就递归复制整个 `$HOME`。

若账户登录 Shell 与当前进程都是 Bash，这是本文预期的常见起点；若已经是 Zsh，仍应继续备份两类配置，避免遗漏以前使用 Bash 时留下的环境设置。若输出为其他 Shell、为空或与组织账户策略不符，应先查明账户来源和登录方式，不要直接执行后面的 `chsh`。

如果当前是唯一的远程管理会话，先再打开一个 SSH 窗口并保持登录。后续任何配置错误都应能从备用会话恢复；不要在尚未测试新登录的情况下退出全部连接。

## 2. 建立并验证私密恢复基线

这份备份用于比较和回退，不是 dotfiles 配置源，也不等于已经完成 Bash 到 Zsh 的迁移。以下命令先把当前 Bash 会话尚未写入文件的历史追加到其原历史文件，再分别复制存在的 Bash、Zsh 配置和历史。备份可能包含主机名、代理、内部路径或敏感命令，因此先用 `umask 077` 收紧新建目录和清单的权限，并在完成后恢复原来的 umask。

如果第 1 节发现了非默认配置目录或历史文件，应先把经过确认的具体路径同时加入下面的复制清单和验证清单。不要把不明确的上层目录直接加入递归复制。

~~~bash
backup_dir="$HOME/.terminal-backups/$(date +%Y%m%d-%H%M%S)"
previous_umask="$(umask)"
umask 077
mkdir -p "$backup_dir"
printf '%s\n' "$previous_umask" > "$backup_dir/umask-before.txt"

for file in \
  "$HOME/.profile" \
  "$HOME/.bash_profile" \
  "$HOME/.bash_login" \
  "$HOME/.bashrc" \
  "$HOME/.bash_aliases" \
  "$HOME/.bash_logout" \
  "$HOME/.inputrc" \
  "$HOME/.zshenv" \
  "$HOME/.zprofile" \
  "$HOME/.zshrc"
do
  [ -f "$file" ] && cp -p "$file" "$backup_dir/"
done

for directory in "$HOME/.config/zsh" "$HOME/.config/ghostty" "$HOME/.config/atuin"; do
  [ -d "$directory" ] && cp -R "$directory" "$backup_dir/"
done

account_shell="$(getent passwd "$(id -un)" | cut -d: -f7)"
printf '%s\n' "$account_shell" > "$backup_dir/login-shell-before.txt"

bash_history_file="$HOME/.bash_history"
if [ -n "${BASH_VERSION:-}" ]; then
  history -a
  bash_history_file="${HISTFILE:-$HOME/.bash_history}"
fi
if [ -f "$bash_history_file" ]; then
  cp -p "$bash_history_file" "$backup_dir/bash-history"
  printf '%s\n' "$bash_history_file" > "$backup_dir/bash-history-source.txt"
fi

zsh_history_file=""
if [ -n "${ZSH_VERSION:-}" ] && [ -n "${HISTFILE:-}" ]; then
  zsh_history_file="$HISTFILE"
elif [ -f "$HOME/.zhistory" ]; then
  zsh_history_file="$HOME/.zhistory"
elif [ -f "$HOME/.zsh_history" ]; then
  zsh_history_file="$HOME/.zsh_history"
fi
if [ -n "$zsh_history_file" ] && [ -f "$zsh_history_file" ]; then
  cp -p "$zsh_history_file" "$backup_dir/zsh-history"
  printf '%s\n' "$zsh_history_file" > "$backup_dir/zsh-history-source.txt"
fi

umask "$previous_umask"
printf 'backup created: %s\n' "$backup_dir"
unset account_shell bash_history_file previous_umask zsh_history_file
~~~

`history -a` 只在当前进程确实是 Bash 时执行，用于把本会话新增命令追加到 Bash 的 `$HISTFILE`；它不会把 Bash 历史转换为 Zsh 格式。若当前不是 Bash，则只复制默认的 `~/.bash_history`（如果存在）。Zsh 历史只在当前进程确实是 Zsh 时采用其 `$HISTFILE`，否则按常见文件名寻找，避免把 Bash 的 `$HISTFILE` 错记成 Zsh 历史。

最后一行出现 `backup created` 只说明命令运行到了打印位置，不能证明此前每次复制都成功。继续之前必须验证目录权限、登录 Shell 记录、原始配置副本、历史副本和 umask 恢复状态：

~~~bash
backup_verified=1

[ -d "$backup_dir" ] || backup_verified=0
[ "$(stat -c '%a' "$backup_dir" 2>/dev/null)" = 700 ] || backup_verified=0
grep -q '[^[:space:]]' "$backup_dir/login-shell-before.txt" 2>/dev/null \
  || backup_verified=0
[ "$(umask)" = "$(cat "$backup_dir/umask-before.txt" 2>/dev/null)" ] \
  || backup_verified=0

for original_path in \
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
  "$HOME/.config/ghostty" \
  "$HOME/.config/atuin"
do
  if [ -e "$original_path" ] \
    && [ ! -e "$backup_dir/${original_path##*/}" ]; then
    printf 'missing backup copy: %s\n' "$original_path" >&2
    backup_verified=0
  fi
done

if [ -s "$backup_dir/bash-history-source.txt" ] \
  && [ ! -f "$backup_dir/bash-history" ]; then
  printf 'missing Bash history backup\n' >&2
  backup_verified=0
fi
if [ -s "$backup_dir/zsh-history-source.txt" ] \
  && [ ! -f "$backup_dir/zsh-history" ]; then
  printf 'missing Zsh history backup\n' >&2
  backup_verified=0
fi

if [ "$backup_verified" -eq 1 ]; then
  printf 'backup verified: %s\n' "$backup_dir"
  unset backup_verified original_path
else
  printf 'STOP: backup verification failed; do not continue the migration\n' >&2
  false
fi
~~~

记录通过验证的 `backup_dir`。启用 `ZDOTDIR` 后，原来的 `~/.zshrc` 会暂时不被读取，但先不要删除它；Bash 启动文件也应保留，供显式启动 Bash、比较迁移内容和回退使用。

备份目录必须留在 dotfiles 仓库之外：不要在其中执行 `git init`，也不要把历史、代理、令牌或机器私有配置复制进稍后建立的仓库。只有看到 `backup verified`，才进入下一节安装基础依赖。

## 3. 安装系统基础依赖

通过 APT 安装 Zsh、Git、GNU Stow、curl、证书和后文审阅安装脚本所需的 `less`：

~~~bash
sudo apt update
sudo apt install -y zsh git stow curl ca-certificates less
~~~

确认路径和版本：

~~~bash
command -v zsh
zsh --version
git --version
stow --version
curl --version
~~~

## 4. 建立或接入 dotfiles Git 工作区

第 2 节的私密备份已经提供原始配置回退；从这一节开始，dotfiles 仓库才负责保存经过筛选、能够跨机器复用的配置源。根据实际状态只选择一条路径：

1. 第一次建立 dotfiles：执行 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库#4. 初始化普通仓库与 package 目录|初始化普通仓库]]，目前只创建仓库根目录和 README，不预建空 package；
2. 已经有可信远端：按 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库#11. 在新机器恢复|新机器恢复流程]]完成 clone 和 Git 现场检查，但暂不执行 Stow 部署，部署留到第 9 节；
3. `DOTFILES_DIR` 已经存在但来源不明：只读检查目录、Git 根和远端，停止后查清所有者；不得删除、覆盖或重新 `git init`。

本文继续用 `$HOME/.dotfiles` 作为示例；若选择了其他位置，只修改 `DOTFILES_DIR`。完成所选路径后统一验证：

~~~bash
DOTFILES_DIR="${DOTFILES_DIR:-$HOME/.dotfiles}"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" rev-parse --show-toplevel
git -C "$DOTFILES_DIR" branch --show-current
git -C "$DOTFILES_DIR" remote -v
~~~

刚执行 `git init` 的仓库没有远端时，`git remote -v` 没有输出是正常现象。建立 Git 工作区也不等于已经形成可回退版本：第一个“已知良好”提交必须等配置完成真实登录与 SSH 验收后再创建。无论仓库是否私有，都不能把第 2 节的原始备份或命令历史复制进来。

## 5. 形成迁移清单与配置落点

在仓库已经有明确根目录后，再逐项阅读 `.profile`、`.bash_profile`、`.bash_login`、`.bashrc` 和 `.bash_aliases`。先按读取时机区分登录、交互和非交互 Shell，再按跨平台、Linux 专属和机器私有确定落点：

- 登录 Shell 所需且可共享的 PATH、SDK 和环境变量写入 `.zprofile`；仅适用于 Linux 的共享登录逻辑在 `.zprofile` 内做平台判断，机器私有或含敏感信息的登录环境写入不入库的 `local.zprofile`；
- 只在交互 Shell 中需要的别名、函数和工具初始化按作用范围拆分：跨平台内容进入 `common.zsh`，Linux 专属内容进入 `linux.zsh`，机器私有内容进入 `local.zsh`；
- Bash 的 `shopt`、`PROMPT_COMMAND`、Readline 按键和其他专属语法不直接复制，也不从 `.zshrc` 整体 `source ~/.bashrc`；
- `.profile` 还可能被桌面会话或其他登录路径使用，完成 Zsh 验证不代表可以删除它。

本节只形成“保留、改写、放入哪个文件或不迁移”的清单，不直接改写当前生效配置。可共享内容在第 7 节写入仓库源；机器私有内容在第 9 节 Stow 部署后写入真实 `local` 文件。只有每一项需要保留的能力都在新 Zsh 会话中验证，才算完成迁移。

后文主路线采用未设置 `XDG_CONFIG_HOME` 时的默认 `~/.config`，并让 `ZDOTDIR` 指向 `~/.config/zsh`。如果第 1 节发现当前机器使用其他路径，应先按 [[现代终端环境搭建概览#先理解 XDG 基础目录规范|XDG 与 ZDOTDIR 目录模型]]决定是迁回默认布局，还是一致调整仓库源、Stow 目标和全部验证路径；不能把两套布局混用后继续照抄命令。

## 6. 安装独立命令行组件与 Antidote

Ubuntu 版本之间的软件仓库差异较大。下面先安装 fzf，再从各项目官方地址下载安装器；下载的脚本先保存到临时文件并阅读，不让安装器接管 Shell 配置。

### fzf

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

### Antidote

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

末行的 `test -r` 用退出状态确认 Antidote 入口文件对当前用户可读，条件命令见 [[Shell 脚本阅读基础#6. 使用 test 表达条件|test 条件判断]]。

不要把 Antidote 安装目录、插件缓存或生成的 `.zsh_plugins.zsh` 提交到配置仓库。跨机器只同步 Zsh 配置、插件清单和可选 snapshot。

## 7. 建立 `zsh` package 并写入迁移配置

第 4 节已经建立或接入 dotfiles 工作区。先确认仓库状态，并检查 `zsh` package 是否已经由远端或以前的配置提供：

~~~bash
DOTFILES_DIR="${DOTFILES_DIR:-$HOME/.dotfiles}"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" ls-files -- zsh
~~~

如果第二条命令已经列出受跟踪的 Zsh 配置，先阅读并按第 5 节迁移清单逐项比较；不要用本文骨架覆盖已有事实来源。确认这是第一次建立 `zsh` package 时，才创建配置源，并预建用于容纳符号链接和本机覆盖的真实目标目录：

~~~bash
DOTFILES_DIR="${DOTFILES_DIR:-$HOME/.dotfiles}"
zsh_source_dir="$DOTFILES_DIR/zsh/.config/zsh"

mkdir -p "$zsh_source_dir" "$HOME/.config/zsh"
touch "$DOTFILES_DIR/zsh/.zshenv"
touch "$zsh_source_dir/.zprofile"
touch "$zsh_source_dir/.zshrc"
touch "$zsh_source_dir/.zsh_plugins.txt"
~~~

这里不创建仓库中的 `local.zprofile` 或 `local.zsh`。它们是部署后位于 `$HOME/.config/zsh` 的本机真实文件，不能成为符号链接源。

### 配置 `~/.zshenv`

将下列内容保存到 `$DOTFILES_DIR/zsh/.zshenv`。若 `~/.zshenv` 原本存在，先按 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库#7. 安全接管已经存在的配置|安全接管已有配置]] 对照备份、选择内容并处理 Stow 冲突：

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

将下列跨平台内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zprofile`；Stow 部署后的运行时目标才是 `~/.config/zsh/.zprofile`：

~~~zsh
# macOS 登录 Shell 会执行；Ubuntu 上因文件不存在而自然跳过。
if [[ -x /opt/homebrew/bin/brew ]]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [[ -x /usr/local/bin/brew ]]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

[[ -r "$ZDOTDIR/local.zprofile" ]] && source "$ZDOTDIR/local.zprofile"
~~~

### 声明插件与 `.zshrc`

将下列内容保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zsh_plugins.txt`：

~~~text
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting kind:clone
~~~

`.zsh_plugins.txt` 是 Antidote 的插件声明清单，不是直接执行的 Shell 脚本。每行先写 GitHub 仓库的 `owner/repo`；稍后的 `.zshrc` 执行 `antidote load "$ZDOTDIR/.zsh_plugins.txt"` 时，Antidote 会下载缺少的仓库，并根据行尾标注决定如何加载：

- `zsh-autosuggestions` 根据历史记录给出输入建议，也可配置为参考补全结果。它没有额外标注，因此使用默认的 `kind:zsh`，由 Antidote 下载并加载；
- `zsh-syntax-highlighting` 在输入命令时进行语法高亮，帮助执行前发现错误。`kind:clone` 表示这里只由 Antidote 下载、不自动加载；稍后的 `.zshrc` 会等其他交互组件初始化完成后再最后 `source` 它，满足该插件的加载顺序要求。

这里只保留两个职责明确的插件。新增插件意味着在每个交互式 Zsh 中执行新的第三方代码，应先确认实际需要并审查其仓库与加载要求。

再将下列骨架保存为 `$DOTFILES_DIR/zsh/.config/zsh/.zshrc`；代码注释仍使用运行时目标路径，便于理解 Zsh 实际读取什么：

~~~zsh
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

# Antidote 本体放在 XDG data；可重新下载的插件仓库与管理缓存放在 XDG cache。
export ANTIDOTE_HOME="${XDG_CACHE_HOME:-$HOME/.cache}/antidote"
antidote_dir="${XDG_DATA_HOME:-$HOME/.local/share}/antidote"

# 入口可读时把 Antidote 管理函数加载到当前 Shell，再处理共享插件清单。
if [[ -r "$antidote_dir/antidote.zsh" ]]; then
  source "$antidote_dir/antidote.zsh"

  # 在自动建议插件加载前设置低亮度显示样式。
  ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=8'

  # 下载缺失的插件仓库并加载普通插件；kind:clone 项只下载，留待后文手动加载。
  antidote load "$ZDOTDIR/.zsh_plugins.txt"
else
  # 本体缺失时只向标准错误报告，保留无插件但仍可使用的基础 Zsh。
  print -u2 "Antidote is missing: $antidote_dir"
fi

# 只清理本体路径临时变量；ANTIDOTE_HOME 继续供 Antidote 的后续命令使用。
unset antidote_dir

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
~~~

写完骨架后回到第 5 节迁移清单，逐项落实已经决定保留的内容：第一条跨平台别名或函数进入仓库源树的 `common.zsh`，第一项 Ubuntu 专属设置进入 `linux.zsh`；没有实际内容时不创建对应文件，前面的可读性判断会自然跳过。机器私有的登录环境和交互配置暂时不写进仓库，等第 9 节部署后再分别写入真实的 `local.zprofile` 和 `local.zsh`。

不要为了目录看起来完整而创建空配置，也不要把第 2 节备份整体复制进 package。完成当前轮写入后检查 Git 现场，确认只有预期的配置源和初始化 README：

~~~bash
git -C "$DOTFILES_DIR" status --short --branch
~~~

## 8. 检查仓库源并隔离测试

先解析 `zsh` package 中已经存在的配置源，包括第 5 节迁入的可选平台文件：

~~~bash
DOTFILES_DIR="${DOTFILES_DIR:-$HOME/.dotfiles}"
zsh_source_dir="$DOTFILES_DIR/zsh/.config/zsh"

source_layout_ok=1
for required_source in \
  "$DOTFILES_DIR/zsh/.zshenv" \
  "$zsh_source_dir/.zprofile" \
  "$zsh_source_dir/.zshrc" \
  "$zsh_source_dir/.zsh_plugins.txt"
do
  if [ ! -f "$required_source" ]; then
    printf 'missing required source: %s\n' "$required_source" >&2
    source_layout_ok=0
  fi
done

config_syntax_ok=1
for config_source in \
  "$DOTFILES_DIR/zsh/.zshenv" \
  "$zsh_source_dir/.zprofile" \
  "$zsh_source_dir/.zshrc" \
  "$zsh_source_dir/common.zsh" \
  "$zsh_source_dir/macos.zsh" \
  "$zsh_source_dir/linux.zsh"
do
  [ -f "$config_source" ] || continue
  if ! zsh -n "$config_source"; then
    config_syntax_ok=0
    break
  fi
done

if [ "$source_layout_ok" -eq 1 ] && [ "$config_syntax_ok" -eq 1 ]; then
  printf 'repository-source syntax verified\n'
  unset config_source config_syntax_ok required_source source_layout_ok zsh_source_dir
else
  printf 'STOP: repository-source syntax verification failed\n' >&2
  false
fi
~~~

语法全部通过后，不修改账户登录 Shell，直接把仓库中的配置目录交给一个新的测试进程：

~~~bash
DOTFILES_DIR="${DOTFILES_DIR:-$HOME/.dotfiles}"

env ZDOTDIR="$DOTFILES_DIR/zsh/.config/zsh" zsh -lic '
  echo "repository-source config loaded"
  command -v antidote starship atuin zoxide fzf
'
~~~

首次加载时 Antidote 会访问 GitHub 并克隆插件。这个测试只证明 `.zprofile`、`.zshrc`、插件和可共享迁移内容能够从仓库源目录加载；它通过环境变量绕过了根目录 `.zshenv`，因此不能证明真实的 `~/.zshenv → ZDOTDIR` 引导链。引导链必须在下一节部署后单独通过验证。

## 9. 用 Stow 接管并验证真实启动链

先模拟部署 `zsh` package。已有目标发生冲突时必须回到 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库#7. 安全接管已经存在的配置|安全接管已有配置]] 完成备份、比较和内容选择，不使用 `--adopt` 跳过判断：

~~~bash
DOTFILES_DIR="${DOTFILES_DIR:-$HOME/.dotfiles}"

stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --verbose=2 zsh
~~~

确认模拟输出只涉及预期路径且没有 conflict 后，再执行实际部署并检查关键链接：

~~~bash
DOTFILES_DIR="${DOTFILES_DIR:-$HOME/.dotfiles}"

stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --verbose=2 zsh

managed_links_ok=1
for managed_path in \
  "$HOME/.zshenv" \
  "$HOME/.config/zsh/.zprofile" \
  "$HOME/.config/zsh/.zshrc"
do
  if [ -L "$managed_path" ]; then
    printf 'managed link: %s -> %s\n' "$managed_path" "$(readlink "$managed_path")"
  else
    printf 'expected managed link: %s\n' "$managed_path" >&2
    managed_links_ok=0
  fi
done

if [ "$managed_links_ok" -eq 1 ]; then
  unset managed_links_ok managed_path
else
  printf 'STOP: managed-link verification failed\n' >&2
  false
fi
~~~

部署后创建权限收紧的本机覆盖，不覆盖已经存在的文件，并恢复执行前的 umask：

~~~bash
previous_umask="$(umask)"
umask 077
[ -e "$HOME/.config/zsh/local.zprofile" ] || : > "$HOME/.config/zsh/local.zprofile"
[ -e "$HOME/.config/zsh/local.zsh" ] || : > "$HOME/.config/zsh/local.zsh"
umask "$previous_umask"
unset previous_umask
~~~

把第 5 节确认的机器私有登录环境和交互配置分别写入这两个真实文件；它们不经过仓库，也不能包含在稍后的 Git 暂存内容中。然后仍不执行 `chsh`，用已部署的真实入口启动一个新的登录 Zsh：

~~~bash
zsh -lic '
  expected_zdotdir="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"
  printf "deployed ZDOTDIR: %s\n" "$ZDOTDIR"
  [[ "$ZDOTDIR" == "$expected_zdotdir" ]] || {
    print -u2 "unexpected ZDOTDIR: $ZDOTDIR"
    exit 1
  }
  command -v antidote starship atuin zoxide fzf
  bindkey "^R"
'
~~~

这一步同时经过 `~/.zshenv`、`ZDOTDIR`、`.zprofile`、`.zshrc` 和本机 `local` 文件。只有它成功，才允许修改账户登录 Shell。

## 10. 固化验证入口后再切换登录 Shell

第一次手工部署和真实启动链都成功后，按 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库#8. 固化可重复的部署和验证入口|dotfiles 部署与验证入口]] 保存 `scripts/deploy` 和 `scripts/verify`，再执行同一 package 的模拟与最低验证：

~~~bash
DOTFILES_DIR="${DOTFILES_DIR:-$HOME/.dotfiles}"

"$DOTFILES_DIR/scripts/deploy" --simulate zsh
"$DOTFILES_DIR/scripts/verify"
~~~

两项都通过后，才检查 Zsh 是否属于系统允许的登录 Shell，并修改当前用户的账户设置：

~~~bash
zsh_path="$(command -v zsh)"
if ! grep -Fx "$zsh_path" /etc/shells >/dev/null; then
  printf 'STOP: Zsh is not listed in /etc/shells: %s\n' "$zsh_path" >&2
  false
else
  chsh -s "$zsh_path"
  account_shell_after="$(getent passwd "$(id -un)" | cut -d: -f7)"
  if [ "$account_shell_after" = "$zsh_path" ]; then
    printf 'account login shell changed: %s\n' "$account_shell_after"
    unset account_shell_after zsh_path
  else
    printf 'STOP: account login shell was not changed: %s\n' \
      "$account_shell_after" >&2
    false
  fi
fi
~~~

不要使用 `sudo chsh` 修改其他账户。`chsh` 只改变后续登录应启动什么，不会把当前 Bash 进程变成 Zsh；完成后保留当前会话，再从另一个终端或 SSH 窗口验证新登录。

## 11. 本地与 SSH 验收

本地新开终端，或者从客户端重新建立 SSH 连接，执行：

~~~zsh
DOTFILES_DIR="${DOTFILES_DIR:-$HOME/.dotfiles}"

"$DOTFILES_DIR/scripts/verify"
git -C "$DOTFILES_DIR" status --short --branch
printf 'account login shell: '
getent passwd "$(id -un)" | cut -d: -f7
printf 'session SHELL variable: %s\n' "$SHELL"
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

Ubuntu Server、远程开发机或其他需要 SSH 的场景，还应从另一台机器验证非交互 PATH；纯本地 Desktop 不使用 SSH 时可把这一项明确记为不适用。执行时将示例主机替换为真实地址：

~~~bash
ssh user@example-host '
  printf "shell=%s ZDOTDIR=%s\n" "$SHELL" "$ZDOTDIR"
  command -v zsh starship atuin zoxide fzf
'
~~~

SSH 远程命令通常不会读取 `.zshrc`，但会经过 Zsh 的 `.zshenv`，因此应能找到这些二进制；提示符、按键和插件只在交互会话加载。再打开一个交互会话，验证 `Ctrl-R`、自动建议、语法高亮、`z`、`zi` 和 `Ctrl-T`，并逐项确认第 5 节决定保留的 PATH、SDK、别名与函数确实可用。

若 Ghostty 客户端中的 `$TERM` 在远端缺少 terminfo，不要在服务器的启动文件中永久伪造 `TERM`。应按 [[Ghostty 常用配置与 Shell 集成]] 检查当前 Ghostty 稳定版支持的 `ssh-env` 与 `ssh-terminfo` 功能，或使用其安全回退。

## 12. 审查并保存已验证配置状态

只有第 11 节的真实新登录以及适用的 SSH 场景通过后，当前配置才有资格成为已知良好状态。先检查未跟踪文件和差异：

~~~bash
DOTFILES_DIR="${DOTFILES_DIR:-$HOME/.dotfiles}"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" diff --check
git -C "$DOTFILES_DIR" diff
~~~

然后按 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库#9. 首次提交与可选远端|首次提交流程]]只暂存经过审查的 README、`zsh` package 和脚本，检查暂存 diff 与秘密边界后再提交。第一次建立的仓库由此形成第一个已知良好提交；从远端恢复的仓库只有在本机确实产生了应共享的配置变化时才创建新提交。远端仍是可选项，不是本地 Git 成立的前提。

第 2 节备份、`local.zprofile`、`local.zsh` 和历史数据都不能出现在 `git ls-files` 中。后续更新、快照和回退见 [[现代终端环境更新、验证与回退]]。

## 13. 按来源导入 Bash 与 Zsh 历史

已知良好配置保存后，先阅读第 2 节备份的历史副本，排除不应进入 Atuin 的敏感命令。再把下列占位目录替换为实际备份目录，并按原始 Shell 格式选择导入器：

~~~bash
backup_dir="$HOME/.terminal-backups/YYYYMMDD-HHMMSS"
if [ -f "$backup_dir/bash-history" ]; then
  HISTFILE="$backup_dir/bash-history" atuin import bash
fi
if [ -f "$backup_dir/zsh-history" ]; then
  HISTFILE="$backup_dir/zsh-history" atuin import zsh
fi
atuin stats
git -C "${DOTFILES_DIR:-$HOME/.dotfiles}" status --short --branch
~~~

`atuin import bash` 和 `atuin import zsh` 解析的是不同历史格式；同一文件不能为了“多导入一些”而交给两种导入器，也不要反复执行同一导入块。新骨架已把 Zsh 的原生 `HISTFILE` 切到 XDG state，因此这里显式指定旧文件副本；切换后直接运行 `atuin import auto` 会根据当前 `$SHELL` 选择 Zsh，并可能只读到新的历史文件，从而遗漏原来的 Bash 历史。导入不会删除备份，也不应改变 dotfiles Git 工作区。服务器命令更可能包含临时令牌、主机名或运维参数，应先阅读 [[Atuin 命令历史管理]] 的过滤规则；注册账户和跨机器同步始终是可选项。

## 14. 可选：Ubuntu Desktop 安装 Ghostty

核心 Shell 与 CLI 已经形成已知良好状态后，Ubuntu Desktop 用户可以再评估 Ghostty；Ubuntu Server 直接跳过本节。Ghostty 项目目前只直接发布 macOS 预编译二进制，Linux 包由发行版或社区维护者构建。Ghostty 官方安装页会列出当前可用的 Linux 包，并明确区分发行版包和风险更高的社区二进制。

推荐顺序是：

1. 优先检查当前 Ubuntu 版本和已配置的受信任仓库是否提供 Ghostty；
2. 在 Ghostty 官方“Binaries and Packages”页面核对该包由谁构建与维护；
3. 只有明确接受维护与供应链风险时，才采用官方页面列出的社区 Ubuntu 包；
4. 不从博客复制不明的一行 `curl | bash`，也不为了终端外观以 root 身份运行未知脚本；
5. 没有合适来源时继续使用 GNOME Terminal，命令行部分完全不受影响。

本文不固定抄写社区 Ubuntu 安装脚本，因为它不是 Ghostty 官方构建产物，维护方式也可能变化。需要跨机器维护 Ghostty 配置时，再按 [[Ghostty 常用配置与 Shell 集成]] 创建、模拟和验证独立 package；不要把可选 GUI 安装重新混入已经验证的基础 Zsh 提交。

## 官方参考资料

- [Ubuntu Server：软件包管理](https://ubuntu.com/server/docs/how-to/software/package-management/)
- [Ubuntu Server：命令行与 Shell 环境变量](https://ubuntu.com/server/docs/tutorial/cli-in-depth/)
- [Ubuntu Manpage：`chsh` 修改登录 Shell](https://manpages.ubuntu.com/manpages/noble/man1/chsh.1.html)
- [Ghostty：预编译二进制与 Linux 包来源说明](https://ghostty.org/docs/install/binary)
- [Ghostty：Linux 平台说明](https://ghostty.org/docs/linux)
- [GNU Bash：启动文件](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html)
- [GNU Bash：历史记录机制](https://www.gnu.org/software/bash/manual/html_node/Bash-History-Facilities.html)
- [Zsh：启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)
- [Antidote：官方安装与使用说明](https://antidote.sh/)
- [Starship：官方安装指南](https://starship.rs/guide/)
- [Atuin：官方安装说明](https://docs.atuin.sh/cli/guide/installation/)
- [Atuin：完整 setup 脚本源码](https://github.com/atuinsh/atuin/blob/main/install.sh)
- [Atuin：release 二进制安装器](https://github.com/atuinsh/atuin/releases/latest/download/atuin-installer.sh)
- [Atuin：导入已有 Shell 历史](https://docs.atuin.sh/main/reference/import/)
- [zoxide：官方安装与 fzf 版本要求](https://github.com/ajeetdsouza/zoxide)
- [fzf：官方安装与 Shell integration](https://github.com/junegunn/fzf)
- [GNU Stow：官方手册](https://www.gnu.org/software/stow/manual/stow.html)

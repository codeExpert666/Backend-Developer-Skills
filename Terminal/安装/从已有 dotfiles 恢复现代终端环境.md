---
title: 从已有 dotfiles 恢复现代终端环境
aliases:
  - 新机器恢复现代终端环境
  - 从远端 dotfiles 恢复终端配置
tags:
  - Terminal
  - Terminal/安装
  - Dotfiles
  - GNU-Stow
  - Zsh
created: 2026-08-30T22:24:37
updated: 2026-08-31T13:18:23
---

这条主线适用于：dotfiles 已经推送到可信远端，现在要在一台新 macOS 或 Ubuntu 上恢复同一套终端环境。单独阅读本文即可完成依赖安装、克隆、冲突处理、部署、验证和回退。

> [!warning]
> 本文不会创建新仓库，也不会把新机器上的文件自动吸收到旧仓库。若还没有远端 dotfiles，请改走 [[macOS 从零搭建现代终端环境]] 或 [[Ubuntu 从零搭建现代终端环境]]。

> [!info] 安装资料核对范围
> 本文于 2026-08-30 核对了 Zsh、Antidote、Atuin、fzf、zoxide 与 Ghostty 的官方资料。资料核对不等于某台新机器已经恢复成功。

## 1. 明确恢复输入与成功标准

开始前应知道：

- 可信的远端仓库 URL；
- 希望恢复的分支，以及最好能记录一个已知可用提交；
- 仓库约定的目录，本文使用 `$HOME/.dotfiles`；
- 当前机器要部署哪些 Stow package，例如 `zsh`、`starship`、`atuin`，桌面机器还可能有 `ghostty`；
- 哪些本机文件、密钥和历史本来就不在 Git 中。

成功不只是 `git clone` 返回 0。还需要证明：目标链接来自预期仓库、Zsh 启动链可用、组件能执行、本机私有数据仍在 Git 外，并且仓库工作树没有因启动 Shell 而产生意外修改。

## 2. 只读检查新机器

先确认系统、账号登录 Shell、当前进程和可能存在的配置：

```sh
uname -a
printf 'HOME=%s\nSHELL=%s\n' "$HOME" "${SHELL-}"
ps -p $$ -o pid=,ppid=,comm=,args=

if command -v getent >/dev/null 2>&1; then
  getent passwd "$USER"
else
  dscl . -read "/Users/$USER" UserShell 2>/dev/null || true
fi

for candidate_path in \
  "$HOME/.dotfiles" \
  "$HOME/.zshenv" \
  "$HOME/.zprofile" \
  "$HOME/.zshrc" \
  "$HOME/.config/zsh"; do
  if [ -e "$candidate_path" ] || [ -L "$candidate_path" ]; then
    ls -ld "$candidate_path"
  fi
done
```

`$SHELL` 通常是继承的登录 Shell 路径，不等于当前进程。`ps` 才观察当前进程；`getent` 或 `dscl` 观察账号记录。

若 `$HOME/.dotfiles` 已存在，先检查而不是删除或覆盖：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

if [ -e "$DOTFILES_DIR" ]; then
  ls -ld "$DOTFILES_DIR"
  git -C "$DOTFILES_DIR" status --short --branch 2>/dev/null || true
  git -C "$DOTFILES_DIR" remote -v 2>/dev/null || true
  git -C "$DOTFILES_DIR" log -1 --oneline 2>/dev/null || true
fi
```

只有确认该目录不存在时，后文才能克隆。若它已经是正确仓库，不要重新克隆；先查清本地改动、分支和远端，再决定是否继续。

## 3. 给原有配置建立私密基线

即使是“新机器”，也可能已有系统或用户配置。先把存在的目标保存到一个只有当前用户可访问的目录：

```sh
previous_umask=$(umask)
umask 077

backup_dir="$HOME/terminal-backups/restore-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"

for source_path in \
  "$HOME/.zshenv" \
  "$HOME/.zprofile" \
  "$HOME/.zshrc" \
  "$HOME/.config/zsh" \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/atuin" \
  "$HOME/.config/ghostty"; do
  if [ -e "$source_path" ] || [ -L "$source_path" ]; then
    cp -a "$source_path" "$backup_dir/"
  fi
done

printf 'backup=%s\n' "$backup_dir"
find "$backup_dir" -maxdepth 2 -ls
umask "$previous_umask"
```

备份目录不进入 dotfiles。若源是符号链接，`cp -a` 会保留链接本身；需要恢复内容时，应同时确认它原来指向哪里。

## 4. 安装平台依赖

只执行与当前系统匹配的小节。Ghostty 只安装在有桌面的本机；SSH 服务器不需要终端模拟器。

### 4.1 macOS

先检查 Homebrew：

```sh
command -v brew || true
```

若尚未安装，先下载并阅读官方安装器：

```sh
installer=$(mktemp)
curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh -o "$installer"
less "$installer"
/bin/bash "$installer"
rm -f "$installer"
```

按安装器提示初始化当前会话，或自动判断常见路径：

```sh
if [ -x /opt/homebrew/bin/brew ]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [ -x /usr/local/bin/brew ]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

brew install git stow starship atuin zoxide fzf
# 仅桌面机器需要：
brew install --cask ghostty
```

macOS 自带 `/bin/zsh`。不要仅为“更新版本”就先改变登录 Shell；先完成真实启动验证。

### 4.2 Ubuntu

```sh
sudo apt update
sudo apt install -y zsh git stow curl ca-certificates less
```

Ubuntu 仓库中的独立工具版本可能不同。若 dotfiles 的 `README.md` 记录了来源和最低版本，以仓库契约为准。下面沿用这套笔记的用户级安装方式。

安装支持 `fzf --zsh` 的 fzf：

```sh
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

安装 Starship：

```sh
mkdir -p "$HOME/.local/bin"
installer=$(mktemp)
curl -fsSL https://starship.rs/install.sh -o "$installer"
less "$installer"
sh "$installer" -b "$HOME/.local/bin"
rm -f "$installer"
```

安装 Atuin，但禁止安装器修改 Shell 文件：

```sh
installer=$(mktemp)
curl --proto '=https' --tlsv1.2 -fsSL \
  https://github.com/atuinsh/atuin/releases/latest/download/atuin-installer.sh \
  -o "$installer"
less "$installer"
ATUIN_NO_MODIFY_PATH=1 sh "$installer"
rm -f "$installer"
```

安装 zoxide：

```sh
installer=$(mktemp)
curl -fsSL https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh -o "$installer"
less "$installer"
sh "$installer"
rm -f "$installer"
```

### 4.3 两个平台共同安装 Antidote

配置约定 Antidote 位于 XDG 数据目录，而不是 dotfiles 仓库：

```sh
antidote_root="${XDG_DATA_HOME:-$HOME/.local/share}"
antidote_dir="$antidote_root/antidote"
mkdir -p "$antidote_root"

if [ ! -d "$antidote_dir/.git" ]; then
  git clone --depth=1 https://github.com/mattmc3/antidote.git "$antidote_dir"
fi

test -r "$antidote_dir/antidote.zsh"
```

检查必要命令。Ubuntu 当前会话可能还需要临时 PATH，持久 PATH 应由恢复后的 `.zshenv` 提供：

```sh
export PATH="$HOME/.local/bin:$HOME/.atuin/bin:$HOME/.local/share/fzf/bin:$PATH"

git --version
stow --version
zsh --version
starship --version
atuin --version
zoxide --version
fzf --version
```

## 5. 克隆并审查恢复输入

交互式读取远端 URL，避免把私人地址写入共享笔记或 Shell 历史示例：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

if [ -e "$DOTFILES_DIR" ]; then
  printf '停止：目标已经存在：%s\n' "$DOTFILES_DIR" >&2
else
  printf 'dotfiles remote URL: '
  IFS= read -r REPO_URL
  git clone "$REPO_URL" "$DOTFILES_DIR"

  git -C "$DOTFILES_DIR" status --short --branch
  git -C "$DOTFILES_DIR" remote -v
  git -C "$DOTFILES_DIR" log -1 --format=fuller
fi
```

看到“停止”后不要继续部署；先按第 2 节查清已有目录。只有 `else` 分支完成 clone 并显示了 Git 现场时才继续。

若事先记录了预期提交，再比较完整提交标识：

```sh
git -C "$DOTFILES_DIR" rev-parse HEAD
```

继续前阅读仓库说明和实际目录，不要猜 package：

```sh
sed -n '1,240p' "$DOTFILES_DIR/README.md"
find "$DOTFILES_DIR" -mindepth 1 -maxdepth 2 \( -type f -o -type d \)
git -C "$DOTFILES_DIR" ls-files
```

确认源文件存在、没有提交历史或密钥，并从 README 与 `.zshrc` 共同推导本机的“启动依赖闭包”：

```sh
grep -nE 'atuin init|starship init|zoxide init|fzf --zsh' \
  "$DOTFILES_DIR/zsh/.config/zsh/.zshrc" 2>/dev/null || true

for package_name in zsh atuin starship ghostty; do
  [ -d "$DOTFILES_DIR/$package_name" ] \
    && printf 'available package: %s\n' "$package_name"
done
unset package_name
```

若 `.zshrc` 会执行 `atuin init`，并且仓库声明 `config.toml` 由 Stow 管理，那么 `atuin` 必须在第一次真实 Zsh 之前部署；Starship 同理。macOS 桌面若要由仓库管理 Ghostty 配置，也必须在首次打开 Ghostty 前加入 `ghostty`。zoxide 数据库与 fzf Shell 生成代码不需要独立软件包。

本套仓库的命令行闭包是 `zsh atuin starship`；macOS 桌面闭包是 `zsh atuin starship ghostty`。若克隆到的是其他布局，以已审查的 README 和启动文件为准；缺少被启动链依赖的受管配置包时先停止修复仓库，不要运行组件让它临时生成普通配置。

## 6. 先暴露冲突，再部署相同列表

检查目标是普通文件、目录还是链接：

```sh
for target_path in \
  "$HOME/.zshenv" \
  "$HOME/.config/zsh/.zprofile" \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/atuin/config.toml" \
  "$HOME/.config/ghostty/config.ghostty"; do
  if [ -e "$target_path" ] || [ -L "$target_path" ]; then
    ls -ld "$target_path"
    readlink "$target_path" 2>/dev/null || true
  fi
done
```

若仓库带有受版本控制的 `scripts/deploy`，优先使用它。先模拟：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
"$DOTFILES_DIR/scripts/deploy" --simulate zsh atuin starship

# macOS 桌面且仓库管理 Ghostty 时，改用：
# "$DOTFILES_DIR/scripts/deploy" --simulate zsh atuin starship ghostty
```

若旧仓库没有该脚本，使用等价的原始 Stow 模拟：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --restow --verbose=2 zsh atuin starship

# macOS 桌面且仓库管理 Ghostty 时，在同一行末尾追加 ghostty。
```

Stow 报冲突时，不要使用 `--adopt` 自动吸收新机器文件。先对照第 3 节备份与仓库源，确认旧目标不再需要后，再把**明确的冲突目标**移动到备份目录。不要移动整个 `$HOME/.config`。

模拟无冲突后，以完全相同的 package 列表应用：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
"$DOTFILES_DIR/scripts/deploy" --apply zsh atuin starship

# macOS 桌面必须与刚才的模拟列表一致：
# "$DOTFILES_DIR/scripts/deploy" --apply zsh atuin starship ghostty
```

没有脚本时：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --restow --verbose=2 zsh atuin starship

# macOS 桌面必须与刚才的模拟列表一致，在同一行末尾追加 ghostty。
```

模拟与应用的列表必须逐字一致。不要先只部署 `zsh` 并运行一次真实 Zsh，再补 `atuin` 或 `starship`；那一次启动就可能抢先创建后续准备由 Stow 管理的目标。

## 7. 创建不入库的本机文件

仓库可能在 `.zshrc` 中按存在性读取 `local.zprofile` 与 `local.zsh`。只在源配置确实约定这些文件时创建：

```sh
previous_umask=$(umask)
umask 077

zsh_config_dir="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"
mkdir -p "$zsh_config_dir"
touch "$zsh_config_dir/local.zprofile" "$zsh_config_dir/local.zsh"
chmod 600 "$zsh_config_dir/local.zprofile" "$zsh_config_dir/local.zsh"

umask "$previous_umask"
```

代理、公司路径、主机别名和秘密只写入这类本机文件或专门的秘密管理系统，不要为了让新机器“完全一致”而提交它们。

## 8. 验证链接、语法与真实启动

若仓库提供 `scripts/verify`，先运行它：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

if [ -x "$DOTFILES_DIR/scripts/verify" ]; then
  "$DOTFILES_DIR/scripts/verify"
fi
```

无论是否有脚本，都先检查启动闭包的关键链接，再执行任何真实 Zsh：

```sh
managed_links_ok=1
for managed_path in \
  "$HOME/.zshenv" \
  "${XDG_CONFIG_HOME:-$HOME/.config}/zsh/.zprofile" \
  "${XDG_CONFIG_HOME:-$HOME/.config}/zsh/.zshrc" \
  "${XDG_CONFIG_HOME:-$HOME/.config}/atuin/config.toml" \
  "${XDG_CONFIG_HOME:-$HOME/.config}/starship.toml"; do
  if [ ! -L "$managed_path" ]; then
    printf 'STOP: expected managed link: %s\n' "$managed_path" >&2
    managed_links_ok=0
    continue
  fi
  ls -ld "$managed_path"
  readlink "$managed_path"
done

if [ "$managed_links_ok" -ne 1 ]; then
  printf 'STOP: configuration closure is incomplete\n' >&2
  false
fi
unset managed_links_ok managed_path

zsh -n "$HOME/.zshenv"
zsh -n "${XDG_CONFIG_HOME:-$HOME/.config}/zsh/.zprofile"
zsh -n "${XDG_CONFIG_HOME:-$HOME/.config}/zsh/.zshrc"

zsh -lic '
  printf "shell=%s\nZDOTDIR=%s\n" "$ZSH_VERSION" "${ZDOTDIR-}"
  command -v starship atuin zoxide fzf
'
```

语法检查不执行启动逻辑；只有上述受管目标都确认是链接后，`zsh -lic` 才第一次经过登录与交互启动链。macOS 桌面还应在首次打开 Ghostty 前确认 `~/.config/ghostty/config.ghostty` 是链接。最后打开一个全新的 Ghostty 窗口或重新建立 SSH 连接，人工检查：

- 没有启动报错；
- PATH 中没有重复且错误的安装来源；
- 提示符、历史搜索、目录跳转和 fzf 触发器按仓库说明工作；
- `git -C "$HOME/.dotfiles" status --short` 没有因运行而新增缓存或历史文件。

完成真实启动后重复链接检查。应用可以在仓库外创建 Atuin 数据库、Antidote 插件、Zsh 历史和缓存，但不得把任何受管配置替换成普通文件。

## 9. 最后才切换 Ubuntu 登录 Shell

macOS 通常已经使用 `/bin/zsh`。Ubuntu 只有在第 8 节全部通过后才执行：

```sh
command -v zsh
grep -Fx "$(command -v zsh)" /etc/shells
chsh -s "$(command -v zsh)"
```

`chsh` 成功只表示账号记录发生变化；退出并重新登录后再次验证：

```sh
getent passwd "$USER"
printf 'inherited SHELL=%s\n' "$SHELL"
ps -p $$ -o comm=,args=
```

## 10. 历史、Atuin 密钥与同步单独恢复

Git 恢复的是声明式配置，不应携带命令历史、Atuin 密钥、数据库或会话。先保留已知可用配置提交，再根据 [[Atuin 命令历史管理]] 选择：

- 只使用本机新历史；
- 登录 Atuin 后由它自己的同步机制恢复；
- 从经过人工脱敏的 Bash 或 Zsh 历史副本导入。

不要把第 3 节的原始备份复制进 dotfiles，也不要在未检查敏感命令前批量导入。

## 11. 完成检查与回退

最终记录当前提交和工作树：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" rev-parse HEAD
git -C "$DOTFILES_DIR" remote -v
```

恢复过程通常不应修改仓库。若出现差异，先确认是否是缓存、生成文件或本机值误入版本控制，不要直接提交。

需要解除部署时，对**实际部署过的同一 package 列表**运行：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --delete --verbose=2 zsh atuin starship
```

macOS 桌面若实际部署过 `ghostty`，解除命令也必须在同一列表末尾追加它。确认链接已经移除后，再从第 3 节的明确备份路径恢复原文件。解除 Stow 不会卸载工具，也不会自动恢复历史或登录 Shell；Ubuntu 如需改回 Bash，先确认 `/bin/bash` 在 `/etc/shells`，再运行 `chsh -s /bin/bash` 并重新登录。

## 进一步理解

- [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]]：源、目标、package、冲突和秘密边界。
- [[Zsh 与 Antidote 跨机器配置管理]]：启动文件与插件生成逻辑。
- [[现代终端环境更新、验证与回退]]：日常更新和故障定位。

## 官方参考资料

- [Git clone documentation](https://git-scm.com/docs/git-clone)
- [GNU Stow Manual](https://www.gnu.org/software/stow/manual/stow.html)
- [Zsh Startup Files](https://zsh.sourceforge.io/Doc/Release/Files.html#Startup_002fShutdown-Files)
- [Antidote Documentation](https://antidote.sh/)
- [Homebrew Installation](https://docs.brew.sh/Installation)

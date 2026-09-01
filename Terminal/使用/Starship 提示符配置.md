---
title: Starship 提示符配置
aliases:
  - Starship 配置
  - Zsh Starship 提示符
  - Starship 后端开发配置
tags:
  - Terminal
  - Terminal/使用
  - Terminal/提示符
  - Starship
  - Zsh
created: 2026-07-19T16:35:26
updated: 2026-09-01T10:54:44
---

Starship 是独立的跨 Shell 提示符程序，只负责根据当前目录、Git 仓库和项目文件渲染 prompt。它不提供 Zsh 插件管理、命令历史、目录数据库、模糊查找或终端主题；这些职责分别属于 Antidote、Atuin、zoxide、fzf 和 Ghostty。完整分层见 [[现代终端环境搭建概览]]，配置源与默认路径的部署关系见 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]]。

> [!tip] 何时打开本文
> 平台主线已经安装并初始化 Starship。只有要定制提示符内容、控制字体依赖或排查大型仓库性能时，才继续本文。

## 1. 确认当前二进制与初始化入口

安装命令只在场景主线维护：[[macOS 从零搭建现代终端环境]]、[[Ubuntu 从零搭建现代终端环境]]、[[从已有 dotfiles 恢复现代终端环境]]。本专题从二进制已经可见开始：

~~~bash
command -v starship
starship --version
~~~

若命令缺失，回到当前平台主线检查原安装来源与 PATH，不要在专题中叠加第二种安装方式。

## 2. 在 Zsh 中只初始化一次

把下面内容放进交互式 `$ZDOTDIR/.zshrc`，不要放进 `.zshenv` 或 `.zprofile`：

~~~zsh
if (( $+commands[starship] )); then
  eval "$(starship init zsh)"
fi
~~~

在本文推荐的组合中，Starship 位于 fzf、Atuin、zoxide 之后；`zsh-syntax-highlighting` 仍需在 `.zshrc` 真正最后加载。完整顺序见 [[Zsh 与 Antidote 跨机器配置管理]]。

把初始化块写进仓库源不会立即执行它。若平台主线已经写入启动基线，可以继续保留；若要定制，则在第 4～6 节中选择一套完整配置。无论选择哪种内容，都先确认第 3 节的配置源并按第 7 节部署，之后才运行 `starship explain`、`exec zsh -l` 或任何完整交互式 Zsh；这样所有受管配置在行为验证前已经拥有明确链接。

从 Oh My Zsh 迁移时，必须移除或禁用原有的 `ZSH_THEME`，也不要再初始化 Powerlevel10k、Pure 等第二套 prompt。多个提示符同时修改 `PROMPT`、`RPROMPT` 和 precmd hook，会产生覆盖、闪烁或重复内容。

## 3. 使用默认配置路径

Starship 默认读取：

~~~text
~/.config/starship.toml
~~~

主路线不直接创建目标文件，而是在 dotfiles 的 `starship` package 中建立配置源，同时确保目标父目录是真实目录：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
starship_source="$DOTFILES_DIR/starship/.config/starship.toml"

git -C "$DOTFILES_DIR" status --short --branch
mkdir -p "${starship_source%/*}" "$HOME/.config"
touch "$starship_source"
~~~

除非现有 dotfiles 布局无法调整，否则不要额外设置 `STARSHIP_CONFIG`。统一默认路径能减少 macOS、Ubuntu、本地终端和 SSH 会话之间的差异。

## 4. 可选：用显式 format 限定后端开发模块

平台主线先写入“最小启动覆盖配置”：它只覆盖少量选项，没有设置顶层 `format`，因此其他模块继续遵循 Starship 默认的 `$all` 格式并按目录条件出现。“最小”指覆盖项少，不表示只会显示配置文件中出现的模块；默认格式和当前模块顺序以 [Starship 官方配置](https://starship.rs/config/#default-prompt-format) 为准。

如果启动基线已经通过验证，并且希望明确限制提示符中允许出现的模块，可以用下面这份完整配置**替换**仓库源。它只展示身份、目录、Git、Java、Go、Docker 上下文和耗时命令。Kubernetes 默认关闭，避免在普通目录持续显示集群信息；需要时再显式启用。符号使用普通文本，不依赖 Nerd Font：

~~~toml
# ~/.config/starship.toml
"$schema" = "https://starship.rs/config-schema.json"

add_newline = false
scan_timeout = 30
command_timeout = 500

format = """
$username\
$hostname\
$directory\
$git_branch\
$git_state\
$git_status\
$java\
$golang\
$docker_context\
$kubernetes\
$cmd_duration\
$line_break\
$character"""

[username]
format = "[$user]($style) "

[hostname]
ssh_only = true
ssh_symbol = ""
format = "[$hostname]($style) "

[directory]
truncation_length = 4
truncate_to_repo = true

[git_branch]
symbol = "git:"
format = "[$symbol$branch]($style) "

[git_status]
format = "([$all_status$ahead_behind]($style) )"

[java]
symbol = "java:"
format = "[$symbol$version]($style) "

[golang]
symbol = "go:"
format = "[$symbol$version]($style) "

[docker_context]
symbol = "docker:"
format = "[$symbol$context]($style) "
only_with_files = true

[kubernetes]
symbol = "k8s:"
format = "[$symbol$context]($style) "
disabled = true

[cmd_duration]
min_time = 2000
format = "[$duration]($style) "

[character]
success_symbol = "[>](bold green)"
error_symbol = "[>](bold red)"
~~~

这套配置有三个刻意的取舍：

1. SSH 会话显示用户名和主机名，降低在错误服务器执行命令的风险；本地普通用户不重复显示身份。
2. Java、Go 和 Docker 模块只在项目上下文满足检测条件时出现，不把所有已安装工具堆进每一行。
3. Kubernetes 需要把 `disabled = true` 改为 `false` 后才显示；启用前先确认 kubeconfig 和上下文切换习惯可靠。

如果只想继续使用 Starship 默认模块，可以保留平台主线的短配置；如果希望采用 Tokyo Night 或 Catppuccin Powerline，则分别选择第 5、6 节。第 4～6 节都是完整方案，不能把多个顶层 `format` 或同名 TOML 表机械拼接。不要在首次启动尚未通过时复制长主题配置，再通过逐项删除寻找故障。

## 5. 从启动基线切换到 Tokyo Night 预设

Tokyo Night 是 [Starship 官方预设列表](https://starship.rs/presets/) 收录的社区预设，本文于 2026-08-31 核对了其页面和 TOML。上游版本使用 Nerd Font、固定深色配色、操作系统、目录、Git、Node.js、Bun、Rust、Go、PHP 与时间模块；下面的派生配置保留其视觉结构，同时按本文的后端开发边界补回 SSH 身份、Git 操作状态、Java、Docker、Kubernetes 和长命令耗时。它不是 Starship 默认配置，也不与第 4 节叠加。

| 审阅项 | 上游 Tokyo Night | 本文派生配置 |
| --- | --- | --- |
| SSH 身份 | 不含 username、hostname | 保留用户名和仅 SSH 显示的主机名 |
| Git | 分支、工作区状态 | 额外显示 merge、rebase 等操作状态 |
| 开发模块 | Node.js、Bun、Rust、Go、PHP | Java、Go、Docker；Kubernetes 默认关闭 |
| 末端信息 | 当前时间 | 长命令耗时和当前时间 |
| 写入位置 | 官网命令直接指定运行时路径 | 先生成临时候选，再更新 dotfiles 仓库源 |

### 5.1 先确认字体与明暗主题边界

[官方 Tokyo Night 预设](https://starship.rs/presets/tokyo-night) 要求终端启用 Nerd Font。Starship 只输出字符和颜色，真正渲染字形的是本地终端：从 macOS Ghostty SSH 到 Ubuntu Server 时，应在 macOS 的 Ghostty 中选择包含所需 glyph 的字体，不需要在无图形服务器上安装桌面字体。IDE 集成终端、其他终端模拟器和本地 Ghostty 需要分别验证。

下面的颜色直接写在 Starship 配置中，不会跟随 Ghostty 的 `light:...,dark:...` 自动切换。先保持 Ghostty 主题不变，分别在明、暗外观下检查可读性；不要在同一轮同时改动 Ghostty 主题和 Starship 配置。字体名称和终端职责见 [[Ghostty 常用配置与 Shell 集成#4. 字体、主题与快捷键的取舍|Ghostty 的字体与主题边界]]。

### 5.2 先生成候选文件，不直接覆盖受管源

官网示例把预设直接输出到 `~/.config/starship.toml`。该路径在本文中是 Stow 管理的链接，直接写入可能改动仓库源，因此先在临时目录生成候选文件并审阅：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
starship_source="$DOTFILES_DIR/starship/.config/starship.toml"
preset_dir="$(mktemp -d)"
preset_candidate="$preset_dir/starship.toml"

test -f "$starship_source"
starship --version
starship preset tokyo-night -o "$preset_candidate"
sed -n '1,220p' "$preset_candidate"
diff -u "$starship_source" "$preset_candidate"
~~~

`diff` 退出码为 `0` 表示内容相同，`1` 表示发现预期差异，大于 `1` 才表示比较失败。在同一 Shell 会话中审阅完成后清理本轮候选文件；变量指向本轮刚创建的临时目录，不要替换成宽泛路径：

~~~bash
rm -f "$preset_candidate"
rmdir "$preset_dir"
unset preset_candidate preset_dir starship_source
~~~

`starship preset` 使用当前已安装 Starship 随附的预设。升级 Starship 后若想跟进上游变化，应再次生成临时候选并比较，不要把官网变化静默覆盖到已验证配置。

### 5.3 使用经过适配的完整配置

下面内容整体保存到 `$DOTFILES_DIR/starship/.config/starship.toml`。它与第 4、6 节是三选一关系：

~~~toml
# 为支持 JSON Schema 的编辑器提供补全与类型检查；这行不控制外观。
"$schema" = "https://starship.rs/config-schema.json"

# 不在相邻两次提示符之间额外插入空行；format 内仍会在输入符号前主动换行。
add_newline = false

# 保留明确的扫描和外部命令超时，便于排查大型仓库中的提示符延迟。
scan_timeout = 30
command_timeout = 500

# 顶层 format 同时决定模块顺序和允许出现的模块。
# 用户名与主机名位于彩色区块之前，只在 SSH 或高权限身份等模块条件满足时出现。
# 目录、Git、语言/容器、耗时/时间分别使用四段 Tokyo Night 色块。
format = """
$username\
$hostname\
[░▒▓](#a3aed2)\
$os\
[](bg:#769ff0 fg:#a3aed2)\
$directory\
[](fg:#769ff0 bg:#394260)\
$git_branch\
$git_state\
$git_status\
[](fg:#394260 bg:#212736)\
$java\
$golang\
$docker_context\
$kubernetes\
[](fg:#212736 bg:#1d2230)\
$cmd_duration\
$time\
[ ](fg:#1d2230)\
\n$character"""

# username 默认不会在普通本地会话中始终显示；SSH 或 root 等场景由模块规则决定。
[username]
style_user = "bold #e0af68"
style_root = "bold #f7768e"
format = "[$user]($style) "

# 主机名只在 SSH 会话中显示，与 username 一起构成醒目的身份安全提示。
[hostname]
ssh_only = true
ssh_symbol = ""
style = "bold #e0af68"
format = "[$hostname]($style) "

# 操作系统图标构成第一段；图标依赖 Nerd Font。
[os]
disabled = false
style = "bg:#a3aed2 fg:#090c0c"
format = "[ $symbol ]($style)"

[os.symbols]
Windows = "󰍲"
Ubuntu = "󰕈"
SUSE = ""
Raspbian = "󰐿"
Mint = "󰣭"
Macos = "󰀵"
Manjaro = ""
Linux = "󰌽"
Gentoo = "󰣨"
Fedora = "󰣛"
Alpine = ""
Amazon = ""
Android = ""
AOSC = ""
Arch = "󰣇"
Artix = "󰣇"
EndeavourOS = ""
CentOS = ""
Debian = "󰣚"
Redhat = "󱄛"
RedHatEnterprise = "󱄛"
Pop = ""

# 目录段最多保留三层，并优先截断到 Git 仓库根目录。
[directory]
style = "fg:#e3e5e5 bg:#769ff0"
format = "[ $path ]($style)"
truncation_length = 3
truncation_symbol = "…/"
truncate_to_repo = true

# 这些替换只匹配同名路径组件；使用其他语言或目录名时保持原名。
[directory.substitutions]
"Documents" = "󰈙 "
"Downloads" = " "
"Music" = " "
"Pictures" = " "

# Git 分支、rebase/merge 等操作状态和工作区状态共用第三段背景色。
[git_branch]
symbol = ""
style = "bg:#394260"
format = '[[ $symbol $branch ](fg:#769ff0 bg:#394260)]($style)'

[git_state]
style = "bg:#394260"
format = '[[ $state( $progress_current/$progress_total) ](fg:#769ff0 bg:#394260)]($style)'

[git_status]
style = "bg:#394260"
format = '[[($all_status$ahead_behind )](fg:#769ff0 bg:#394260)]($style)'

# Java、Go 与容器上下文只在各自检测条件满足时进入第四段。
[java]
symbol = ""
style = "bg:#212736"
format = '[[ $symbol ($version) ](fg:#769ff0 bg:#212736)]($style)'

[golang]
symbol = ""
style = "bg:#212736"
format = '[[ $symbol ($version) ](fg:#769ff0 bg:#212736)]($style)'

[docker_context]
symbol = ""
style = "bg:#212736"
format = '[[ $symbol $context ](fg:#769ff0 bg:#212736)]($style)'
only_with_files = true

# Kubernetes 先保留样式但默认禁用；确认 kubeconfig 与上下文切换习惯后再改为 false。
[kubernetes]
symbol = "󱃾"
style = "bg:#212736"
format = '[[ $symbol $context ](fg:#769ff0 bg:#212736)]($style)'
disabled = true

# 超过两秒的命令显示耗时；24 小时时钟沿用上游 Tokyo Night 的最后一段。
[cmd_duration]
min_time = 2000
style = "bg:#1d2230"
format = '[[ 󱎫 $duration ](fg:#a0a9cb bg:#1d2230)]($style)'

[time]
disabled = false
time_format = "%R"
style = "bg:#1d2230"
format = '[[  $time ](fg:#a0a9cb bg:#1d2230)]($style)'

# 输入符号单独位于第二行；成功、失败与 Vim normal mode 使用不同颜色或方向。
[character]
success_symbol = "[❯](bold #9ece6a)"
error_symbol = "[❯](bold #f7768e)"
vimcmd_symbol = "[❮](bold #9ece6a)"
~~~

与上游相比，本配置没有保留 Node.js、Bun、Rust 和 PHP；只有确实用于日常项目时，才把对应模块同时加入顶层 `format` 和语言色块配置。若不需要始终显示时间，不能只把 `[time]` 改为禁用：还要从顶层 `format` 移除 `$time`，并决定是否保留最后一段。最后一段只剩 `$cmd_duration` 时，普通短命令之后会出现没有内容的色块，必要时应把耗时模块移出 Powerline 色块再单独显示。

## 6. 从启动基线切换到 Catppuccin Powerline 预设

[Catppuccin Powerline](https://starship.rs/zh-CN/presets/catppuccin-powerline) 是 Starship 官方预设列表收录的社区预设。官方说明它是在 Gruvbox Rainbow 预设上做最小修改，并改用 Catppuccin 配色；本文于 2026-09-01 核对了其页面和 TOML。它是一份带顶层 `format` 的完整配置，使用 Powerline 分隔符和 Nerd Font 图标，不能叠加到第 4、5 节的完整配置上。

与 Tokyo Night 的固定色值不同，这个预设通过 Starship palette（调色板）集中维护 Catppuccin 颜色。本方案直接采用上游默认的 Mocha flavor（风味，即 Catppuccin 的配色变体），并延续第 4、5 节已经确定的 SSH 身份、Git 操作状态、后端模块和性能边界：

| 审阅项 | 上游 Catppuccin Powerline | 本文派生配置 |
| --- | --- | --- |
| 配色 | 默认 Mocha，同时定义四种 flavor | 固定使用 `catppuccin_mocha`，只保留当前实际引用的 Mocha 颜色 |
| SSH 身份 | 始终显示用户名，不含主机名 | 用户名仅在 SSH、root 等场景出现；SSH 同时显示主机名 |
| Git | 分支、工作区状态 | 额外显示 merge、rebase 等操作状态 |
| 开发模块 | C、Rust、Go、Node.js、Bun、PHP、Java、Kotlin、Haskell、Python 和 Conda | Java、Go、Docker；Kubernetes 保留样式但默认关闭 |
| 命令耗时 | 显示毫秒，超过 45 秒时触发桌面通知 | 超过两秒才显示，不显示毫秒，不触发桌面通知 |
| 行布局 | 禁用 `line_break`，输入符号保持在同一行 | 启用 `line_break`，输入符号单独位于第二行；通过 `add_newline` 在相邻提示符之间保留空行 |
| 写入位置 | 官网命令直接指定运行时路径 | 先生成临时候选进行上游对照，再更新 dotfiles 仓库源 |

### 6.1 先确认字体与 Mocha 明暗边界

官方预设要求显示它的终端已经启用 Nerd Font。字体仍由本地 Ghostty、IDE 集成终端或其他终端模拟器渲染；通过本地终端 SSH 到服务器时，不需要为了提示符图标在无图形服务器上安装桌面字体。

Mocha 是 Catppuccin 最深的配色变体，也是该预设的默认值；下面的派生配置固定设置 `palette = 'catppuccin_mocha'`。Starship 的顶层 `palette` 不会跟随 Ghostty 的 `light:...,dark:...` 自动切换，因此以深色背景作为主要使用场景，同时仍应在实际启用的浅色外观中检查文字对比度。自动切换需要额外的配置生成或 Shell 逻辑，不属于本方案。

### 6.2 先生成上游候选，不直接覆盖受管源

官网命令会把完整预设直接写到 `~/.config/starship.toml`。该路径在本文中是 Stow 管理的链接，直接写入可能改动仓库源，因此先在临时目录生成上游候选并审阅：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
starship_source="$DOTFILES_DIR/starship/.config/starship.toml"
preset_dir="$(mktemp -d)"
preset_candidate="$preset_dir/catppuccin-powerline.toml"

git -C "$DOTFILES_DIR" status --short --branch
test -f "$starship_source"
starship --version
starship preset catppuccin-powerline -o "$preset_candidate"
sed -n '1,$p' "$preset_candidate"
diff -u "$starship_source" "$preset_candidate"
~~~

`starship preset` 使用当前已安装 Starship 随附的预设，因此输出可能随 Starship 版本变化。`diff` 退出码为 `0` 表示内容相同，`1` 表示发现预期差异，大于 `1` 才表示比较失败。若当前二进制不认识 `catppuccin-powerline`，先回到平台主线核对安装来源和版本，不要从不明站点复制一份无法追溯版本的 TOML。

审阅完成后清理本轮候选；变量必须仍指向本轮 `mktemp -d` 创建的目录：

~~~bash
rm -f "$preset_candidate"
rmdir "$preset_dir"
unset preset_candidate preset_dir starship_source
~~~

如果 `rmdir` 提示目录非空，先检查残留内容，不要改用递归删除。

### 6.3 使用经过适配的 Mocha 完整配置

下面内容整体保存到 `$DOTFILES_DIR/starship/.config/starship.toml`。它与第 4、5 节是三选一关系：

~~~toml
# 为支持 JSON Schema 的编辑器提供补全与类型检查；这行不控制外观。
"$schema" = "https://starship.rs/config-schema.json"

# 在相邻两次提示符之间插入一个空行；line_break 仍会把输入符号放到第二行。
add_newline = true

# 保留明确的扫描和外部命令超时，便于排查大型仓库中的提示符延迟。
scan_timeout = 30
command_timeout = 500

# 固定使用 Catppuccin Mocha，并通过顶层 format 限定允许出现的模块和顺序。
palette = "catppuccin_mocha"
format = """
[](red)\
$os\
$username\
$hostname\
[](bg:peach fg:red)\
$directory\
[](bg:yellow fg:peach)\
$git_branch\
$git_state\
$git_status\
[](fg:yellow bg:green)\
$java\
$golang\
[](fg:green bg:sapphire)\
$docker_context\
$kubernetes\
[](fg:sapphire bg:lavender)\
$time\
[ ](fg:lavender)\
$cmd_duration\
$line_break\
$character"""

# 操作系统图标始终构成第一个色块；图标依赖 Nerd Font。
[os]
disabled = false
style = "bg:red fg:crust"
format = "[ $symbol ]($style)"

[os.symbols]
Windows = ""
Ubuntu = "󰕈"
SUSE = ""
Raspbian = "󰐿"
Mint = "󰣭"
Macos = "󰀵"
Manjaro = ""
Linux = "󰌽"
Gentoo = "󰣨"
Fedora = "󰣛"
Alpine = ""
Amazon = ""
Android = ""
AOSC = ""
Arch = "󰣇"
Artix = "󰣇"
CentOS = ""
Debian = "󰣚"
Redhat = "󱄛"
RedHatEnterprise = "󱄛"

# 用户名只在 SSH、root 等 Starship 触发条件满足时显示；root 使用粗体强调。
[username]
show_always = false
style_user = "bg:red fg:crust"
style_root = "bold bg:red fg:crust"
format = "[ $user]($style)"

# 主机名只在 SSH 会话显示，与用户名组成 user@host 身份提示。
[hostname]
ssh_only = true
ssh_symbol = ""
style = "bg:red fg:crust"
format = "[@$hostname ]($style)"

# 目录段最多保留三层，并优先截断到 Git 仓库根目录。
[directory]
style = "bg:peach fg:crust"
format = "[ $path ]($style)"
truncation_length = 3
truncation_symbol = "…/"
truncate_to_repo = true

[directory.substitutions]
"Documents" = "󰈙 "
"Downloads" = " "
"Music" = "󰝚 "
"Pictures" = " "

# Git 分支、操作状态和工作区状态共用黄色色块。
[git_branch]
symbol = ""
style = "bg:yellow fg:crust"
format = "[ $symbol $branch ]($style)"

[git_state]
style = "bg:yellow fg:crust"
format = "[ $state( $progress_current/$progress_total) ]($style)"

[git_status]
style = "bg:yellow fg:crust"
format = "[($all_status$ahead_behind )]($style)"

# Java 与 Go 只在各自项目检测条件满足时进入绿色色块。
[java]
symbol = ""
style = "bg:green fg:crust"
format = "[ $symbol ($version) ]($style)"

[golang]
symbol = ""
style = "bg:green fg:crust"
format = "[ $symbol ($version) ]($style)"

# Docker 与 Kubernetes 共用 sapphire 色块；Docker 仅在相关文件存在时显示。
[docker_context]
symbol = ""
style = "bg:sapphire fg:crust"
format = "[ $symbol $context ]($style)"
only_with_files = true

# Kubernetes 保留样式但默认关闭；确认上下文切换习惯可靠后再显式启用。
[kubernetes]
symbol = "󱃾"
style = "bg:sapphire fg:crust"
format = "[ $symbol $context ]($style)"
disabled = true

# 24 小时时钟构成最后一个 Powerline 色块。
[time]
disabled = false
time_format = "%R"
style = "bg:lavender fg:crust"
format = "[  $time ]($style)"

# 超过两秒才显示耗时；不显示毫秒，也不触发系统桌面通知。
[cmd_duration]
min_time = 2000
show_milliseconds = false
show_notifications = false
style = "fg:lavender"
format = "[ $duration]($style) "

# 明确启用换行，使输入符号单独位于第二行。
[line_break]
disabled = false

[character]
success_symbol = "[❯](bold green)"
error_symbol = "[❯](bold red)"
vimcmd_symbol = "[❮](bold green)"
vimcmd_replace_one_symbol = "[❮](bold lavender)"
vimcmd_replace_symbol = "[❮](bold lavender)"
vimcmd_visual_symbol = "[❮](bold yellow)"

# 本方案固定使用 Mocha，只定义上面实际引用的颜色名。
[palettes.catppuccin_mocha]
red = "#f38ba8"
peach = "#fab387"
yellow = "#f9e2af"
green = "#a6e3a1"
sapphire = "#74c7ec"
lavender = "#b4befe"
crust = "#11111b"
~~~

这份派生配置已经落实以下决定：Mocha 是唯一活动 palette；用户名与 SSH 主机名保留安全提示但不在普通本地会话重复出现；`git_state`、Java、Go、Docker 和默认关闭的 Kubernetes 延续第 4、5 节边界；`show_notifications = false` 禁止桌面通知；`line_break.disabled = false` 把输入符号放到第二行；`add_newline = true` 在相邻两次提示符之间插入一个空行，避免上一条命令或输出与下一条提示符紧贴。`line_break` 控制单个提示符内部的双行布局，`add_newline` 控制相邻提示符块之间的垂直间距，两者职责不同。

上游包含的其他语言模块没有进入顶层 `format`。以后确实需要某个模块时，应把模块变量和匹配当前色块的配置一起加入，而不是把另一份完整预设拼接进来。

## 7. 部署所选配置

可以保留平台主线的启动基线，也可以在第 4～6 节中只选择一套完整配置保存到仓库源。保存后先模拟再部署；已有目标发生冲突时，按 dotfiles 专题备份和比较，不直接覆盖：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --restow --verbose=2 starship
~~~

确认模拟输出没有 conflict 且只涉及 Starship 配置后，再实际部署：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --restow --verbose=2 starship

test -L "$HOME/.config/starship.toml"
readlink "$HOME/.config/starship.toml"
~~~

## 8. 字体与 Ghostty 的关系

第 4 节使用 `git:`、`java:`、`go:` 等文本符号，普通等宽字体即可；第 5、6 节都使用 Powerline 分隔符和 Nerd Font 图标。选择任一预设时应同时满足：

1. 系统使用 UTF-8 locale。
2. Ghostty 选择的字体确实包含对应 glyph，字体名称以 `ghostty +list-fonts` 为准。
3. 本地 Ghostty 和其他终端模拟器分别验证，不要把字体问题归因于 Zsh 或 Starship。

Ghostty 的字体、主题和配置分层见 [[Ghostty 常用配置与 Shell 集成]]。提示符模块配色由 Starship 控制，终端背景和基础调色板由 Ghostty 控制；二者可以协调，但不要在两处复制同一份主题配置。

## 9. 控制提示符性能

平台主线的启动基线使用默认 `$all`，可能按上下文检查较多模块；第 4～6 节的完整配置则用显式 `format` 限定允许参与提示符的模块。遇到提示符变慢时，先测量，再减少模块或修复对应命令：

~~~bash
starship explain
env STARSHIP_LOG=trace starship timings
time zsh -i -c exit
~~~

| 现象 | 优先检查 | 不建议的做法 |
| --- | --- | --- |
| 大型 Git 仓库变慢 | `git_status`、仓库文件系统和网络挂载 | 直接把所有 timeout 调得很大 |
| 每个目录都执行外部命令 | 自定义模块的 `command`、`when` 和检测条件 | 在 prompt 中访问网络 |
| 语言版本模块超时 | 对应版本管理器或二进制 PATH | 仅隐藏警告而不定位慢命令 |
| 启动 Shell 变慢 | 整体 `.zshrc` 加载顺序 | 只测 Starship 渲染时间就下结论 |

`scan_timeout` 控制文件扫描，`command_timeout` 控制模块执行外部命令的上限。超时不是越大越好；如果某个模块不重要，直接从 `format` 移除通常更稳定。

## 10. 验证配置

保存后先让 Starship 解析当前目录上下文：

~~~bash
starship explain
~~~

再检查 Zsh 配置并替换当前登录 Shell：

~~~bash
zsh -n "$HOME/.config/zsh/.zshrc"
exec zsh -l
~~~

先确认 `~/.config/starship.toml` 指向预期 dotfiles 仓库并审查对应 Git diff。然后进入普通目录、Git 仓库，以及所选顶层 `format` 对应的至少一个真实项目，确认模块只在检测条件满足时出现。只有配置包含 `$git_state` 时，才在发生 merge 或 rebase 的测试仓库中检查操作状态；只有配置包含 `$hostname` 时，才在 SSH 测试会话中检查用户名与主机名。

选择 Tokyo Night 或 Catppuccin Powerline 时，再直接打印两套配置使用的代表性 glyph，确认没有方框、问号或错位：

~~~bash
printf 'Powerline=%s%s%s OS=%s Git=%s Java=%s Go=%s Docker=%s K8s=%s Time=%s Duration=%s\n' \
  '' '' '' '󰕈' '' '' '' '' '󱃾' '' ''
~~~

分别在 Ghostty 明、暗外观和实际使用的 IDE 终端中检查文字对比度、分隔符衔接与所选单行或双行布局；再在普通目录和大型 Git 仓库运行 `starship timings` 比较耗时。若出现 TOML 解析错误、图标缺失或性能明显回退，先把仓库源恢复到已知良好内容并重新 `stow --restow`，再逐段定位；不要同时修改 Ghostty 主题和 Starship 配置。

Starship 的日志默认位于缓存目录，不进入 Git。更新、基准测试和回退流程见 [[现代终端环境更新、验证与回退]]。

## 官方参考资料

- [Starship：安装与 Zsh 初始化](https://starship.rs/guide/)
- [Starship：配置文件与全局选项](https://starship.rs/config/)
- [Starship：预设列表](https://starship.rs/presets/)
- [Starship：Tokyo Night 预设](https://starship.rs/presets/tokyo-night)
- [Starship：Catppuccin Powerline 预设](https://starship.rs/zh-CN/presets/catppuccin-powerline)
- [Catppuccin：官方 palette 与 flavor](https://catppuccin.com/palette/)
- [Nerd Fonts：官方网站](https://www.nerdfonts.com/)
- [Starship：性能诊断与字体排障](https://starship.rs/faq/)
- [GNU Stow：官方手册](https://www.gnu.org/software/stow/manual/stow.html)

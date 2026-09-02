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
updated: 2026-09-02T17:02:36
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

把初始化块写进仓库源不会立即执行它。平台主线先用最小启动覆盖配置完成首次部署、真实 Zsh 验收和已知良好提交；随后按第 7 节建立三配置布局：第 4 节写入默认路径，第 5、6 节作为备用 profile 同时保留。三份配置都先由 Stow 部署，再运行 `starship explain`、`exec zsh -l` 或任何完整交互式 Zsh；这样所有受管配置在行为验证前已经拥有明确链接。

从 Oh My Zsh 迁移时，必须移除或禁用原有的 `ZSH_THEME`，也不要再初始化 Powerlevel10k、Pure 等第二套 prompt。多个提示符同时修改 `PROMPT`、`RPROMPT` 和 precmd hook，会产生覆盖、闪烁或重复内容。

## 3. 保留默认路径并建立备用配置目录

Starship 默认读取：

~~~text
~/.config/starship.toml
~~~

第 4 节继续占用这个默认路径；第 5、6 节放在同一 `starship` package 的 `profiles/` 目录。`profile` 在本文中表示一份可以独立解析和启用的完整配置，不是可以拼接到其他配置上的 TOML 片段；`profiles/` 只是本套 dotfiles 的归档目录，Starship 不会自动扫描其中的文件：

~~~text
$DOTFILES_DIR/starship/.config/
├── starship.toml
└── starship/
    └── profiles/
        ├── tokyo-night.toml
        └── catppuccin-mocha.toml
~~~

主路线不直接创建 `$HOME` 下的目标文件，而是在 dotfiles 中准备配置源，同时确保目标父目录是真实目录：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
starship_source="$DOTFILES_DIR/starship/.config/starship.toml"
starship_profiles_dir="$DOTFILES_DIR/starship/.config/starship/profiles"

git -C "$DOTFILES_DIR" status --short --branch
mkdir -p "${starship_source%/*}" "$starship_profiles_dir" "$HOME/.config"
touch "$starship_source"
~~~

截至 2026-09-01，Starship 官方支持用 [`STARSHIP_CONFIG`](https://starship.rs/config/#config-file-location) 环境变量选择非默认配置路径。本文只在明确选择备用 profile 时设置它；未设置时始终回到 `~/.config/starship.toml`，不为默认配置增加额外选择状态。`STARSHIP_CONFIG` 必须在 Starship 初始化之前生效，第 7 节给出临时和本机持久两种用法。

## 4. 默认配置：用显式 format 限定后端开发模块

平台主线先写入“最小启动覆盖配置”：它只覆盖少量选项，没有设置顶层 `format`，因此其他模块继续遵循 Starship 默认的 `$all` 格式并按目录条件出现。“最小”指覆盖项少，不表示只会显示配置文件中出现的模块；默认格式和当前模块顺序以 [Starship 官方配置](https://starship.rs/config/#default-prompt-format) 为准。

启动基线通过验证并形成已知良好提交后，用下面这份完整配置替换 `$DOTFILES_DIR/starship/.config/starship.toml`。它是三配置布局的默认配置：未设置 `STARSHIP_CONFIG` 时自动生效。它只展示身份、目录、Git、Java、Go、Docker 上下文和耗时命令。Kubernetes 默认关闭，避免在普通目录持续显示集群信息；需要时再显式启用。符号使用普通文本，不依赖 Nerd Font：

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

第 4 节只在默认路径维护一份，不再复制为 `backend.toml`。第 5、6 节分别进入两个备用 profile；三份配置都含顶层 `format`，不能把多个顶层 `format` 或同名 TOML 表机械拼接。不要在首次启动尚未通过时替换最小启动配置，再通过逐项删除长配置寻找故障。

## 5. 备用配置：Tokyo Night 派生预设

Tokyo Night 是 [Starship 官方预设列表](https://starship.rs/presets/) 收录的社区预设，本文于 2026-09-02 核对了其页面、TOML 与相关配置选项。上游版本使用 Nerd Font、固定深色配色、操作系统、目录、Git、Node.js、Bun、Rust、Go、PHP 与时间模块；下面的派生配置保留其分段配色，并把 SSH 身份并入首个浅蓝色块，同时按本文的后端开发边界补回 Git 操作状态、Java、Docker、Kubernetes 和长命令耗时。它不是 Starship 默认配置，也不与第 4 节叠加。

| 审阅项 | 上游 Tokyo Night | 本文派生配置 |
| --- | --- | --- |
| SSH 身份 | 不含 username、hostname | 用户名与仅 SSH 显示的主机名组成 user@host，与系统图标共用浅蓝色块 |
| Git | 分支、工作区状态 | 额外显示 merge、rebase 等操作状态 |
| 开发模块 | Node.js、Bun、Rust、Go、PHP | Java、Go、Docker；Kubernetes 默认关闭 |
| 末端信息 | 当前时间 | 长命令耗时和当前时间 |
| 行布局 | 通过 `\n$character` 使用双行布局 | 保留双行，并显式在每次新提示符前留一个空行 |
| 写入位置 | 官网命令直接指定运行时路径 | 先生成临时候选，再更新 dotfiles 中的备用 profile |

### 5.1 先确认字体与明暗主题边界

[官方 Tokyo Night 预设](https://starship.rs/presets/tokyo-night) 要求终端启用 Nerd Font。Starship 只输出字符和颜色，真正渲染字形的是本地终端：从 macOS Ghostty SSH 到 Ubuntu Server 时，应在 macOS 的 Ghostty 中选择包含所需 glyph 的字体，不需要在无图形服务器上安装桌面字体。IDE 集成终端、其他终端模拟器和本地 Ghostty 需要分别验证。

下面的颜色直接写在 Starship 配置中，不会跟随 Ghostty 的 `light:...,dark:...` 自动切换。先保持 Ghostty 主题不变，分别在明、暗外观下检查可读性；不要在同一轮同时改动 Ghostty 主题和 Starship 配置。字体名称和终端职责见 [[Ghostty 常用配置与 Shell 集成#4. 字体、主题与快捷键的取舍|Ghostty 的字体与主题边界]]。

### 5.2 先生成候选文件，不直接覆盖受管源

官网示例把预设直接输出到 `~/.config/starship.toml`。该路径在本文中是 Stow 管理的链接，直接写入可能改动仓库源，因此先在临时目录生成候选文件并审阅：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
starship_profile_source="$DOTFILES_DIR/starship/.config/starship/profiles/tokyo-night.toml"
preset_dir="$(mktemp -d)"
preset_candidate="$preset_dir/starship.toml"

mkdir -p "${starship_profile_source%/*}"
starship --version
starship preset tokyo-night -o "$preset_candidate"
sed -n '1,220p' "$preset_candidate"

if [ -f "$starship_profile_source" ]; then
  diff -u "$starship_profile_source" "$preset_candidate"
else
  printf 'new profile target: %s\n' "$starship_profile_source"
fi
~~~

`diff` 退出码为 `0` 表示内容相同，`1` 表示发现预期差异，大于 `1` 才表示比较失败。在同一 Shell 会话中审阅完成后清理本轮候选文件；变量指向本轮刚创建的临时目录，不要替换成宽泛路径：

~~~bash
rm -f "$preset_candidate"
rmdir "$preset_dir"
unset preset_candidate preset_dir starship_profile_source
~~~

`starship preset` 使用当前已安装 Starship 随附的预设。升级 Starship 后若想跟进上游变化，应再次生成临时候选并比较，不要把官网变化静默覆盖到已验证配置。

### 5.3 使用经过适配的完整配置

提示符采用“第一行显示状态、第二行输入命令”的双行布局，并在上一条命令的输出与下一次提示符之间留一个空行。系统图标与满足显示条件的 `user@host` 共用首个浅蓝色块，再衔接目录等后续色块。

下面内容整体保存到 `$DOTFILES_DIR/starship/.config/starship/profiles/tokyo-night.toml`。保存文件不会自动启用它；第 7 节通过 `STARSHIP_CONFIG` 显式选择：

~~~toml
# 为支持 JSON Schema 的编辑器提供补全与类型检查；这行不控制外观。
"$schema" = "https://starship.rs/config-schema.json"

# 在每次新提示符前插入一个空行；format 末尾的 \n 继续把输入符号放到第二行。
add_newline = true

# 保留明确的扫描和外部命令超时，便于排查大型仓库中的提示符延迟。
scan_timeout = 30
command_timeout = 500

# 顶层 format 同时决定模块顺序和允许出现的模块。
# 系统图标与按条件显示的 user@host 共用首个浅蓝色块。
# 目录、Git、语言/容器、耗时/时间分别使用四段 Tokyo Night 色块。
# 首块右侧的背景色空格统一写在顶层，身份模块隐藏时仍保留完整留白。
format = """
[░▒▓](#a3aed2)\
$os\
$username\
$hostname\
[ ](bg:#a3aed2)\
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
# 普通身份采用深色文字；root 使用同一浅蓝背景上的深红色强调。
# 用户名前的两个空格拉开与系统图标的距离，并随用户名模块一起显示或隐藏。
[username]
show_always = false
style_user = "bold fg:#090c0c bg:#a3aed2"
style_root = "bold fg:#8c2432 bg:#a3aed2"
format = "[  $user]($style)"

# 主机名只在 SSH 会话中显示；@ 随本模块一起隐藏，与用户名共用背景色。
[hostname]
ssh_only = true
ssh_symbol = ""
style = "bold fg:#090c0c bg:#a3aed2"
format = "[@$hostname]($style)"

# 操作系统图标与身份信息共用第一段；图标依赖 Nerd Font，右侧留白由顶层统一补齐。
[os]
disabled = false
style = "bg:#a3aed2 fg:#090c0c"
format = "[ $symbol]($style)"

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

`add_newline = true` 控制新提示符前的空行，`\n$character` 控制单个提示符内部的换行。二者同时保留后，命令输出与下一次提示符有清楚的分隔，输入符号仍位于第二行。

身份显示条件保持按需触发：普通本地用户未触发 username 模块时，首块只显示系统图标；SSH 会话显示 `user@host`；root 的用户名使用深红色。系统图标与用户名之间保留两个空格，留白写在 username 的 `format` 内，身份隐藏时一起消失。`@` 放在 hostname 的 `format` 内，主机名隐藏时不会留下孤立的 `@`。模块内的空格与顶层 `[ ](bg:#a3aed2)` 都带背景色，让各场景下的首块连续，并在分隔符前保留一个空格。

启用后，先在 SSH 会话连续运行 `pwd` 和 `printf 'ok\n'`，检查两次新提示符前都有留白、身份信息位于同一色块、输入符号仍在第二行；再在普通本地用户会话和较窄窗口中检查身份隐藏后的留白及色块衔接。完整的配置选择、字体和恢复检查见第 7、10 节。

与上游相比，本配置没有保留 Node.js、Bun、Rust 和 PHP；只有确实用于日常项目时，才把对应模块同时加入顶层 `format` 和语言色块配置。若不需要始终显示时间，不能只把 `[time]` 改为禁用：还要从顶层 `format` 移除 `$time`，并决定是否保留最后一段。最后一段只剩 `$cmd_duration` 时，普通短命令之后会出现没有内容的色块，必要时应把耗时模块移出 Powerline 色块再单独显示。

## 6. 备用配置：Catppuccin Mocha Powerline

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
| 写入位置 | 官网命令直接指定运行时路径 | 先生成临时候选进行上游对照，再更新 dotfiles 中的备用 profile |

### 6.1 先确认字体与 Mocha 明暗边界

官方预设要求显示它的终端已经启用 Nerd Font。字体仍由本地 Ghostty、IDE 集成终端或其他终端模拟器渲染；通过本地终端 SSH 到服务器时，不需要为了提示符图标在无图形服务器上安装桌面字体。

Mocha 是 Catppuccin 最深的配色变体，也是该预设的默认值；下面的派生配置固定设置 `palette = 'catppuccin_mocha'`。Starship 的顶层 `palette` 不会跟随 Ghostty 的 `light:...,dark:...` 自动切换，因此以深色背景作为主要使用场景，同时仍应在实际启用的浅色外观中检查文字对比度。自动切换需要额外的配置生成或 Shell 逻辑，不属于本方案。

### 6.2 先生成上游候选，不直接覆盖受管源

官网命令会把完整预设直接写到 `~/.config/starship.toml`。该路径在本文中是 Stow 管理的链接，直接写入可能改动仓库源，因此先在临时目录生成上游候选并审阅：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
starship_profile_source="$DOTFILES_DIR/starship/.config/starship/profiles/catppuccin-mocha.toml"
preset_dir="$(mktemp -d)"
preset_candidate="$preset_dir/catppuccin-powerline.toml"

git -C "$DOTFILES_DIR" status --short --branch
mkdir -p "${starship_profile_source%/*}"
starship --version
starship preset catppuccin-powerline -o "$preset_candidate"
sed -n '1,$p' "$preset_candidate"

if [ -f "$starship_profile_source" ]; then
  diff -u "$starship_profile_source" "$preset_candidate"
else
  printf 'new profile target: %s\n' "$starship_profile_source"
fi
~~~

`starship preset` 使用当前已安装 Starship 随附的预设，因此输出可能随 Starship 版本变化。`diff` 退出码为 `0` 表示内容相同，`1` 表示发现预期差异，大于 `1` 才表示比较失败。若当前二进制不认识 `catppuccin-powerline`，先回到平台主线核对安装来源和版本，不要从不明站点复制一份无法追溯版本的 TOML。

审阅完成后清理本轮候选；变量必须仍指向本轮 `mktemp -d` 创建的目录：

~~~bash
rm -f "$preset_candidate"
rmdir "$preset_dir"
unset preset_candidate preset_dir starship_profile_source
~~~

如果 `rmdir` 提示目录非空，先检查残留内容，不要改用递归删除。

### 6.3 使用经过适配的 Mocha 完整配置

下面内容整体保存到 `$DOTFILES_DIR/starship/.config/starship/profiles/catppuccin-mocha.toml`。保存文件不会自动启用它；第 7 节通过 `STARSHIP_CONFIG` 显式选择：

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

这份 profile 已经落实以下决定：Mocha 是其中唯一的 palette；用户名与 SSH 主机名保留安全提示但不在普通本地会话重复出现；`git_state`、Java、Go、Docker 和默认关闭的 Kubernetes 延续第 4、5 节边界；`show_notifications = false` 禁止桌面通知；`line_break.disabled = false` 把输入符号放到第二行；`add_newline = true` 在相邻两次提示符之间插入一个空行，避免上一条命令或输出与下一条提示符紧贴。`line_break` 控制单个提示符内部的双行布局，`add_newline` 控制相邻提示符块之间的垂直间距，两者职责不同。

上游包含的其他语言模块没有进入顶层 `format`。以后确实需要某个模块时，应把模块变量和匹配当前色块的配置一起加入，而不是把另一份完整预设拼接进来。

## 7. 部署三套配置并选择活动配置

只有平台主线已经用最小启动配置通过真实新会话验收并形成已知良好提交，才进入本节。此时把第 4 节写入默认源，把第 5、6 节写入各自 profile；最小启动配置不再作为第四份长期配置留在当前工作树。先确认三份源都存在且非空：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
starship_sources_ok=1

git -C "$DOTFILES_DIR" status --short --branch
for starship_source in \
  "$DOTFILES_DIR/starship/.config/starship.toml" \
  "$DOTFILES_DIR/starship/.config/starship/profiles/tokyo-night.toml" \
  "$DOTFILES_DIR/starship/.config/starship/profiles/catppuccin-mocha.toml"; do
  if ! test -s "$starship_source"; then
    printf 'missing Starship source: %s\n' "$starship_source" >&2
    starship_sources_ok=0
  fi
done

if [ "$starship_sources_ok" -ne 1 ]; then
  unset starship_source starship_sources_ok
  false
else
  unset starship_source starship_sources_ok
fi
~~~

三份文件仍属于同一个 `starship` package。保存后先模拟再部署；已有目标发生冲突时，按 dotfiles 专题备份和比较，不直接覆盖：

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

实际部署后检查三份目标都由 Stow 管理：

~~~bash
managed_links_ok=1
for managed_path in \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/starship/profiles/tokyo-night.toml" \
  "$HOME/.config/starship/profiles/catppuccin-mocha.toml"; do
  if ! test -L "$managed_path"; then
    printf 'expected managed link: %s\n' "$managed_path" >&2
    managed_links_ok=0
    continue
  fi
  readlink "$managed_path"
done

if [ "$managed_links_ok" -ne 1 ]; then
  unset managed_links_ok managed_path
  false
else
  unset managed_links_ok managed_path
fi
~~~

### 7.1 默认配置不保存额外选择状态

第 4 节位于 Starship 默认读取的 `~/.config/starship.toml`。没有 `STARSHIP_CONFIG` 时，它自然成为默认配置：

~~~zsh
unset STARSHIP_CONFIG
exec zsh -l
~~~

`exec` 会用新的登录 Zsh 替换当前 Shell；先确认当前终端没有需要继续保留的前台任务。新 Shell 中仍可能由 `local.zsh` 再次设置 `STARSHIP_CONFIG`，因此持久配置存在时应按第 7.3 节处理。

### 7.2 临时选择备用 profile

只在当前终端会话选择 Tokyo Night：

~~~zsh
export STARSHIP_CONFIG="$HOME/.config/starship/profiles/tokyo-night.toml"
exec zsh -l
~~~

选择 Catppuccin Mocha 时改用：

~~~zsh
export STARSHIP_CONFIG="$HOME/.config/starship/profiles/catppuccin-mocha.toml"
exec zsh -l
~~~

环境变量只决定 Starship 本轮读取哪个文件，不改写任何 TOML、Stow 链接或 Git 源。新开的独立终端是否继承该选择取决于它的父进程；需要跨新会话保留时使用下一节的本机配置。

### 7.3 在单台机器持久选择备用 profile

三条搭建与迁移主线都按“共享 → 平台 → local → Starship”的顺序加载交互配置。因此，单台机器要长期使用备用 profile 时，先检查不入库的 `$HOME/.config/zsh/local.zsh`，再加入其中一行：

~~~zsh
# 二选一，不要同时保留两行。
export STARSHIP_CONFIG="$HOME/.config/starship/profiles/tokyo-night.toml"
# export STARSHIP_CONFIG="$HOME/.config/starship/profiles/catppuccin-mocha.toml"
~~~

保存后执行 `exec zsh -l`。要恢复第 4 节默认配置，应从 `local.zsh` 删除或注释这行并重新进入登录 Zsh；不要在共享配置中追加一条相反的 `unset`。`STARSHIP_CONFIG` 是交互提示符选择，不放进所有 Zsh 都会读取的 `.zshenv`。

切换时不要用 `ln -sfn` 替换 `~/.config/starship.toml`，也不要把备用内容复制到默认文件：前者会破坏 Stow 所有权，后者会把临时选择变成仓库内容修改。

## 8. 字体与 Ghostty 的关系

第 4 节默认配置使用 `git:`、`java:`、`go:` 等文本符号，普通等宽字体即可；第 5、6 节备用 profile 都使用 Powerline 分隔符和 Nerd Font 图标。选择任一备用 profile 时应同时满足：

1. 系统使用 UTF-8 locale。
2. Ghostty 选择的字体确实包含对应 glyph，字体名称以 `ghostty +list-fonts` 为准。
3. 本地 Ghostty 和其他终端模拟器分别验证，不要把字体问题归因于 Zsh 或 Starship。

Ghostty 的字体、主题和配置分层见 [[Ghostty 常用配置与 Shell 集成]]。提示符模块配色由 Starship 控制，终端背景和基础调色板由 Ghostty 控制；二者可以协调，但不要在两处复制同一份主题配置。

## 9. 控制提示符性能

平台主线的最小启动基线使用默认 `$all`，可能按上下文检查较多模块；最终默认配置和两个备用 profile 都用显式 `format` 限定允许参与提示符的模块。遇到提示符变慢时，先确认实际选择，再测量、减少模块或修复对应命令：

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

先确认所有配置源和部署目标仍然一一对应，再让 Starship 分别解析三份配置。命令前缀中的 `STARSHIP_CONFIG=...` 只影响紧随其后的 `starship explain`，不会持久切换当前 Shell：

~~~bash
starship_configs_ok=1
for starship_config in \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/starship/profiles/tokyo-night.toml" \
  "$HOME/.config/starship/profiles/catppuccin-mocha.toml"; do
  if ! test -L "$starship_config"; then
    printf 'expected managed link: %s\n' "$starship_config" >&2
    starship_configs_ok=0
    continue
  fi
  if ! STARSHIP_CONFIG="$starship_config" starship explain >/dev/null; then
    printf 'Starship could not parse: %s\n' "$starship_config" >&2
    starship_configs_ok=0
  fi
done

if [ "$starship_configs_ok" -ne 1 ]; then
  unset starship_config starship_configs_ok
  false
else
  unset starship_config starship_configs_ok
fi
~~~

三份都能被读取后，检查 Zsh 配置并替换当前登录 Shell：

~~~bash
zsh -n "$HOME/.config/zsh/.zshrc"
exec zsh -l
~~~

进入新 Shell 后再打印实际选择并运行 `starship explain`：

~~~zsh
print -r -- "effective Starship config: ${STARSHIP_CONFIG:-$HOME/.config/starship.toml}"
starship explain
~~~

确认有效路径与实际外观属于同一份配置。然后进入普通目录、Git 仓库，以及所选顶层 `format` 对应的至少一个真实项目，确认模块只在检测条件满足时出现。只有配置包含 `$git_state` 时，才在发生 merge 或 rebase 的测试仓库中检查操作状态；只有配置包含 `$hostname` 时，才在 SSH 测试会话中检查用户名与主机名。

选择 Tokyo Night 或 Catppuccin Powerline 时，再直接打印两套配置使用的代表性 glyph，确认没有方框、问号或错位：

~~~bash
printf 'Powerline=%s%s%s OS=%s Git=%s Java=%s Go=%s Docker=%s K8s=%s Time=%s Duration=%s\n' \
  '' '' '' '󰕈' '' '' '' '' '󱃾' '' ''
~~~

分别在 Ghostty 明、暗外观和实际使用的 IDE 终端中检查文字对比度、分隔符衔接与所选单行或双行布局；再在普通目录和大型 Git 仓库运行 `starship timings` 比较耗时。每次切换后重复三份链接检查和 `git -C "$HOME/.dotfiles" status --short`，确认选择动作没有改写链接或仓库源。若出现 TOML 解析错误、图标缺失或性能明显回退，先清除本轮 `STARSHIP_CONFIG` 或从 `local.zsh` 移除选择，回到第 4 节默认配置；配置源本身有误时才从已知良好提交恢复明确文件，只有源路径或部署关系改变时才重新运行 Stow。不要同时修改 Ghostty 主题和 Starship 配置。

三份配置和默认选择全部通过后，把这次 profile 布局作为启动基线之后的一次独立 dotfiles 变更审阅并提交；具体提交边界见 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库#9. Git 提交与远端边界]]。`local.zsh` 中的单机选择仍不进入 Git，也不应成为共享配置已验证的替代证据。

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

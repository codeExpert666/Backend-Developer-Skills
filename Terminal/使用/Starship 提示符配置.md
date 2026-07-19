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
updated: 2026-07-19T16:53:12
---

Starship 是独立的跨 Shell 提示符程序，只负责根据当前目录、Git 仓库和项目文件渲染 prompt。它不提供 Zsh 插件管理、命令历史、目录数据库、模糊查找或终端主题；这些职责分别属于 Antidote、Atuin、zoxide、fzf 和 Ghostty。完整分层见 [[现代终端环境搭建概览]]。

## 1. 安装 Starship

macOS 使用 Homebrew：

~~~bash
brew install starship
~~~

Ubuntu 25.04 及以上可使用发行版仓库：

~~~bash
sudo apt update
sudo apt install starship
~~~

较旧 Ubuntu 若仓库没有 `starship`，可使用官方安装脚本或官方 Release 二进制。先把官方脚本下载到临时文件，审阅后再执行：

~~~bash
mkdir -p "$HOME/.local/bin"
starship_installer="$(mktemp)"
curl --fail --silent --show-error --location \
  https://starship.rs/install.sh \
  --output "$starship_installer"
less "$starship_installer"
sh "$starship_installer" -b "$HOME/.local/bin"
rm -f -- "$starship_installer"
unset starship_installer
~~~

这里安装到已经纳入本方案最小 PATH 的 `~/.local/bin`，不要求写入系统目录。在受管或生产环境中，应由管理员决定实际安装目录；审阅未通过时不要执行后续 `sh`。不要从不明镜像复制二进制。平台完整安装流程见 [[macOS 从零搭建现代终端环境]] 与 [[Ubuntu 从零搭建现代终端环境]]。

安装后确认二进制可见：

~~~bash
starship --version
command -v starship
~~~

## 2. 在 Zsh 中只初始化一次

把下面内容放进交互式 `$ZDOTDIR/.zshrc`，不要放进 `.zshenv` 或 `.zprofile`：

~~~zsh
if (( $+commands[starship] )); then
  eval "$(starship init zsh)"
fi
~~~

在本文推荐的组合中，Starship 位于 fzf、Atuin、zoxide 之后；`zsh-syntax-highlighting` 仍需在 `.zshrc` 真正最后加载。完整顺序见 [[Zsh 与 Antidote 跨机器配置管理]]。

从 Oh My Zsh 迁移时，必须移除或禁用原有的 `ZSH_THEME`，也不要再初始化 Powerlevel10k、Pure 等第二套 prompt。多个提示符同时修改 `PROMPT`、`RPROMPT` 和 precmd hook，会产生覆盖、闪烁或重复内容。

## 3. 使用默认配置路径

Starship 默认读取：

~~~text
~/.config/starship.toml
~~~

创建文件：

~~~bash
mkdir -p "$HOME/.config"
touch "$HOME/.config/starship.toml"
~~~

除非现有 dotfiles 布局无法调整，否则不要额外设置 `STARSHIP_CONFIG`。统一默认路径能减少 macOS、Ubuntu、本地终端和 SSH 会话之间的差异。

## 4. 最小后端开发配置

下面配置只展示身份、目录、Git、Java、Go、Docker 上下文和耗时命令。Kubernetes 默认关闭，避免在普通目录持续显示集群信息；需要时再显式启用。符号使用普通文本，不依赖 Nerd Font：

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

如果只想使用 Starship 默认样式，`starship.toml` 可以保持为空。不要一开始复制数百行主题配置，再通过逐项删除寻找性能问题。

## 5. 字体与 Ghostty 的关系

本文配置使用 `git:`、`java:`、`go:` 等文本符号，因此普通等宽字体即可。若以后换用带图标的 Starship preset，应同时满足：

1. 系统使用 UTF-8 locale。
2. Ghostty 选择的字体确实包含对应 glyph，字体名称以 `ghostty +list-fonts` 为准。
3. 本地 Ghostty 和其他终端模拟器分别验证，不要把字体问题归因于 Zsh 或 Starship。

Ghostty 的字体、主题和配置分层见 [[Ghostty 常用配置与 Shell 集成]]。提示符配色应由 Starship 控制模块样式，终端背景和基础调色板则由 Ghostty 控制；不要在两处重复维护同一套主题文件。

## 6. 控制提示符性能

Starship 只运行顶层 `format` 中出现的模块。遇到提示符变慢时，先测量，再减少模块或修复对应命令：

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

## 7. 验证配置

保存后先让 Starship 解析当前目录上下文：

~~~bash
starship explain
~~~

再检查 Zsh 配置并替换当前登录 Shell：

~~~bash
zsh -n "$HOME/.config/zsh/.zshrc"
exec zsh -l
~~~

分别进入普通目录、Git 仓库、Java 项目和 Go 项目验证模块按需出现；SSH 到测试主机时确认用户名与主机名可见。若出现 TOML 解析错误，先回退到空 `starship.toml`，再逐段恢复；不要同时修改 Ghostty 主题和 Starship 配置。

Starship 的日志默认位于缓存目录，不进入 Git。更新、基准测试和回退流程见 [[现代终端环境更新、验证与回退]]。

## 官方参考资料

- [Starship：安装与 Zsh 初始化](https://starship.rs/guide/)
- [Starship：配置文件与全局选项](https://starship.rs/config/)
- [Starship：性能诊断与字体排障](https://starship.rs/faq/)

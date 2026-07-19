---
title: Ghostty 常用配置与 Shell 集成
aliases:
  - Ghostty 配置与 Shell 集成
  - Ghostty 跨机器配置
tags:
  - Terminal
  - Terminal/使用
  - Terminal/终端模拟器
  - Ghostty
  - Zsh
created: 2026-07-19T16:35:26
updated: 2026-07-19T16:35:26
---

Ghostty 是本地计算机上的终端模拟器，负责窗口、字体、配色、标签页、分屏和终端协议；它不负责 Zsh 插件、提示符、命令历史或目录跳转。完整组件关系见 [[现代终端环境搭建概览]]，Shell 配置见 [[Zsh 与 Antidote 跨机器配置管理]]。

连接 Ubuntu Server 时，Ghostty 仍然运行在本地 macOS 或 Linux 桌面端，远端只运行 Shell 和命令行程序。无图形界面的服务器通常不需要安装 Ghostty。Linux 安装方式应以具体发行版维护渠道为准，不要把第三方安装脚本描述成 Ghostty 官方预编译包。

## 1. 先以零配置运行

Ghostty 的默认配置已经包含可用字体、配色和 Zsh 自动集成。第一次启动后，先直接使用默认值并确认基础环境：

~~~zsh
printf 'shell=%s\nterm=%s\n' "$SHELL" "$TERM"
zsh --version
~~~

正常情况下，在 Ghostty 中启动的终端会使用 `xterm-ghostty`。如果默认字体、字号和快捷键已经合适，就不必为了“完整配置”重复声明默认值；显式配置越少，跨版本维护成本越低。

## 2. 使用统一的 XDG 配置路径

当前 Ghostty 优先使用以下 XDG 配置文件：

~~~text
~/.config/ghostty/config.ghostty
~~~

旧版本还使用 `~/.config/ghostty/config`，当前版本仍会识别它。新配置统一采用 `config.ghostty`，不要同时维护两个文件。macOS 还支持 `~/Library/Application Support/com.mitchellh.ghostty/` 下的配置，而且会在 XDG 文件之后加载；若两处都存在，后加载的 macOS 文件可能覆盖 XDG 配置，因此跨机器方案只保留 XDG 路径。

先创建目录和主配置文件：

~~~bash
mkdir -p "$HOME/.config/ghostty"
touch "$HOME/.config/ghostty/config.ghostty"
~~~

Ghostty 的语法是每行一个 `key = value`。建议先只设置真正主观的两项：

~~~ini
# ~/.config/ghostty/config.ghostty
font-size = 14
theme = dark:Catppuccin Frappe,light:Catppuccin Latte
~~~

主题名称和字体名称必须以本机实际列表为准：

~~~bash
ghostty +list-themes
ghostty +list-fonts
~~~

macOS 默认使用 `Cmd+Shift+,` 重新加载配置，Linux 默认使用 `Ctrl+Shift+,`。部分选项只影响新建终端或必须重启 Ghostty，不能仅凭当前标签页判断配置是否无效。

## 3. 使用 `config-file` 分层

需要同步多台桌面设备时，把上一节的字号和主题移入 `common.ghostty`，再让主文件只作为加载入口：

~~~bash
touch "$HOME/.config/ghostty/common.ghostty"
~~~

~~~ini
# ~/.config/ghostty/common.ghostty
font-size = 14
theme = dark:Catppuccin Frappe,light:Catppuccin Latte
~~~

~~~ini
# ~/.config/ghostty/config.ghostty
config-file = common.ghostty
config-file = ?platform.ghostty
config-file = ?local.ghostty
~~~

建议的职责如下：

| 文件 | 是否进入 Git | 适合保存的内容 |
| --- | --- | --- |
| `config.ghostty` | 是 | 固定加载顺序 |
| `common.ghostty` | 是 | 跨平台主题、通用字号和明确验证过的键位 |
| `macos.ghostty` / `linux.ghostty` | 是 | 平台专属选项模板 |
| `platform.ghostty` | 否 | 指向当前平台模板的符号链接，或当前平台的少量设置 |
| `local.ghostty` | 否 | 本机字体、屏幕字号和临时实验 |

`?` 表示文件不存在时忽略。相对路径以包含 `config-file` 的文件为基准。Ghostty 会在当前文件的其他设置之后处理所有 `config-file`，并按加载顺序让后面的值覆盖前面的值，因此这里形成 `common -> platform -> local` 的覆盖关系。

如果确实需要平台模板，可在每台机器上只创建一个不会进入 Git 的链接：

~~~bash
cd "$HOME/.config/ghostty"

# macOS 与 Linux 桌面二选一；目标已存在时命令会失败，不会静默覆盖。
ln -s macos.ghostty platform.ghostty
# ln -s linux.ghostty platform.ghostty
~~~

没有平台差异时，不创建 `platform.ghostty` 即可。不要为了目录看起来完整而复制空配置。

## 4. 字体、主题与快捷键的取舍

| 配置对象 | 推荐做法 | 常见问题 |
| --- | --- | --- |
| 字体 | 先使用默认字体；需要图标时再从 `ghostty +list-fonts` 选择本机存在的字体 | 字体名称写错会回退；图标乱码通常不是 Zsh 故障 |
| 主题 | 优先使用 `ghostty +list-themes` 中的内置主题 | 自定义主题也是配置文件，可能修改颜色之外的选项；使用前应审阅 |
| 快捷键 | 先运行 `ghostty +list-keybinds --default`，只覆盖稳定习惯 | 终端先截获按键；占用 `Ctrl-R`、`Ctrl-T` 会破坏 Atuin 或 fzf |
| 平台修饰键 | macOS 主要使用 `Cmd`，Linux 主要使用 `Ctrl` | 把平台键位硬写进 common 文件会降低可移植性 |

Ghostty 自定义键位的基本格式是：

~~~ini
keybind = trigger=action
~~~

不要先复制整套他人键位。新增一条后立即重新加载并验证终端应用、Zsh 行编辑器以及 TUI 程序中的行为；若同一个按键在三层都有含义，应明确只让其中一层拥有它。

## 5. 保持默认的 Zsh Shell Integration

Ghostty 默认以 `detect` 模式为 Zsh 自动注入 Shell integration，可提供工作目录继承、提示符标记、提示符跳转和更可靠的复杂提示符重绘。正常情况下，不需要在 `.zshrc` 再手工 `source` 一次。

在 Ghostty 内可进行基础检查：

~~~zsh
printf 'TERM=%s\n' "$TERM"
print -r -- "GHOSTTY_RESOURCES_DIR=${GHOSTTY_RESOURCES_DIR:-not-set}"
~~~

只有在 Ghostty 中再次切换 Shell、进入某些隔离环境，或日志明确显示自动注入失败时，才考虑手工加载 Zsh 集成：

~~~zsh
if [[ -n "${GHOSTTY_RESOURCES_DIR:-}" ]]; then
  source "$GHOSTTY_RESOURCES_DIR/shell-integration/zsh/ghostty-integration"
fi
~~~

这段代码应放在其他 Shell 配置之前，并替代自动注入，而不是与自动注入长期并存。若只是想临时排除集成问题，可在 Ghostty 配置中设置 `shell-integration = none`，验证后再恢复默认的 `detect`。

## 6. 处理 SSH 与远端 terminfo

Ghostty 在本地设置 `TERM=xterm-ghostty`。若远端系统没有相应 terminfo，`less`、`vim`、`tmux` 等程序可能报告 `unknown terminal type`。不要在所有机器上全局覆盖 `TERM`；按远端主机处理。

### 优先安装远端 terminfo

本地和远端分别存在 `infocmp`、`tic` 时，可执行：

~~~bash
infocmp -x xterm-ghostty | ssh example.com -- tic -x -
~~~

随后重新连接并检查：

~~~bash
infocmp xterm-ghostty >/dev/null && printf 'terminfo ready\n'
~~~

### 使用 Ghostty 1.3.x 的可选 SSH 集成

SSH 自动处理默认关闭。确认本机版本和行为后，可在 Ghostty 配置中启用：

~~~ini
shell-integration-features = ssh-env,ssh-terminfo
~~~

这只包装当前交互式 Shell 中直接执行的 `ssh`；Git、`scp`、`rsync -e ssh`、脚本、子进程和非交互 Shell 不会继承 Shell 函数。不要据此假设所有 SSH 流量都已处理。

直接命令 `ghostty +ssh` 在 Ghostty 1.3.x 中不可用，它属于 tip 版本并计划进入 1.4。只有下面的探测成功后，才能按当前版本文档使用它：

~~~bash
ghostty +ssh --help
~~~

### 对旧远端保守回退

无法安装 terminfo 且客户端 OpenSSH 不低于 8.7 时，只为对应主机设置：

~~~sshconfig
# ~/.ssh/config
Host legacy-example
  HostName example.com
  SetEnv TERM=xterm-256color
~~~

回退会丢失 Ghostty 超出 `xterm-256color` 的终端能力。远端升级并具备 `xterm-ghostty` 后，应删除该主机的 `SetEnv`。

## 7. 最小验证清单

每次调整后依次检查：

~~~bash
ghostty +version
ghostty +show-config
ghostty +list-fonts
ghostty +list-themes
~~~

然后新建一个标签页，确认工作目录继承、[[Starship 提示符配置|Starship 提示符]]、`Ctrl-R` 的 Atuin 搜索以及 `Ctrl-T` 的 fzf 选择均正常。SSH 问题应在本地终端、远端 terminfo 和目标主机配置之间逐层定位；完整回退流程见 [[现代终端环境更新、验证与回退]]。

## 官方参考资料

- [Ghostty：配置文件、XDG 路径与 config-file](https://ghostty.org/docs/config)
- [Ghostty：配置项参考](https://ghostty.org/docs/config/reference)
- [Ghostty：Shell Integration](https://ghostty.org/docs/features/shell-integration)
- [Ghostty：SSH 集成与版本边界](https://ghostty.org/docs/features/ssh)
- [Ghostty：Terminfo 排障](https://ghostty.org/docs/help/terminfo)
- [Ghostty：主题](https://ghostty.org/docs/features/theme)
- [Ghostty：自定义快捷键](https://ghostty.org/docs/config/keybind)

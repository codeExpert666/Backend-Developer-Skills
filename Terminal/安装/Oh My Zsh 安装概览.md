---
title: Oh My Zsh 安装概览
aliases:
  - Oh My Zsh 安装入口
  - Zsh 与 Oh My Zsh 入门
  - 终端 Oh My Zsh 配置指南
tags:
  - Terminal
  - Terminal/安装
  - Terminal/Shell
  - Zsh
  - Oh-My-Zsh
created: 2026-07-14T00:04:29
updated: 2026-07-19T16:29:30
---

本文是 Linux（以 Ubuntu 为例）和 macOS 默认终端安装 Oh My Zsh 的入口。目标不是堆叠主题和插件，而是先得到一套可验证、可维护、可回退的 Zsh 环境，再按需要完成效率优化。

如果你正在使用 Ubuntu，请阅读 [[Ubuntu 从零安装 Oh My Zsh]]；如果你使用 Mac 自带的 Terminal，请阅读 [[macOS 从零配置 Oh My Zsh]]。两个平台完成基础安装后，再继续 [[Oh My Zsh 常用优化配置]]。

如果你更重视跨机器复现、组件边界和按需升级，可从 [[现代终端环境搭建概览]] 进入 Ghostty、Zsh、Antidote、Starship、Atuin、zoxide 与 fzf 组成的模块化路线。两条路线都以 Zsh 为基础，不需要同时加载 Oh My Zsh 与 Antidote；已有 Oh My Zsh 配置可按 [[从 Oh My Zsh 迁移到 Antidote]] 渐进迁移。

## 先区分终端、Shell 与 Oh My Zsh

“终端”经常被用来泛指整个命令行环境，但以下四层承担的职责不同：

| 名称 | 它解决什么问题 | Ubuntu 中的常见形式 | macOS 中的常见形式 |
| --- | --- | --- | --- |
| 终端模拟器 | 提供窗口、字体、颜色和键盘输入输出 | GNOME Terminal、Konsole | Terminal.app |
| Shell | 解释命令、变量、脚本和补全 | Bash 或 Zsh | macOS 10.15 及以后通常为 Zsh |
| Zsh | 一种功能丰富的 Shell | 通过 apt 安装或已安装 | 系统自带 `/bin/zsh` |
| Oh My Zsh | 管理 Zsh 主题、内置插件和自定义配置的框架 | 安装到用户目录 | 安装到用户目录 |

因此，安装 Oh My Zsh 不会替换 GNOME Terminal 或 Terminal.app，也不会替你选择终端字体和配色；它主要生成并加载 Zsh 配置。主题中的图标显示异常时，通常是终端字体问题，而不是 Oh My Zsh 安装失败。

## 推荐的阅读与操作顺序

1. 先在平台笔记中确认当前 Shell、安装 Zsh，并让新开的终端真正以 Zsh 启动。
2. 备份已有配置后，安装 Oh My Zsh，并执行最小验证。
3. 只启用一个轻量主题和少量内置插件，确认启动速度和命令行为正常。
4. 再按需安装自动建议、语法高亮或 fzf 等外部组件。
5. 后续升级、故障定位、还原和卸载统一参见 [[Oh My Zsh 更新、备份与卸载]]。

> [!important] 安装 Oh My Zsh 不等于更换所有命令行工具
> 先保持系统自带的 `ls`、`cat`、`grep` 等命令与既有脚本不变。`eza`、`bat`、`fzf` 之类的工具应作为单独、可选的增强项；不要为了“美化”在开发机或服务器上批量替换基础命令。

## 安装后会出现哪些文件

默认安装会把框架放在用户主目录的 `~/.oh-my-zsh` 下，并使用 `~/.zshrc` 作为交互式终端的主要配置入口。

| 路径 | 用途 | 是否建议手工修改 |
| --- | --- | --- |
| `~/.oh-my-zsh` | Oh My Zsh 框架、内置主题和内置插件 | 不直接修改；升级会覆盖框架内容 |
| `~/.oh-my-zsh/custom` | 自定义主题、插件与覆盖配置 | 推荐放自己的扩展 |
| `~/.zshrc` | 每个交互式 Zsh 会话读取的用户配置 | 是，主题、插件、历史和别名主要在这里 |
| `~/.zprofile` | 登录 Shell 的配置入口 | 仅放登录环境需要的 PATH 或环境变量 |
| `~/.zsh_history` | 命令历史文件，具体路径由 `HISTFILE` 决定 | 不手工编辑，注意其中可能含敏感命令 |

Zsh 的启动文件顺序并不等同于 Bash：`~/.zshenv` 会被所有 Zsh 进程读取，`~/.zprofile` 只用于登录 Shell，`~/.zshrc` 用于交互式 Shell。日常终端插件和提示符配置应留在 `~/.zshrc`，不要把耗时逻辑塞进 `~/.zshenv`。

## 开始前的共同检查

在任何平台开始前，先记录当前状态。这些命令不会修改配置：

~~~bash
printf 'configured login shell: %s\\n' "$SHELL"
ps -p $$ -o pid=,comm=
command -v zsh || true
zsh --version 2>/dev/null || true
git --version
curl --version
~~~

`$SHELL` 表示账户配置的登录 Shell，而 `ps -p $$ -o comm=` 显示当前窗口实际正在运行的 Shell；两者暂时不同并不一定是错误，例如刚执行完 `chsh` 但尚未新开终端时。

## 安全与备份原则

Oh My Zsh 官方提供一行安装命令，但它本质上仍会下载并执行远程脚本。个人开发机可以使用官方命令，然而第一次接触该项目、受管设备或网络环境异常时，建议先下载、检查，再执行。两个平台笔记都给出了这一方式。

安装器通常会把已有的 `~/.zshrc` 改名为 `~/.zshrc.pre-oh-my-zsh`。这不是完整的配置管理策略：如果你已有个人函数、公司代理、SDK PATH 或 SSH 环境变量，仍应在安装前自行备份并逐项迁移，而不是直接覆盖到新文件里。

不要将令牌、密码、私钥内容或生产数据库连接串写进 `~/.zshrc`。即使配置文件权限正确，命令历史、进程参数、备份和同步工具仍可能扩大暴露范围。

## 最小成功标准

完成任一平台的安装后，新开一个终端并执行：

~~~bash
echo "$ZSH"
omz version
grep '^ZSH_THEME=' ~/.zshrc
~~~

预期是：`$ZSH` 指向 `~/.oh-my-zsh`，`omz version` 能输出版本信息，且 `~/.zshrc` 中存在主题设置。若任一项不符合预期，先不要继续增加插件；请按 [[Oh My Zsh 更新、备份与卸载]] 的“诊断与回退”部分确认配置到底有没有被读取。

## 后续路径

- 基础安装：[[Ubuntu 从零安装 Oh My Zsh]]、[[macOS 从零配置 Oh My Zsh]]
- 日常效率配置：[[Oh My Zsh 常用优化配置]]
- 更新、故障处理和卸载：[[Oh My Zsh 更新、备份与卸载]]
- 模块化替代路线：[[现代终端环境搭建概览]]
- 从当前配置迁移：[[从 Oh My Zsh 迁移到 Antidote]]

## 官方参考资料

- [Oh My Zsh：官方安装与使用说明](https://github.com/ohmyzsh/ohmyzsh)
- [Oh My Zsh：官方网站](https://ohmyz.sh/)
- [Zsh：启动文件说明](https://zsh.sourceforge.io/Doc/Release/Files.html)
- [Apple：在 Mac 上将 zsh 设为默认 Shell](https://support.apple.com/102360)

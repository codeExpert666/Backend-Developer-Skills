---
title: XDG 基础目录与终端配置边界
aliases:
  - XDG Base Directory 基础
  - 终端配置数据状态缓存目录
  - XDG 与 ZDOTDIR
tags:
  - Terminal
  - Terminal/使用
  - XDG
  - Zsh
  - Dotfiles
created: 2026-08-30T22:24:37
updated: 2026-08-31T13:18:23
---

平台搭建主线默认使用 XDG 约定的默认目录。只有在你想理解路径边界、修改默认目录，或排查“程序到底读了哪里”时，才需要打开这篇笔记。

## 1. XDG 解决什么问题

[XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/) 为配置、数据、状态和缓存定义了不同职责。未设置对应环境变量时，常见默认值如下：

| 环境变量 | 默认目录 | 终端环境中的例子 |
| --- | --- | --- |
| `XDG_CONFIG_HOME` | `$HOME/.config` | Zsh、Starship、Atuin、Ghostty 的可编辑配置 |
| `XDG_DATA_HOME` | `$HOME/.local/share` | Atuin 数据、应用下载的数据文件 |
| `XDG_STATE_HOME` | `$HOME/.local/state` | Zsh 历史、可跨会话保留但不适合作为配置的状态 |
| `XDG_CACHE_HOME` | `$HOME/.cache` | Antidote 生成文件、补全缓存、可重新生成的数据 |

这个分类回答的是“文件承担什么职责”，不等于这些目录中的所有内容都要进入 Git。

## 2. 默认值展开不等于设置环境变量

Shell 中常见下面的写法：

```zsh
config_root="${XDG_CONFIG_HOME:-$HOME/.config}"
```

`${name:-fallback}` 由当前 Shell 展开：当 `name` 未设置或为空时使用 `fallback`。它只给变量 `config_root` 赋值，并不会自动 `export XDG_CONFIG_HOME`，也不会改变其他进程看到的环境。

先用只读命令观察当前状态：

```zsh
printf 'XDG_CONFIG_HOME=%q\n' "${XDG_CONFIG_HOME-}"
printf 'effective config root=%q\n' "${XDG_CONFIG_HOME:-$HOME/.config}"
```

因此，应用能在 `$HOME/.config` 工作，不代表系统中一定显式设置了 `XDG_CONFIG_HOME`。

## 3. Git、Stow 与 XDG 不能互相替代

| 机制 | 回答的问题 |
| --- | --- |
| XDG | 目标文件按职责应该位于哪个目录 |
| GNU Stow | 仓库中的源文件如何映射到目标目录 |
| Git | 哪些源文件形成版本历史并可推送到远端 |

例如 Starship 的源文件可以是：

```text
$DOTFILES_DIR/starship/.config/starship.toml
```

部署后的目标路径是：

```text
$HOME/.config/starship.toml
```

XDG 解释目标为什么在 `.config`，Stow 创建这层映射，Git 只跟踪仓库中的源文件。完整的源、目标、软件包和冲突模型见 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]]。

路径职责还决定不了“谁先创建文件”。若 `~/.config/atuin/config.toml` 或 `~/.config/ghostty/config.ghostty` 准备由 Stow 管理，就必须先创建仓库源并部署链接，再运行会读取或生成配置的组件。否则应用可能先写出普通文件，Stow 随后只能报告冲突。data、state 与 cache 则应继续由应用创建，不需要为了 Stow 预建或链接。

## 4. Zsh 为什么还需要 ZDOTDIR

Zsh 默认先寻找 `$HOME/.zshenv`。读取它后，如果设置了 `ZDOTDIR`，后续启动文件会转到该目录。最小入口通常是：

```zsh
# ~/.zshenv，由 Stow 链接到 dotfiles 中的源文件
export ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"
```

于是读取过程变为：

1. Zsh 从 `$HOME/.zshenv` 得到 `ZDOTDIR`；
2. 登录 Shell 再从 `$ZDOTDIR/.zprofile` 读取登录环境；
3. 交互式 Shell 从 `$ZDOTDIR/.zshrc` 读取插件、补全、提示符和交互行为。

这也是 `.zshenv` 必须保持最小的原因：它会影响脚本和非交互 Zsh。启动文件的完整职责与 Antidote 加载顺序见 [[Zsh 与 Antidote 跨机器配置管理]]。

## 5. 默认目录下的配置、状态与缓存

一套常见布局如下：

```text
$HOME/
├── .zshenv                         # 指向 dotfiles 的入口链接
├── .config/
│   ├── zsh/                        # 可版本化的 Zsh 配置与本机私有文件并存
│   ├── starship.toml               # 可版本化配置
│   ├── atuin/config.toml            # 可版本化的非秘密配置
│   └── ghostty/config.ghostty       # 可版本化配置
├── .local/
│   ├── share/                      # 应用数据
│   └── state/zsh/history           # 历史等状态
└── .cache/                         # 可重新生成的缓存
```

是否进入 Git 应按内容判断：

| 内容 | 通常进入 Git | 原因 |
| --- | --- | --- |
| Zsh、Starship、Atuin、Ghostty 的稳定非秘密配置 | 是 | 它们是希望在机器间复现的声明 |
| `local.zsh`、`local.zprofile` | 否 | 可能含机器路径、代理和账号差异 |
| Antidote 本体、插件 clone 与静态加载缓存 | 否 | 本体放在 data；clone 与生成文件可重建，放在 cache |
| Zsh 历史、zoxide 数据库 | 否 | 是持续变化的个人状态 |
| Atuin 密钥、会话和数据库 | 否 | 包含秘密或个人历史，应走 Atuin 自己的机制 |
| 缓存、日志和临时文件 | 否 | 可重新生成且会制造无意义差异 |

即使 dotfiles 是私有仓库，也不应把它当作密钥仓库。

## 6. 使用非默认 XDG 目录前如何判断

先观察变量和启动文件中已有的设置：

```zsh
printenv | grep -E '^(XDG_CONFIG_HOME|XDG_DATA_HOME|XDG_STATE_HOME|XDG_CACHE_HOME|ZDOTDIR)='
grep -RInE 'XDG_(CONFIG|DATA|STATE|CACHE)_HOME|ZDOTDIR' \
  "$HOME/.zshenv" "$HOME/.zprofile" "$HOME/.zshrc" "$HOME/.config/zsh" 2>/dev/null
```

决定修改前逐项确认：

1. 目标程序是否遵循该 XDG 变量，还是使用自己的固定路径；
2. 变量必须影响所有进程，还是只影响交互式 Shell；
3. 旧目录中是否已有需要迁移的数据；
4. Stow 的目标是否随之改变；
5. 登录、SSH、脚本和 GUI 启动的进程能否得到同一个变量；
6. 如何在回退变量后恢复旧目录。

没有明确收益时，优先保留默认值。默认值更容易与系统文档、远程主机和排障命令对齐。

## 7. 最小验证

检查一个干净环境中的 Zsh 是否能找到入口和后续目录：

```zsh
env -i HOME="$HOME" PATH="/usr/bin:/bin" /bin/zsh -c '
  printf "ZDOTDIR=%s\n" "${ZDOTDIR-}"
  printf "config=%s\n" "${XDG_CONFIG_HOME:-$HOME/.config}"
'

ls -ld "$HOME/.zshenv" "${XDG_CONFIG_HOME:-$HOME/.config}/zsh"
readlink "$HOME/.zshenv"
```

第一段只证明非交互 Zsh 能读取 `.zshenv`；它不证明 `.zshrc`、插件或提示符可用。完整验收仍应回到所选平台主线或 [[现代终端环境更新、验证与回退]]。

## 官方资料

- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)
- [Zsh Startup Files](https://zsh.sourceforge.io/Doc/Release/Files.html#Startup_002fShutdown-Files)
- [GNU Stow Manual](https://www.gnu.org/software/stow/manual/stow.html)

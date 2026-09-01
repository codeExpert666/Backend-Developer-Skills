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
updated: 2026-09-01T15:50:00
---

平台搭建主线使用 XDG 约定的默认目录，但**不要求先显式建立四个 `XDG_*_HOME` 环境变量**。变量未设置或为空时使用规范默认值，本身就是正常且完整的配置。只有在你想理解路径边界、修改默认目录，或排查“程序到底读了哪里”时，才需要打开这篇笔记。

## 1. XDG 解决什么问题

[XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/) 为配置、数据、状态和缓存定义了不同职责。未设置对应环境变量时，常见默认值如下：

| 环境变量 | 默认目录 | 终端环境中的例子 |
| --- | --- | --- |
| `XDG_CONFIG_HOME` | `$HOME/.config` | Zsh、Starship、Atuin、Ghostty 的可编辑配置 |
| `XDG_DATA_HOME` | `$HOME/.local/share` | Atuin 数据、应用下载的数据文件 |
| `XDG_STATE_HOME` | `$HOME/.local/state` | Zsh 历史、可跨会话保留但不适合作为配置的状态 |
| `XDG_CACHE_HOME` | `$HOME/.cache` | Antidote 生成文件、补全缓存、可重新生成的数据 |

规范还定义了三个容易与上表混淆的变量：

| 环境变量 | 未设置或为空时的行为 | 边界 |
| --- | --- | --- |
| `XDG_CONFIG_DIRS` | 使用 `/etc/xdg` | 按优先级搜索系统级配置，不替代用户的 `XDG_CONFIG_HOME` |
| `XDG_DATA_DIRS` | 使用 `/usr/local/share:/usr/share` | 按优先级搜索系统级数据，不替代用户的 `XDG_DATA_HOME` |
| `XDG_RUNTIME_DIR` | **没有默认值** | 由登录会话或系统建立，属于当前用户、权限通常为 `0700`，注销或重启后可以消失 |

四个 `XDG_*_HOME` 与 `XDG_RUNTIME_DIR` 若被设置，值必须是绝对路径；`XDG_CONFIG_DIRS`、`XDG_DATA_DIRS` 中以冒号分隔的每一项也必须是绝对路径。相对路径不符合规范，应被程序忽略。尤其不要在 `.zshenv` 中用 `mkdir` 和 `export` 人工合成 `XDG_RUNTIME_DIR`：远程登录、图形会话、容器与服务管理器各自有生命周期和权限边界，缺失时应检查会话环境或让具体程序按其文档降级。

`$HOME/.local/bin` 常与 XDG 默认目录一起出现，但它不是 `XDG_*_HOME` 对应的基础目录，也没有一个需要设置的 XDG 环境变量。本套主线把直接安装到用户账号的命令统一放在这里，并只把这一处加入 PATH；Homebrew、APT 等包管理器仍使用自己的前缀。

这个分类回答的是“文件承担什么职责”，不等于这些目录中的所有内容都要进入 Git，也不表示每个应用都会自动遵循所有 XDG 变量。

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

因此，应用能在 `$HOME/.config` 工作，不代表系统中一定显式设置了 `XDG_CONFIG_HOME`。对采用默认布局的机器，额外写入下面四行通常只是在重复规范默认值，并会新增一处必须让登录、SSH、脚本和 GUI 会话保持一致的配置源：

```zsh
export XDG_CONFIG_HOME="$HOME/.config"
export XDG_DATA_HOME="$HOME/.local/share"
export XDG_STATE_HOME="$HOME/.local/state"
export XDG_CACHE_HOME="$HOME/.cache"
```

本套主线因此**不写这四行**，而是在每个实际引用处使用 `${XDG_*_HOME:-默认值}`。若确实选择非默认目录，就必须把它当作一项路径迁移，而不是为了让变量“看起来完整”而增加赋值。

## 3. Git、Stow 与 XDG 不能互相替代

| 机制 | 回答的问题 |
| --- | --- |
| XDG | 目标文件按职责应该位于哪个目录 |
| GNU Stow | 仓库中的源文件如何映射到目标目录 |
| Git | 哪些源文件形成版本历史并可推送到远端 |

例如 Starship 的默认源与两个备用 profile 可以是：

```text
$DOTFILES_DIR/starship/.config/starship.toml
$DOTFILES_DIR/starship/.config/starship/profiles/tokyo-night.toml
$DOTFILES_DIR/starship/.config/starship/profiles/catppuccin-mocha.toml
```

部署后的目标路径分别是：

```text
$HOME/.config/starship.toml
$HOME/.config/starship/profiles/tokyo-night.toml
$HOME/.config/starship/profiles/catppuccin-mocha.toml
```

XDG 解释这些目标为什么在 `.config`，Stow 创建这层映射，Git 只跟踪仓库中的源文件。未设置 `STARSHIP_CONFIG` 时，Starship 读取默认文件；设置后读取指定的备用 profile。`STARSHIP_CONFIG` 是 Starship 自己的配置选择变量，不属于 XDG 规范；它只选择配置，不会把 profile 变成 state 或 cache，也不会改变 Stow 所有权。完整的源、目标、软件包和冲突模型见 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]]。

路径职责还决定不了“谁先创建文件”。若 `~/.config/atuin/config.toml` 或 `~/.config/ghostty/config.ghostty` 准备由 Stow 管理，就必须先创建仓库源并部署链接，再运行会读取或生成配置的组件。否则应用可能先写出普通文件，Stow 随后只能报告冲突。data、state 与 cache 则应继续由应用创建，不需要为了 Stow 预建或链接。

## 4. Zsh 为什么还需要 ZDOTDIR

Zsh 启动时若尚未设置 `ZDOTDIR`，会用 `$HOME` 寻找 `.zshenv`。读取这个根入口后，在当前进程设置 `ZDOTDIR`，后续启动文件才转到该目录。最小入口通常是：

```zsh
# ~/.zshenv，由 Stow 链接到 dotfiles 中的源文件
# 不导出；子 Zsh 需要重新从 $HOME/.zshenv 取得入口与早期启动开关。
ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"
```

这里不能机械写成 `export ZDOTDIR=...`。若已配置的父 Zsh 把 `ZDOTDIR` 传给子进程，子 Zsh 会直接寻找 `$ZDOTDIR/.zshenv`；当前布局没有管理第二份 `.zshenv`，于是根入口以及其中非导出的 `skip_global_compinit` 都会被跳过。保持 `ZDOTDIR` 为当前 Zsh 的普通参数，既能重定向本次启动的后续文件，也能让每个子 Zsh 重新经过同一个根入口。

于是读取过程变为：

1. Zsh 在 `ZDOTDIR` 尚未设置时从 `$HOME/.zshenv` 得到当前进程的 `ZDOTDIR`；
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
│   ├── starship/profiles/          # 可版本化的 Starship 备用配置
│   ├── atuin/
│   │   ├── config.toml              # 可版本化的非秘密配置链接
│   │   └── atuin-receipt.json       # 安装器生成的本机元数据，不进入 Git
│   └── ghostty/config.ghostty       # 可版本化配置
├── .local/
│   ├── bin/                         # 直接安装的用户级命令，只把这一处加入 PATH
│   ├── share/                       # Atuin 数据、Antidote 本体与其他应用数据
│   └── state/
│       ├── zsh/history              # 原生 Zsh 历史
│       └── atuin/logs/              # Atuin 可跨进程保留的日志
└── .cache/                         # 可重新生成的缓存
```

是否进入 Git 应按内容判断：

| 内容 | 通常进入 Git | 原因 |
| --- | --- | --- |
| Zsh、Starship、Atuin、Ghostty 的稳定非秘密配置 | 是 | 它们是希望在机器间复现的声明 |
| `local.zsh`、`local.zprofile` | 否 | 可能含机器路径、代理和账号差异 |
| Antidote 本体、插件 clone 与静态加载缓存 | 否 | 本体放在 data；clone 与生成文件可重建，放在 cache |
| Zsh 历史、Atuin 日志 | 否 | 是持续变化、需要跨进程保留的本机状态 |
| zoxide 数据库 | 否 | Linux/BSD 通常位于 XDG data；macOS 默认位于原生 Application Support，都是本机运行数据 |
| Atuin 密钥、会话和数据库 | 否 | 包含秘密或个人历史，应走 Atuin 自己的机制 |
| 缓存和临时文件 | 否 | 可重新生成且会制造无意义差异 |

即使 dotfiles 是私有仓库，也不应把它当作密钥仓库。

“某个文件承担 cache 职责”不表示程序会自动遵循 `XDG_CACHE_HOME`。两个常见默认行为尤其容易把生成文件写回配置目录：

- 裸 `compinit` 默认把 `.zcompdump` 写到 `$ZDOTDIR` 或 `$HOME`；本方案使用 `compinit -d "${XDG_CACHE_HOME:-$HOME/.cache}/..."` 显式改写目标，并在 Ubuntu 上通过早期 `skip_global_compinit` 防止系统级 `zshrc` 抢先初始化；
- 单参数 `antidote load "$ZDOTDIR/.zsh_plugins.txt"` 会从清单名推导相邻的 `.zsh_plugins.zsh`；本方案把 XDG cache 中的静态脚本路径作为第二个参数传入。

因此，在 `$ZDOTDIR` 看到普通 `.zcompdump` 或 `.zsh_plugins.zsh` 时，结论不是“它可以进入 dotfiles”，而是“至少有一次初始化仍走默认生成路径”。应先定位系统级与用户级调用，再移动可重建旧文件；完整顺序和检查命令见 [[Zsh 与 Antidote 跨机器配置管理]]。

应用原生路径不一定构成偏离：zoxide 在 macOS 默认把数据库放入 `~/Library/Application Support`，这符合平台惯例，本套主线不为了目录外观强制迁移；Ghostty 在 macOS 还会读取同一目录下的配置，而且该配置可能晚于 XDG 配置加载，因此主线会专门检查并停用会覆盖受管 XDG 配置的旧文件。判断重点是职责、加载顺序和所有权，而不是把所有路径机械改成同一种形状。

## 6. 使用非默认 XDG 目录前如何判断

四条标准搭建主线固定使用默认布局：变量可以未设置、为空，或显式等于规范默认值；若发现相对路径或其他绝对路径，就在写文件、安装用户级命令或运行 Stow 之前停止。它们复用下面的只读门禁：

```sh
xdg_layout_ok=1

for xdg_name in \
  XDG_CONFIG_HOME \
  XDG_DATA_HOME \
  XDG_STATE_HOME \
  XDG_CACHE_HOME; do
  case "$xdg_name" in
    XDG_CONFIG_HOME) xdg_default="$HOME/.config" ;;
    XDG_DATA_HOME)   xdg_default="$HOME/.local/share" ;;
    XDG_STATE_HOME)  xdg_default="$HOME/.local/state" ;;
    XDG_CACHE_HOME)  xdg_default="$HOME/.cache" ;;
  esac

  # 名称来自上面的固定列表；读取当前 Shell 参数，也能发现尚未 export 的旧赋值。
  eval "xdg_value=\${$xdg_name-}"
  printf '%s=%s; effective=%s\n' \
    "$xdg_name" "${xdg_value:-not set}" "${xdg_value:-$xdg_default}"

  case "$xdg_value" in
    "") ;;
    /*)
      if [ "$xdg_value" != "$xdg_default" ]; then
        printf 'STOP: standard mainline uses the default XDG layout: %s\n' \
          "$xdg_name" >&2
        xdg_layout_ok=0
      fi
      ;;
    *)
      printf 'STOP: %s must be an absolute path: %s\n' \
        "$xdg_name" "$xdg_value" >&2
      xdg_layout_ok=0
      ;;
  esac
done

xdg_runtime_value="${XDG_RUNTIME_DIR-}"
printf 'XDG_RUNTIME_DIR=%s\n' "${xdg_runtime_value:-not set}"
case "$xdg_runtime_value" in
  ""|/*) ;;
  *)
    printf 'STOP: XDG_RUNTIME_DIR must be an absolute path: %s\n' \
      "$xdg_runtime_value" >&2
    xdg_layout_ok=0
    ;;
esac

if [ "$xdg_layout_ok" -ne 1 ]; then
  unset xdg_default xdg_layout_ok xdg_name xdg_runtime_value xdg_value
  false
else
  unset xdg_default xdg_layout_ok xdg_name xdg_runtime_value xdg_value
fi
```

这段命令不会建立或清空任何 XDG 变量；`XDG_RUNTIME_DIR` 不要求存在或等于某个默认值，只在已经设置时检查它是否为绝对路径。出现 `STOP` 时，不要为了继续主线而临时 `unset` 变量，否则应用仍可能在旧目录留下配置、数据或状态，GUI 与 SSH 会话也可能继续获得不同环境。

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

选择非默认目录后，需要同时调整并迁移 Git/Stow 目标、已有配置、data、state、cache，以及登录、SSH、脚本和 GUI 的变量来源；然后分别验证旧路径和新路径。标准主线没有覆盖这项自定义迁移。没有明确收益时，优先保留默认值，它更容易与系统文档、远程主机和排障命令对齐。

## 7. 最小验证

检查一个干净环境中的 Zsh 是否能找到入口和后续目录：

```zsh
env -i HOME="$HOME" PATH="/usr/bin:/bin" /bin/zsh -c '
  printf "ZDOTDIR=%s\n" "${ZDOTDIR-}"
  printf "config=%s\n" "${XDG_CONFIG_HOME:-$HOME/.config}"
'

ls -ld "$HOME/.zshenv" "${XDG_CONFIG_HOME:-$HOME/.config}/zsh"
readlink "$HOME/.zshenv"
command -v starship atuin zoxide fzf 2>/dev/null || true
```

直接安装的用户级工具应解析到 `$HOME/.local/bin`；Homebrew 或 APT 安装的工具则应解析到对应包管理器前缀。第一段只证明非交互 Zsh 能读取 `.zshenv`；它不证明 `.zshrc`、插件或提示符可用。完整验收仍应回到所选平台主线或 [[现代终端环境更新、验证与回退]]。

## 官方资料

- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)
- [Zsh Startup Files](https://zsh.sourceforge.io/Doc/Release/Files.html#Startup_002fShutdown-Files)
- [Zsh Completion System](https://zsh.sourceforge.io/Doc/Release/Completion-System.html)
- [Antidote Commands](https://antidote.sh/commands)
- [GNU Stow Manual](https://www.gnu.org/software/stow/manual/stow.html)

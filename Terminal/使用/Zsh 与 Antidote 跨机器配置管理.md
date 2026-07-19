---
title: Zsh 与 Antidote 跨机器配置管理
aliases:
  - Zsh 跨机器配置
  - Antidote 插件管理
  - Zsh 与 Antidote 配置骨架
tags:
  - Terminal
  - Terminal/使用
  - Terminal/Shell
  - Zsh
  - Antidote
  - Dotfiles
created: 2026-07-19T16:35:26
updated: 2026-07-19T16:53:12
---

本文给出一套可在 macOS 与 Ubuntu 间复用的 Zsh 配置骨架。Antidote 只管理 Zsh 插件；Starship、Atuin、zoxide 和 fzf 都是独立二进制，由系统包管理器安装，再从 `.zshrc` 接入。整体安装顺序见 [[现代终端环境搭建概览]]，从 Oh My Zsh 切换时先阅读 [[从 Oh My Zsh 迁移到 Antidote]]。

## 配置目标

| 层 | 组件 | 本文中的职责 |
| --- | --- | --- |
| 终端模拟器 | Ghostty | 本地窗口、字体、配色与终端协议，见 [[Ghostty 常用配置与 Shell 集成]] |
| Shell | Zsh | 启动文件、环境变量、补全、行编辑和基础历史 |
| 插件管理 | Antidote | 下载并加载少量 Zsh 插件 |
| 提示符 | Starship | 渲染 prompt，见 [[Starship 提示符配置]] |
| 交互工具 | Atuin、zoxide、fzf | 历史搜索、目录跳转、模糊选择 |

跨机器维护的关键不是把整个主目录同步，而是只同步声明式配置，让安装目录、缓存、数据库、密钥和本机覆盖留在各自设备。

## 1. 明确 Zsh 启动文件边界

| 文件 | 读取时机 | 适合保存的内容 |
| --- | --- | --- |
| `~/.zshenv` | 每次启动 Zsh，包括非交互命令 | `ZDOTDIR` 和必须对 SSH 非交互命令可见的最小 PATH |
| `$ZDOTDIR/.zprofile` | 登录 Shell | 登录会话环境，不放别名、插件和提示符 |
| `$ZDOTDIR/.zshrc` | 交互式 Shell | 选项、补全、插件、按键、Atuin、zoxide、fzf、Starship |

`ZDOTDIR` 尚未设置时，Zsh 先从主目录读取 `~/.zshenv`。在这里设置 `ZDOTDIR` 后，后续用户启动文件会改从 `~/.config/zsh/` 读取。不要再同时维护根目录下的 `.zprofile` 和 `.zshrc`。

建议的目录结构为：

~~~text
~/.zshenv
~/.config/zsh/
├── .zprofile
├── .zshrc
├── .zsh_plugins.txt
├── common.zsh
├── macos.zsh
├── linux.zsh
├── local.zsh
├── local.zprofile
└── snapshots/
    └── antidote.txt
~~~

先创建目录：

~~~bash
mkdir -p "$HOME/.config/zsh/snapshots"
mkdir -p "${XDG_DATA_HOME:-$HOME/.local/share}"
~~~

## 2. 保持 `.zshenv` 足够小

`~/.zshenv` 只负责重定向配置目录，以及暴露确实要在 `ssh host command` 等非交互场景中使用的命令目录：

~~~zsh
# ~/.zshenv
export ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"

typeset -U path PATH
[[ -d /usr/local/bin ]] && path=(/usr/local/bin $path)
[[ -d /opt/homebrew/bin ]] && path=(/opt/homebrew/bin $path)
[[ -d "$HOME/.local/share/fzf/bin" ]] && path=("$HOME/.local/share/fzf/bin" $path)
[[ -d "$HOME/.atuin/bin" ]] && path=("$HOME/.atuin/bin" $path)
[[ -d "$HOME/.local/bin" ]] && path=("$HOME/.local/bin" $path)
export PATH
~~~

不要在 `.zshenv` 中运行 `brew shellenv`、语言版本管理器、网络命令、补全、插件或提示符初始化。它会影响脚本、IDE 后端和非交互 SSH，任何耗时或输出都会被放大。

如果某个工具只需要在登录终端使用，把可共享部分直接放进 `$ZDOTDIR/.zprofile`，本机差异放进 `local.zprofile`：

~~~zsh
# ~/.config/zsh/.zprofile
# 只在登录 Shell 中加载 Homebrew 的完整环境；Ubuntu 上会自然跳过。
if [[ -x /opt/homebrew/bin/brew ]]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [[ -x /usr/local/bin/brew ]]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

[[ -r "$ZDOTDIR/local.zprofile" ]] && source "$ZDOTDIR/local.zprofile"
~~~

## 3. 安装到跨平台一致的 Antidote 路径

共享 `.zshrc` 不应依赖 Homebrew 或某个 Linux 包的安装位置。主路线统一把 Antidote 本体放到 XDG 数据目录：

~~~bash
antidote_root="${XDG_DATA_HOME:-$HOME/.local/share}"
antidote_dir="$antidote_root/antidote"
mkdir -p "$antidote_root"
if [ ! -d "$antidote_dir/.git" ]; then
  git clone --depth=1 https://github.com/mattmc3/antidote.git "$antidote_dir"
fi
test -r "$antidote_dir/antidote.zsh"
~~~

这个目录是可重新下载的工具安装目录，不进入 dotfiles 仓库。若已通过 Homebrew 安装 Antidote，可以按包管理器说明加载其路径，但不要把该平台路径写进共享 `.zshrc`。

## 4. 声明最小插件清单

创建 `$ZDOTDIR/.zsh_plugins.txt`：

~~~text
# 输入建议由 Antidote 正常加载。
zsh-users/zsh-autosuggestions

# 只克隆，不在这里 source；必须等所有 ZLE widget 创建后再最后加载。
zsh-users/zsh-syntax-highlighting kind:clone
~~~

这里只保留两个职责单一的插件。Zsh 已有内置补全，因此基础方案不额外安装 `zsh-completions`。若以后加入 `kind:fpath` 类型的补全插件，必须先让 Antidote 把目录加入 `fpath`，再执行 `compinit`；不能照搬本文“先内置 compinit、后加载插件”的顺序。

常用 bundle 标注如下：

| 写法 | 作用 | 使用建议 |
| --- | --- | --- |
| `owner/repo` | 按默认 `kind:zsh` 查找并加载插件文件 | 普通 Zsh 插件 |
| `kind:clone` | 只克隆，不自动加载 | 需要稍后手工加载或只取资源时 |
| `kind:fpath path:src` | 把子目录加入 `fpath` | 必须放在 `compinit` 之前生效 |
| `path:plugins/name` | 只加载仓库中的指定路径 | 从框架中选取单个插件时 |
| `branch:name` | 跟随指定分支 | 仅在上游明确要求时使用 |
| `pin:<完整提交 SHA>` | 锁定到提交，`antidote update` 会跳过 | 对可复现性要求严格时使用 |
| `conditional:function_name` | 零参数函数返回成功时才加载 | 平台或能力判断；函数须在 `antidote load` 前定义 |

例如，只在 macOS 加载一个经过确认的插件路径：

~~~text
ohmyzsh/ohmyzsh path:plugins/macos conditional:antidote_is_macos
~~~

对应函数必须先出现在 `.zshrc`：

~~~zsh
antidote_is_macos() {
  [[ "$OSTYPE" == darwin* ]]
}
~~~

不要同时使用 `branch:` 和 `pin:` 表达两种相反的更新策略。锁定时使用完整提交 SHA，不要用可能移动的短分支名代替版本。

## 5. 使用唯一、明确的 `.zshrc` 加载顺序

下面的骨架可直接保存为 `~/.config/zsh/.zshrc`。它使用 Zsh 内置补全，让 fzf 保留 `Ctrl-T`，让 Atuin 独占 `Ctrl-R`，让 zoxide 的 `zi` 统一负责交互式目录选择，并确保语法高亮真正最后加载：

~~~zsh
# ~/.config/zsh/.zshrc

# 1. 交互选项与原生历史回退。
bindkey -e

zsh_state_dir="${XDG_STATE_HOME:-$HOME/.local/state}/zsh"
mkdir -p "$zsh_state_dir"
HISTFILE="$zsh_state_dir/history"
HISTSIZE=50000
SAVEHIST=20000
setopt append_history
setopt hist_expire_dups_first
setopt hist_ignore_dups
setopt hist_ignore_space
setopt hist_reduce_blanks
unset zsh_state_dir

# 2. common -> platform -> local；local 文件不进入 Git。
[[ -r "$ZDOTDIR/common.zsh" ]] && source "$ZDOTDIR/common.zsh"
case "$OSTYPE" in
  darwin*) [[ -r "$ZDOTDIR/macos.zsh" ]] && source "$ZDOTDIR/macos.zsh" ;;
  linux*)  [[ -r "$ZDOTDIR/linux.zsh" ]] && source "$ZDOTDIR/linux.zsh" ;;
esac
[[ -r "$ZDOTDIR/local.zsh" ]] && source "$ZDOTDIR/local.zsh"

# 3. 基础方案只使用 Zsh 内置补全。
zsh_cache_dir="${XDG_CACHE_HOME:-$HOME/.cache}/zsh"
mkdir -p "$zsh_cache_dir"
autoload -Uz compinit
compinit -d "$zsh_cache_dir/zcompdump-$ZSH_VERSION"
unset zsh_cache_dir

# 4. Antidote 管理 Zsh 插件；本体路径在 macOS 与 Ubuntu 上保持一致。
export ANTIDOTE_HOME="${XDG_CACHE_HOME:-$HOME/.cache}/antidote"
antidote_root="${XDG_DATA_HOME:-$HOME/.local/share}/antidote"
antidote_is_macos() {
  [[ "$OSTYPE" == darwin* ]]
}

if [[ -r "$antidote_root/antidote.zsh" ]]; then
  source "$antidote_root/antidote.zsh"
  ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=8'
  antidote load "$ZDOTDIR/.zsh_plugins.txt"
fi
unset antidote_root
unfunction antidote_is_macos 2>/dev/null

# 5. fzf 提供 Ctrl-T 和模糊补全；Ctrl-R 给 Atuin，目录交互统一使用 zi。
if (( $+commands[fzf] )) && fzf --zsh >/dev/null 2>&1; then
  FZF_CTRL_R_COMMAND= FZF_ALT_C_COMMAND= source <(fzf --zsh)
fi

# 6. Atuin 独占 Ctrl-R；保留原生上箭头，并显式关闭额外 AI 键位。
if (( $+commands[atuin] )); then
  eval "$(atuin init zsh --disable-up-arrow --disable-ai)"
fi

# 7. zoxide 必须在 compinit 之后初始化，zi 可复用 fzf。
if (( $+commands[zoxide] )); then
  eval "$(zoxide init zsh)"
fi

# 8. Starship 只负责提示符。
if (( $+commands[starship] )); then
  eval "$(starship init zsh)"
fi

# 9. 所有可能创建 ZLE widget 的工具完成后，最后加载语法高亮。
if (( $+functions[antidote] )); then
  zsh_highlight_root="$(antidote path zsh-users/zsh-syntax-highlighting 2>/dev/null)"
  if [[ -r "$zsh_highlight_root/zsh-syntax-highlighting.zsh" ]]; then
    source "$zsh_highlight_root/zsh-syntax-highlighting.zsh"
  fi
  unset zsh_highlight_root
fi
~~~

这里不能只把 `zsh-syntax-highlighting` 写在插件清单最后。fzf、Atuin 和 zoxide 会在它之后创建或替换 ZLE widget，因此必须使用 `kind:clone`，等所有交互工具初始化完成后再手工 `source`。

`fzf --zsh` 只存在于较新的 fzf。若条件判断未通过，先查看 `fzf --help` 以及当前包管理器提供的集成文件，不要在共享配置里硬编码某台机器的 `/usr/share/...` 或 Homebrew 路径。这里还禁用了 fzf 的 `Alt-C`，避免与 zoxide 的目录导航形成两套入口；交互式选目录统一使用 `zi`。Atuin 和 zoxide 的具体使用见 [[Atuin 命令历史管理]] 与 [[zoxide 与 fzf 导航和模糊查找]]。

## 6. 划分 common、platform 与 local

`common.zsh` 只放所有机器都成立的交互习惯，例如安全别名和通用环境变量：

~~~zsh
# ~/.config/zsh/common.zsh
alias gss='git status --short'
alias glog='git log --oneline --graph --decorate -20'
~~~

`macos.zsh` 和 `linux.zsh` 只保存平台差异；不要用它们复制一整份 `.zshrc`。主机名、公司代理、临时 SDK、私有仓库地址和秘密放入 `local.zsh` 或专用秘密管理工具，并让 `local.zsh` 保持不跟踪。

## 7. 选择 pin 或 snapshot 保持可复现

少量关键插件可以直接在 `.zsh_plugins.txt` 使用 `pin:<完整提交 SHA>`。如果更希望正常更新、但保留可回退点，则使用 Antidote snapshot：

~~~zsh
mkdir -p "$ZDOTDIR/snapshots"
antidote snapshot save "$ZDOTDIR/snapshots/antidote.txt"
~~~

在另一台机器恢复已记录的提交：

~~~zsh
antidote snapshot restore "$ZDOTDIR/snapshots/antidote.txt"
~~~

snapshot 文件可以进入 Git；Antidote 默认数据目录中的自动快照不必全部提交。每次更新前保存一个经过验证的快照即可，升级和回退流程见 [[现代终端环境更新、验证与回退]]。

## 8. 明确 Git 跟踪边界

| 进入 Git | 不进入 Git |
| --- | --- |
| `~/.zshenv` 的模板或符号链接源文件 | `.zsh_plugins.zsh` 生成文件 |
| `.zprofile`、`.zshrc` | Antidote 本体、克隆缓存和下载目录 |
| `.zsh_plugins.txt` | `local.zsh`、`local.zprofile` |
| `common.zsh`、`macos.zsh`、`linux.zsh` | 原生 Zsh 历史文件 |
| 有意保存的 Antidote snapshot | Atuin 历史数据库、密钥和会话数据 |
| `starship.toml`、Ghostty 配置、脱敏后的 Atuin 配置 | zoxide 数据库、Starship 缓存和日志 |

秘密、令牌、私有代理和机器专属路径既不应进入 Git，也不应伪装成“跨平台默认值”。安装目录和缓存丢失后应能由声明式配置重新生成。

## 9. 新机器恢复与验证

先部署 dotfiles，再安装独立二进制，最后克隆 Antidote：

~~~bash
antidote_root="${XDG_DATA_HOME:-$HOME/.local/share}"
antidote_dir="$antidote_root/antidote"
mkdir -p "$antidote_root"
if [ ! -d "$antidote_dir/.git" ]; then
  git clone --depth=1 https://github.com/mattmc3/antidote.git "$antidote_dir"
fi

zsh -n "$HOME/.zshenv"
ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"
zsh -n "$ZDOTDIR/.zprofile"
zsh -n "$ZDOTDIR/.zshrc"
~~~

验证非交互 SSH 可见的最小 PATH，不依赖当前终端已经导出的环境：

~~~bash
env -i HOME="$HOME" PATH=/usr/bin:/bin:/usr/sbin:/sbin SHELL=/bin/zsh \
  /bin/zsh -c 'printf "ZDOTDIR=%s\nPATH=%s\n" "$ZDOTDIR" "$PATH"'
~~~

再启动登录交互 Shell，逐项检查组件和按键归属：

~~~zsh
exec zsh -l
~~~

~~~zsh
command -v atuin zoxide fzf starship
antidote list
bindkey '^R'
type z zi
time zsh -i -c exit
~~~

`Ctrl-R` 应打开 Atuin，`Ctrl-T` 应由 fzf 选择文件，`Alt-C` 不应被 fzf 绑定，`z` / `zi` 应由 zoxide 提供；输入有效和无效命令时应出现语法着色。任何一项失败，都先注释最后一个初始化块并重新执行 `zsh -n`，不要同时重排所有组件。

## 官方参考资料

- [Zsh：启动文件](https://zsh.sourceforge.io/Doc/Release/Files.html)
- [Zsh：选项](https://zsh.sourceforge.io/Doc/Release/Options.html)
- [Antidote：安装、bundle 语法、条件与 snapshot](https://antidote.sh/)
- [fzf：Zsh 集成与禁用 Ctrl-R](https://github.com/junegunn/fzf)
- [Atuin：Shell Integration](https://docs.atuin.sh/cli/guide/shell-integration/)
- [Atuin：默认键位开关](https://docs.atuin.sh/cli/faq/)
- [zoxide：Zsh 初始化顺序](https://github.com/ajeetdsouza/zoxide)
- [Starship：Zsh 初始化](https://starship.rs/guide/)
- [zsh-syntax-highlighting：加载顺序](https://github.com/zsh-users/zsh-syntax-highlighting)

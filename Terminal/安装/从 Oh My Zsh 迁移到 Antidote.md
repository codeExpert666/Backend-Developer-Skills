---
title: 从 Oh My Zsh 迁移到 Antidote
aliases:
  - Oh My Zsh 迁移 Antidote
  - Zsh 框架迁移
  - Oh My Zsh 现代化迁移
tags:
  - Terminal
  - Terminal/安装
  - Terminal/迁移
  - Zsh
  - Oh-My-Zsh
  - Antidote
created: 2026-07-19T16:30:50
updated: 2026-07-19T17:05:02
---

本文把现有 Oh My Zsh 环境迁移为 Ghostty + Zsh + Antidote + Starship + Atuin + zoxide + fzf。迁移的重点不是删除 Oh My Zsh，而是先盘点现有能力，在新的 `~/.config/zsh` 中并行重建，显式测试后再切换 `ZDOTDIR`；旧 `~/.zshrc` 与 `~/.oh-my-zsh` 会保留到观察期结束。

新环境的分层与文件边界见 [[现代终端环境搭建概览]]。平台安装命令分别见 [[macOS 从零搭建现代终端环境]] 和 [[Ubuntu 从零搭建现代终端环境]]；迁移完成后的维护见 [[现代终端环境更新、验证与回退]]。

## 迁移原则

1. 先记录主题、插件、别名、函数、PATH 和自定义目录，再决定每项去向。
2. 在 `~/.config/zsh` 准备新配置，不覆盖仍在工作的 `~/.zshrc`。
3. Oh My Zsh 主题由 Starship 替代，历史搜索由 Atuin 负责，目录跳转由 zoxide 负责。
4. 只有确实依赖的 Oh My Zsh 子插件才通过 Antidote 选择性加载，不再启动整个框架。
5. Antidote 只管理 Zsh 插件；Ghostty、Starship、Atuin、zoxide 和 fzf 仍由平台包管理器或官方安装器管理。
6. 至少保留一个已登录的终端或 SSH 会话，直到新开的会话通过全部验证。
7. 观察期结束前不卸载 Oh My Zsh、不删除旧配置，也不清理历史文件。

## 1. 记录当前状态

在当前 Oh My Zsh 会话执行：

~~~zsh
printf 'login shell: %s\n' "$SHELL"
printf 'current shell: '
ps -p $$ -o comm=
printf 'Oh My Zsh root: %s\n' "$ZSH"
omz version
printf 'theme: %s\n' "$ZSH_THEME"
print -rl -- ${plugins[@]}

command -v starship atuin zoxide fzf 2>/dev/null || true
~~~

再检查主配置中的环境与手工加载项：

~~~bash
grep -nE '^(export |path=|PATH=|ZSH_THEME=|plugins=|source |alias |function )' \
  "$HOME/.zshrc" || true

find "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}" \
  -maxdepth 3 -type d -name .git -print 2>/dev/null
~~~

第二条命令只列出 custom 目录中的 Git 仓库，不会修改插件。不要只抄 `plugins=(...)`：公司代理、SDK 初始化、自定义函数和 PATH 往往才是迁移后“命令不见了”的真正原因。

## 2. 创建带清单的备份

执行：

~~~zsh
backup_dir="$HOME/.terminal-backups/omz-migration-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"

for file in "$HOME/.zshenv" "$HOME/.zprofile" "$HOME/.zshrc"; do
  [[ -f "$file" ]] && cp -p "$file" "$backup_dir/"
done

[[ -f "$HOME/.zshenv" ]] || touch "$backup_dir/no-original-zshenv"

legacy_history_file="${HISTFILE:-$HOME/.zsh_history}"
if [[ ! -f "$legacy_history_file" && -f "$HOME/.zhistory" ]]; then
  legacy_history_file="$HOME/.zhistory"
fi
printf '%s\n' "$legacy_history_file" > "$backup_dir/zsh-history-source.txt"
[[ -f "$legacy_history_file" ]] && cp -p "$legacy_history_file" "$backup_dir/zsh-history"
unset legacy_history_file

if [[ -d "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}" ]]; then
  cp -R "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}" "$backup_dir/omz-custom"
fi

print -rl -- ${plugins[@]} > "$backup_dir/omz-plugins.txt"
alias > "$backup_dir/aliases.txt"
printf 'backup created: %s\n' "$backup_dir"
~~~

`no-original-zshenv` 用于区分“迁移前没有 `.zshenv`”与“忘记备份”。记下 `backup_dir` 的真实值，回退命令需要使用它。备份可能包含代理、内部域名或其他敏感信息，不要直接提交到公开仓库。

## 3. 把旧能力映射到新组件

先逐项决定保留、替代或删除：

| Oh My Zsh 中的能力 | 新环境去向 | 迁移建议 |
| --- | --- | --- |
| `ZSH_THEME` 与第三方主题 | Starship | 新配置删除 `ZSH_THEME`，只保留一次 `starship init zsh` |
| `z`、`zsh-z` 或 autojump | zoxide | 初始化后用 `z`、`zi`，可导入旧目录数据 |
| fzf 插件或手工 `key-bindings.zsh` | 独立 fzf | 只初始化一次；禁用 fzf 的 `Ctrl-R` 与 `Alt-C` |
| 原生 `Ctrl-R` 或历史搜索插件 | Atuin | `Ctrl-R` 交给 Atuin，上箭头保持原生；同步可选 |
| `zsh-autosuggestions` | Antidote 同名插件 | 在插件清单正常加载 |
| `zsh-syntax-highlighting` | Antidote 下载、`.zshrc` 最后加载 | 清单使用 `kind:clone`，不能提前 source |
| `git`、`sudo`、`extract` 等 OMZ 插件 | Antidote 选择性加载 OMZ 子目录 | 只保留确实使用的别名或函数 |
| 自定义 alias 与 function | `common.zsh`、`macos.zsh`、`linux.zsh` | 先去重，再按平台拆分 |
| PATH、SDK 和登录环境 | `.zshenv`、`.zprofile`、`local.zprofile` | `.zshenv` 只留最小 PATH，登录初始化进入 `.zprofile` |
| 代理、令牌和机器私有目录 | `local.zsh` 或秘密管理工具 | 不提交到配置仓库 |

以下内容通常不应原样迁移：Oh My Zsh 自动更新设置、`source "$ZSH/oh-my-zsh.sh"`、旧主题变量、OMZ 的 `z` 插件、重复的 fzf 初始化，以及由旧框架生成的补全缓存。

## 4. 先安装新组件，不修改旧启动链路

按当前平台笔记完成以下安装步骤，但暂时不要写入或替换 `~/.zshenv`：

- macOS：安装 Ghostty、Starship、Atuin、zoxide 和 fzf，参见 [[macOS 从零搭建现代终端环境]]；
- Ubuntu：区分 Desktop 与 Server，安装命令行组件，参见 [[Ubuntu 从零搭建现代终端环境]]；
- 两个平台都把 Antidote 安装到 `${XDG_DATA_HOME:-$HOME/.local/share}/antidote`。

检查：

~~~zsh
antidote_dir="${XDG_DATA_HOME:-$HOME/.local/share}/antidote"
test -r "$antidote_dir/antidote.zsh"

for command_name in starship atuin zoxide fzf; do
  if (( $+commands[$command_name] )); then
    printf '%-10s %s\n' "$command_name" "$commands[$command_name]"
  else
    printf '%-10s missing\n' "$command_name"
  fi
done
~~~

此时旧 Oh My Zsh 仍是当前会话的启动框架。不要把 Antidote 再 `source` 到旧 `~/.zshrc`，否则会形成两个插件管理体系同时工作。

## 5. 在新目录准备配置

创建新目录和文件：

~~~bash
mkdir -p "$HOME/.config/zsh"
touch "$HOME/.config/zsh/.zprofile"
touch "$HOME/.config/zsh/.zshrc"
touch "$HOME/.config/zsh/.zsh_plugins.txt"
touch "$HOME/.config/zsh/common.zsh"
touch "$HOME/.config/zsh/macos.zsh"
touch "$HOME/.config/zsh/linux.zsh"
touch "$HOME/.config/zsh/local.zprofile"
touch "$HOME/.config/zsh/local.zsh"
~~~

从对应平台笔记复制当前完整的 `.zprofile` 与 `.zshrc` 骨架。迁移旧配置时遵循：

- 跨平台 alias 和函数进入 `common.zsh`；
- macOS 专属命令进入 `macos.zsh`，Ubuntu 专属命令进入 `linux.zsh`；
- 当前机器的代理和私有 SDK 路径进入 `local.zprofile` 或 `local.zsh`；
- 非交互 SSH 命令也必须看到的最小 PATH，留待切换时合并进 `~/.zshenv`；
- 不复制 `ZSH`、`ZSH_THEME`、`plugins=(...)` 或 Oh My Zsh 的 source 行。

## 6. 选择插件清单

### 推荐的纯净清单

如果不依赖 Oh My Zsh 的别名和函数，使用：

~~~text
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting kind:clone
~~~

### 暂时保留少量 Oh My Zsh 子插件

若盘点确认仍需要 `git`、`sudo` 或 `extract`，Antidote 官方建议先加载兼容层，再选择具体子目录：

~~~text
getantidote/use-omz
ohmyzsh/ohmyzsh path:lib
ohmyzsh/ohmyzsh path:plugins/git
ohmyzsh/ohmyzsh path:plugins/sudo
ohmyzsh/ohmyzsh path:plugins/extract

zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting kind:clone
~~~

这不会读取原来的 `~/.oh-my-zsh`，而是由 Antidote 克隆并加载选中的上游目录。不要为了“可能会用”复制全部 OMZ 插件；每增加一项都要在新会话确认别名冲突和启动耗时。

无论选择哪份清单，`.zshrc` 的关键尾部都应保持以下顺序：

~~~zsh
antidote_dir="${XDG_DATA_HOME:-$HOME/.local/share}/antidote"
if [[ -r "$antidote_dir/antidote.zsh" ]]; then
  export ANTIDOTE_HOME="${XDG_CACHE_HOME:-$HOME/.cache}/antidote"
  source "$antidote_dir/antidote.zsh"
  ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=8'
  antidote load "$ZDOTDIR/.zsh_plugins.txt"
fi
unset antidote_dir

if (( $+commands[fzf] )) && fzf --zsh >/dev/null 2>&1; then
  FZF_CTRL_R_COMMAND= FZF_ALT_C_COMMAND= source <(fzf --zsh)
fi

(( $+commands[atuin] )) && eval "$(atuin init zsh --disable-up-arrow --disable-ai)"
(( $+commands[zoxide] )) && eval "$(zoxide init zsh)"
(( $+commands[starship] )) && eval "$(starship init zsh)"

if (( $+functions[antidote] )); then
  zsh_highlight_root="$(antidote path zsh-users/zsh-syntax-highlighting 2>/dev/null)"
  if [[ -r "$zsh_highlight_root/zsh-syntax-highlighting.zsh" ]]; then
    source "$zsh_highlight_root/zsh-syntax-highlighting.zsh"
  fi
  unset zsh_highlight_root
fi
~~~

`zsh-syntax-highlighting` 虽然写在清单最后，仍必须使用 `kind:clone` 并在全部交互工具之后显式加载。这样 Atuin、fzf、zoxide 或 Starship 后续创建的 ZLE 部件才不会破坏高亮。

## 7. 不切换 `~/.zshenv`，先并行测试

先解析新文件：

~~~bash
zsh -n "$HOME/.config/zsh/.zprofile"
zsh -n "$HOME/.config/zsh/.zshrc"
~~~

再把 `ZDOTDIR` 只传给一个新的测试进程：

~~~bash
env ZDOTDIR="$HOME/.config/zsh" zsh -lic '
  echo "parallel config loaded"
  command -v antidote starship atuin zoxide fzf
  bindkey "^R"
'
~~~

该命令不会更改账户登录 Shell，也不会让当前窗口退出 Oh My Zsh。首次运行会由 Antidote 克隆插件；网络失败时先修复下载问题，不要因此删除旧框架。

还应测试日常依赖：

~~~bash
env ZDOTDIR="$HOME/.config/zsh" zsh -lic '
  git --version
  printf "PATH=%s\n" "$PATH"
  type z
  type zi
  echo "prompt and plugins loaded"
'
~~~

在这个阶段逐项补齐旧配置遗漏。不要用 `source ~/.zshrc` 测试，因为那仍指向旧文件，也会把两个框架混进同一个 Shell 进程。

## 8. 切换最小 `~/.zshenv`

只有并行测试通过后，才把以下两类内容合并进主目录的 `~/.zshenv`：

1. `ZDOTDIR` 引导；
2. 对 SSH 非交互命令不可缺少、且经过存在性判断的最小 PATH。

核心引导是：

~~~zsh
export ZDOTDIR="${XDG_CONFIG_HOME:-$HOME/.config}/zsh"
~~~

完整的 macOS 或 Ubuntu PATH 写法应从相应平台笔记复制。不要把旧 `~/.zshrc` 中的主题、插件、别名、SDK 初始化和输出语句全部搬进 `.zshenv`。

写入后，当前 Oh My Zsh 会话不会被立即替换。保留它，另开一个全新终端或新的 SSH 连接进行验证。

## 9. 验证切换结果

在全新会话执行：

~~~zsh
printf 'ZDOTDIR: %s\n' "$ZDOTDIR"
printf 'current shell: '
ps -p $$ -o comm=

if (( $+functions[omz] )); then
  print 'unexpected: Oh My Zsh is still loaded'
else
  print 'ok: full Oh My Zsh framework is not loaded'
fi

antidote list
bindkey '^R'
type z
type zi
starship --version
atuin doctor
fzf --version
zsh -lic 'echo startup-ok'
~~~

然后人工验证：

1. `Ctrl-R` 打开 Atuin；上方向键仍按原生方式逐条浏览历史；
2. 空提示符下的 `?` 不触发 Atuin AI；
3. `Ctrl-T` 打开 fzf，`Alt-C` 不绑定，目录交互选择使用 `zi`；
4. 自动建议与语法高亮正常，且没有插件重复加载警告；
5. Starship 只出现一个提示符；
6. 旧 alias、函数、SDK 和 SSH PATH 中真正需要的部分仍可使用；
7. macOS、Ubuntu Desktop 或第二个 SSH 会话都能正常进入 Zsh。

如果仍出现 Oh My Zsh prompt，优先搜索新目录是否残留旧初始化：

~~~bash
grep -RInE 'oh-my-zsh|ZSH_THEME|oh-my-zsh\.sh' "$HOME/.config/zsh"
~~~

通过 Antidote 选择性保留 OMZ 子插件时，清单中的 `ohmyzsh/ohmyzsh` 属于预期结果；新 `.zshrc` 中不应再出现 `source "$ZSH/oh-my-zsh.sh"`。

## 10. 导入历史与目录数据

先确认新会话稳定，再导入数据。把示例中的占位目录替换为第 2 步实际备份路径；Atuin 本地模式不要求注册账户：

~~~bash
backup_dir="$HOME/.terminal-backups/omz-migration-YYYYMMDD-HHMMSS"
if [ -f "$backup_dir/zsh-history" ]; then
  HISTFILE="$backup_dir/zsh-history" atuin import zsh
fi
atuin stats
~~~

新骨架已经把原生 `HISTFILE` 改到 XDG state；显式导入备份副本，才能避免 `atuin import auto` 只读取新的空历史文件。Atuin 导入后仍会保留和更新新的原生 Zsh 历史。跨机器同步是可选功能，应在检查历史过滤与密钥保存方式后单独启用。

如果原来使用 OMZ `z` 或 `zsh-z`，zoxide 可自动识别其常见数据位置：

~~~bash
zoxide import z
# 原来使用 zsh-z 时改为：zoxide import zsh-z
~~~

导入会写入 zoxide 自己的数据库，不删除旧插件数据。若自动检测不到文件，先查看 `zoxide import --help`，不要猜测并移动原数据库。

## 11. 保存已验证状态

基础体验稳定后，可让 Antidote 保存一份固定到提交的 snapshot：

~~~zsh
mkdir -p "$ZDOTDIR/snapshots"
antidote snapshot save "$ZDOTDIR/snapshots/antidote.txt"
~~~

配置仓库可同时跟踪 `.zsh_plugins.txt` 和该 snapshot：前者描述“加载什么和如何加载”，后者记录“当时验证的是哪些提交”。Antidote 安装目录、插件缓存和生成的 `.zsh_plugins.zsh` 不跟踪。

## 12. 回退到原 Oh My Zsh

若新会话出现问题，从一直保留的旧会话执行。先把本节的示例路径换成第 2 步真实的 `backup_dir`：

~~~bash
backup_dir="$HOME/.terminal-backups/omz-migration-YYYYMMDD-HHMMSS"

if [ -f "$backup_dir/.zshenv" ]; then
  cp -p "$backup_dir/.zshenv" "$HOME/.zshenv"
elif [ -f "$backup_dir/no-original-zshenv" ] && [ -f "$HOME/.zshenv" ]; then
  mv "$HOME/.zshenv" "$HOME/.zshenv.antidote-disabled.$(date +%Y%m%d-%H%M%S)"
fi

[ -f "$backup_dir/.zprofile" ] && cp -p "$backup_dir/.zprofile" "$HOME/.zprofile"
[ -f "$backup_dir/.zshrc" ] && cp -p "$backup_dir/.zshrc" "$HOME/.zshrc"
~~~

然后关闭并重新打开终端。若要在当前窗口立即验证旧配置，必须清除当前环境继承的 `ZDOTDIR`：

~~~bash
exec env -u ZDOTDIR zsh -l
~~~

该回退不删除 `~/.config/zsh`，因此修正后仍可重复第 7 步并行测试。若新 `.zshrc` 错误导致交互启动异常，也可从备用会话运行 `zsh -f`，跳过交互配置后再修复文件。

## 13. 何时清理 Oh My Zsh

至少经过一段覆盖日常开发、Git 仓库、IDE 终端和 SSH 的观察期，再考虑清理。清理前确认：

- 新配置不再引用 `~/.oh-my-zsh`；
- 必要 alias、函数、SDK 和补全均已迁移；
- Atuin 与原生历史都可访问；
- 回退备份仍可读取；
- Antidote snapshot 已保存；
- 多台机器的新会话都通过验证。

之后按 [[Oh My Zsh 更新、备份与卸载]] 使用官方卸载流程。不要直接 `rm -rf ~/.oh-my-zsh`；旧 custom 目录可能仍包含尚未迁移的函数或私有插件。

## 官方参考资料

- [Oh My Zsh：官方安装、配置与卸载说明](https://github.com/ohmyzsh/ohmyzsh)
- [Antidote：官方安装、插件清单与 snapshot](https://antidote.sh/)
- [Antidote：选择性使用 Zsh 框架子插件](https://antidote.sh/using-zsh-frameworks)
- [zsh-syntax-highlighting：官方加载顺序说明](https://github.com/zsh-users/zsh-syntax-highlighting)
- [Starship：官方 Zsh 初始化说明](https://starship.rs/guide/)
- [Atuin：导入已有历史](https://docs.atuin.sh/cli/guide/import/)
- [Atuin：按键配置](https://docs.atuin.sh/cli/configuration/key-binding/)
- [zoxide：导入旧目录数据](https://github.com/ajeetdsouza/zoxide)
- [fzf：Shell integration 与按键禁用](https://github.com/junegunn/fzf)

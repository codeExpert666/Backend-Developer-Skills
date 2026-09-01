---
title: Atuin 命令历史管理
aliases:
  - Atuin 使用指南
  - Atuin 历史搜索与同步
  - Zsh 命令历史管理
tags:
  - Terminal
  - Terminal/使用
  - Terminal/Shell
  - Zsh
  - Atuin
  - 命令历史
created: 2026-07-19T16:33:48
updated: 2026-09-01T15:50:00
---

Atuin 将命令、执行时间、工作目录、退出状态和耗时保存在本地数据库中，并提供比普通 Shell 历史更容易检索的界面。本方案把它作为 **local-first 的命令历史工具**：不注册账号也能完整使用，跨机器同步是后续按需启用的可选能力。

本文默认已经按照 [[现代终端环境搭建概览]] 配置 Zsh。Atuin 配置源与本地数据的部署边界见 [[使用 Git 与 GNU Stow 搭建 dotfiles 仓库]]，插件加载和跨机器文件边界见 [[Zsh 与 Antidote 跨机器配置管理]]，与 fzf 的按键分工见 [[zoxide 与 fzf 导航和模糊查找]]，升级和数据恢复见 [[现代终端环境更新、验证与回退]]。

> [!tip] 何时打开本文
> 主线已经安装 Atuin 并接入 `Ctrl-R`。只有要导入旧历史、调整过滤规则、理解本地数据与密钥，或决定是否启用同步时，才继续本文。

## 1. 先明确职责边界

Atuin 是独立可执行程序，不是提示符、终端模拟器或 Antidote 插件。即使使用 Antidote，也应安装 Atuin 二进制，再由 `.zshrc` 调用 `atuin init zsh`；不要同时通过 Antidote 加载 `atuinsh/atuin`，否则可能重复注册历史钩子和快捷键。

| 组件 | 负责什么 | 本方案中的快捷键 |
| --- | --- | --- |
| Zsh 原生历史 | 保留传统 `HISTFILE`，供兼容和应急恢复 | 上方向键保持原生行为 |
| Atuin | 结构化记录、排序和搜索历史 | 独占 `Ctrl-R` |
| fzf | 文件选择和通用模糊过滤 | 保留 `Ctrl-T`，禁用其 `Ctrl-R` |

Atuin 导入旧历史后，原来的 Zsh 历史文件仍会继续更新。不要急于删除 `HISTFILE` 或原有历史选项；两者并存能降低迁移和回退成本。

## 2. 在加载 Atuin 设置前部署受管配置

Atuin 不只是读取配置：当配置目录或 `config.toml` 不存在时，加载设置的命令可能创建目录并写入示例配置。因此，如果该文件准备由 Stow 管理，不能先运行 `atuin info`、`atuin doctor`、`atuin init` 或真实交互式 Zsh。

> [!info] 配置语义核对范围
> 本节于 2026-09-01 按 [Atuin v18.21.0](https://github.com/atuinsh/atuin/releases/tag/v18.21.0) 和官方 `main` 文档核对了安装目录、同步、匹配方式、过滤范围、workspace、日志目录与有效配置查询。资料核对不表示任何具体机器已经运行或验证这套配置。

主线已经完成这一顺序；单独新增 Atuin 软件包时，在仓库中创建或编辑配置源：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
atuin_source="$DOTFILES_DIR/atuin/.config/atuin/config.toml"

git -C "$DOTFILES_DIR" status --short --branch
mkdir -p "${atuin_source%/*}" "$HOME/.config/atuin"
~~~

将下面的统一 local-first 受管基线保存到 `$DOTFILES_DIR/atuin/.config/atuin/config.toml`。平台安装主线与迁移主线有意使用相同的有效配置；它不是等待后续替换的临时最小文件：

~~~toml
# 不自动同步历史；需要跨机器同步时再改为 true，或手动运行 `atuin sync`。
# 这不关闭由 update_check 单独控制的版本更新检查。
auto_sync = false

# 选中历史后先回填到 Shell 命令行，便于检查或编辑，不直接执行。
enter_accept = false

# 使用精简界面；可改为 auto（按终端高度切换）或 full（完整界面）。
style = "compact"

# 限制界面最多占 20 行；设为 0 时使用全部可用高度。
inline_height = 20

# 使用模糊匹配搜索命令内容；它决定“怎样匹配”，不决定“搜索哪些历史”。
# 在搜索界面可按 Ctrl-S 临时循环其他匹配模式。
search_mode = "fuzzy"

# 默认先搜索全部历史；当前还支持 host、session、directory、workspace 和 session-preload。
# Ctrl-R 循环的是 search.filters 中启用的过滤范围；本基线不覆盖该列表。
filter_mode = "global"

# 启用 workspace 过滤能力；它不会把默认过滤范围从 global 改为 workspace。
# 不在 Git 仓库时，workspace 模式会被跳过。
workspaces = true

# 启用 Atuin 内置的凭据格式过滤；它只是安全网，不能覆盖所有秘密格式。
secrets_filter = true

# history_filter 使用正则表达式；按自己的实际命令扩展。
history_filter = [
  "^export .*(_TOKEN|_PASSWORD|_SECRET)=",
  "^curl .*Authorization:",
]

# 如需排除整个目录，取消注释并替换成真实的绝对路径。
# cwd_filter = ["^/absolute/path/to/private-workspace"]

[logs]
# Atuin 当前默认写入 ~/.atuin/logs；日志属于可跨进程保留但不应进入 Git 的本机状态。
dir = "~/.local/state/atuin/logs"
~~~

`search_mode` 决定输入文字怎样匹配历史，`filter_mode` 决定打开界面时先搜索哪一部分历史；`workspaces = true` 只是让 workspace 成为可用范围。这三项分别控制匹配算法、初始范围和范围能力，不是互相覆盖的同一开关。

这里的 local-first 表示历史默认不自动同步，不表示 Atuin 完全离线。若环境明确禁止 Atuin 发起更新检查，再单独设置 `update_check = false`；不要仅因为关闭同步就默认加入它。

已有 `~/.config/atuin/config.toml` 时先停止并判断它的来源：若是普通文件，先在仓库外备份、检查私有路径并与配置源比较，不直接覆盖，也不使用 `stow --adopt`。确认目标可由新源接管后，先模拟再部署：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --restow --verbose=2 atuin
~~~

确认模拟输出没有 conflict 且只涉及 Atuin 配置后，再实际部署并验证所有权：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --restow --verbose=2 atuin

test -L "$HOME/.config/atuin/config.toml"
readlink "$HOME/.config/atuin/config.toml"
~~~

`history_filter` 和 `cwd_filter` 是正则表达式，默认可在字符串任意位置匹配；需要匹配命令开头或结尾时，应显式使用 `^` 或 `$`。不要直接复制过于宽泛的规则，否则正常历史也会被排除。

Atuin 也遵守“命令前加空格则不记入历史”的惯例，但这只是降低误记录概率，不是秘密管理方案。密码和令牌应通过交互式输入、密码管理器、受控环境文件或专用 CLI 传递。

> [!warning] 过滤规则不会自动清除旧记录
> 新增过滤规则只影响后续记录。`atuin history prune` 会依据当前过滤规则删除既有历史，属于数据删除操作；先确认私有备份和规则范围，再单独执行，不要把它写进启动脚本。

## 3. 确认当前二进制与数据位置

配置目标已经由 Stow 占位后，才运行会加载设置的检查：

~~~bash
command -v atuin
atuin --version
atuin info
~~~

macOS 的 Homebrew 安装应解析到 Homebrew 前缀；Ubuntu 主线给官方安装器同时设置 `ATUIN_INSTALL_DIR="$HOME/.local/bin"` 与 `ATUIN_NO_MODIFY_PATH=1`，因此应解析到 `$HOME/.local/bin/atuin`。`$HOME/.atuin/bin` 是安装器的上游默认位置，只作为旧布局识别，不再加入新 PATH。

若命令缺失，回到当前平台主线检查原安装来源与 PATH，不要重复运行安装器。`atuin info` 用于核对当前配置、数据库与密钥的实际位置；XDG 默认目录只是默认值，备份和恢复必须以当前输出为准。运行后再次确认 `config.toml` 仍是指向 dotfiles 的链接，而数据库、密钥和 session 位于仓库外。

安装器还可能在配置目录生成 `atuin-receipt.json`。它记录本机安装信息，不是跨机器配置；主线使用 Stow `--no-folding`，因此真实的 `~/.config/atuin` 目录可以同时容纳受管 `config.toml` 链接与本机回执。不要把回执复制回 dotfiles 源。

## 4. 让 Atuin 接管 Ctrl-R

在 `.zshrc` 的交互式工具区使用下面这一个初始化块。若此前运行过会自动配置 Shell 的 setup 脚本，并已存在 `eval "$(atuin init zsh)"`，应将该行替换掉，而不是在其他位置再追加一次：

~~~zsh
if command -v atuin >/dev/null 2>&1; then
  eval "$(atuin init zsh --disable-up-arrow --disable-ai)"
fi
~~~

这里显式做了三项选择：

- Atuin 保留默认的 `Ctrl-R` 历史搜索绑定。
- `--disable-up-arrow` 让上方向键继续使用 Zsh 原生历史，不突然打开全屏搜索界面。
- `--disable-ai` 关闭空提示符下由 `?` 触发的 Atuin AI；这套配置只把 Atuin 当作本地历史工具。

fzf 的 Shell 集成必须先加载，并在加载当时禁用它的 `Ctrl-R`；随后再初始化 Atuin。完整顺序见 [[zoxide 与 fzf 导航和模糊查找]]。同一条 `atuin init` 只能出现一次。

保存后检查并新开终端：

~~~bash
zsh_config_dir="$(zsh -c 'print -r -- "${ZDOTDIR:-$HOME}"')"
zsh -n "$zsh_config_dir/.zshrc"
exec zsh -l
~~~

## 5. 只从人工脱敏副本导入一次

先确认旧历史的真实来源。当前仍是旧 Zsh 时，先把内存中的新记录追加到当前历史文件；已经切换到 XDG state 时，则使用迁移主线保存的私密历史副本：

~~~zsh
current_history_file="${HISTFILE:-$HOME/.zsh_history}"
if [[ -n "${ZSH_VERSION:-}" && -n "$current_history_file" ]]; then
  fc -AI "$current_history_file"
fi

printf 'old or backed-up Zsh history path: '
IFS= read -r legacy_history_file

if [[ ! -f "$legacy_history_file" ]]; then
  print -u2 "STOP: history file does not exist: $legacy_history_file"
  false
fi
unset current_history_file
~~~

输入应来自迁移主线实际打印的备份路径或旧配置记录，不要猜测。接下来创建权限收紧的工作副本；原始历史保持不变：

~~~zsh
previous_umask=$(umask)
umask 077

history_work_dir="$HOME/terminal-backups/atuin-import-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$history_work_dir"
sanitized_history="$history_work_dir/zsh-history.sanitized"
cp -p "$legacy_history_file" "$sanitized_history"
chmod 600 "$sanitized_history"

umask "$previous_umask"
unset previous_umask
printf 'review and redact: %s\n' "$sanitized_history"
~~~

用编辑器逐行删除令牌、密码、连接串、内部主机名和其他不应进入 Atuin 的命令。确认副本已经脱敏后，才明确使用 Zsh 解析器导入：

~~~zsh
less "$sanitized_history"
HISTFILE="$sanitized_history" atuin import zsh
atuin stats
unset legacy_history_file sanitized_history
~~~

`atuin import auto` 会根据当前 Shell 和当前历史路径推断输入；切换到新 XDG 历史后，它可能读错文件，因此迁移场景使用显式 `import zsh`。导入不会删除原文件，也不会替你识别全部秘密；这是一次性操作，不能放入 `.zshrc` 或重复执行。

## 6. 日常搜索

在新终端按 `Ctrl-R` 打开 Atuin：

- 输入任意片段，默认按 `search_mode = "fuzzy"` 进行模糊匹配；按 `Ctrl-S` 可临时循环其他匹配方式。
- 在搜索界面再次按 `Ctrl-R`，可循环 `search.filters` 中启用的全局、主机、会话、目录等过滤范围。
- 按 `Tab` 将选中的命令带回命令行编辑；本方案还设置了 `enter_accept = false`，避免误执行旧命令。
- 可直接运行 `atuin search <关键词>` 做非快捷键搜索。

历史命令可能来自另一台机器、另一个目录或较早的软件版本。涉及删除、部署、数据库和生产环境时，选择后必须先核对参数再执行。

## 7. 按需启用跨机器同步

本地搜索稳定后再决定是否启用同步。没有账号时，Atuin 不需要联网也能记录和搜索历史。

首次注册：

~~~text
atuin register -u <用户名> -e <邮箱>
~~~

注册后会生成端到端加密所需的密钥，可用 `atuin key` 查看。密钥丢失后服务端无法替你恢复，因此应保存到密码管理器；不要放入 dotfiles、Git、聊天记录或普通云盘。

在另一台机器登录时：

~~~text
atuin login -u <用户名>
~~~

命令会交互式询问密码和加密密钥。登录成功后，把 `config.toml` 中的 `auto_sync` 改为 `true`，再手动执行一次：

~~~bash
atuin sync
~~~

只有在正常同步后仍明确缺少数据时，才使用耗时更长的完整同步：

~~~bash
atuin sync -f
~~~

同步是跨机器数据通道，不替代加密密钥备份，也不替代本地历史数据的私有备份。

## 8. IDE 与交互式 Shell

Atuin 的记录钩子只会在交互式 Shell 加载 `.zshrc` 并执行 `atuin init` 后生效。普通 Ghostty 窗口能记录、IDE 内置终端不能记录时，先在两个环境分别运行：

~~~bash
printf 'shell flags: %s\n' "$-"
atuin doctor
~~~

交互式 Zsh 的 flags 应包含 `i`，`atuin doctor` 中的 `shell.preexec` 不应为 `none`。IDE 的“终端”配置可使用交互式 Zsh，例如 `/bin/zsh -i`；不要把构建任务、脚本运行器或 CI 强行改成交互式 Shell，它们本来就不应读取个人 `.zshrc`。

## 9. 验证与数据边界

新开终端后依次检查：

~~~zsh
atuin --version
atuin config get auto_sync --verbose
atuin config get search_mode --verbose
atuin config get filter_mode --verbose
atuin config get workspaces --verbose
atuin config get logs.dir --verbose
atuin doctor
bindkey '^R'
~~~

`atuin config get ... --verbose` 同时帮助区分配置文件中的值与叠加默认值、环境变量后的有效值。本基线预期上述五项分别解析为 `false`、`fuzzy`、`global`、`true` 和 `~/.local/state/atuin/logs` 对应的有效路径；若文件值与有效值不同，先检查环境变量和实际加载的配置路径。

再执行一条无副作用的测试命令，按 `Ctrl-R` 搜索其中的唯一文本。若 `Ctrl-R` 打开 fzf 或行为反复变化，说明 fzf 与 Atuin 都绑定了同一按键，回到 [[zoxide 与 fzf 导航和模糊查找]] 只保留一个初始化顺序。

| 内容 | 是否进入 Git | 原因 |
| --- | --- | --- |
| `$DOTFILES_DIR/atuin/.config/atuin/config.toml` | 可以 | 由 Stow 部署到默认路径，只保存通用偏好和过滤规则 |
| Atuin 历史数据库 | 不可以 | 包含实际命令、目录和执行元数据 |
| Atuin 加密密钥 | 不可以 | 是解密同步历史的安全边界 |
| Atuin session 文件 | 不可以 | 本质上是服务端会话令牌 |
| `~/.config/atuin/atuin-receipt.json` | 不可以 | 安装器生成的本机元数据，可与受管链接并存 |
| `~/.local/state/atuin/logs` | 不可以 | 属于持续变化的本机状态，且可能含排障信息 |
| Atuin 缓存 | 不可以 | 可重新生成且会制造无意义差异 |

Atuin 默认把数据库、密钥和 session 放在 `~/.local/share/atuin`，除非使用 XDG 环境变量或配置覆盖；本基线另外把日志从上游默认的 `~/.atuin/logs` 定向到 XDG state。data 和 state 可以进入受保护的私有备份，但绝不能提交到 dotfiles 仓库。最后确认运行时 `config.toml` 是指向预期仓库的链接，Atuin 自检通过，并且 Git diff 只包含经过审查的通用配置。

## 官方参考资料

- [Atuin：安装](https://docs.atuin.sh/cli/guide/installation/)
- [Atuin：官方 release](https://github.com/atuinsh/atuin/releases)
- [Atuin：导入旧历史](https://docs.atuin.sh/cli/guide/import/)
- [Atuin：Shell 集成与 IDE 排障](https://docs.atuin.sh/cli/guide/shell-integration/)
- [Atuin：按键配置](https://docs.atuin.sh/main/configuration/key-binding/)
- [Atuin：配置、过滤与数据路径](https://docs.atuin.sh/main/configuration/config/)
- [Atuin：高级搜索与过滤范围](https://docs.atuin.sh/main/guide/advanced-usage/)
- [Atuin：查询有效配置](https://docs.atuin.sh/main/reference/config/)
- [Atuin：可选同步与密钥边界](https://docs.atuin.sh/cli/guide/sync/)
- [Atuin：doctor 命令](https://docs.atuin.sh/cli/reference/doctor/)
- [Atuin：设置加载与缺失配置创建逻辑](https://github.com/atuinsh/atuin/blob/main/crates/atuin-client/src/settings.rs#L1465-L1533)
- [GNU Stow：官方手册](https://www.gnu.org/software/stow/manual/stow.html)

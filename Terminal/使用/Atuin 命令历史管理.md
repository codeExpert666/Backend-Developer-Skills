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
updated: 2026-07-19T17:05:02
---

Atuin 将命令、执行时间、工作目录、退出状态和耗时保存在本地数据库中，并提供比普通 Shell 历史更容易检索的界面。本方案把它作为 **local-first 的命令历史工具**：不注册账号也能完整使用，跨机器同步是后续按需启用的可选能力。

本文默认已经按照 [[现代终端环境搭建概览]] 配置 Zsh。插件加载和跨机器文件边界见 [[Zsh 与 Antidote 跨机器配置管理]]，与 fzf 的按键分工见 [[zoxide 与 fzf 导航和模糊查找]]，升级和数据恢复见 [[现代终端环境更新、验证与回退]]。

## 1. 先明确职责边界

Atuin 是独立可执行程序，不是提示符、终端模拟器或 Antidote 插件。即使使用 Antidote，也应安装 Atuin 二进制，再由 `.zshrc` 调用 `atuin init zsh`；不要同时通过 Antidote 加载 `atuinsh/atuin`，否则可能重复注册历史钩子和快捷键。

| 组件 | 负责什么 | 本方案中的快捷键 |
| --- | --- | --- |
| Zsh 原生历史 | 保留传统 `HISTFILE`，供兼容和应急恢复 | 上方向键保持原生行为 |
| Atuin | 结构化记录、排序和搜索历史 | 独占 `Ctrl-R` |
| fzf | 文件选择和通用模糊过滤 | 保留 `Ctrl-T`，禁用其 `Ctrl-R` |

Atuin 导入旧历史后，原来的 Zsh 历史文件仍会继续更新。不要急于删除 `HISTFILE` 或原有历史选项；两者并存能降低迁移和回退成本。

## 2. 安装 Atuin 二进制

macOS 上若使用 Homebrew：

~~~bash
brew install atuin
~~~

Ubuntu 或其他 Unix 系统可以使用 Atuin 官方 release 中的二进制安装器。先下载到临时文件并阅读，再执行已经审查的本地副本：

~~~bash
atuin_installer="$(mktemp)"
curl --proto '=https' --tlsv1.2 -LsSf \
  https://github.com/atuinsh/atuin/releases/latest/download/atuin-installer.sh \
  -o "$atuin_installer"
less "$atuin_installer"
ATUIN_NO_MODIFY_PATH=1 sh "$atuin_installer"
rm -f "$atuin_installer"
~~~

在 `less` 中检查来源与内容，按 `q` 返回后才继续。这里直接使用官方二进制安装器，并以 `ATUIN_NO_MODIFY_PATH=1` 禁止它自动修改 `.profile`、`.zshrc`、`.zshenv` 等文件；历史导入、同步账号、PATH 和 Shell 初始化都留到后文手工完成。安装器默认把二进制放入 `~/.atuin/bin`。若新终端仍找不到命令，应先按 [[Zsh 与 Antidote 跨机器配置管理]] 检查 PATH，而不是重复运行安装器。

这是从网络取得并执行官方脚本；受管服务器应先按组织的软件来源和审计要求确认是否允许。

重新打开终端后确认：

~~~bash
command -v atuin
atuin --version
~~~

## 3. 让 Atuin 接管 Ctrl-R

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

## 4. 一次性导入旧历史

如果仍沿用旧配置，先用实际的旧 `HISTFILE`；若已经切到本文配套的 XDG Zsh 骨架，默认旧文件通常是 `~/.zsh_history`，也可以直接使用迁移时保存的 `zsh-history` 副本。下面先显式指定并备份这个输入文件：

~~~zsh
legacy_history_file="$HOME/.zsh_history"
if [[ ! -f "$legacy_history_file" && -f "$HOME/.zhistory" ]]; then
  legacy_history_file="$HOME/.zhistory"
fi
if [[ -f "$legacy_history_file" ]]; then
  history_backup="${legacy_history_file}.before-atuin.$(date +%Y%m%d-%H%M%S)"
  cp -p "$legacy_history_file" "$history_backup"
  printf 'history backup: %s\n' "$history_backup"
fi
~~~

旧配置使用自定义路径时，应先替换 `legacy_history_file`，不要猜测。然后明确以 Zsh 格式导入同一个文件：

~~~zsh
if [[ -f "$legacy_history_file" ]]; then
  HISTFILE="$legacy_history_file" atuin import zsh
fi
unset legacy_history_file
~~~

只有当前 `HISTFILE` 仍然指向旧文件时，`atuin import auto` 才能作为等价简写。现代骨架会把 `HISTFILE` 切到新的 XDG state 路径，因此迁移后应像上面一样显式指定旧文件。这是一次性迁移步骤，不应写入 `.zshrc` 或在每次启动时重复执行。

导入不会替你识别所有敏感命令。若旧历史中可能含令牌、密码或连接串，应在启用远程同步前先审查和清理。

## 5. 使用一套克制的本地配置

Atuin 的主配置通常位于 `~/.config/atuin/config.toml`。先创建目录：

~~~bash
mkdir -p "$HOME/.config/atuin"
~~~

可从下面的最小配置开始：

~~~toml
# 默认只使用本地数据库；决定启用同步后再改为 true。
auto_sync = false

# 搜索结果先回到命令行编辑，不直接执行历史命令。
enter_accept = false

style = "compact"
inline_height = 20
filter_mode = "global"
workspaces = true

# 保留 Atuin 内置的常见秘密模式过滤。
secrets_filter = true

# 按自己的实际命令扩展；这些是正则表达式。
history_filter = [
  "^export .*(_TOKEN|_PASSWORD|_SECRET)=",
  "^curl .*Authorization:",
]

# 如需排除整个目录，取消注释并替换成真实的绝对路径。
# cwd_filter = ["^/absolute/path/to/private-workspace"]
~~~

`history_filter` 和 `cwd_filter` 是正则表达式，默认可在字符串任意位置匹配；需要匹配命令开头或结尾时，应显式使用 `^` 或 `$`。不要直接复制过于宽泛的规则，否则正常历史也会被排除。

Atuin 也遵守“命令前加空格则不记入历史”的惯例，但这只是降低误记录概率，不是秘密管理方案。密码和令牌应通过交互式输入、密码管理器、受控环境文件或专用 CLI 传递。

> [!warning] 过滤规则不会自动清除旧记录
> 新增过滤规则只影响后续记录。`atuin history prune` 会依据当前过滤规则删除既有历史，属于数据删除操作；先确认私有备份和规则范围，再单独执行，不要把它写进启动脚本。

## 6. 日常搜索

在新终端按 `Ctrl-R` 打开 Atuin：

- 输入任意片段进行模糊搜索。
- 在搜索界面再次按 `Ctrl-R`，可轮换全局、主机、会话、目录等过滤范围。
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
atuin doctor
bindkey '^R'
~~~

再执行一条无副作用的测试命令，按 `Ctrl-R` 搜索其中的唯一文本。若 `Ctrl-R` 打开 fzf 或行为反复变化，说明 fzf 与 Atuin 都绑定了同一按键，回到 [[zoxide 与 fzf 导航和模糊查找]] 只保留一个初始化顺序。

| 内容 | 是否进入 Git | 原因 |
| --- | --- | --- |
| 审查后的 `~/.config/atuin/config.toml` | 可以 | 只保存通用偏好和过滤规则 |
| Atuin 历史数据库 | 不可以 | 包含实际命令、目录和执行元数据 |
| Atuin 加密密钥 | 不可以 | 是解密同步历史的安全边界 |
| Atuin session 文件 | 不可以 | 本质上是服务端会话令牌 |
| Atuin 日志和缓存 | 不可以 | 属于运行时数据，且可能含本机信息 |

Atuin 默认把数据库、密钥和 session 放在 `~/.local/share/atuin`，除非使用 XDG 环境变量或配置覆盖。该目录可以进入受保护的私有备份，但绝不能提交到 dotfiles 仓库。

## 官方参考资料

- [Atuin：安装](https://docs.atuin.sh/cli/guide/installation/)
- [Atuin：官方 release](https://github.com/atuinsh/atuin/releases)
- [Atuin：导入旧历史](https://docs.atuin.sh/cli/guide/import/)
- [Atuin：Shell 集成与 IDE 排障](https://docs.atuin.sh/cli/guide/shell-integration/)
- [Atuin：按键配置](https://docs.atuin.sh/cli/configuration/key-binding/)
- [Atuin：配置、过滤与数据路径](https://docs.atuin.sh/cli/configuration/config/)
- [Atuin：可选同步与密钥边界](https://docs.atuin.sh/cli/guide/sync/)
- [Atuin：doctor 命令](https://docs.atuin.sh/cli/reference/doctor/)

---
title: zoxide 与 fzf 导航和模糊查找
aliases:
  - zoxide 与 fzf 使用指南
  - Zsh 快速目录导航
  - 终端模糊查找配置
tags:
  - Terminal
  - Terminal/使用
  - Terminal/Shell
  - Zsh
  - zoxide
  - fzf
  - 命令行效率
created: 2026-07-19T16:33:48
updated: 2026-07-19T16:33:48
---

zoxide 与 fzf 都能减少路径输入，但解决的问题不同：zoxide 根据使用频率记住目录，fzf 从候选文本中进行模糊选择。本方案让 zoxide 负责目录跳转、fzf 负责文件选择，并把历史搜索明确交给 [[Atuin 命令历史管理]]。

安装全套环境见 [[现代终端环境搭建概览]]，Zsh 与 Antidote 的配置边界见 [[Zsh 与 Antidote 跨机器配置管理]]，更新和回退见 [[现代终端环境更新、验证与回退]]。

## 1. 先固定职责和按键

| 入口 | 负责什么 | 是否保留 |
| --- | --- | --- |
| `z <关键词>` | 跳转到 zoxide 评分最高的匹配目录 | 保留 |
| `zi [关键词]` | 通过 fzf 交互选择 zoxide 已记录的目录 | 保留 |
| `Ctrl-T` | 通过 fzf 选择文件或目录，并插入当前命令行 | 保留 |
| `Ctrl-R` | 搜索结构化命令历史 | 只交给 Atuin |
| `Alt-C` | 通过 fzf 从当前目录树选择目录并跳转 | 默认禁用，避免与 `zi` 重复 |
| 直接运行 `fzf` | 对标准输入或当前目录候选做通用模糊过滤 | 保留 |

`zi` 根据已经访问过的目录及其使用频率选择；fzf 的 `Alt-C` 更偏向扫描当前目录树。若你确实依赖后者，可以保留 `Alt-C`，但不应再把两个入口描述成同一种能力。

## 2. 安装二进制

macOS 使用 Homebrew：

~~~bash
brew install zoxide fzf
~~~

Ubuntu 应优先按 [[Ubuntu 从零搭建现代终端环境]] 选择当前上游兼容的 fzf 安装方式。发行版仓库只作为备选；先查看候选版本：

~~~bash
apt-cache policy fzf
~~~

只有候选版本至少为 `0.51.0`，并且该包提供 `fzf --zsh` 时，才使用 `sudo apt install fzf`。版本较旧时不要先安装再叠加另一份 fzf，否则 PATH 先后顺序会让实际版本难以判断。

zoxide 上游指出 Debian 和 Ubuntu 仓库中的版本更新较慢，当前推荐使用其官方安装器。先下载到临时文件并阅读，再执行本地副本：

~~~bash
zoxide_setup="$(mktemp)"
curl -sSfL https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh -o "$zoxide_setup"
less "$zoxide_setup"
sh "$zoxide_setup"
rm -f "$zoxide_setup"
~~~

在 `less` 中检查内容，按 `q` 返回后才继续。这仍是从网络取得并执行官方脚本；受管服务器应先确认软件来源策略。若不允许，可改用 zoxide 官方列出的 Cargo、Nix 或 Linuxbrew 安装方式，不要随意添加来源不明的 APT 仓库。

确认版本：

~~~bash
zoxide --version
fzf --version
~~~

zoxide 当前要求用于交互选择的 fzf 至少为 `0.51.0`。Ubuntu 仓库版本过旧时，`z` 仍可使用，但应按 fzf 官方安装说明升级后再启用 `zi` 和嵌入式 `fzf --zsh` 集成。

## 3. 使用唯一的初始化顺序

以下片段应位于 `.zshrc` 的交互式工具区，并满足两个前提：

1. 补全系统已经由基础配置完成 `compinit`，不要在这里重复执行。
2. Antidote 已加载普通插件，但最终的语法高亮仍留在所有 ZLE 工具之后加载。

~~~zsh
# 1. fzf 提供 Ctrl-T 和模糊补全；加载当时关闭冲突按键。
if command -v fzf >/dev/null 2>&1 && fzf --zsh >/dev/null 2>&1; then
  FZF_CTRL_R_COMMAND= FZF_ALT_C_COMMAND= source <(fzf --zsh)
fi

# 2. Atuin 后加载，因此独占 Ctrl-R；保留原生上方向键并关闭 AI 快捷键。
if command -v atuin >/dev/null 2>&1; then
  eval "$(atuin init zsh --disable-up-arrow --disable-ai)"
fi

# 3. zoxide 必须位于 compinit 之后，才能正确提供 Zsh 补全。
if command -v zoxide >/dev/null 2>&1; then
  eval "$(zoxide init zsh)"
fi

# 4. Starship 在上述工具之后初始化。
if command -v starship >/dev/null 2>&1; then
  eval "$(starship init zsh)"
fi

# 5. zsh-syntax-highlighting 或同类插件应在所有 ZLE 组件之后最后加载。
~~~

fzf 官方要求在 `source <(fzf --zsh)` **执行当时** 把 `FZF_CTRL_R_COMMAND` 或 `FZF_ALT_C_COMMAND` 设为空；在下一行才设置变量不会撤销已经注册的按键。这里保留 `Ctrl-T`，关闭 fzf 的 `Ctrl-R` 和 `Alt-C`。

若你决定保留 `Alt-C`，只删除该行前的 `FZF_ALT_C_COMMAND=`：

~~~zsh
FZF_CTRL_R_COMMAND= source <(fzf --zsh)
~~~

不要再通过 Oh My Zsh 的 `z` 插件、另一个目录跳转插件或手写函数定义同名的 `z`、`zi`。重复定义通常表现为命令行为随加载顺序变化。

## 4. 使用 zoxide 导航

zoxide 在初始化后根据目录切换积累评分。先正常访问几个真实目录，再尝试：

~~~text
z project
z backend skills
zi
zi project
~~~

- `z project` 直接跳转到评分最高的匹配项。
- 多个关键词使用“同时匹配”的方式缩小范围。
- `zi` 或 `zi project` 调用 fzf，在 zoxide 已记录的目录中交互选择。
- `z ..`、`z -` 和绝对路径仍可处理常见的目录跳转。

默认保留 `cd`，不要一开始就使用 `zoxide init zsh --cmd cd` 覆盖它。显式的 `z` 更容易理解、排障和在脚本与交互命令之间保持边界。

### 从旧工具导入目录数据

若此前使用 Oh My Zsh 的 `z` 插件，可一次性执行：

~~~bash
zoxide import z
~~~

若此前使用 `zsh-z`，则改为：

~~~bash
zoxide import zsh-z
~~~

也可以在 Atuin 已导入足够历史后，从其中提取目录：

~~~bash
zoxide import atuin
~~~

只选择与你实际旧数据来源一致的命令；导入是迁移操作，不要写入 `.zshrc`。

## 5. 使用 fzf 选择文件

加载 Shell 集成后，在一条尚未执行的命令中按 `Ctrl-T`，选择结果会被插入当前命令行。例如先输入 `vim `，再按 `Ctrl-T` 选择文件；选择完成后仍应核对路径，再按 Enter 执行。

fzf 也可以直接过滤标准输入：

~~~bash
printf '%s\n' alpha beta gamma | fzf
~~~

默认的模糊补全触发符是 `**` 后接 Tab，例如：

~~~text
vim **<Tab>
cd ~/Programming/**<Tab>
ssh **<Tab>
~~~

这里的 `<Tab>` 表示按下 Tab 键，不是需要输入的文字。候选内容可能来自文件系统、进程、主机或环境变量，具体取决于光标前的命令。

## 6. 可选的显示设置

若希望 `Ctrl-T` 与 `zi` 都使用较紧凑的浮层，可在初始化之前设置：

~~~zsh
export FZF_CTRL_T_OPTS='--height 40% --layout=reverse --border'
export _ZO_FZF_OPTS='--height 40% --layout=reverse --border'
~~~

这些变量只调整显示，不改变数据来源。先让默认行为稳定，再增加预览、隐藏文件或自定义 walker；复杂预览命令会增加外部依赖和路径转义风险。

## 7. 验证按键和加载顺序

修改后先做语法与非交互检查：

~~~bash
zsh_config_dir="$(zsh -c 'print -r -- "${ZDOTDIR:-$HOME}"')"
zsh -n "$zsh_config_dir/.zshrc"
zsh -ic 'whence -w z zi; bindkey "^T"; bindkey "^R"'
~~~

然后在新终端手动确认：

1. 访问两个实际目录后，`z <唯一片段>` 能跳转。
2. `zi` 打开目录候选界面。
3. `Ctrl-T` 打开 fzf，并把选中路径插入命令行。
4. `Ctrl-R` 打开 Atuin，而不是 fzf。
5. `Alt-C` 默认没有 fzf 跳转绑定；若主动保留，则确认它与 `zi` 的用途不同。

常见问题对应关系如下：

| 症状 | 优先检查 |
| --- | --- |
| `zi` 提示找不到 fzf | `fzf --version` 是否至少为 `0.51.0`，PATH 是否一致 |
| `z` 没有匹配项 | 初始化后是否实际访问过目录，或是否需要执行一次旧数据导入 |
| zoxide 补全缺失 | `zoxide init zsh` 是否位于唯一一次 `compinit` 之后 |
| `Ctrl-R` 打开 fzf | fzf 加载时是否已将 `FZF_CTRL_R_COMMAND` 设为空，Atuin 是否随后初始化 |
| 语法高亮异常 | 高亮插件是否在 fzf、Atuin、zoxide 和 Starship 之后最后加载 |

zoxide 的目录评分数据库属于本机运行数据，不应提交到 Git；fzf 本身不维护需要同步的历史数据库。应进入 dotfiles 的只是上述环境变量和初始化逻辑。

## 官方参考资料

- [zoxide：安装、Zsh 初始化、命令与数据导入](https://github.com/ajeetdsouza/zoxide)
- [fzf：安装、Zsh 集成、按键与禁用方式](https://github.com/junegunn/fzf)
- [Atuin：按键配置](https://docs.atuin.sh/cli/configuration/key-binding/)
- [Zsh：补全系统](https://zsh.sourceforge.io/Doc/Release/Completion-System.html)

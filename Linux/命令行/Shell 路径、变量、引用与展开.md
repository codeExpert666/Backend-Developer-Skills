---
title: Shell 路径、变量、引用与展开
aliases:
  - Bash 路径变量与引号
  - Shell 变量引用和通配符
tags:
  - Linux
  - Linux/命令行
  - Shell
  - Bash
  - 路径
  - 变量
created: 2026-07-19T23:48:38
updated: 2026-07-20T00:49:15
---

本文解释 Shell 怎样把路径、变量、引号、命令替换和通配符变成最终传给命令的参数。目标是读懂常见命令并安全处理路径，不是穷举 Bash 的所有展开功能。

## 本篇掌握目标

| 层级 | 内容 |
| --- | --- |
| 必须熟练 | 绝对与相对路径、变量赋值、`${name}`、单引号、双引号，以及路径变量默认写成 `"$path"` |
| 理解会查 | Shell 变量与环境变量、`export`、命令替换、基础通配符和 PATH 查找 |
| 认识即可 | 完整展开顺序、花括号展开、数组、进程替换和改变默认通配行为的 Shell 选项 |

本篇优先建立“展开会改变最终参数”的模型。低频展开语法在真实脚本中遇到后再查询。

## 1. Shell 传递的是参数，不是“整行字符串”

输入下面一行时：

```bash
grep -n "server name" "$HOME/notes today.txt"
```

Shell 不会把整行原样交给 `grep`。它会识别引号和变量，完成展开，最后大致形成四个参数：

```text
参数 0：grep
参数 1：-n
参数 2：server name
参数 3：/home/当前用户/notes today.txt
```

双引号本身通常不会成为参数内容。它们是在告诉 Shell：空格属于当前参数，并且允许 `$HOME` 展开。

因此，读命令时不要只问“这行做什么”，还要问：

- 最终有几个参数？
- 每个路径在展开后是什么？
- 哪些字符由 Shell 解释，哪些字符交给目标命令？

## 2. 路径先以当前工作目录为参照

Linux 路径使用 `/` 分隔目录。

| 写法 | 含义 | 示例 |
| --- | --- | --- |
| `/` | 根目录 | `/etc/ssh` 从根目录开始 |
| 绝对路径 | 不依赖当前目录 | `/home/alice/src` |
| 相对路径 | 从当前工作目录开始 | `notes/day-1.txt` |
| `.` | 当前目录 | `./script.sh` |
| `..` | 父目录 | `../config` |
| `~` | 由 Shell 展开的当前用户主目录 | `~/src` |
| `$HOME` | 保存主目录路径的变量 | `"$HOME/src"` |

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
pwd
printf 'HOME=%s\n' "$HOME"
printf 'tilde expanded=%s\n' ~
realpath .
realpath ..
```

`~` 和 `$HOME` 经常得到相同结果，但处理阶段不同：

- 未引用、位于单词开头的 `~` 通常由 Shell 做波浪号展开。
- `"~"` 是普通波浪号字符，不会展开。
- `"$HOME"` 会在双引号内进行变量展开，处理包含特殊字符的路径时更稳定。

不要根据提示符猜当前目录。执行相对路径命令前先用 `pwd`，必要时把目标变成经过核对的绝对路径。

## 3. 变量保存 Shell 需要复用的值

### 赋值与读取

变量赋值的等号两边不能随意加空格：

```bash
project_name='eventhub'
project_dir="$HOME/src/$project_name"
printf 'project_dir=%s\n' "$project_dir"
```

`project_name = 'eventhub'` 不再是赋值；Shell 会尝试运行名为 `project_name` 的命令，并把 `=` 等内容作为参数。

读取变量常用 `$name` 或 `${name}`。花括号可以消除名称边界歧义：

```bash
topic='shell'
printf '%s\n' "${topic}_notes"
```

若写成 `$topic_notes`，Shell 会尝试读取名为 `topic_notes` 的变量。

### 未设置、空字符串与默认值

未设置变量和已设置为空在某些操作中含义不同。常见的安全读取形式是：

```bash
printf 'value=%s\n' "${OPTIONAL_VALUE:-default}"
```

`${name:-default}` 在变量未设置或为空时使用默认值。参数展开还有很多形式，初学阶段只需掌握变量边界和默认值，其他形式看到时查 Bash 手册。

### Shell 变量与环境变量

普通 Shell 变量只属于当前 Shell。`export` 把变量标记为环境变量，使之后启动的子进程能够继承它：

```bash
study_scope='current shell'
bash -c 'printf "before export: %s\n" "${study_scope-unset}"'

export study_scope
bash -c 'printf "after export: %s\n" "$study_scope"'

unset study_scope
```

**执行位置：Ubuntu 主机（任意目录；只改变当前 Shell 及其子进程环境）**

预期结果：第一个子 Bash 输出 `unset`，第二个子 Bash 输出 `current shell`；`unset` 最后移除当前会话中的练习变量。

`export` 不会自动永久修改登录环境。只有把内容写入 Shell 启动文件，后续新会话才可能继续加载；修改启动文件属于另一项持久变更，应在理解加载顺序后进行。

## 4. 引用决定哪些字符保留特殊含义

Shell 中常说的 quoting 可以理解为“引用”或“加引号”。它控制空格、`$`、`*`、反斜杠等字符是否继续由 Shell 解释。

| 写法 | 主要效果 | 常见用途 |
| --- | --- | --- |
| `'...'` | 其中所有字符基本按字面保留 | 固定文本、正则表达式 |
| `"..."` | 保留整体参数，同时允许变量和命令替换 | 路径、变量输出 |
| `\x` | 让下一个字符按字面处理；行尾 `\` 可续行 | 转义单个字符、拆分长命令 |

### 单引号

```bash
name='Linux'
printf '%s\n' '$name is a variable'
```

输出是字面量 `$name is a variable`。单引号内部不能直接再包含单引号；复杂固定文本可以拆成多个相邻片段，或改用其他安全表达方式。

### 双引号

```bash
name='Linux learner'
printf '%s\n' "$name"
```

双引号允许 `$name` 展开，同时保证展开结果仍是一个参数。双引号内并非所有字符都失去特殊含义，`$`、反引号和某些反斜杠用法仍可能参与处理。

### 反斜杠

```bash
printf '%s\n' one\ word
```

反斜杠让空格失去分隔参数的含义，输出 `one word`。长命令末尾的未引用反斜杠会与换行一起被移除，表示下一行仍属于同一条命令；反斜杠后若还有空格，续行可能失效。

## 5. 为什么变量通常写成 `"$variable"`

未引用的变量展开结果可能继续发生单词分割和文件名展开。变量中若包含空格、换行或 `*`，最终参数数量和内容可能改变。

**执行位置：Ubuntu 主机（任意目录；只改变当前 Shell 变量）**

```bash
file_name='weekly report.txt'

printf 'unquoted=<%s>\n' $file_name
printf 'quoted=<%s>\n' "$file_name"

unset file_name
```

预期结果：未引用形式通常输出两个参数 `weekly` 和 `report.txt`；引用形式输出一个完整参数 `weekly report.txt`。

安全习惯是：

- 路径变量默认写成 `"$path"`。
- 多个独立参数使用 Bash 数组，而不是把整条命令塞进一个字符串。
- 只有明确需要单词分割或文件名展开时，才有意识地省略引号。

> [!warning] 空变量与删除命令
> 不要把未经验证的变量直接拼入递归删除命令。执行文件变更前先检查变量是否非空、打印规范化目标，并把允许范围限制在明确目录内。加引号能避免分割和通配符展开，但不能证明目标路径本身正确。

## 6. 命令替换把输出放入当前位置

`$(command)` 先执行括号中的命令，再把其标准输出用于外层命令：

```bash
kernel_name="$(uname -s)"
printf 'kernel=%s\n' "$kernel_name"
```

命令替换常用于获取当前状态，但有两个边界：

- 末尾连续换行会被移除。
- 输出若包含多个文件名或多行记录，不应依赖未引用的命令替换进行拆分。

优先使用 `$()`，因为嵌套和阅读通常比旧式反引号清楚。命令替换由子 Shell 环境执行；其中对普通变量或当前目录的修改不会直接改变外层 Shell。

## 7. 通配符由 Shell 展开文件名

常见文件名模式：

| 模式 | 含义 | 不匹配路径分隔符 `/` |
| --- | --- | --- |
| `*` | 零个或多个字符 | 是 |
| `?` | 任意一个字符 | 是 |
| `[abc]` | 集合中的一个字符 | 是 |
| `[0-9]` | 指定范围中的一个字符 | 是 |

例如：

```bash
printf '%s\n' /etc/*.conf
```

`*.conf` 通常先由 Shell 展开成若干路径，`printf` 接收到的是展开后的路径列表。它不是正则表达式，也不是由 `printf` 搜索目录。

需要注意：

- `*` 默认不会匹配名称开头的 `.`，例如 `.env`。
- 模式没有匹配时，Bash 默认可能把原模式文本保留下来；Shell 选项可以改变此行为。
- 把模式放进双引号后会抑制文件名展开，例如 `"*.txt"` 表示字面文本。
- 模式范围必须在执行前核对，尤其是与 `rm`、`chmod -R`、`chown -R` 等变更命令组合时。

花括号展开如 `file-{1,2}.txt` 和算术展开如 `$((count + 1))` 也属于 Bash 常见展开。现阶段认识它们即可，读脚本时再按上下文查手册。

## 8. 展开顺序的实用模型

Bash 的完整处理规则有许多细节。初学阅读命令时，可以先使用下面的实用模型：

1. 根据未引用的空白和操作符识别单词、管道与重定向，同时遵守引号。
2. 进行花括号和波浪号等早期展开。
3. 展开变量、算术表达式和命令替换。
4. 对未被双引号保护的结果进行单词分割。
5. 对未被引用的模式进行文件名展开。
6. 移除只用于控制 Shell 的引号，再把最终参数交给命令。

这个模型能解释大多数日常命令，但不是 Bash 规范的替代品。遇到数组、进程替换、here-document 或复杂嵌套时，应查对应章节。

## 9. PATH 决定外部命令的查找目录

`PATH` 是由冒号分隔的目录列表。Shell 需要查找外部命令时，会按顺序寻找匹配的可执行文件。

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
printf 'PATH=%s\n' "$PATH"
command -v bash
command -v ls
type -a printf
```

当前目录通常不会自动包含在 PATH 中，因此运行当前目录下的脚本常写 `./script.sh`。不要为解决一次 `command not found` 就把 `.` 放进 PATH；这可能在进入不受信任目录时执行同名恶意文件。

临时或永久修改 PATH 前，应先确认：

- 新目录由谁控制。
- 新目录应放在原 PATH 前还是后。
- 是否会遮蔽系统已有命令。
- 修改只影响当前会话，还是写入了启动文件。

命令来源检查见 [[Shell 命令结构、类型与帮助系统]]。

## 10. 安全练习：观察引用和通配符

下面只在 `/tmp` 下创建唯一练习目录与普通文本文件，不使用 `sudo`。练习目录会保留，便于复查。

**执行位置：Ubuntu 主机（Bash；会在 `/tmp` 创建练习文件）**

```bash
if practice_dir="$(mktemp -d /tmp/shell-expansion.XXXXXX)"; then
  mkdir "$practice_dir/reports 2026"

  printf 'alpha\n' > "$practice_dir/reports 2026/report 1.txt"
  printf 'beta\n' > "$practice_dir/reports 2026/report-2.txt"
  printf 'hidden\n' > "$practice_dir/reports 2026/.draft.txt"

  report_dir="$practice_dir/reports 2026"
  printf 'quoted directory=<%s>\n' "$report_dir"
  printf 'matched=<%s>\n' "$report_dir"/*.txt
  printf 'quoted pattern=<%s>\n' "$report_dir/*.txt"
  printf 'practice_dir=%s\n' "$practice_dir"
else
  printf '%s\n' '无法创建练习目录，已停止文件操作。' >&2
fi
```

预期结果：

- 带空格的目录始终作为一个路径使用。
- 未引用的 `*.txt` 匹配两个非隐藏文本文件。
- 双引号中的 `*.txt` 作为字面模式输出。
- `.draft.txt` 不在默认 `*` 匹配结果中。

练习后尝试先预测、再运行：

```bash
if test -n "${report_dir:-}" && test -d "$report_dir"; then
  printf 'all entries=<%s>\n' "$report_dir"/.* "$report_dir"/*
fi
```

注意 `.*` 在不同 Shell 和选项下可能包含需要谨慎处理的结果。这里只交给 `printf` 输出，属于只读观察；不要直接把宽泛通配符接到删除或递归权限命令后。

### 不看笔记自测

1. `~`、`"~"`、`$HOME` 和 `"$HOME"` 在展开方式上有什么不同？
2. 为什么 `project_name = value` 不是变量赋值？
3. 单引号和双引号分别会怎样处理 `$name`？
4. 未引用的变量为什么可能从一个参数变成多个参数？
5. `"$directory/*.txt"` 与 `"$directory"/*.txt` 的匹配行为为什么不同？
6. `export` 会影响哪些进程，为什么它本身不能永久保存变量？

## 11. 常见误解

### “有引号的变量不会展开”

单引号会抑制变量展开，双引号不会。双引号通常正是安全展开变量的方式。

### “`~` 在任何位置都会变成 HOME”

不会。是否展开取决于位置和引用状态；需要稳定表达主目录路径时常用 `"$HOME/..."`。

### “通配符由命令处理”

日常 Bash 命令中，通配符通常先由 Shell 展开。目标命令往往只看到最终路径列表。

### “加上双引号就证明文件操作安全”

双引号解决参数分割和文件名展开问题，不会验证变量指向的是否是正确目录。变更操作仍要核对规范化路径、所有者和范围。

### “export 会永久保存变量”

`export` 只影响当前 Shell 后续启动的子进程。永久化需要修改相应启动配置，并在新会话验证。

## 完成标准

- [ ] 能区分绝对路径、相对路径、`.`、`..`、`~` 和 `$HOME`。
- [ ] 能正确书写变量赋值，并用 `${name}` 消除名称边界歧义。
- [ ] 能解释 Shell 变量与环境变量的区别。
- [ ] 能说明单引号、双引号和反斜杠的主要差异。
- [ ] 路径变量默认使用 `"$variable"`，并知道引用不能替代范围验证。
- [ ] 能解释命令替换和 `*`、`?` 的基本行为。
- [ ] 知道 PATH 影响外部命令查找，不随意把当前目录加入 PATH。

## 相关笔记

- [[Linux 命令行学习路线与命令地图]]
- [[Shell 命令结构、类型与帮助系统]]
- [[Shell 标准流、管道、重定向与退出状态]]
- [[Shell 脚本阅读基础]]
- [[Linux 文件与目录常用命令]]

## 官方参考资料

以下资料于 **2026-07-19** 核对：

- [GNU Bash：Shell 展开](https://www.gnu.org/software/bash/manual/html_node/Shell-Expansions.html)
- [GNU Bash：引用](https://www.gnu.org/software/bash/manual/html_node/Quoting.html)
- [GNU Bash：Shell 参数](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameters.html)
- [GNU Bash：环境](https://www.gnu.org/software/bash/manual/html_node/Environment.html)
- [GNU Bash：命令查找与执行](https://www.gnu.org/software/bash/manual/html_node/Command-Search-and-Execution.html)

---
title: Shell 标准流、管道、重定向与退出状态
aliases:
  - Bash 管道重定向与退出状态
  - Shell 标准输入输出基础
tags:
  - Linux
  - Linux/命令行
  - Shell
  - Bash
  - 管道
  - 重定向
created: 2026-07-19T23:48:38
updated: 2026-08-18T23:25:52
---

本文解释一条命令运行时，数据怎样进入和离开进程，以及进程结束后怎样用退出状态报告结果。目标是读懂日常管道、重定向和命令列表，能够区分“信息去了哪里”与“命令是否成功”，并在覆盖文件前识别风险。

开始前，应先完成 [[Shell 命令结构、类型与帮助系统]] 和 [[Shell 路径、变量、引用与展开]] 的最小主线。本文只负责命令之间的数据连接与状态组合，不重复解释参数、变量和引用规则。

## 本篇掌握目标

| 层级 | 内容 |
| --- | --- |
| 必须熟练 | stdin、stdout、stderr；`<`、`|`、`>`、`>>`、`2>`；退出状态；`&&`、`||`、`;` |
| 理解会查 | `2>&1` 的顺序、`tee`、`/dev/null`、管道默认状态和 `pipefail` |
| 认识即可 | Bash 的 `PIPESTATUS` 数组 |

本篇必须熟练的是判断数据去向、覆盖风险和成功失败。自定义文件描述符、here-string、进程替换等进阶连接方式不属于本篇最小主线，遇到时再查对应 Shell 手册。

## 1. 一条命令同时具有数据通道和退出状态

程序启动时通常会继承三个已经打开的文件描述符：

| 编号 | 名称 | 默认连接 | 用途 |
| --- | --- | --- | --- |
| 0 | 标准输入 stdin | 终端键盘或上游数据 | 程序读取输入 |
| 1 | 标准输出 stdout | 终端屏幕 | 正常结果 |
| 2 | 标准错误 stderr | 终端屏幕 | 诊断和错误信息 |

“标准流”是命令运行期间的输入输出通道，不等于某个固定文件。Shell 可以在命令启动前把这些通道重新连接到文件、管道或设备。命令结束后还会返回一个退出状态，供父 Shell 判断本次执行是否成功。

```text
输入数据
  -> stdin（0）
  -> 进程
      -> stdout（1）：正常结果
      -> stderr（2）：诊断信息

进程结束
  -> 退出状态
  -> 父 Shell
```

数据通道与退出状态是两个不同维度：命令显示了文字不等于执行成功，命令没有输出也不等于执行失败；重定向只改变信息去向，通常不会改变命令本身的退出状态。

下面的命令会向两个不同通道写内容，但默认都显示在当前终端：

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
printf 'normal output\n'
printf 'error output\n' >&2
```

`>&2` 把本次 `printf` 的标准输出改接到标准错误。终端外观可能相同，但重定向和管道可以把两个通道分开处理。

## 2. 重定向由 Shell 建立

`<`、`>`、`>>` 和 `2>` 等符号是 Shell 语法，不是普通参数。Shell 会先建立连接，再启动目标命令；目标命令通常不知道输入原本来自键盘还是文件，也不知道输出最终显示在终端还是写入文件。

基础骨架：

| 语法 | 含义 | 主要风险 |
| --- | --- | --- |
| `command < file` | 从文件连接标准输入 | 默认只读该文件 |
| `command > file` | 标准输出写入文件 | 文件存在时先截断 |
| `command >> file` | 标准输出追加到文件 | 内容会增长，可能写错目标 |
| `command 2> file` | 标准错误写入文件 | 文件存在时先截断 |
| `command 2>> file` | 标准错误追加到文件 | 内容会增长 |

### 标准输入不一定来自键盘

许多文本工具既能读取文件参数，也能从标准输入读取：

```bash
wc -l /etc/os-release
wc -l < /etc/os-release
```

两者都统计行数，但显示可能不同：

- 第一行把文件路径作为参数交给 `wc`，输出通常包含文件名。
- 第二行由 Shell 把文件连接到 stdin，`wc` 不知道文件名，通常只输出数字。

这一区别解释了为什么同一个工具既能独立处理文件，也能放在管道中处理上游结果。

### `>` 会截断已有文件

```text
some-command > important.txt
```

Shell 在执行命令前就会尝试打开并截断 `important.txt`。即使命令随后启动失败，原内容也可能已经丢失。

执行前至少确认：

- 当前目录和目标绝对路径。
- 目标是否已经存在。
- 需要覆盖还是追加。
- 命令失败时是否还需要保留旧文件。

对于重要文件，先写入同目录临时文件，验证成功后再受控替换，通常比直接覆盖安全。

### 重定向顺序会改变结果

Shell 从左到右处理重定向：

```text
command > combined.log 2>&1
```

先让 stdout 指向 `combined.log`，再让 stderr 复制 stdout 当前的去向，因此两者都进入文件。

```text
command 2>&1 > stdout.log
```

先让 stderr 复制 stdout 当时的去向，通常仍是终端；随后只有 stdout 改到文件。因此 stderr 不会跟着进入 `stdout.log`。

把 `2>&1` 理解为“让文件描述符 2 复制文件描述符 1 **此刻** 的连接”，比把它死记成“合并输出”更准确。

### `sudo` 不会提升当前 Shell 的重定向

下面的 `>` 仍由当前 Shell 在启动 `sudo` 前处理：

```text
sudo some-command > /root/result.txt
```

因此，打开 `/root/result.txt` 的通常还是当前用户，命令可能得到 `Permission denied`。若确实需要以管理员权限写系统文件，应先确认目标、内容和恢复方案，再让实际打开文件的程序获得相应权限，例如受控使用 `sudo tee`。不要为了绕过权限错误而盲目给整个 Shell 提权；`tee` 的数据流向见第 4 节。

## 3. 管道只连接 stdout 与 stdin

```text
producer | consumer
```

Shell 创建管道，让左侧命令的标准输出成为右侧命令的标准输入。两侧通常是独立进程，并可能并发运行。

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
printf 'alpha\nbeta\n' | wc -l
```

数据流可以画成：

```text
printf stdout
  -> wc stdin / stdout
  -> 当前终端
```

`wc -l` 已在上一节用于统计标准输入的行数。这里预期输出 `2`，重点只观察“左侧 stdout 成为右侧 stdin”，不额外引入尚未学习的排序和去重命令。

默认情况下，左侧命令的 stderr 不进入管道，仍连接到终端。需要把两者都送入管道时，必须有意识地重定向；这也会让正常数据与诊断信息混合，可能破坏下游格式。

这一节只回答数据怎样流动。上游失败后，下游仍可能成功处理空输入或部分输入，因此“终端最后有输出”不能证明整条管道成功；第 7 节会在理解退出状态后继续分析管道的状态流。

## 4. `tee` 与 `/dev/null` 改变输出去向

### `tee` 同时写终端和文件

`tee` 从标准输入读取内容，把它同时写到标准输出和一个或多个文件：

```text
producer | tee result.log
```

默认会覆盖目标文件；`tee -a result.log` 表示追加。它适合一边观察输出、一边保存记录，但仍然属于文件变更操作。

### `/dev/null` 丢弃不需要保留的输出

`/dev/null` 是一个特殊设备：它接受写入但不保存内容。把输出重定向到这里，相当于有意识地丢弃该通道的信息：

```text
command > /dev/null
command 2> /dev/null
command > /dev/null 2>&1
```

它适合明确不关心某类输出的检查，但不应作为“让报错消失”的默认办法。排障时通常应先保留 stderr 和退出状态，理解失败原因后再决定是否静默。

例如：

```bash
getent passwd "$USER" >/dev/null
```

这类检查不需要显示记录内容，而是通过退出状态判断用户是否存在。若命令可能因权限、网络或配置失败，单纯丢弃错误会让不同原因变得难以区分。

把原命令的输出交给 `tee` 或重定向到 `/dev/null`，不会自动改变原命令的退出状态。不过 `tee` 本身也是命令，写文件失败时会返回自己的非零状态；第 7 节会说明管道怎样汇总这些状态。下一节先建立单条命令的退出状态模型。

## 5. 退出状态表示命令结果

进程结束时会向父进程返回一个整数退出状态：

- `0` 通常表示成功。
- 非 `0` 表示某种失败或“不满足条件”，具体含义由命令定义。

`$?` 保存最近一个前台管道的退出状态。它会被下一条命令迅速覆盖，因此需要立即保存：

```bash
grep -q '^NAME=' /etc/os-release
grep_status=$?
printf 'grep_status=%s\n' "$grep_status"
```

`grep` 是一个典型例子：找到匹配通常返回 0，未找到通常返回 1，真正执行错误通常返回 2。不能把所有非零状态都简单翻译成“程序崩溃”；应查命令的 EXIT STATUS。

不要这样检查：

```text
some-command
printf 'finished\n'
printf '%s\n' "$?"
```

最后看到的是 `printf` 的状态，不再是 `some-command` 的状态。

## 6. 用命令列表决定后续是否执行

`;`、`&&` 和 `||` 都能把命令组织成列表，但只有后两者根据左侧状态决定是否运行右侧：

| 连接符 | 何时执行右侧 | 常见用途 |
| --- | --- | --- |
| `first ; second` | 无论 `first` 成败 | 明确顺序但不设成功条件 |
| `first && second` | `first` 成功时 | 前一步成功后继续 |
| `first || second` | `first` 失败时 | 失败处理或备用路径 |

`true` 不输出文字并返回状态 `0`，`false` 不输出文字并返回非零状态。先用它们隔离观察连接符本身：

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
true && printf 'left side succeeded\n'
false || printf 'left side failed\n'
false ; printf 'semicolon still continues\n'
```

真实命令可能用多个非零状态表达“没有匹配”“条件不成立”或“执行错误”。连接符只看零与非零，不理解这些业务含义；使用前仍应查询左侧命令的退出状态约定。路径、字符串和整数条件怎样通过 `test` 产生状态，见 [[Shell 脚本阅读基础#6. 使用 test 表达条件|test 条件判断]]。

这些连接符本身不等于完整错误处理。若右侧也可能失败，仍要检查它的状态。

### `|| true` 会故意把失败变成成功

```text
optional-command 2>/dev/null || true
```

这表示：无论 `optional-command` 为什么失败，整个列表最终都以 `true` 成功结束。它只适用于失败确实可忽略的可选探测，并应有上下文说明。

不要在安装、复制、权限变更、数据迁移或服务重启等关键步骤后随意添加 `|| true`；它会让后续步骤在错误前提下继续。

## 7. 管道同时具有数据流和状态流

第 3 节说明了管道的数据流：左侧 stdout 进入右侧 stdin。状态流是另一件事；Bash 默认把整条管道的状态设为最后一个命令的状态：

```bash
bash -c 'false | true; printf "default pipeline status=%s\n" "$?"'
```

预期输出是 `0`，尽管左侧 `false` 已失败。

启用 `pipefail` 后，只要管道中有命令失败，管道会返回最右侧失败命令的状态；全部成功才返回 0：

```bash
bash -o pipefail -c 'false | true; printf "pipefail status=%s\n" "$?"'
```

预期输出非 0。示例在子 Bash 中启用选项，不会改变当前交互 Shell。

Bash 的 `PIPESTATUS` 数组还能显示刚完成管道中各命令的状态，但它会被后续命令覆盖。初学阶段必须理解默认状态和 `pipefail`，`PIPESTATUS` 认识即可。

## 8. 安全练习：分别观察数据流与状态流

数据流练习会在 `/tmp` 创建唯一目录和练习文件，不使用 `sudo`；状态流练习只运行固定的成功、失败和子 Bash 命令，不写文件。先预测每一步，再执行并核对结果。

### 8.1 数据流练习：分离 stdout、stderr 并比较顺序

**执行位置：Ubuntu 主机（Bash；会在 `/tmp` 创建练习文件）**

```bash
if practice_dir="$(mktemp -d /tmp/shell-streams.XXXXXX)"; then
  missing_path="$practice_dir/not-created"

  printf 'alpha\nbeta\n' > "$practice_dir/input.txt"

  ls -l "$practice_dir/input.txt" \
    > "$practice_dir/stdout.txt" \
    2> "$practice_dir/stderr.txt"
  first_status=$?

  ls -l "$missing_path" \
    > "$practice_dir/missing-stdout.txt" \
    2> "$practice_dir/missing-stderr.txt"
  missing_status=$?

  printf 'first_status=%s missing_status=%s\n' \
    "$first_status" "$missing_status"
  wc -c < "$practice_dir/stdout.txt"
  wc -c < "$practice_dir/stderr.txt"
  wc -c < "$practice_dir/missing-stdout.txt"
  wc -c < "$practice_dir/missing-stderr.txt"

  printf 'one\ntwo\n' | tee "$practice_dir/tee-copy.txt"

  bash -c 'printf "out\n"; printf "err\n" >&2' \
    > "$practice_dir/combined.txt" 2>&1

  bash -c 'printf "out\n"; printf "err\n" >&2' \
    2>&1 > "$practice_dir/stdout-only.txt"

  printf '%s\n' '--- combined.txt ---'
  cat "$practice_dir/combined.txt"
  printf '%s\n' '--- stdout-only.txt ---'
  cat "$practice_dir/stdout-only.txt"
  printf 'practice_dir=%s\n' "$practice_dir"
else
  printf '%s\n' '无法创建练习目录，已停止文件操作。' >&2
fi
```

预期结果：

- 第一次 `ls` 成功，正常信息进入 `stdout.txt`，错误文件通常为空。
- 第二次 `ls` 失败，错误信息进入 `missing-stderr.txt`，正常输出文件通常为空。
- `first_status` 为 `0`，`missing_status` 为非 `0`。
- `tee` 在终端显示两行，并把同样内容写入 `tee-copy.txt`。
- 重定向顺序练习的第二条命令会把 `err` 直接显示在终端，而 `stdout-only.txt` 中只有 `out`；`combined.txt` 则包含两行。

### 8.2 状态流练习：比较命令列表与管道状态

**执行位置：Ubuntu 主机（Bash；任意目录，只读）**

```bash
true && printf 'left side succeeded\n'
false || printf 'left side failed\n'
false ; printf 'semicolon still continues\n'

bash -c 'false | true; printf "default pipeline status=%s\n" "$?"'
bash -o pipefail -c 'false | true; printf "pipefail status=%s\n" "$?"'
```

预期结果：

- `&&` 右侧只在左侧成功时执行，`||` 右侧只在左侧失败时执行，`;` 右侧始终执行。
- 默认管道状态为 `0`，因为最后一个命令 `true` 成功。
- 启用 `pipefail` 后管道状态为非 `0`，因为上游 `false` 失败。

### 8.3 核对并清理练习目录

数据流练习故意保留目录，便于先检查文件。请在运行练习的同一个 Bash 会话中继续下面步骤；若变量不存在，保护分支会停止。下面只读核对变量形状、目录类型和目录内容；其中六个 `?` 对应 `mktemp` 模板中的六个 `X`。这段保护结构会在下一篇 [[Shell 脚本阅读基础]] 中展开，此处只需按顺序核对每个停止条件。

```bash
case "${practice_dir:-}" in
  /tmp/shell-streams.??????)
    if test -d "$practice_dir" && ! test -L "$practice_dir"; then
      printf '待清理的练习目录：%s\n' "$practice_dir"
      ls -la -- "$practice_dir"
    else
      printf '停止：目标不是预期的非符号链接目录。\n' >&2
    fi
    ;;
  *)
    printf '停止：练习目录变量不符合预期模式。\n' >&2
    ;;
esac
```

确认上一步只显示本次练习生成的文件后，再运行下面的清理代码。它只删除已知文件名，再用 `rmdir` 删除空目录；如果目录中出现其他内容，`rmdir` 会失败并保留未预期内容供检查，不会递归删除未知文件。

```bash
case "${practice_dir:-}" in
  /tmp/shell-streams.??????)
    if test -d "$practice_dir" && ! test -L "$practice_dir"; then
      rm -f -- \
        "$practice_dir/input.txt" \
        "$practice_dir/stdout.txt" \
        "$practice_dir/stderr.txt" \
        "$practice_dir/missing-stdout.txt" \
        "$practice_dir/missing-stderr.txt" \
        "$practice_dir/tee-copy.txt" \
        "$practice_dir/combined.txt" \
        "$practice_dir/stdout-only.txt"

      if rmdir -- "$practice_dir"; then
        if test ! -e "$practice_dir" && test ! -L "$practice_dir"; then
          printf '练习目录已清理：%s\n' "$practice_dir"
        else
          printf '目录项仍然存在，请停止并检查：%s\n' "$practice_dir" >&2
        fi
      else
        printf '目录未清理，可能存在未预期内容：%s\n' "$practice_dir" >&2
        ls -la -- "$practice_dir"
      fi
    else
      printf '停止：目标不是预期的非符号链接目录。\n' >&2
    fi
    ;;
  *)
    printf '停止：练习目录变量不符合预期模式。\n' >&2
    ;;
esac
```

### 8.4 不看笔记自测

1. 为什么 stdout 和 stderr 默认都显示在屏幕上，却仍是两个通道？
2. `>` 与 `>>` 对已存在文件的影响有什么不同？
3. `command > file 2>&1` 和 `command 2>&1 > file` 为什么不等价？
4. 管道默认连接左侧的哪个通道与右侧的哪个通道？
5. 为什么读取 `$?` 前要先保存它？
6. `;`、`&&` 和 `||` 分别怎样决定右侧是否执行？
7. `|| true` 与 `2>/dev/null` 分别改变了状态还是信息去向？
8. 默认管道状态与启用 `pipefail` 后有什么区别？
9. 清理练习目录前为什么要先核对路径形状、目录类型和实际内容？

## 9. 常见误解

### “屏幕上看到的都是标准输出”

stderr 默认也显示在终端。需要重定向后才能清楚区分。

### “`>` 是命令的参数”

它通常由 Shell 处理，而且会在命令运行前打开或截断文件。

### “管道会自动传递错误信息”

默认只连接左侧 stdout 到右侧 stdin，stderr 仍走原来的通道。

### “最后一个命令成功，整条管道就没有问题”

Bash 默认状态确实来自管道最后一个命令，但上游仍可能失败。关键脚本应考虑 `pipefail` 和分步骤验证。

### “隐藏 stderr 可以让命令成功”

重定向只改变信息去向，不改变原命令的退出状态。`|| true` 才会进一步把列表结果变成成功，但也会掩盖故障。

## 完成标准

- [ ] 能解释 stdin、stdout、stderr 和文件描述符 0、1、2。
- [ ] 能区分 `>`、`>>`、`2>`、`2>&1`，并知道 `>` 会截断文件。
- [ ] 能说明重定向顺序为什么重要。
- [ ] 能解释管道默认只连接 stdout 与 stdin。
- [ ] 能用退出状态区分成功与失败，并及时保存 `$?`。
- [ ] 能说明 `;` 无条件继续，`&&` 与 `||` 分别根据成功和失败决定是否继续。
- [ ] 能区分 `2>/dev/null` 与 `|| true` 改变的是信息去向还是最终状态。
- [ ] 知道 Bash 管道默认返回最后一个命令的状态，以及 `pipefail` 解决什么问题。

## 相关笔记

- [[Linux 命令行学习路线与命令地图]]
- [[Shell 命令结构、类型与帮助系统]]
- [[Shell 路径、变量、引用与展开]]
- [[Shell 脚本阅读基础]]
- [[Linux 文本查看、搜索与处理常用命令]]

## 官方参考资料

以下资料于 **2026-07-19** 核对：

- [GNU Bash：重定向](https://www.gnu.org/software/bash/manual/html_node/Redirections.html)
- [GNU Bash：管道](https://www.gnu.org/software/bash/manual/html_node/Pipelines.html)
- [GNU Bash：命令列表](https://www.gnu.org/software/bash/manual/html_node/Lists.html)
- [GNU Bash：退出状态](https://www.gnu.org/software/bash/manual/html_node/Exit-Status.html)
- [GNU Coreutils：tee](https://www.gnu.org/software/coreutils/manual/html_node/tee-invocation.html)

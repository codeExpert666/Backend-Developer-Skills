---
title: Shell 脚本阅读基础
aliases:
  - Bash 脚本阅读入门
  - Shell 判断循环与函数基础
tags:
  - Linux
  - Linux/命令行
  - Shell
  - Bash
  - Shell/脚本
created: 2026-07-19T23:48:38
updated: 2026-08-18T23:22:57
---

本文帮助初学者读懂 Ubuntu Server 笔记中常见的 Bash 脚本：输入从哪里来、哪些条件会停止、哪些步骤修改系统、怎样验证结果、退出时怎样清理。本文不是完整 Bash 编程教程，也不要求立即独立编写复杂自动化脚本。

开始前，应先完成 [[Shell 路径、变量、引用与展开]] 和 [[Shell 标准流、管道、重定向与退出状态]] 的最小主线。本文会直接使用变量、引用、命令替换、重定向和退出状态，不再重复展开这些前置概念。

## 本篇掌握目标

| 层级 | 内容 |
| --- | --- |
| 必须熟练 | 识别解释器和调用方式，读懂位置参数、`test`、`[ ... ]`、`if`、`case`、`for`、退出状态和明显变更命令 |
| 理解会查 | `while read`、函数、`local`、`trap`、here-document（此处文档）、`set -u` 与 `pipefail` |
| 认识即可 | 数组高级操作、进程替换、作业控制、复杂算术和 Bash 调试设施 |

学习目标是“能安全审阅和小幅修改”，不是背诵所有语法。

## 1. 先看一个完整脚本

先完整看一遍下面的 `inspect-paths.sh`，暂时不要逐行查语法，也不要立即执行：

```bash
#!/usr/bin/env bash

set -u

if [ "$#" -eq 0 ]; then
  printf 'usage: %s PATH...\n' "$0" >&2
  exit 2
fi

overall_status=0

for path in "$@"; do
  if [ -f "$path" ]; then
    line_count="$(wc -l < "$path")"
    printf 'file lines=%s path=%s\n' "$line_count" "$path"
  elif [ -d "$path" ]; then
    printf 'directory path=%s\n' "$path"
  else
    printf 'unsupported path=%s\n' "$path" >&2
    overall_status=1
  fi
done

exit "$overall_status"
```

这段脚本接收一个或多个路径：普通文件会报告行数，目录会报告类型，不存在或不属于这两类的路径会写出错误。它只读取传入路径，不修改这些目标。

| 调用结果 | 输出与退出状态 |
| --- | --- |
| 没有传入路径 | 把用法写到 stderr，并以状态 `2` 退出 |
| 所有路径都是普通文件或目录 | 把结果写到 stdout，并以状态 `0` 退出 |
| 至少一个路径未被识别 | 继续检查其余路径，最终以状态 `1` 退出 |

第一次阅读不需要记住所有符号。先建立整体印象，下一节再按固定顺序把它拆开。

## 2. 用固定顺序建立脚本地图

面对较长脚本，不要逐字符从头背到尾。第一遍只做结构扫描：

1. **解释器和 Shell 选项**：它要求 Bash 还是通用 `sh`，是否启用 `set` 选项。
2. **输入**：位置参数、环境变量、标准输入、配置文件和交互式 `read`。
3. **前置检查**：身份、命令是否安装、路径是否存在、变量是否合法。
4. **控制流**：哪些条件选择分支，哪些循环会重复执行操作。
5. **变更操作**：创建、覆盖、删除、安装、权限、服务、网络和远程操作。
6. **验证与输出**：状态、文件内容、stdout、stderr、服务日志和外部可用性。
7. **失败与清理**：`exit`、`return`、`trap`、临时文件和恢复逻辑。

把这套顺序应用到第 1 节脚本，可以先得到一张地图：

| 位置 | 阅读结论 |
| --- | --- |
| `#!/usr/bin/env bash` | 直接执行时需要 Bash |
| `set -u` | 使用未设置变量通常会报错 |
| `"$#"`、`"$0"`、`"$@"` | 输入来自位置参数；脚本名用于用法提示 |
| 第一个 `if` | 没有参数时立即停止，不进入循环 |
| `for path in "$@"` | 每个原始参数都会被检查一次 |
| `if`、`elif`、`else` | 每个路径只进入一个类型分支 |
| `wc` 与 `printf` | 前者读取文件并计算行数，后者报告结果或错误 |
| `overall_status` 与最后的 `exit` | 单个失败不会中断循环，但会影响脚本最终状态 |

这一步还应得出一个重要结论：脚本主体没有修改传入路径，也没有清理任务。第二遍再沿文本顺序阅读每个分支，确认上述判断是否成立。后续各节会解释这张地图中的语法。

## 3. 先确认解释器、调用方式与影响范围

脚本第一行常见解释器声明（shebang）：

```bash
#!/usr/bin/env bash
```

它表示直接执行文件时，通过 `env` 在 PATH 中查找 `bash` 作为解释器。另一种常见写法是：

```bash
#!/bin/bash
```

它使用固定路径。选择哪种方式取决于部署约束；阅读时先确认项目约定和目标机器上的实际路径。

三种调用方式含义不同：

| 调用方式 | 解释器与环境 | 影响范围 |
| --- | --- | --- |
| `./script.sh` | 内核根据 shebang 选择解释器，文件还需可执行权限 | 新进程 |
| `bash script.sh` | 明确让当前 PATH 找到的 Bash 读取文件，通常不要求可执行权限 | 新 Bash 进程 |
| `source script.sh` 或 `. script.sh` | 由当前 Shell 在当前环境执行 | 可直接改变当前目录、变量、函数和 Shell 选项 |

不要为了“运行成功”随意把 `bash` 换成 `sh`。Ubuntu 的 `/bin/sh` 通常不是 Bash；数组、`[[ ... ]]`、进程替换等 Bash 语法可能失败或产生不同结果。

> [!warning] source 的影响更直接
> `source` 不是普通的脚本启动方式。被读取文件中的 `cd`、`export`、`unset`、函数定义和 Shell 选项都会影响当前会话。只 source 你已经审阅并明确需要加载到当前 Shell 的文件。

## 4. 找出脚本输入及参数边界

脚本输入不只来自命令行。阅读时应同时检查位置参数、环境变量、标准输入、配置文件和交互式输入。本节先处理第 1 节脚本使用的位置参数和特殊参数：

| 写法 | 类别 | 含义 |
| --- | --- | --- |
| `$1`、`$2` | 位置参数 | 第 1、2 个调用参数 |
| `$0` | 特殊参数 | 脚本或 Shell 的名称 |
| `$#` | 特殊参数 | 位置参数数量 |
| `"$@"` | 特殊参数 | 保持参数边界地传递全部位置参数 |
| `$?` | 特殊参数 | 最近一个前台管道的退出状态 |

第 1 节脚本中的参数数量检查是：

```text
if [ "$#" -eq 0 ]; then
  printf 'usage: %s PATH...\n' "$0" >&2
  exit 2
fi
```

随后，`for path in "$@"` 逐个取得调用参数。`"$@"` 很重要：若调用者传入两个参数，其中一个包含空格，`"$@"` 仍保留两个参数；未引用的 `$@` 或 `$*` 可能重新分割内容。

环境变量也是输入。脚本看到 `PROJECT_DIR`、`HTTP_PROXY` 或 `JAVA_HOME` 时，应继续找：

- 未设置时是否有默认值。
- 是否在使用前验证格式和范围。
- 是否可能包含密码、令牌或其他敏感内容。
- 是否会被打印到日志。

标准输入和配置文件也可能改变脚本行为。看到 `read`、`< FILE`、管道或 here-document 时，应继续追踪数据来源；看到脚本读取配置文件时，还要检查路径由谁决定、内容是否可信。`while read` 与 here-document 分别在第 9、13 节展开。

## 5. 先理解“退出状态 → 条件 → 分支”

Shell 不要求 `if` 后面出现一种专门的布尔表达式。它会执行 `if` 与 `then` 之间的命令，并根据命令的退出状态选择分支：状态 `0` 表示条件成立，非 `0` 表示没有成功满足该条件。

```bash
if test -f "$target_path"; then
  printf 'regular file: %s\n' "$target_path"
elif test -d "$target_path"; then
  printf 'directory: %s\n' "$target_path"
else
  printf 'unsupported path: %s\n' "$target_path" >&2
fi
```

这段代码包含三个连续角色：

1. `test` 检查一个条件，并用退出状态报告结果，通常不输出文字。
2. `if` 和 `elif` 依次检查这些状态，第一个成立的分支会被执行。
3. 前面的条件都不成立时进入 `else`，`fi` 结束整个结构。

因此，阅读条件代码时不要只问“表达式是真是假”，还要问“实际执行了哪个命令、哪些非零状态可能出现”。`&&`、`||` 与 `!` 也根据退出状态组织流程，通用规则见 [[Shell 标准流、管道、重定向与退出状态#5. 退出状态表示命令结果]]。下一节深入整理 `test`，第 7 节再说明 `if` 的分支边界。

## 6. 使用 test 表达条件

`test` 不会输出 `true` 或 `false`，而是计算一个条件并通过退出状态报告结果：条件成立通常返回 `0`，不成立通常返回 `1`；表达式写错还可能返回其他非零状态并输出诊断。`if`、`!`、`&&` 与 `||` 正是根据这个状态决定下一步，退出状态的通用规则见 [[Shell 标准流、管道、重定向与退出状态#5. 退出状态表示命令结果]]。

第一次阅读先掌握 `test`、`[ ... ]` 的命令形式，以及 `-e`、`-f`、`-d`、`-n`、`-z` 和整数比较。符号链接、访问条件、`sudo test` 与 `[[ ... ]]` 的边界保留在本节，遇到对应脚本时再回来查询。

最小骨架是：

```text
test 条件表达式
[ 条件表达式 ]
```

在 Bash 和 Zsh 中，`test` 与 `[` 通常都是 Shell 内建命令；系统也可能提供同名外部程序。`[ ... ]` 是 `test` 的另一种命令形式，末尾的 `]` 也是必需参数，不是排版括号。因此所有运算符、操作数和结尾的 `]` 都必须分别成为参数：

```bash
if [ -f "$target_path" ]; then
  printf 'regular file: %s\n' "$target_path"
fi
```

错误写法 `[-f "$target_path"]` 缺少必要空格，Shell 会把它当成另一个命令名。

### 6.1 检查路径是否存在、属于什么类型或能否访问

| 条件 | 成立条件 | 当前目标 |
| --- | --- | --- |
| `-e PATH` | 路径解析后指向一个存在的对象 | 必须熟练 |
| `-f PATH` | 路径解析后指向普通文件 | 必须熟练 |
| `-d PATH` | 路径解析后指向目录 | 必须熟练 |
| `-L PATH` | 路径最后一段本身是符号链接 | 理解会查 |
| `-r PATH` | 以当前执行身份判断为可读 | 理解会查 |
| `-w PATH` | 以当前执行身份判断为可写 | 理解会查 |
| `-x PATH` | 文件可执行；目录可进入或搜索 | 理解会查 |
| `-s PATH` | 路径存在且大小大于 `0` | 理解会查 |

除 `-L`（以及同义的 `-h`）外，这些文件条件通常会跟随符号链接，检查链接指向的对象。因此，指向普通文件的符号链接可以同时满足 `-f` 和 `-L`；指向不存在目标的悬空链接通常不满足 `-e`，但仍满足 `-L`。

这解释了安全脚本中常见的组合：

```bash
if test -e "$target_path" || test -L "$target_path"; then
  printf '路径已经被对象或符号链接占用：%s\n' "$target_path" >&2
fi
```

`-e || -L` 把悬空链接也视为“路径已占用”，适合在创建文件、目录或链接前避免覆盖已有目录项。

```bash
if ! test -f "$key_path" || test -L "$key_path"; then
  printf '不是预期的非符号链接普通文件：%s\n' "$key_path" >&2
fi
```

这段保护条件会拒绝不存在的路径、目录、其他特殊文件以及符号链接；只有路径本身不是符号链接、解析结果又是普通文件时才继续。

`-r`、`-w` 和 `-x` 反映的是执行 `test` 的身份所看到的访问条件，不保证后续操作一定成功。父目录权限、只读文件系统、ACL（访问控制列表）、竞态和后续命令本身都可能改变结果。看到 `sudo test ...` 时，表示只让这一次 `test` 以提升后的身份运行；它不会自动提升同一行之外的其他命令，也不会改变 Shell 自己处理的重定向。

### 6.2 检查字符串和整数

| 条件 | 成立条件 | 当前目标 |
| --- | --- | --- |
| `-n STRING` | 字符串长度不为 `0` | 必须熟练 |
| `-z STRING` | 字符串长度为 `0` | 必须熟练 |
| `STRING_A = STRING_B` | 两个字符串相等 | 必须熟练 |
| `STRING_A != STRING_B` | 两个字符串不相等 | 理解会查 |
| `INTEGER_A -eq INTEGER_B` | 两个整数相等 | 理解会查 |
| `-ne`、`-lt`、`-le`、`-gt`、`-ge` | 整数不等、小于、小于等于、大于、大于等于 | 理解会查 |

来自变量的操作数应引用，并在允许变量未设置时显式提供默认值：

```bash
test -n "${JAVA_HOME:-}"
test "$rollback_mode" = restore
test "${retry_count:-0}" -ge 3
```

字符串比较使用 `=`，整数比较使用 `-eq` 等运算符；不要用 `=` 比较整数，也不要把 `-eq` 用于任意文本。

### 6.3 用 Shell 组合条件

以下三种写法分别表示正向判断、Shell 取反和失败时执行右侧命令：

```bash
if test -f "$config_file"; then
  printf 'config exists\n'
fi

if ! test -s "$config_file"; then
  printf 'config is missing or empty\n' >&2
fi

test -d "$work_dir" || printf 'work directory is unavailable\n' >&2
```

`! test -e "$path"` 中的 `!` 由 Shell 处理；`test ! -e "$path"` 则把 `!` 作为 `test` 表达式的一部分。两者都能表达取反，但阅读多条命令组成的条件时，优先明确每一层由 Shell 还是 `test` 解释。

不要为了把所有条件塞进一次 `test` 而依赖容易混淆的 `-a`、`-o` 或括号表达式。仓库中的可读写法优先使用多个完整命令，并由 Shell 的 `&&`、`||` 和分行控制组合；它们还能明确是否需要短路执行右侧条件。

### 6.4 区分 `test`、`[ ... ]` 与 `[[ ... ]]`

| 形式 | 本质 | 主要边界 |
| --- | --- | --- |
| `test 表达式` | 命令 | 适合脚本和交互检查，也可以写成 `sudo test ...` |
| `[ 表达式 ]` | `test` 的命令形式 | 每一部分都要留空格，最后的 `]` 不可省略 |
| `[[ 表达式 ]]` | Bash、Zsh 等 Shell 的条件语法 | 支持更强的模式处理，但不是通用 `/bin/sh` 语法，也不能直接写成 `sudo [[ ... ]]` |

看到 `[[ ... ]]` 时先确认 shebang 或当前 Shell。不要仅为追求写法统一，就在面向 `/bin/sh` 的脚本中引入它；也不要假设三种形式在模式匹配、引用和扩展运算符上完全相同。

## 7. `if` 根据状态选择分支

Shell 的 `if` 判断的是命令或命令列表的退出状态，不要求条件必须写成布尔表达式。完整结构可以包含零个或多个 `elif`，以及可选的 `else`：

```text
if 第一组命令; then
  第一条分支
elif 第二组命令; then
  第二条分支
else
  兜底分支
fi
```

Shell 依次检查 `if` 和各个 `elif`，执行第一个状态为 `0` 的分支；已经进入一个分支后，不再继续匹配后续分支。下面的例子只有 `if` 和 `else`：

```bash
if grep -q '^NAME=' /etc/os-release; then
  printf 'NAME field exists\n'
else
  printf 'NAME field is absent or grep failed\n' >&2
fi
```

这段代码把 `grep` 的所有非零状态都归入 `else`。严谨脚本有时需要区分“没有匹配”和“读取失败”，因此还应查看目标命令的退出状态约定。

阅读 `if` 时按顺序问：

1. 条件部分实际运行了什么？
2. 哪些退出状态进入 `then`？
3. 后续 `elif` 是否仍有机会执行？
4. `else` 是否把多种失败原因混在一起？
5. 各个分支分别修改什么？
6. `fi` 之后是否假定某个分支已经成功？

## 8. `case` 适合有限取值和模式检查

```text
case "$action" in
  start|stop|restart)
    printf 'accepted action: %s\n' "$action"
    ;;
  '')
    printf 'action is empty\n' >&2
    exit 2
    ;;
  *)
    printf 'unsupported action: %s\n' "$action" >&2
    exit 2
    ;;
esac
```

`case` 的模式不是完整正则表达式，而是 Shell 模式。`*` 表示兜底分支。涉及服务名、用户名或动作选择时，明确允许值通常比“接受任意输入再拼接命令”安全。

每个模式后的 `)` 开始对应分支，`;;` 结束该分支，`esac` 结束整个结构。初学阶段先能找出允许值、兜底分支和每个分支的退出行为，不必背诵低频终止符变体。

## 9. 循环会放大单次操作的范围

### 9.1 `for` 保留参数边界

```bash
for path in "$@"; do
  printf 'checking: %s\n' "$path"
done
```

`"$@"` 让每个原始参数成为一次循环值。阅读循环时必须确认集合从哪里来；若来源是宽泛通配符或未引用变量，一次错误会被重复应用到许多文件。

### 9.2 `while IFS= read -r` 按行读取

```bash
while IFS= read -r line || [ -n "$line" ]; do
  printf 'line=<%s>\n' "$line"
done < /etc/os-release
```

- `IFS=` 避免删除行首和行尾的 IFS 空白。
- `read -r` 避免把反斜杠当成转义字符。
- `|| [ -n "$line" ]` 让最后一行即使缺少换行符，只要仍有内容也会被处理。
- `< /etc/os-release` 让循环从文件的标准输入读取。

这适合一般文本行，但不适合无损遍历任意 Linux 文件名，因为文件名可以包含换行。需要批量处理文件时，优先使用能传递 NUL（空字符）分隔记录的工具模式，并在实际场景中查对应手册。

## 10. 函数封装步骤，但不会自动隔离影响

### 10.1 函数仍在当前 Shell 上下文执行

```bash
print_error() {
  local message=$1
  printf 'error: %s\n' "$message" >&2
}
```

函数在当前 Shell 上下文执行。函数内的 `cd`、变量和 Shell 选项可能影响后续代码；`local` 可以把变量限制在函数调用范围内。

| 语句 | 作用范围 |
| --- | --- |
| `return STATUS` | 结束当前函数，并把状态交给调用者 |
| `exit STATUS` | 结束整个脚本或当前 Shell 进程 |

读取函数时不要只看定义位置，还要搜索所有调用点、参数来源和返回状态是否被检查。

### 10.2 圆括号分组可以限制部分会话影响

圆括号分组会在子 Shell 环境执行：

```bash
(
  cd /tmp || exit 1
  pwd
)

pwd
```

子 Shell 中的 `cd` 不会改变外层 Shell 的目录。花括号分组 `{ ...; }` 通常在当前 Shell 执行，影响范围不同；初学阶段认识这个边界即可。

## 11. `trap` 为退出安排清理

脚本常创建临时文件，随后用 `trap` 注册清理函数：

**执行位置：Bash（会在 `/tmp` 创建并删除一个临时目录）**

```bash
(
  if ! work_dir="$(mktemp -d /tmp/script-work.XXXXXX)"; then
    printf '%s\n' '无法创建临时目录，已停止。' >&2
    exit 1
  fi
  readonly work_dir

  cleanup() {
    case "$work_dir" in
      /tmp/script-work.*)
        rm -rf -- "$work_dir"
        ;;
      *)
        printf 'skip cleanup: unexpected path: %s\n' "$work_dir" >&2
        ;;
    esac
  }

  trap cleanup EXIT
  printf '临时目录会在这个子 Shell 退出时清理：%s\n' "$work_dir"
)
```

`EXIT` 表示当前 Shell 进程退出时调用函数。示例显式放在圆括号创建的子 Shell 中，因此粘贴到交互终端后，`trap` 不会残留到当前登录 Shell。这里的保护点包括：

- 路径由 `mktemp -d` 创建，而不是自行拼接固定名称。
- 变量设为只读，避免后续被意外改写。
- 清理前再次限制路径模式。
- `rm` 使用 `--` 结束选项。

`trap` 不等于事务回滚。脚本若已修改服务、用户、数据库或远程系统，删除临时目录不会撤销这些变化。真正的恢复路径必须针对每项持久变更设计和验证。

本节只建立 `EXIT` 清理模型，不展开中断信号 `INT`、终止信号 `TERM` 等信号处理。遭遇不可捕获的强制终止信号 `SIGKILL`、机器掉电或清理逻辑没有获得执行机会时，清理函数仍可能无法运行。

## 12. `set -euo pipefail` 不是自动安全开关

常见脚本开头：

```text
set -euo pipefail
```

| 选项 | 主要作用 | 需要注意 |
| --- | --- | --- |
| `-e` | 在部分未处理的非零状态后退出 | 在条件、列表和其他语境中有例外，不能替代显式检查 |
| `-u` | 展开未设置变量时通常报错 | 可选变量需要显式默认值 |
| `-o pipefail` | 管道中有命令失败时让管道反映失败 | 仍要理解每个命令允许哪些非零状态 |

这些选项能暴露部分错误，但也可能改变原有控制流。不要只看见 `set -e` 就认定“所有失败都会停止”，也不要在旧脚本中机械添加后认为任务完成。

关键变更更适合显式写出：

```text
if ! some-important-command; then
  printf 'important command failed\n' >&2
  exit 1
fi
```

随后还应验证预期状态，而不是只相信退出码。

## 13. 读懂逻辑行、注释与多行输入

脚本中的一条逻辑命令不一定只占一个物理行，多行文本也不一定来自文件。阅读跨行代码时，先识别续行、注释和 here-document 的边界。

### 13.1 长命令续行

```bash
printf 'first=%s second=%s\n' \
  'one' \
  'two'
```

未引用的行尾 `\` 与紧随其后的换行一起被移除。反斜杠后不能再留普通空格，否则可能不再续行。

### 13.2 注释

未被引用的 `#` 在合适位置开始注释。引号中的 `#` 是普通字符：

```bash
printf '%s\n' '# not a comment here'
```

注释应该解释原因、风险或约束，不应重复显而易见的命令名称。

### 13.3 here-document

here-document 把多行文本连接到命令的标准输入，数据流向仍遵循 [[Shell 标准流、管道、重定向与退出状态]]：

```bash
cat <<'TEXT'
$HOME is not expanded in this block.
TEXT
```

分隔符 `TEXT` 被单引号保护，因此块内不会做变量、命令和算术展开。未引用分隔符的行为不同，阅读生成配置文件的脚本时必须先检查这一点。

## 14. 从审阅到执行：检查、练习与清理

### 14.1 语法检查之前先完成范围审查

准备运行脚本时，先确认来源和解释器，再按第 2 节的顺序找出输入、权限、文件、服务、网络、远程操作以及失败后的残留。语法工具只能补充这次人工审阅，不能替代它。

对已经确认应由 Bash 解释的脚本，可以先做语法检查：

**执行位置：保存待检查脚本的目录（只读取脚本，不执行普通命令）**

```bash
bash -n script.sh
```

`-n` 读取并检查语法而不执行普通命令。它能发现缺少 `fi`、引号未闭合等语法错误，但不能证明脚本安全或逻辑正确，也不会验证路径是否存在、权限是否足够、远程服务是否可用。

若当前环境已经安装 ShellCheck，还可以进行静态分析：

```bash
shellcheck script.sh
```

ShellCheck 能提示许多未引用变量和可疑写法，但诊断需要结合目标 Shell 和脚本意图判断。不要为了消除告警机械改变行为。下载的脚本即使通过 `bash -n` 和 ShellCheck，也仍可能删除文件、上传数据或修改系统；语法通过不是信任证明。

### 14.2 在临时目录复现同一个脚本

下面的练习会在 `/tmp` 下创建唯一目录、两个练习对象和第 1 节脚本，不使用 `sudo`，也不修改系统配置。here-document 的分隔符已引用，因此当前 Shell 不会提前展开脚本正文中的变量。

**执行位置：Ubuntu 主机（Bash；会在 `/tmp` 创建练习文件）**

```bash
if practice_dir="$(mktemp -d /tmp/bash-reading.XXXXXX)"; then
  script_path="$practice_dir/inspect-paths.sh"

  printf 'alpha\nbeta\n' > "$practice_dir/sample file.txt"
  mkdir "$practice_dir/sample-dir"

  cat > "$script_path" <<'BASH'
#!/usr/bin/env bash

set -u

if [ "$#" -eq 0 ]; then
  printf 'usage: %s PATH...\n' "$0" >&2
  exit 2
fi

overall_status=0

for path in "$@"; do
  if [ -f "$path" ]; then
    line_count="$(wc -l < "$path")"
    printf 'file lines=%s path=%s\n' "$line_count" "$path"
  elif [ -d "$path" ]; then
    printf 'directory path=%s\n' "$path"
  else
    printf 'unsupported path=%s\n' "$path" >&2
    overall_status=1
  fi
done

exit "$overall_status"
BASH

  if bash -n "$script_path"; then
    bash "$script_path" \
      "$practice_dir/sample file.txt" \
      "$practice_dir/sample-dir"
    first_status=$?

    bash "$script_path" "$practice_dir/not-created"
    missing_status=$?

    bash "$script_path"
    usage_status=$?

    printf 'first_status=%s missing_status=%s usage_status=%s\n' \
      "$first_status" "$missing_status" "$usage_status"
    printf 'practice_dir=%s\n' "$practice_dir"
  else
    printf '%s\n' '语法检查失败，未运行练习脚本。' >&2
  fi
else
  printf '%s\n' '无法创建练习目录，已停止文件操作。' >&2
fi
```

预期结果：

- `bash -n` 没有输出并返回 `0`，表示语法检查通过。
- 第一次运行识别一个两行文件和一个目录，`first_status=0`。
- 第二次运行把不受支持的缺失路径写到 stderr，`missing_status=1`。
- 第三次运行没有传入路径，把用法写到 stderr，`usage_status=2`。
- 包含空格的文件路径始终作为一个参数传递。
- 脚本不需要可执行权限，因为示例明确使用 `bash "$script_path"`。
- 外层练习负责创建文件和目录；`inspect-paths.sh` 本身仍只读取传入路径。

### 14.3 先检查目标，再清理练习目录

练习故意保留目录，便于先查看生成结果。下面的代码只读检查变量、目录类型和目录内容；路径模式中的每个 `?` 匹配一个字符，六个 `?` 对应 `mktemp` 模板中的六个 `X`。若变量不符合这个形状，或目标不是非符号链接目录，就停止检查：

```bash
case "${practice_dir:-}" in
  /tmp/bash-reading.??????)
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

确认上一步只显示本次练习对象后，再运行清理。下面的 `rm -rf` 只在变量仍符合 `/tmp/bash-reading.??????` 形状、目标当前仍为非符号链接目录时运行；最后同时使用 `-e` 与 `-L` 验证目录项已经消失：

```bash
case "${practice_dir:-}" in
  /tmp/bash-reading.??????)
    if test -d "$practice_dir" && ! test -L "$practice_dir"; then
      rm -rf -- "$practice_dir"
      if test ! -e "$practice_dir" && test ! -L "$practice_dir"; then
        printf '练习目录已清理：%s\n' "$practice_dir"
      else
        printf '清理后目标仍然存在，请停止并检查：%s\n' "$practice_dir" >&2
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

### 14.4 不看笔记自测

读完后不看解释，尝试回答：

1. `inspect-paths.sh` 的输入从哪里进入？
2. 内层脚本是否修改传入路径？外层练习中的哪些命令会创建文件或目录？
3. 哪个变量保存整体结果？
4. 为什么循环使用 `"$@"`？
5. 为什么 `wc -l` 的输出不包含文件名？
6. 哪种情况返回 `2`，哪种情况返回 `1`？
7. 为什么语法检查不能替代变更范围审查？
8. 清理前为什么同时检查路径模式、目录类型和符号链接？

## 15. 常见误解

### “脚本有 `.sh` 后缀，所以一定由 Bash 执行”

不成立。直接执行取决于 shebang；显式 `bash file` 或 `sh file` 又会覆盖这一选择。

### “`bash -n` 成功就可以放心运行”

它只检查语法，不判断命令来源、路径范围、权限、业务逻辑和恶意行为。

### “`[` 是括号，空格可有可无”

`[` 是命令形式，参数边界依赖空格。缺少空格会改变命令名或参数。

### “循环中的命令只影响一个目标”

循环会把同一操作应用到整个输入集合。先审查集合来源，再审查循环体。

### “trap 一定能恢复现场”

`trap` 只能执行注册的处理；进程遭遇无法捕获的 `SIGKILL`、机器掉电或持久变更已经发生时，恢复能力取决于具体设计。

### “set -e 会捕获所有错误”

`-e` 有语境规则和例外，且无法验证命令虽然返回 0、结果却不符合预期的情况。

## 完成标准

- [ ] 能根据 shebang 和调用方式判断脚本由哪个 Shell 解释。
- [ ] 能用 `bash -n` 做语法检查，并知道它不是安全审计。
- [ ] 能按输入、前置检查、变更、验证、失败和清理的顺序阅读脚本。
- [ ] 能解释 `$#`、`$1`、`"$@"` 和位置参数边界。
- [ ] 能解释 `test` 为什么通常没有输出，并区分 `-e`、`-f`、`-d`、`-L`、`-n` 与 `-s`。
- [ ] 能读懂基础 `if`、`case`、`for` 和 `while IFS= read -r`。
- [ ] 能区分 `return` 与 `exit`，知道函数通常在当前 Shell 上下文运行。
- [ ] 能说明 `trap` 和 `set -euo pipefail` 分别解决什么问题、不能保证什么。
- [ ] 执行脚本前会定位所有文件、权限、服务、网络和远程变更操作。

## 相关笔记

- [[Linux 命令行学习路线与命令地图]]
- [[Shell 命令结构、类型与帮助系统]]
- [[Shell 路径、变量、引用与展开]]
- [[Shell 标准流、管道、重定向与退出状态]]
- [[Linux 用户、用户组、sudo 与文件权限]]

## 官方参考资料

Shell 脚本通用资料于 **2026-07-19** 核对；`test`、`[`、`[[ ... ]]` 与符号链接条件于 **2026-07-25** 再次核对：

- [GNU Bash：Shell 脚本](https://www.gnu.org/software/bash/manual/html_node/Shell-Scripts.html)
- [GNU Bash：条件结构](https://www.gnu.org/software/bash/manual/html_node/Conditional-Constructs.html)
- [GNU Bash：条件表达式](https://www.gnu.org/software/bash/manual/html_node/Bash-Conditional-Expressions.html)
- [GNU Bash：循环结构](https://www.gnu.org/software/bash/manual/html_node/Looping-Constructs.html)
- [GNU Bash：Shell 函数](https://www.gnu.org/software/bash/manual/html_node/Shell-Functions.html)
- [GNU Bash：Bourne Shell 内建命令](https://www.gnu.org/software/bash/manual/html_node/Bourne-Shell-Builtins.html)
- [POSIX：test](https://pubs.opengroup.org/onlinepubs/9699919799.2016edition/utilities/test.html)
- [ShellCheck 官方仓库](https://github.com/koalaman/shellcheck)

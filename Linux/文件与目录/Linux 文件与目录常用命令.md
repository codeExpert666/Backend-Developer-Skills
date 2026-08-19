---
title: Linux 文件与目录常用命令
aliases:
  - Linux 文件操作命令
  - Linux 目录操作命令
  - Linux 路径与文件查找命令
tags:
  - Linux
  - Linux/文件与目录
  - Linux/命令行
  - Bash
  - 文件系统
created: 2026-07-19T23:47:26
updated: 2026-08-18T21:54:01
---

本文建立一套处理文件与目录的最小工作流：先定位当前上下文并只读核对目标，再执行最小范围的创建、复制、移动或删除，随后验证结果；若中途失败，则保留现场并判断恢复方式。目标不是背完所有参数，而是能够安全完成日常操作，并在忘记选项时知道怎样查证。

本篇默认使用 Ubuntu Server 中的 Bash 普通用户会话。Shell 如何展开路径、变量、引号和通配符，见 [[Shell 路径、变量、引用与展开]]；如何查看或搜索文件内容，见 [[Linux 文本查看、搜索与处理常用命令]]；权限和磁盘布局分别由对应专题负责。

## 1. 先理解路径与文件对象

Linux 把文件组织成一棵从 `/` 开始的目录树。命令操作的不是“屏幕上看到的名字”，而是某个路径经过 Shell 展开后指向的对象。

- **当前工作目录**：Shell 解释相对路径时采用的起点，可用 `pwd` 查看。
- **绝对路径**：从 `/` 开始，例如 `/etc/ssh/sshd_config`。
- **相对路径**：从当前工作目录开始，例如 `docs/readme.md`。
- `.`：当前目录。
- `..`：上一级目录。
- `~`：由 Shell 展开的当前用户主目录。

目录、普通文件和符号链接是不同类型的对象。相同命令对不同类型可能产生不同结果；例如复制目录通常需要递归选项，删除符号链接本身不会删除它指向的目标。路径文字相同也不代表结果相同：只要当前工作目录发生变化，相对路径的含义就会随之改变。

## 2. 掌握目标与命令选择

| 层级 | 应达到的程度 | 命令与能力 |
| --- | --- | --- |
| 必须熟练 | 能先核对目标，再完成基本文件操作并验证结果 | `pwd`、`cd`、`ls`、`mkdir`、`touch`、`cp`、`mv`、`rm` |
| 理解会查 | 知道它们回答什么问题，复杂参数按需查 | `rmdir`、`file`、`stat`、`realpath`、`basename`、`dirname`、`find`、`du`、`df` |
| 认识即可 | 看到时知道用途，不在本篇展开 | 符号链接、硬链接、访问控制列表（ACL）、扩展属性、`find` 的复杂表达式与动作 |

“必须熟练”不等于背下全部选项，而是能在执行前回答：源路径是什么、目标路径是什么、是否会覆盖、是否会递归、如何验证、失败后还能保留什么。

按问题选择命令，比按字母记忆更容易迁移：

| 要回答的问题 | 优先想到的命令 |
| --- | --- |
| 我在哪里，目录中有哪些对象 | `pwd`、`cd`、`ls` |
| 这个路径指向什么对象 | `file`、`stat`、`realpath` |
| 某类文件位于哪里 | `find` |
| 目录用了多少空间，所在文件系统还剩多少空间 | `du`、`df` |
| 创建、复制、移动或删除对象 | `mkdir`、`touch`、`cp`、`mv`、`rm`、`rmdir` |

忘记参数时，外部命令通常先查看 `COMMAND --help` 或 `man COMMAND`；`cd` 等 Shell 内建命令可以使用 `help cd`。命令结构和帮助系统的完整方法见 [[Shell 命令结构、类型与帮助系统]]。

## 3. 统一安全操作循环

文件操作不应从“我要输入哪条命令”开始，而应按同一循环处理：

```text
确认执行位置
    -> 核对源路径、目标路径和对象类型
    -> 判断覆盖、递归、跨文件系统与删除风险
    -> 执行最小范围的操作
    -> 检查退出状态和操作后状态
    -> 失败时保留现场并选择恢复方式
```

### 3.1 执行前核对

至少确认以下问题：

1. 当前 Shell 位于哪个目录，相对路径将从哪里解析？
2. 源对象是否存在，它是文件、目录还是符号链接？
3. 目标路径是否已经存在；若存在，它是什么类型？目标若是失效的符号链接，也不能当作名称完全空闲。
4. 命令是否会覆盖、递归、跨文件系统或删除数据？
5. 计划用什么结果证明操作符合预期？

通用的只读检查骨架是：

```text
pwd
printf '%q\n' "$PATH_TO_CHECK"
ls -ld -- "$SOURCE_PATH" "$DESTINATION_PATH"
file -- "$SOURCE_PATH"
stat -- "$SOURCE_PATH"
realpath -- "$SOURCE_PATH"
```

这里的变量名是说明性占位符，必须先赋为真实路径，不能把整段当成可直接执行的脚本。`--` 表示后面的内容按操作数解释，可避免以 `-` 开头的文件名被误当成选项。目标尚不存在时，`ls`、`file`、`stat` 或 `realpath` 可能返回非零状态；这本身就是需要纳入判断的结果，而不是应被隐藏的噪声。

### 3.2 执行后验证

退出状态为 `0` 只表示命令认为自己成功完成，不自动证明结果符合人的意图。例如，`mv source existing-directory` 可以成功，但结果可能是把 `source` 放进该目录，而不是把它重命名成预期名称。

因此，变更后还要检查后置条件：

```text
test -e "$EXPECTED_PATH"
test ! -e "$REMOVED_PATH" && test ! -L "$REMOVED_PATH"
ls -ld -- "$RESULT_PATH"
stat -- "$RESULT_PATH"
find "$CHECK_ROOT" -maxdepth "$MAX_DEPTH" -print
```

`test` 通过退出状态表达真假。`test -e` 会跟随符号链接，失效链接可能得到假值；如果要证明该路径名对应的目录项已经消失，还要同时确认 `test ! -L`。完整语法见 [[Shell 脚本阅读基础#6. 使用 test 表达条件|test 条件判断]]。标准输出、标准错误和退出状态见 [[Shell 标准流、管道、重定向与退出状态]]。

### 3.3 按风险选择操作方式

| 命令 | 默认性质 | 主要风险与处理重点 |
| --- | --- | --- |
| `pwd`、`ls`、`file`、`stat`、`realpath`、`basename`、`dirname` | 只读观察 | 输出可能暴露用户名、路径和文件名 |
| `find`、`du`、`df` | 本篇用法只读 | 扫描大目录可能较慢；`find` 另有会修改内容的动作 |
| `cd` | 改变当前 Shell 状态 | 后续相对路径的含义随之改变 |
| `mkdir`、`touch` | 创建或更新时间戳 | 可能在错误目录创建对象；`touch` 会修改已有对象的时间 |
| `cp` | 写入目标 | 可能覆盖目标、耗尽磁盘或丢失部分元数据 |
| `mv` | 改变名称或位置 | 可能覆盖目标；跨文件系统移动可能涉及复制和删除 |
| `rm`、`rmdir` | 删除 | `rm` 通常不可撤销；递归和通配符会扩大范围 |

在系统目录中加入 `sudo` 不只是“让命令成功”，而是让变更以管理员权限发生。遇到权限错误时，应先检查对象归属、路径各级权限和正确管理方式，不要为了省事给日常项目操作加 `sudo`。详见 [[Linux 用户、用户组、sudo 与文件权限]]。

## 4. 只读观察与诊断

### 4.1 定位并查看目录

先确认当前位置，再进入 `$HOME` 指向的当前用户主目录并重新核对：

```bash
pwd
ls
ls -la
cd "$HOME"
pwd
```

- `pwd` 输出当前工作目录的绝对路径。
- `cd` 改变当前 Shell 的工作目录，因此它是 Shell 内建命令；子进程不能替父 Shell 改目录。
- `ls` 列出目录项；`-l` 使用长格式，`-a` 同时显示以 `.` 开头的名称。
- `cd -` 返回前一个工作目录；单独执行 `cd` 通常回到当前用户主目录。

`ls -l` 显示的是目录项元数据摘要，不等于文件内容，也不能证明当前用户真的可以完成某项操作。查看和搜索文件内容应使用 [[Linux 文本查看、搜索与处理常用命令]]；权限问题应结合 [[Linux 用户、用户组、sudo 与文件权限]] 判断。

### 4.2 识别对象并解析路径

```bash
file /etc/os-release
stat /etc/os-release
stat -c 'type=%F owner=%U:%G mode=%a size=%s path=%n' /etc/os-release
realpath /etc/os-release
basename /var/log/syslog
dirname /var/log/syslog
```

- `file` 读取内容特征，判断文本、可执行文件、压缩包等类型；它不只依赖扩展名。
- `stat` 显示类型、大小、权限、所有者和时间戳等元数据。
- `realpath` 规范化路径，并在允许的情况下解析其中的符号链接；缺失或不可访问的路径组件可能使它失败，应同时检查退出状态。
- `basename` 只取路径最后一段。
- `dirname` 只取最后一段之前的部分。

`basename` 和 `dirname` 处理的是路径字符串，不负责验证对象是否存在；`file`、`stat` 和 `realpath` 则会访问文件系统。需要核对符号链接时，不要只看 `realpath` 的最终结果，还要用 `ls -ld` 或 `stat` 区分链接本身和目标对象。

### 4.3 按条件定位对象

下面假设 `$HOME/src` 已存在；若实际工作区不同，应先替换并核对起始路径：

```bash
find "$HOME/src" -maxdepth 2 -type d -print
find "$HOME/src" -type f -name '*.log' -print
find "$HOME/src" -type f -size +100M -print
```

`find` 的基本顺序是“起始路径 → 测试条件 → 动作”。`'*.log'` 必须引用，否则 Shell 可能在 `find` 运行前就在当前目录展开通配符。初学阶段优先使用只读的 `-print`；`-delete` 和 `-exec` 等动作要在明确执行范围后再学习。

`find` 主要根据名称、类型、大小等条件查找文件系统对象；在文本内容中找匹配行通常使用 `grep` 或其他文本搜索工具，见 [[Linux 文本查看、搜索与处理常用命令#4.3 用 grep 找到需要的行|使用 grep 搜索文本]]。

### 4.4 检查目录占用与文件系统容量

下面同样假设 `$HOME/src` 已存在：

```bash
du -sh "$HOME/src"
du -h --max-depth=1 "$HOME/src"
df -hT "$HOME/src"
```

- `du` 汇总目录树中文件占用，回答“这个目录用了多少空间”。
- `df` 查看路径所在文件系统的整体容量与剩余空间，回答“这个文件系统还有多少空间”。
- `du` 与 `df` 的统计边界不同，结果不必完全一致。已删除但仍被进程打开的文件、文件系统保留空间和稀疏文件都可能造成差异。

扫描大目录可能耗时，也可能因权限不足而得到不完整结果。磁盘、挂载和文件系统概念见 [[Linux 磁盘、分区、文件系统与 LVM 基础]]。

## 5. 创建、复制、移动与删除

下面的语法骨架使用中文占位符，因此声明为 `text`，不能原样执行。实际操作时应代入已经核对的路径，并按照“预检 → 操作 → 后置验证”执行。

### 5.1 创建文件和目录

```text
mkdir 目录路径
mkdir -p 多级目录路径
touch 文件路径
```

- `mkdir` 创建目录。
- `mkdir -p` 同时创建缺失的父目录；目标已经作为目录存在时不会因此报错，但路径中的已有对象若不是目录，命令仍会失败。
- `touch` 的本义是更新时间戳；目标不存在时通常会创建空文件。

创建前先用 `pwd` 和 `ls -ld -- 父目录路径` 核对落点；创建后使用 `test -d 目录路径`、`test -f 文件路径` 或 `stat` 检查对象类型。`touch` 不会为文件写入正文，也不是编辑器；若目标已经存在，它会改变时间戳，因此不能把它当成完全无影响的“存在性检查”。

### 5.2 复制文件或目录树

```text
cp 源文件 目标文件
cp -a 源目录 目标目录
```

- `cp` 保留源对象并创建目标副本。
- `cp -a` 适合复制目录树，并尽量保留链接、权限和时间等属性；是否能完整保留仍受目标文件系统和权限限制。

目标路径的形态会改变结果：如果目标是已有目录，源对象通常被放进该目录；如果目标不存在，命令通常以该名称创建副本。执行前必须分别核对源和目标，不能因为目标不存在使 `ls` 返回非零状态，就跳过“它预期应当不存在”的判断。

复制完成后，至少检查命令退出状态、目标是否存在及类型是否正确。迁移重要数据时，还应采用适合数据类型的内容比对或校验方式；看到一个同名目标，并不能证明内容完整。

### 5.3 移动或重命名

```text
mv 源路径 目标路径
```

`mv` 可以重命名，也可以把对象移动到另一个目录。与 `cp` 一样，已有目录会改变目标解释，已有文件还可能面临覆盖风险。

同一文件系统内的普通重命名通常只改变目录项；跨文件系统执行 `mv` 时，底层可能变成“复制后删除源对象”，不再是同一文件系统中的原子重命名。迁移重要目录时，优先采用“复制 → 验证目标 → 单独处理源目录”的可恢复流程。

操作后应同时验证两项：新路径符合预期，旧路径也处于预期状态。只检查其中一端，无法证明移动或重命名完整完成。

### 5.4 删除明确目标

```text
rm -- 待删除文件
rmdir -- 待删除空目录
rm -r -- 待递归删除目录
```

- `rm` 删除文件或符号链接；默认不能删除目录。
- `rmdir` 只删除空目录，适合用“目录必须为空”作为保护条件。
- `rm -r` 递归删除目录树，影响范围明显更大。

> [!warning] `rm` 默认不会进入回收站
> 在服务器终端中，删除通常不可直接撤销。不要对尚未展开确认的变量、通配符、`/`、`$HOME` 或陌生目录运行递归删除。先用 `printf '%q\n' "$TARGET"`、`ls -ld -- "$TARGET"` 和 `find "$TARGET" -maxdepth 2 -print` 核对目标；确认存在有效备份或目标确实可重建后，再执行最小范围删除。

`rm -f` 的主要作用是忽略不存在的文件并取消部分交互提示，不代表“更彻底”，也不会让危险操作更安全。删除后同时使用 `test ! -e 目标路径` 和 `test ! -L 目标路径` 验证目录项确实消失；如果删除失败，不要未经核对就升级为 `sudo rm -rf`，而应先检查对象类型、目录内容、权限和挂载边界。

删除不存在通用的命令级回滚。恢复通常依赖备份、版本控制、快照或应用自身的恢复机制，因此恢复能力必须在删除前建立，而不是删除后补救。

## 6. 分阶段安全练习

下面的练习只操作 `mktemp` 创建的独立临时目录，不需要 `sudo`。请在同一个 Bash 会话中按顺序执行；每一阶段开始前先预测将看到什么，不要跳过路径核对。

### 6.1 创建练习目录并完成操作

代码块中的操作使用 `&&` 串联：任一步返回非零状态，后续步骤就不会继续。失败时临时目录会保留，便于检查，而不会自动清理证据。

```bash
if LAB_DIR="$(mktemp -d /tmp/file-command-lab.XXXXXX)"; then
  printf 'lab=%s\n' "$LAB_DIR"

  if (
    cd -- "$LAB_DIR" &&
    mkdir -p project/docs &&
    touch project/docs/readme.txt &&
    printf '%s\n' 'Linux command lab' > project/docs/readme.txt &&
    pwd &&
    ls -la project/docs &&
    file project/docs/readme.txt &&
    stat -c 'type=%F size=%s path=%n' project/docs/readme.txt &&
    cp project/docs/readme.txt project/docs/readme.backup.txt &&
    test -f project/docs/readme.backup.txt &&
    mv project/docs/readme.backup.txt project/docs/readme.old.txt &&
    test ! -e project/docs/readme.backup.txt &&
    test ! -L project/docs/readme.backup.txt &&
    test -f project/docs/readme.old.txt &&
    find project -maxdepth 3 -print &&
    du -sh project &&
    df -hT "$LAB_DIR"
  ); then
    printf '%s\n' 'PASS: 操作阶段完成，练习目录已保留供复查。'
  else
    printf 'FAIL: 操作阶段中止，请检查并保留目录：%s\n' "$LAB_DIR" >&2
  fi
else
  printf '%s\n' 'FAIL: 无法创建练习目录，未执行后续文件操作。' >&2
fi
```

成功时，`find` 应显示 `project/`、`project/docs/`、`readme.txt` 和 `readme.old.txt`，但不再显示 `readme.backup.txt`。`du` 与 `df` 的具体数字取决于运行环境，不要求相同。若出现 `FAIL`，先使用输出的 `lab=` 路径检查最后成功到哪一步，不要立即运行清理阶段。

### 6.2 删除前只读预览

以下保护条件要求变量非空、目标是目录而不是符号链接，并且路径符合本练习由 `mktemp` 生成的固定格式：

```bash
if test -n "${LAB_DIR:-}" &&
   test -d "$LAB_DIR" &&
   test ! -L "$LAB_DIR" &&
   [[ "$LAB_DIR" == /tmp/file-command-lab.?????? ]]; then
  printf 'cleanup-target=%q\n' "$LAB_DIR"
  find "$LAB_DIR" -maxdepth 3 -print
else
  printf '%s\n' 'FAIL: 练习目录校验失败，拒绝预览和删除。' >&2
fi
```

只有当输出路径和对象集合都符合预期，并且不再需要保留失败现场时，才继续清理。

### 6.3 按明确顺序清理并验证

本练习故意不使用 `rm -r`。先删除两个已知文件，再用 `rmdir` 逐层删除空目录；若出现额外对象，`rmdir` 会失败并保留现场。

```bash
if test -n "${LAB_DIR:-}" &&
   test -d "$LAB_DIR" &&
   test ! -L "$LAB_DIR" &&
   [[ "$LAB_DIR" == /tmp/file-command-lab.?????? ]]; then
  if rm -- "$LAB_DIR/project/docs/readme.old.txt" &&
     rm -- "$LAB_DIR/project/docs/readme.txt" &&
     rmdir -- "$LAB_DIR/project/docs" &&
     rmdir -- "$LAB_DIR/project" &&
     rmdir -- "$LAB_DIR"; then
    printf '%s\n' 'PASS: 练习目录已按明确目标清理。'
  else
    printf 'FAIL: 清理未完成，请重新检查：%s\n' "$LAB_DIR" >&2
  fi
else
  printf '%s\n' 'FAIL: 练习目录校验失败，拒绝删除。' >&2
fi
```

最后执行：

```bash
if test -n "${LAB_DIR:-}" &&
   test ! -e "$LAB_DIR" &&
   test ! -L "$LAB_DIR"; then
  printf '%s\n' 'PASS: 练习目录已不存在。'
else
  printf '%s\n' 'FAIL: 无法证明练习目录已删除。' >&2
fi
```

## 7. 不看笔记自测

1. 当前目录为 `/home/dev/src` 时，`../logs` 指向哪里？
2. 一次文件变更的“执行前核对”和“执行后验证”分别要回答什么问题？
3. 为什么在复制或移动前，要先判断目标路径是否已经是目录？
4. `touch existing.txt` 为什么不算严格只读？
5. 什么时候应优先使用 `rmdir`，什么时候才需要 `rm -r`？
6. 为什么 `find . -name '*.log'` 中的通配符要加引号？`find` 与 `grep` 分别在找什么？
7. `basename`、`dirname` 与 `realpath` 是否都会检查对象存在？
8. `du -sh /var/log` 与 `df -hT /var/log` 分别回答什么问题？
9. 为什么命令退出状态为 `0`，仍不能完全替代对操作后状态的检查？

能用自己的话回答，并在临时目录中完成“预检 → 操作 → 验证 → 受控清理”，可以认为达到了本篇当前阶段的学习目标。记不住选项时使用帮助系统，是正常工作方式。

## 8. 常见误解

| 误解 | 正确认识 |
| --- | --- |
| `ls` 没显示就是目录为空 | 隐藏名称需要 `ls -a`；权限也可能限制读取 |
| `ls` 可以查看文件内容 | `ls` 查看目录项及其元数据摘要，文件内容应使用文本查看工具 |
| 文件扩展名决定类型 | Linux 不强制依赖扩展名，`file` 会检查内容特征 |
| `touch` 只是创建空文件 | 它也会更新已有对象的时间戳 |
| `cp -r` 与 `cp -a` 永远等价 | `-a` 还试图保留链接和多种元数据 |
| `mv` 永远只是改目录项 | 跨文件系统时可能包含复制与删除 |
| `rm -f` 更适合日常使用 | 它会减少保护性反馈，不会降低删除风险 |
| `find` 与 `grep` 都是在找文件 | `find` 主要定位文件系统对象，`grep` 主要匹配文本内容 |
| `du` 和 `df` 应给出同一个数字 | 两者观察范围和统计来源不同 |
| `test ! -e PATH` 能证明任何类型的目录项都不存在 | 它可能把失效符号链接判断为不存在；还要结合 `test ! -L PATH` |
| 退出状态为 `0` 就证明结果符合意图 | 它只证明命令认为执行成功，仍需核对目标路径和后置条件 |
| 路径以 `/` 开头只是写法不同 | 它是从根目录解析的绝对路径，与当前目录无关 |

## 相关笔记

- [[Linux 命令行学习路线与命令地图]]
- [[Shell 命令结构、类型与帮助系统]]
- [[Shell 路径、变量、引用与展开]]
- [[Shell 标准流、管道、重定向与退出状态]]
- [[Shell 脚本阅读基础]]
- [[Linux 文本查看、搜索与处理常用命令]]
- [[Linux 用户、用户组、sudo 与文件权限]]
- [[Linux 开发工作区与本地文件系统规划]]
- [[Linux 磁盘、分区、文件系统与 LVM 基础]]

## 官方参考资料

以下资料于 **2026-07-19** 核对：

- [GNU Coreutils Manual](https://www.gnu.org/software/coreutils/manual/coreutils.html)
- [GNU Findutils Manual](https://www.gnu.org/software/findutils/manual/html_mono/find.html)
- [Ubuntu Manpage：file](https://manpages.ubuntu.com/manpages/noble/en/man1/file.1.html)
- [Ubuntu Manpage：realpath](https://manpages.ubuntu.com/manpages/noble/en/man1/realpath.1.html)

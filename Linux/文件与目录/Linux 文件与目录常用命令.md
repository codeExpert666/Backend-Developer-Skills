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
updated: 2026-07-20T00:49:15
---

本文建立一套处理文件与目录的最小命令框架：先确认自己在哪里和目标是什么，再创建、复制、移动或删除，最后检查结果。目标不是记住所有参数，而是能够安全完成日常操作，并在需要时读懂帮助。

## 1. 先理解路径与文件对象

Linux 把文件组织成一棵从 `/` 开始的目录树。命令操作的不是“屏幕上看到的名字”，而是某个路径指向的对象。

- **当前工作目录**：Shell 解释相对路径时采用的起点，可用 `pwd` 查看。
- **绝对路径**：从 `/` 开始，例如 `/etc/ssh/sshd_config`。
- **相对路径**：从当前工作目录开始，例如 `docs/readme.md`。
- `.`：当前目录。
- `..`：上一级目录。
- `~`：由 Shell 展开的当前用户主目录。

目录、普通文件和符号链接是不同类型的对象。相同命令对不同类型可能有不同结果；例如复制目录通常需要递归选项，删除符号链接本身不会删除它指向的目标。

路径、通配符与引用的完整规则见 [[Shell 路径、变量、引用与展开]]。本篇示例默认使用 Bash，运行位置为 Ubuntu Server 的普通用户会话。

## 2. 掌握目标

| 层级 | 应达到的程度 | 命令与能力 |
| --- | --- | --- |
| 必须熟练 | 能先核对目标，再完成基本文件操作 | `pwd`、`cd`、`ls`、`mkdir`、`touch`、`cp`、`mv`、`rm` |
| 理解会查 | 知道它们回答什么问题，复杂参数按需查 | `rmdir`、`file`、`stat`、`realpath`、`basename`、`dirname`、`find`、`du`、`df` |
| 认识即可 | 看到时知道用途，不在本篇展开 | 符号链接、硬链接、ACL、扩展属性、`find` 的复杂表达式与动作 |

“必须熟练”不等于背下全部选项，而是能在执行前回答：源路径是什么、目标路径是什么、是否会覆盖、是否会递归、如何验证。

## 3. 命令骨架

大多数文件命令可以放进以下骨架：

```text
命令 [选项] 路径...
```

需要区分源和目标的命令通常是：

```text
cp [选项] 源路径 目标路径
mv [选项] 源路径 目标路径
```

查找与容量命令则以“从哪里开始、按什么条件观察”为核心：

```text
find 起始路径 [测试条件] [动作]
du [选项] 路径...
df [选项] [路径...]
```

示例中的“源路径”“目标路径”是说明性占位符，不能原样输入。真实路径中含空格或来自变量时，应使用双引号。

## 4. 五种高频任务模式

### 4.1 定位并查看目录

先确认当前位置，再查看目标目录内容：

```bash
pwd
ls
ls -la
cd "$HOME/src"
pwd
```

- `pwd` 输出当前工作目录的绝对路径。
- `cd` 改变当前 Shell 的工作目录，因此它是 Shell 内建命令；子进程不能替父 Shell 改目录。
- `ls` 列出目录项；`-l` 使用长格式，`-a` 同时显示以 `.` 开头的名称。
- `cd -` 返回前一个工作目录；单独执行 `cd` 通常回到当前用户主目录。

`ls -l` 显示的是目录项元数据摘要，不等于文件内容，也不能证明当前用户真的可以完成某项操作。权限问题应结合 [[Linux 用户、用户组、sudo 与文件权限]] 判断。

### 4.2 创建文件和目录

```text
mkdir reports
mkdir -p reports/2026/07
touch reports/2026/07/summary.txt
```

- `mkdir` 创建目录。
- `mkdir -p` 同时创建缺失的父目录；目标已经作为目录存在时不会因此报错，但路径中的已有对象若不是目录，命令仍会失败。
- `touch` 的本义是更新时间戳；目标不存在时通常会创建空文件。

`touch` 不会为文件写入正文，也不是编辑器。若目标已经存在，它会改变时间戳，因此不能把它当成完全无影响的“存在性检查”。只检查路径是否存在，应使用 `test -e` 或 `[ -e ... ]`，见 [[Shell 脚本阅读基础]]。

### 4.3 复制与移动

```text
cp notes.txt notes.backup.txt
cp -a project project.backup
mv draft.txt final.txt
mv final.txt archive/
```

- `cp` 保留源对象并创建目标副本。
- `cp -a` 适合复制目录树并尽量保留链接、权限和时间等属性；是否能完整保留仍受目标文件系统和权限限制。
- `mv` 可以重命名，也可以把对象移动到另一个目录。

目标路径的形态会改变结果：如果目标是已有目录，源对象通常被放进该目录；如果目标不存在，命令通常以该名称创建或重命名。执行前可先检查：

```bash
ls -ld -- source-path destination-path
```

这里的路径同样需要替换成真实值。`--` 表示后面的内容按操作数解释，可避免以 `-` 开头的文件名被误当成选项。

跨文件系统执行 `mv` 时，底层可能变成“复制后删除源对象”，不再是同一文件系统中的原子重命名。迁移重要目录时，应先复制、验证，再单独处理源目录。

### 4.4 删除明确目标

```text
rm -- 待删除文件
rmdir -- 待删除空目录
rm -r -- 待递归删除目录
```

- `rm` 删除文件或链接；默认不能删除目录。
- `rmdir` 只删除空目录，适合用“目录必须为空”作为保护条件。
- `rm -r` 递归删除目录树，影响范围明显更大。

> [!warning] `rm` 默认不会进入回收站
> 在服务器终端中，删除通常不可直接撤销。不要对尚未展开确认的变量、通配符、`/`、`$HOME` 或陌生目录运行递归删除。先用 `printf '%q\n' "$TARGET"`、`ls -ld -- "$TARGET"` 和 `find "$TARGET" -maxdepth 2 -print` 核对目标。

`rm -f` 的主要作用是忽略不存在的文件并取消部分交互提示，不代表“更彻底”，也不会让危险操作更安全。日常删除优先使用最小范围的明确路径。

### 4.5 确认类型、真实路径、位置与容量

查看对象类型和元数据：

```bash
file ./artifact
stat ./artifact
stat -c 'type=%F owner=%U:%G mode=%a size=%s path=%n' ./artifact
realpath ./artifact
basename /var/log/syslog
dirname /var/log/syslog
```

- `file` 读取内容特征，判断文本、可执行文件、压缩包等类型；它不只依赖扩展名。
- `stat` 显示类型、大小、权限、所有者和时间戳等元数据。
- `realpath` 规范化路径，并通常解析其中的符号链接；结果可能指向原路径之外。
- `basename` 只取路径最后一段。
- `dirname` 只取最后一段之前的部分。

`basename` 和 `dirname` 处理的是路径字符串，不负责验证对象是否真的存在。

按条件查找：

```bash
find "$HOME/src" -maxdepth 2 -type d -print
find "$HOME/src" -type f -name '*.log' -print
find "$HOME/src" -type f -size +100M -print
```

`find` 的基本顺序是“起始路径 → 测试条件 → 动作”。`'*.log'` 必须引用，否则 Shell 可能在 `find` 运行前就在当前目录展开通配符。初学阶段优先使用只读的 `-print`；`-delete` 和 `-exec` 等动作要在明确执行范围后再学习。

检查目录占用与文件系统容量：

```bash
du -sh "$HOME/src"
du -h --max-depth=1 "$HOME/src"
df -hT "$HOME/src"
```

- `du` 汇总目录树中文件占用，回答“这个目录用了多少空间”。
- `df` 查看路径所在文件系统的整体容量与剩余空间，回答“这个文件系统还有多少空间”。
- `du` 与 `df` 的统计边界不同，结果不必完全一致。已删除但仍被进程打开的文件、文件系统保留空间和稀疏文件都可能造成差异。

磁盘、挂载和文件系统概念见 [[Linux 磁盘、分区、文件系统与 LVM 基础]]。

## 5. 只读、变更与风险边界

| 命令 | 默认性质 | 主要风险 |
| --- | --- | --- |
| `pwd`、`ls`、`file`、`stat`、`realpath`、`basename`、`dirname` | 只读 | 输出可能暴露用户名、路径和文件名 |
| `find`、`du`、`df` | 本篇用法只读 | 扫描大目录可能慢；`find` 另有会修改内容的动作 |
| `cd` | 改变当前 Shell 状态 | 后续相对路径的含义随之改变 |
| `mkdir`、`touch` | 创建或更新时间戳 | 可能在错误目录创建对象；`touch` 会改已有文件时间 |
| `cp` | 写入目标 | 可能覆盖目标、耗尽磁盘或丢失部分元数据 |
| `mv` | 改变名称或位置 | 可能覆盖目标；跨文件系统移动涉及复制和删除 |
| `rm`、`rmdir` | 删除 | `rm` 通常不可撤销；递归和通配符会扩大范围 |

在系统目录中加入 `sudo` 不只是“让命令成功”，而是让变更以管理员权限发生。先确认文件归属与正确管理方式，不要为了省事给日常项目操作加 `sudo`。

## 6. 安全练习

下面的练习只操作 `mktemp` 创建的独立临时目录，不需要 `sudo`。每一步执行前，先预测 `pwd` 或 `find` 将输出什么。

```bash
if LAB_DIR="$(mktemp -d /tmp/file-command-lab.XXXXXX)"; then
  printf 'lab=%s\n' "$LAB_DIR"

  (
    cd -- "$LAB_DIR" || exit 1
    mkdir -p project/docs
    touch project/docs/readme.txt
    printf '%s\n' 'Linux command lab' > project/docs/readme.txt

    pwd
    ls -la project/docs
    file project/docs/readme.txt
    stat -c 'type=%F size=%s path=%n' project/docs/readme.txt

    cp project/docs/readme.txt project/docs/readme.backup.txt
    mv project/docs/readme.backup.txt project/docs/readme.old.txt
    find project -maxdepth 3 -print
    du -sh project
    df -hT "$LAB_DIR"

    rm -- project/docs/readme.old.txt
    rm -- project/docs/readme.txt
    rmdir -- project/docs
    rmdir -- project
  ) && rmdir -- "$LAB_DIR"
else
  printf '%s\n' '无法创建练习目录，已停止文件操作。' >&2
fi
```

练习结束后执行 `test -n "${LAB_DIR:-}" && test ! -e "$LAB_DIR"`。没有输出且退出状态为 `0`，表示临时目录已经删除。标准输出、重定向和退出状态见 [[Shell 标准流、管道、重定向与退出状态]]。

## 7. 不看笔记自测

1. 当前目录为 `/home/dev/src` 时，`../logs` 指向哪里？
2. 为什么在复制或移动前，要先判断目标路径是否已经是目录？
3. `touch existing.txt` 为什么不算严格只读？
4. 什么时候应优先使用 `rmdir`，什么时候才需要 `rm -r`？
5. 为什么 `find . -name '*.log'` 中的通配符要加引号？
6. `du -sh /var/log` 与 `df -h /var/log` 分别回答什么问题？
7. `basename`、`dirname` 与 `realpath` 是否都会检查目标存在？

能用自己的话回答并在临时目录中验证，才算掌握；记不住选项时使用 `COMMAND --help` 或 `man COMMAND` 是正常工作方式。

## 8. 常见误解

| 误解 | 正确认识 |
| --- | --- |
| `ls` 没显示就是目录为空 | 隐藏名称需要 `ls -a`；权限也可能限制读取 |
| 文件扩展名决定类型 | Linux 不强制依赖扩展名，`file` 会检查内容特征 |
| `touch` 只是创建空文件 | 它也会更新已有文件的时间戳 |
| `cp -r` 与 `cp -a` 永远等价 | `-a` 还试图保留链接和多种元数据 |
| `mv` 永远只是改目录项 | 跨文件系统时可能包含复制与删除 |
| `rm -f` 更适合日常使用 | 它会减少保护性反馈，不会降低删除风险 |
| `du` 和 `df` 应给出同一个数字 | 两者观察范围和统计来源不同 |
| 路径以 `/` 开头只是写法不同 | 它是从根目录解析的绝对路径，与当前目录无关 |

## 相关笔记

- [[Linux 命令行学习路线与命令地图]]
- [[Shell 命令结构、类型与帮助系统]]
- [[Shell 路径、变量、引用与展开]]
- [[Shell 标准流、管道、重定向与退出状态]]
- [[Linux 用户、用户组、sudo 与文件权限]]
- [[Linux 开发工作区与本地文件系统规划]]
- [[Linux 磁盘、分区、文件系统与 LVM 基础]]

## 官方参考资料

以下资料于 **2026-07-19** 核对：

- [GNU Coreutils Manual](https://www.gnu.org/software/coreutils/manual/coreutils.html)
- [GNU Findutils Manual](https://www.gnu.org/software/findutils/manual/html_mono/find.html)
- [Ubuntu Manpage：file](https://manpages.ubuntu.com/manpages/noble/en/man1/file.1.html)
- [Ubuntu Manpage：realpath](https://manpages.ubuntu.com/manpages/noble/en/man1/realpath.1.html)

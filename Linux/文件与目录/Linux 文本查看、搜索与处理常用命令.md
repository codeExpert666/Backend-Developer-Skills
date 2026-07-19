---
title: Linux 文本查看、搜索与处理常用命令
aliases:
  - Linux 文本处理命令
  - Linux 日志查看与文本搜索命令
  - grep 与常用文本命令
tags:
  - Linux
  - Linux/文件与目录
  - Linux/命令行
  - Bash
  - 文本处理
created: 2026-07-19T23:47:26
updated: 2026-07-20T00:49:15
---

本文建立一条处理文本的最小路径：先选择合适的查看方式，再筛选需要的行，最后按需要统计、排序或写入结果。重点是理解数据如何在命令之间流动，而不是背诵复杂的 `sed` 或 `awk` 程序。

## 1. 文本命令处理的是字节流和行

Linux 命令常把输入看成连续的数据流，并以换行符把它分成多行。很多工具既能读取文件，也能从**标准输入**接收上一条命令的输出：

```text
文件或标准输入
    -> 查看或筛选
    -> 统计或变换
    -> 终端、文件或下一条命令
```

管道 `|` 只把前一条命令的标准输出交给后一条命令，不会自动传递标准错误。`>`、`>>` 和 `tee` 会写文件，风险与纯查看命令不同。完整模型见 [[Shell 标准流、管道、重定向与退出状态]]。

本篇讨论 UTF-8 文本和常见日志。二进制文件、超大数据集、结构化 JSON/YAML 以及带表头的 CSV，通常需要对应的专用工具，不能假设所有内容都适合逐行切割。

## 2. 掌握目标

| 层级 | 应达到的程度 | 命令与能力 |
| --- | --- | --- |
| 必须熟练 | 能选择合适的查看与搜索方式 | `cat`、`less`、`head`、`tail`、`grep` |
| 理解会查 | 能组合常见的一行一记录数据 | `wc`、`sort`、`uniq`、`cut`、`tee` |
| 认识即可 | 看懂用途，用到复杂表达式时查专题或手册 | `sed`、`awk`；开发环境中的可选工具 `rg` |

初学阶段应优先掌握“原始输入是什么、命令保留了什么、输出去了哪里”。只有在简单工具组合明显变得难读时，才需要更强的文本语言。

## 3. 命令骨架

查看指定文件：

```text
cat [选项] 文件...
less [选项] 文件...
head [选项] [文件...]
tail [选项] [文件...]
```

搜索匹配行：

```text
grep [选项] 模式 [文件...]
```

从标准输入继续处理：

```text
生成文本的命令 | grep 模式 | sort | uniq -c
生成文本的命令 | tee 输出文件 | 下一条命令
```

若省略文件参数，多数工具会等待标准输入。终端看似“卡住”时，可能只是在等你输入；按 `Ctrl-D` 发送输入结束，或按 `Ctrl-C` 中断当前命令。

## 4. 五种高频任务模式

### 4.1 查看完整小文件或分页阅读

```bash
cat /etc/os-release
less /var/log/syslog
```

- `cat` 把一个或多个文件依次输出到标准输出，适合内容很短且希望直接进入管道的情况。
- `less` 提供交互式分页，适合较长内容；按 `/` 搜索，按 `n` 跳到下一个匹配，按 `q` 退出。

不要用 `cat` 把未知大小的文件全部刷到终端，也不要直接查看未知二进制文件。先检查类型和大小：

```bash
file ./candidate
stat -c 'size=%s path=%n' ./candidate
```

文件类型和路径操作见 [[Linux 文件与目录常用命令]]。

### 4.2 只看开头、结尾或持续增长的日志

```bash
head -n 20 application.log
tail -n 50 application.log
tail -f application.log
```

- `head -n 20` 输出开头 20 行。
- `tail -n 50` 输出最后 50 行。
- `tail -f` 在输出末尾后继续等待新增内容，适合观察正在写入的日志；按 `Ctrl-C` 退出。

持续跟踪按文件路径工作时，日志轮转可能替换原文件。GNU `tail -F` 会按文件名重试并重新打开，更适合长期跟踪可能被轮转的日志，但仍不能替代日志系统自身的查询能力。systemd 服务日志优先使用 [[systemd 服务与日志基础]] 中的 `journalctl`。

### 4.3 用 grep 找到需要的行

从最明确的匹配方式开始：

```bash
grep -nF 'connection refused' application.log
grep -niF 'error' application.log
grep -nE 'WARN|ERROR' application.log
grep -rF --include='*.conf' 'listen' /etc/nginx
```

常用选项按问题理解：

| 问题 | 选项 | 含义 |
| --- | --- | --- |
| 匹配发生在哪一行 | `-n` | 显示行号 |
| 是否忽略大小写 | `-i` | 忽略字母大小写差异 |
| 模式只是普通文本 | `-F` | 固定字符串，不解释正则元字符 |
| 需要 `A|B` 等扩展正则 | `-E` | 使用扩展正则表达式 |
| 是否递归搜索目录 | `-r` | 递归读取目录下文件 |
| 只关心有没有匹配 | `-q` | 不输出匹配行，以退出状态表达结果 |

固定文本默认优先 `-F`，避免把 `.`, `*`, `[ ]` 等字符误当正则。使用 `-q` 时，退出状态 `0` 表示找到，`1` 表示没有找到，大于 `1` 通常表示发生错误；不能把“没有匹配”和“命令执行失败”混为一谈。

搜索日志可能暴露令牌、请求参数、IP 地址和个人信息。分享输出前要脱敏。

### 4.4 统计、排序、去重与取字段

```bash
wc -l access.log
cut -d ':' -f 1 /etc/passwd
sort names.txt
sort names.txt | uniq
sort names.txt | uniq -c | sort -nr
```

- `wc -l` 统计换行符数量，常被理解为行数；最后一行没有换行符时要留意差异。
- `cut -d ':' -f 1` 以 `:` 为分隔符，取第一个字段。
- `sort` 对输入行排序；默认排序受区域设置影响。
- `uniq` 只合并**相邻**重复行，所以全局去重通常先 `sort`。
- `uniq -c` 在每组相邻重复行前显示数量。

例如，统计日志中每种级别的出现次数：

```bash
cut -d ' ' -f 2 application.log | sort | uniq -c | sort -nr
```

这个例子只有在“第二个以空格分隔的字段就是级别”时才成立。连续空格、字段内含空格、引号、转义和 CSV 规则都会破坏这种假设。先观察真实格式，再选择工具。

### 4.5 同时显示并保存输出

```text
grep -nF 'ERROR' application.log | tee errors.txt
grep -nF 'WARN' application.log | tee -a findings.txt
```

- `tee FILE` 把标准输入同时写到终端和文件，默认覆盖文件。
- `tee -a FILE` 追加到文件末尾。

当目标需要管理员权限时，下面两个**不可直接执行的结构骨架**具有不同的权限边界：

```text
sudo 生成内容的命令 > 受保护的目标路径
生成内容的命令 | sudo tee 受保护的目标路径
```

第一种结构中的重定向由当前 Shell 执行，`sudo` 只作用于生成内容的命令，因此当前用户通常仍无权打开受保护路径；第二种结构由以管理员权限运行的 `tee` 打开目标文件。理解这一点不代表应直接覆盖系统配置：真实操作必须先确认目标归属，保存原内容，验证新内容并准备恢复方式。

## 5. sed、awk 与 rg 的位置

### sed：按规则编辑文本流

看到下面命令时，应能识别它在替换文本，但不要求现在掌握复杂地址、分支或原地编辑：

```bash
sed 's/old/new/' input.txt
```

默认输出变换后的文本，不修改原文件。`sed -i` 会原地改文件，风险明显更高，应先在标准输出验证规则并保留可恢复副本。

### awk：按记录和字段编程

```bash
awk '{print $1}' input.txt
```

它默认逐行读取，并把空白分隔的字段放进 `$1`、`$2` 等变量。条件、聚合和多字段格式适合 `awk`，但当脚本变长时应写成可读脚本并测试，而不是堆成难以审查的一行。

### rg：开发环境中的可选搜索工具

`rg`（ripgrep）适合在源码仓库中快速递归搜索，默认遵循 `.gitignore` 并跳过部分隐藏或二进制内容：

```bash
rg -nF 'TODO' "$HOME/src/project"
```

它不是 Ubuntu 最小系统一定预装的工具，也不能替代对 `grep` 的基础理解。系统维护文档应优先给出可用性明确的命令；开发工作区中可以按项目约定使用 `rg`。

## 6. 只读、变更与风险边界

| 命令 | 默认性质 | 主要风险 |
| --- | --- | --- |
| `cat`、`less`、`head`、`tail` | 只读查看 | 大文件或二进制内容污染终端；输出可能含敏感信息 |
| `tail -f`、`tail -F` | 持续只读 | 长时间占用终端；所见内容可能跨日志轮转变化 |
| `grep`、`wc`、`sort`、`uniq`、`cut` | 本篇用法只读 | 大输入会消耗 CPU、内存或时间；错误格式假设会得到错误结论 |
| `tee` | 写入文件 | 默认覆盖目标；`-a` 会持续追加并可能占满磁盘 |
| `sed`、`awk` | 本篇示例只输出结果 | `sed -i` 或输出重定向会修改文件 |
| `rg` | 本篇用法只读 | 默认忽略规则可能让搜索范围与预期不同 |

管道中每一段都有自己的退出状态。默认情况下，Shell 通常只把最后一条命令的状态作为整个管道状态；重要脚本需要理解 `pipefail`，见 [[Shell 标准流、管道、重定向与退出状态]]。

## 7. 安全练习

下面只在临时目录中创建和处理练习文本，不需要 `sudo`：

```bash
if LAB_DIR="$(mktemp -d /tmp/text-command-lab.XXXXXX)"; then
  printf 'lab=%s\n' "$LAB_DIR"

  printf '%s\n' \
    '2026-07-19 INFO service started' \
    '2026-07-19 WARN retry scheduled' \
    '2026-07-19 ERROR connection refused' \
    '2026-07-19 WARN retry scheduled' \
    > "$LAB_DIR/application.log"

  head -n 2 "$LAB_DIR/application.log"
  tail -n 2 "$LAB_DIR/application.log"
  grep -nE 'WARN|ERROR' "$LAB_DIR/application.log"
  wc -l "$LAB_DIR/application.log"

  cut -d ' ' -f 2 "$LAB_DIR/application.log" \
    | sort \
    | uniq -c \
    | sort -nr \
    | tee "$LAB_DIR/level-counts.txt"

  cat "$LAB_DIR/level-counts.txt"
else
  printf '%s\n' '无法创建练习目录，已停止文件操作。' >&2
fi
```

尝试把其中一个 `WARN` 改为 `INFO`，在不看笔记的情况下预测统计结果，再重新运行管道。

练习目录位于 `/tmp`，并由代码块输出唯一的实际路径。保留它便于复查，由系统的 `/tmp` 清理策略后续处理即可；本篇不再提供可脱离创建步骤单独粘贴的删除块。若要练习逐项核对和删除，使用 [[Linux 文件与目录常用命令#6. 安全练习]] 中创建并清理同一临时目录的完整代码块。

## 8. 不看笔记自测

1. 读取一个 500 MB 日志时，为什么 `less` 通常比 `cat` 更合适？
2. `tail -n 50` 与 `tail -f` 的运行方式有什么不同？
3. 搜索包含点号的普通字符串时，为什么可优先使用 `grep -F`？
4. `grep -q` 没有输出时，如何区分“没找到”和“执行出错”？
5. 为什么 `uniq` 前面经常有 `sort`？
6. `cut -d ' ' -f 2` 对什么输入格式并不可靠？
7. `tee FILE` 与 `tee -a FILE` 的风险差异是什么？
8. 为什么 `sudo command > /etc/file` 不会让重定向自动获得管理员权限？

## 9. 常见误解

| 误解 | 正确认识 |
| --- | --- |
| `cat` 是所有文本查看的默认选择 | 小文件适合 `cat`，长内容更适合 `less`、`head` 或 `tail` |
| `grep` 的模式永远是普通文本 | 默认模式是基本正则；纯文本可明确使用 `-F` |
| `grep -q` 退出码非零都表示没匹配 | `1` 通常是没匹配，更大的值通常是错误 |
| `uniq` 能直接删除所有重复行 | 它只处理相邻重复行 |
| `cut` 能正确解析所有 CSV | 引号、转义和字段内分隔符需要真正的 CSV 解析器 |
| 管道会同时传递标准输出和标准错误 | `|` 默认只连接标准输出 |
| `tee` 只是“边看边存”，没有风险 | 默认会覆盖目标，追加也可能持续占用磁盘 |
| 学会 `sed`、`awk` 就不需要基础工具 | 简单任务用简单工具更容易解释、验证和维护 |
| `rg` 与 `grep -r` 的默认搜索范围完全相同 | `rg` 默认考虑忽略规则和隐藏内容，结果可能不同 |

## 相关笔记

- [[Linux 命令行学习路线与命令地图]]
- [[Shell 命令结构、类型与帮助系统]]
- [[Shell 标准流、管道、重定向与退出状态]]
- [[Shell 路径、变量、引用与展开]]
- [[Linux 文件与目录常用命令]]
- [[systemd 服务与日志基础]]

## 官方参考资料

以下资料于 **2026-07-19** 核对：

- [GNU Coreutils Manual](https://www.gnu.org/software/coreutils/manual/coreutils.html)
- [GNU grep Manual](https://www.gnu.org/software/grep/manual/)
- [GNU sed Manual](https://www.gnu.org/software/sed/manual/sed.html)
- [GNU awk User's Guide](https://www.gnu.org/software/gawk/manual/gawk.html)
- [ripgrep User Guide](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md)

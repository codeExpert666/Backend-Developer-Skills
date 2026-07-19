---
title: Linux 进程与系统资源常用命令
aliases:
  - Linux 进程查看与终止命令
  - Linux CPU 内存与负载命令
  - ps top kill free 常用命令
tags:
  - Linux
  - Linux/系统管理
  - Linux/命令行
  - 进程
  - 系统资源
created: 2026-07-19T23:47:26
updated: 2026-07-20T00:49:15
---

本文提供一套观察进程和系统资源的最小方法：先确认对象身份，再观察状态和资源，最后才决定是否发送信号。目标是能回答“哪个进程在做什么、系统是否真的有压力、应由谁管理它”，而不是看到占用高就立即强制结束。

## 1. 程序、进程、PID 与服务不是同一概念

- **程序**：磁盘上的可执行文件或脚本，是静态内容。
- **进程**：程序一次运行形成的执行实例，拥有地址空间、打开文件和运行状态。
- **PID**：内核为当前进程实例分配的数字标识；进程退出后，数字可以被后续进程复用。
- **父进程 PID（PPID）**：创建当前进程的进程标识，可帮助理解启动关系。
- **服务**：由 systemd 等服务管理器描述和管理的长期能力；一个服务可能包含一个或多个进程，也可能在失败后重新拉起进程。

因此，“程序已安装”不能证明它正在运行，“知道 PID”也不能证明它属于某个服务。对 systemd 服务，应通过 unit 检查和改变生命周期，见 [[systemd 服务与日志基础]]；直接结束其中某个进程可能触发自动重启或留下不完整状态。

## 2. 掌握目标

| 层级 | 应达到的程度 | 命令与能力 |
| --- | --- | --- |
| 必须熟练 | 能找到并核对进程，先发可处理的终止信号 | `ps`、`pgrep`、`kill` 的最小骨架，理解 PID、PPID、`SIGTERM` |
| 理解会查 | 能交互观察进程，并读懂 CPU、内存、负载和采样结果 | `top`、`free`、`uptime`、`nproc`、`lscpu`、`vmstat`，理解常见信号 |
| 认识即可 | 知道存在，复杂排障时按需学习 | 线程、调度优先级、cgroup、`/proc`、`strace`、性能剖析工具 |

初学阶段不要追求记住所有 `ps` 列或 `top` 快捷键。先建立固定顺序：身份 → 状态 → 资源 → 管理边界 → 最小操作 → 再验证。

## 3. 命令骨架

按 PID 或条件定位进程：

```text
ps [选择条件] [输出格式]
pgrep [选项] 匹配条件
top [选项]
```

向已经核对过的进程发送信号：

```text
kill [-信号] PID...
```

观察系统资源：

```text
free [选项]
uptime
nproc
lscpu
vmstat [采样间隔秒数 [采样次数]]
```

命令输出是一个时间点或一段时间的采样，不是永久结论。进程可能在你查看后立即退出，PID 也可能被复用；执行变更前需要再次核对。

## 4. 五种高频任务模式

### 4.1 按名称找到候选进程，再按 PID 核对

```bash
pgrep -a -x sshd
ps -p 1 -o pid=,ppid=,user=,stat=,etime=,comm=,args=
```

- `pgrep` 按进程属性查找 PID。
- `-x` 要求进程名称完整匹配，减少模糊命中的候选。
- `-a` 同时显示命令行，便于初步核对。
- `ps -p PID` 只查看指定 PID；`-o` 明确选择输出列。

命令行参数可能包含路径、用户名甚至误传入的密码或令牌。复制 `ps`、`pgrep -a` 输出前要脱敏。

进程名称可能被截断或与命令行不同。不要仅凭一次模糊的 `pgrep keyword` 就把输出 PID 交给 `kill`。对重要目标还应核对父进程、运行用户、已运行时间和服务归属。

### 4.2 查看进程快照或交互式排序

查看常用列，并按 CPU 使用率排序：

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,etime,comm --sort=-%cpu
```

交互观察：

```bash
top
```

在常见 `top` 界面中：

- 按 `P` 以 CPU 使用率排序。
- 按 `M` 以内存使用率排序。
- 按 `q` 退出。

`ps` 是执行瞬间的快照；`top` 按间隔刷新，更适合观察趋势。单次 CPU 尖峰不等于持续故障，排序第一也不等于该进程有问题。先结合业务行为、持续时间和日志判断。

常见状态列 `STAT` 中，`R` 表示正在运行或可运行，`S` 表示可中断睡眠，`D` 表示不可中断睡眠，`T` 表示停止，`Z` 表示僵尸。状态还可能附带额外字符；需要精确解释时查 `man ps`，不要只根据一个字母采取操作。

### 4.3 优雅终止自己确认过的进程

```bash
kill -TERM PID
ps -p PID -o pid=,stat=,etime=,comm=,args=
```

将 `PID` 替换为核对后的真实数字。`kill PID` 默认通常就是发送 `SIGTERM`，它请求进程完成清理后退出，并不保证立即结束。

查看本机支持的信号名称：

```bash
kill -l
```

初学阶段先理解四个常见信号：

| 信号 | 常见含义 | 注意事项 |
| --- | --- | --- |
| `SIGINT` | 交互式中断，终端中常由 `Ctrl-C` 触发 | 进程可以捕获并清理 |
| `SIGTERM` | 请求终止 | 默认首选，进程可以捕获、忽略或延迟处理 |
| `SIGHUP` | 终端断开；部分守护进程约定用来重载 | 具体行为必须查目标程序文档 |
| `SIGKILL` | 内核立即结束进程 | 不能被捕获或清理，只作为最后手段 |

> [!warning] 不要把 `kill -KILL` 当作第一步
> 强制结束会跳过应用清理，可能中断写入、遗留临时状态或触发服务管理器反复重启。先确认是否应使用 `systemctl stop SERVICE`，再发 `SIGTERM`，等待并检查日志；只有进程无法正常退出且影响可控时才考虑 `SIGKILL`。

PID 是暂时身份。若在“查看”和“发送信号”之间等待很久，原进程可能已经退出，数字也可能复用。操作前应再次执行 `ps -p PID ...`。

### 4.4 判断 CPU、内存与负载是否有压力

```bash
free -h
uptime
nproc
lscpu
```

`free -h` 的重点不是追求 `free` 列很大：Linux 会把空闲内存用于缓存。对“还能否启动新工作”的粗略判断，`available` 通常比完全空闲的 `free` 更有意义；仍需结合 swap、应用工作集和持续趋势。

`uptime` 最后的三个数是最近 1、5、15 分钟的系统负载平均值。它反映可运行任务和不可中断睡眠任务的数量，不是 CPU 百分比。判断是否偏高需要结合可用 CPU 数、I/O 等待和持续时间。

- `nproc` 报告当前进程可用的处理单元数量，可能受 CPU 亲和性或容器限制影响。
- `lscpu` 展示系统 CPU 架构与拓扑信息。

因此，不能机械地认为 `nproc` 与 `lscpu` 中所有 CPU 数字必须相同。

### 4.5 用 vmstat 观察短时间趋势

```bash
vmstat 1 5
```

这里表示每 1 秒采样一次，共输出 5 次。常见关注点：

| 区域 | 列 | 初步含义 |
| --- | --- | --- |
| `procs` | `r` | 等待运行的任务数量 |
| `procs` | `b` | 处于不可中断睡眠的任务数量 |
| `memory` | `si`、`so` | 每秒换入、换出量 |
| `io` | `bi`、`bo` | 块设备输入、输出量 |
| `cpu` | `us`、`sy`、`id`、`wa` | 用户、内核、空闲和 I/O 等待占比 |

没有采样间隔时，`vmstat` 只给出自启动以来的汇总。带间隔时，第一行数据也通常代表自启动以来的平均值，后续行才反映相邻采样区间。不要根据第一行或单个非零值直接下结论。

资源排查应把现象关联起来：负载高但 CPU 空闲可能与 I/O 等待有关；内存可用量低但没有持续换页不一定是故障；容器中的视图还可能受 cgroup 限制。

## 5. 服务进程的固定处理边界

如果进程由 systemd 管理，优先执行只读检查：

```bash
systemctl status ssh.service --no-pager
systemctl show ssh.service -p MainPID -p ExecStart -p Restart
sudo journalctl -u ssh.service -b -n 80 --no-pager
```

三者分别回答：服务当前状态是什么、systemd 记录的主进程与启动/重启策略是什么、当前启动以来最近发生了什么。

需要停止服务时，应在确认依赖和恢复通道后使用：

```bash
sudo systemctl stop SERVICE_NAME.service
```

这里的 `SERVICE_NAME` 是占位符。对 SSH、网络、防火墙、数据库等远程或有状态服务，不能在唯一访问通道中直接试验停止操作。

## 6. 只读、变更与风险边界

| 命令 | 默认性质 | 主要风险 |
| --- | --- | --- |
| `ps`、`pgrep` | 只读 | 命令行可能泄露敏感参数；模糊匹配容易误认对象 |
| `top` | 默认用于观察 | 交互界面也可能提供发信号等操作；只使用已理解的按键 |
| `free`、`uptime`、`nproc`、`lscpu`、`vmstat` | 只读 | 单次采样容易被误读；频繁采样本身也有少量开销 |
| `kill` | 改变进程状态 | 错误 PID 或信号会中断错误对象、造成数据或连接损失 |
| `systemctl stop/restart` | 改变服务状态 | 影响全部服务进程和依赖方，远程服务可能导致失联 |

普通用户通常只能向自己有权限控制的进程发送信号。不要因为收到 `Operation not permitted` 就立即加 `sudo`；先确认目标身份以及你是否真的负责它。

## 7. 安全练习

下面只启动并终止当前 Shell 创建的 `sleep` 练习进程，不需要 `sudo`：

```bash
(
LAB_PID=''

cleanup_lab_process() {
  if test -n "$LAB_PID" && kill -0 "$LAB_PID" 2>/dev/null; then
    kill -TERM "$LAB_PID" 2>/dev/null || true
    wait "$LAB_PID" 2>/dev/null || true
  fi
}
trap cleanup_lab_process EXIT

sleep 300 &
LAB_PID=$!
printf 'lab_pid=%s\n' "$LAB_PID"

ps -p "$LAB_PID" -o pid=,ppid=,user=,stat=,etime=,comm=,args=
top -b -n 1 -p "$LAB_PID"

if kill -TERM "$LAB_PID"; then
  wait "$LAB_PID" 2>/dev/null
  wait_status=$?
  printf 'wait_status=%s；练习子进程已经被当前 Shell 回收。\n' "$wait_status"
  LAB_PID=''
else
  printf '%s\n' 'SIGTERM 发送失败；退出清理仍会再次核对该子进程。' >&2
fi
)
```

练习位于子 Shell 中，并为 `EXIT` 注册清理；即使在中途按 `Ctrl-C` 或某条观察命令失败，也会尝试终止这个代码块自己创建的 `sleep`，不会把 `trap` 留在登录 Shell。进程因 `SIGTERM` 结束时，`wait_status` 常见为 `143`（`128 + 15`）；这里的非零值表示子进程被信号终止，不代表清理未完成。

随后执行以下只读观察，并尝试用自己的话解释每组输出：

```bash
free -h
uptime
nproc
lscpu
vmstat 1 3
```

不要在练习中选择陌生系统进程，也不要使用 `pkill`、`killall` 或 `SIGKILL` 扩大操作范围。

## 8. 不看笔记自测

1. 程序、进程和 systemd 服务分别是什么？
2. 为什么看到一个 PID 后，发送信号前还要再次核对？
3. `ps` 与 `top` 的观察方式有什么不同？
4. 为什么 `kill PID` 不等于“内核立即强制杀死进程”？
5. `SIGTERM` 与 `SIGKILL` 的关键差异是什么？
6. 为什么系统内存的 `free` 很小不一定代表内存不足？
7. 负载平均值是不是 CPU 使用百分比？判断它时还要看什么？
8. `vmstat 1 5` 的第一行为什么不能直接代表刚刚这一秒？
9. 服务内的进程异常时，为什么通常先用 `systemctl` 和 `journalctl`？

## 9. 常见误解

| 误解 | 正确认识 |
| --- | --- |
| 可执行文件存在就代表程序正在运行 | 文件是程序内容，进程才是运行实例 |
| 一个服务永远只有一个 PID | 服务可以派生多个进程，主 PID 也会在重启后变化 |
| `pgrep` 返回的都可以直接结束 | 匹配只产生候选，必须继续核对身份和管理边界 |
| `kill` 默认就是强制杀死 | 默认一般发送 `SIGTERM`，请求进程正常终止 |
| `SIGKILL` 更快，所以应该优先 | 它不给进程清理机会，只应作为最后手段 |
| CPU 排名第一就是故障进程 | 高占用可能是正常工作，要看持续时间、业务和日志 |
| `free` 列越大越健康 | Linux 会利用内存做缓存，`available` 往往更有参考意义 |
| 负载为 4 就一定很高 | 需要结合可用 CPU 数、I/O 和持续趋势 |
| `vmstat` 输出一个非零值就能定位根因 | 它是系统级线索，需要多次采样并与进程和日志关联 |
| systemd 服务异常就直接杀主进程 | 服务管理器可能重启它，应通过 unit 状态与日志处理 |

## 相关笔记

- [[Linux 命令行学习路线与命令地图]]
- [[Shell 命令结构、类型与帮助系统]]
- [[Shell 标准流、管道、重定向与退出状态]]
- [[systemd 服务与日志基础]]
- [[Ubuntu Server 初始化与基础安全]]
- [[Linux 文本查看、搜索与处理常用命令]]

## 官方参考资料

以下资料于 **2026-07-19** 核对：

- [procps-ng：命令与手册源文件](https://gitlab.com/procps-ng/procps)
- [Ubuntu Manpage：pgrep(1)](https://manpages.ubuntu.com/manpages/noble/man1/pgrep.1.html)
- [Ubuntu Manpage：free(1)](https://manpages.ubuntu.com/manpages/noble/man1/free.1.html)
- [Ubuntu Manpage：vmstat(8)](https://manpages.ubuntu.com/manpages/noble/man8/vmstat.8.html)
- [Linux man-pages：signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html)
- [Linux Kernel Documentation：The /proc Filesystem](https://docs.kernel.org/filesystems/proc.html)
- [GNU Coreutils Manual：nproc](https://www.gnu.org/software/coreutils/manual/html_node/nproc-invocation.html)
- [util-linux：lscpu 手册源文件](https://github.com/util-linux/util-linux/blob/master/sys-utils/lscpu.1.adoc)
- [systemd：systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)

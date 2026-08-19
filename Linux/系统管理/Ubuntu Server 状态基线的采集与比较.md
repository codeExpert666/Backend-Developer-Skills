---
title: Ubuntu Server 状态基线的采集与比较
aliases:
  - Ubuntu Server 系统状态基线
  - Ubuntu 主机状态记录与比较
tags:
  - Linux
  - Linux/系统管理
  - Ubuntu
created: 2026-08-02T16:10:31
updated: 2026-08-18T23:22:57
---

本文说明如何把 Ubuntu Server 在某个时点的关键状态采集到受保护的本地文本文件，并在后续变更前后进行比较。重点是理解“为什么采集、采集什么、怎样确认完整、如何解释差异”，而不是保存一份看不懂的命令输出。

> [!info] 核对日期与适用范围
> 本文于 **2026-08-02** 核对 Bash、GNU Coreutils 和 systemd 官方资料。完整示例面向使用 systemd、OpenSSH 和 UFW 的 Ubuntu Server，并假定已经完成 [[Ubuntu Server 初始化与基础安全]] 第 1～10 节。不同发行版、Ubuntu 版本或安全策略应先调整采集项，再在目标主机验证。

## 本篇掌握目标

- **必须熟练**：说明状态基线与备份的区别；创建权限位仅授予所有者的基线文件；确认采集是否完整，并识别不应公开的主机信息。
- **理解会查**：理解子 Shell、`set -e`、`pipefail`、`umask`、`mktemp`、命令组和 `tee` 在完整示例中的职责；使用 `diff` 比较两个时点。
- **认识即可**：为其他发行版、生产合规基线或自动化资产系统设计更严格的机器可读格式和字段规范。

## 1. 状态基线是什么

状态基线（state baseline）是在一个明确时点，对一组经过选择的系统事实所做的记录。它主要回答两个问题：

1. 变更前，这台主机是什么状态？
2. 变更后，哪些字段发生了变化，变化是否符合预期？

它与其他保护或观察手段的职责不同。

| 手段 | 主要回答的问题 | 能否直接恢复系统 |
| --- | --- | --- |
| 状态基线 | 当时的版本、配置摘要和运行状态是什么 | 不能 |
| 文件或系统备份 | 文件丢失或系统损坏后如何恢复数据 | 可以，取决于备份范围和恢复验证 |
| VM 快照或完整副本 | 如何把虚拟机回退或恢复到某个磁盘状态 | 可以，取决于快照或副本是否可用 |
| 监控 | 状态是否持续越过阈值，是否需要告警 | 不能 |
| journal 日志 | 某段时间内发生了哪些事件 | 不能 |

基线也不是安全合规证明。文件末尾出现 `status=complete`，只表示本文的采集流程运行到了完成标记；它不表示所有服务健康、所有配置安全或所有输出都符合预期。

```mermaid
flowchart LR
    A["明确采集目的"] --> B["选择必要字段"]
    B --> C["写入受保护文件"]
    C --> D["检查完整性与权限"]
    D --> E["变更后重新采集"]
    E --> F["比较并解释差异"]
```

## 2. 什么时候采集

适合采集状态基线的时点包括：

- Ubuntu Server 首次初始化完成后。
- 操作系统、内核、OpenSSH、UFW、Docker 或其他关键组件变更前后。
- 创建 VM 快照或完整副本前。
- 问题排除后，需要保留新的已知状态时。

一次采集必须写明用途，例如：

```text
post-initialization
before-os-upgrade
after-os-upgrade
before-vm-backup
```

不要把同一个旧文件不断覆盖成“当前状态”。每次采集都应创建新文件，保留时间和用途，才能形成可比较的历史。

## 3. 采集内容与隐私边界

本文的初始化后基线包含以下摘要。

| 类别 | 采集内容 | 主要用途 |
| --- | --- | --- |
| 系统 | Ubuntu 版本、内核、机器与软件包架构 | 判断运行环境是否变化 |
| 身份 | 当前用户、组和静态主机名 | 确认执行身份与主机对象 |
| 时间与资源 | 时区、同步状态、根文件系统和内存摘要 | 发现时间或资源条件变化 |
| 网络 | 接口地址、路由和 DNS 状态 | 比较连接路径与名称解析条件 |
| systemd | failed unit 和 enabled unit 文件 | 发现失败状态与启用关系变化 |
| 管理入口 | OpenSSH unit 摘要和 UFW 状态 | 比较远程入口与主机防火墙状态 |

本文不主动采集：

- 密码、令牌、私钥、Cookie、恢复码或 `/etc/shadow`。
- 完整环境变量、Shell 命令历史或应用配置全文。
- 完整 journal、请求内容、数据库数据或业务日志。
- `hostnamectl` 的完整输出，因为这里只需要静态主机名，不需要机器 ID、启动 ID 等额外标识。

> [!warning] “不主动采集凭据”不等于“可以公开”
> 基线仍会包含用户名、主机名、地址、路由、DNS 和防火墙规则等主机信息。目录和文件的普通权限位不得向同组或其他用户授予访问权；这不会限制 root 或其他特权机制。未经逐项审查和脱敏，不要提交到 Git、放入公开笔记、上传到公开网盘或直接发送给他人。

第 10 节检查的完整 warning 日志不进入本基线。日志可能包含路径、地址、请求参数或凭据片段，应只在排障现场按需读取，不能因为它“来自系统”就默认安全。

## 4. 采集一份完整的初始化后基线

### 4.1 先确认执行环境和目标目录

完整示例使用 Bash 的 `pipefail`，因此必须在 Bash 中执行。先做只读检查：

**执行位置：Ubuntu Server（当前普通用户会话，任意目录；只读检查）**

```bash
ps -p "$$" -o comm=
printf 'HOME=%s\n' "$HOME"

if test -e "$HOME/system-baselines" || test -L "$HOME/system-baselines"; then
  ls -ld -- "$HOME/system-baselines"
else
  printf '%s\n' '目标目录尚不存在，将由完整示例创建。'
fi
```

第一条命令预期显示 `bash`。如果当前不是 Bash，先运行 `bash` 进入一个 Bash 子 Shell；完成本文操作后可以运行 `exit` 返回原 Shell。

如果 `$HOME/system-baselines` 已存在，应确认它是自己创建的真实目录，不是符号链接或普通文件。完整示例遇到符号链接或同名非目录对象会停止，不会覆盖它。

### 4.2 执行完整示例

下面代码会刷新当前用户的 sudo 认证、创建或收紧 `$HOME/system-baselines` 目录权限，并创建一个新基线文件。它不会修改 systemd、OpenSSH、UFW 或网络配置；除目录和文件创建外，其余采集命令都是只读查询。

**执行位置：Ubuntu Server（当前普通用户的 Bash 会话，任意目录）**

```bash
(
set -e
set -o pipefail
umask 077

# 在创建文件前验证当前用户能读取需要 sudo 的 UFW 状态。
sudo -v

printf '请输入本次基线用途，例如 post-initialization 或 before-vm-backup：'
IFS= read -r baseline_purpose

case "$baseline_purpose" in
  ''|*[!a-z0-9-]*|-*|*-)
    printf '%s\n' '停止：用途只能包含小写字母、数字和连字符，且不能以连字符开头或结尾。' >&2
    exit 1
    ;;
esac

captured_at="$(date --iso-8601=seconds)"
filename_time="$(date +%Y%m%dT%H%M%S)"
baseline_dir="$HOME/system-baselines"
readonly baseline_purpose captured_at filename_time baseline_dir

if test -L "$baseline_dir"; then
  printf '停止：目标路径是符号链接：%s\n' "$baseline_dir" >&2
  exit 1
fi

if test -e "$baseline_dir" && ! test -d "$baseline_dir"; then
  printf '停止：目标路径已存在但不是目录：%s\n' "$baseline_dir" >&2
  exit 1
fi

mkdir -p -- "$baseline_dir"
chmod 0700 -- "$baseline_dir"

if ! baseline_file="$(
  mktemp "$baseline_dir/${baseline_purpose}-${filename_time}-XXXXXX.txt"
)"; then
  printf '%s\n' '停止：无法创建唯一的基线文件。' >&2
  exit 1
fi
readonly baseline_file
printf 'baseline_file=%s\n' "$baseline_file"

{
  printf '[capture]\n'
  printf 'captured_at=%s\n' "$captured_at"
  printf 'purpose=%s\n' "$baseline_purpose"

  printf '\n[os]\n'
  sed -n -e '/^PRETTY_NAME=/p' -e '/^VERSION_ID=/p' /etc/os-release
  uname -r
  uname -m
  dpkg --print-architecture

  printf '\n[identity]\n'
  id

  printf '\n[hostname]\n'
  hostnamectl --static

  printf '\n[time]\n'
  timedatectl status

  printf '\n[storage]\n'
  df -hT /

  printf '\n[memory]\n'
  free -h

  printf '\n[network]\n'
  ip -brief address
  ip route
  resolvectl status

  printf '\n[systemd-failed]\n'
  systemctl --failed --no-pager

  printf '\n[systemd-enabled-unit-files]\n'
  systemctl list-unit-files --state=enabled --no-pager

  printf '\n[openssh-units]\n'
  systemctl show ssh.socket ssh.service \
    --property=Id --property=LoadState \
    --property=UnitFileState --property=ActiveState \
    --no-pager

  printf '\n[ufw]\n'
  sudo ufw status verbose
} | tee "$baseline_file"

chmod 0600 -- "$baseline_file"

printf '\n[capture-result]\nstatus=complete\n' | tee -a "$baseline_file"
stat -c 'mode=%A owner=%U group=%G path=%n' "$baseline_file"
)
```

输入 `post-initialization` 表示首次初始化后的基线；在 VM 备份前执行时可以输入 `before-vm-backup`。成功时，终端会打印 `baseline_file=...`、文件内容、`status=complete` 和最终权限。记录输出的文件路径，后续检查和比较会使用它。

## 5. 按执行顺序理解完整示例

### 5.1 先隔离 Shell 选项和权限掩码

最外层圆括号创建子 Shell。代码块中的 `set` 和 `umask` 只影响这个子 Shell，不会永久改变当前登录 Shell。

- `set -e`：在这段顺序执行的代码中，未被条件结构处理的命令返回非零时停止子 Shell。它存在例外，不是“自动保证脚本安全”的开关，详见 [[Shell 脚本阅读基础#12. set -euo pipefail 不是自动安全开关]]。
- `set -o pipefail`：让后面的采集命令组或 `tee` 任一方失败时，整条管道返回非零。默认情况下，管道状态通常只来自最后一个命令，详见 [[Shell 标准流、管道、重定向与退出状态#7. 管道同时具有数据流和状态流]]。
- `umask 077`：限制随后新建文件和目录的默认权限，避免同组用户或其他用户获得访问权。

`sudo -v` 放在创建文件之前。认证失败时不会留下一个看似可用的基线文件；认证成功只刷新 sudo 凭据缓存，不代表后续 UFW 查询一定成功。

### 5.2 用用途、时间和随机后缀创建新文件

`baseline_purpose` 进入文件名和文件正文，因此先限制为小写字母、数字和连字符。`captured_at` 保存带时区的实际采集时间，`filename_time` 提供便于排序的文件名时间。

目标目录固定为 `$HOME/system-baselines`：

- 遇到符号链接或同名普通文件时停止，避免把输出写到未核对的位置。
- `mkdir -p` 创建目录；`chmod 0700` 不向同组或其他普通用户授予目录访问权。
- `mktemp` 根据模板安全创建一个真实且名称唯一的文件，而不是先猜一个可能冲突的文件名。
- 随机后缀避免同一秒内重复采集时发生覆盖；本文不会改写旧基线。

### 5.3 用命令组汇总输出，再交给 tee

花括号把多条查询命令的标准输出汇总成一条数据流。每个 `[section]` 标题标记后续输出属于哪个观察维度。管道右侧的 `tee` 同时把内容显示在终端并写入刚创建的文件。

这段采集刻意使用摘要而不是复制整个配置：

- `hostnamectl --static` 只读取静态主机名，不采集完整 `hostnamectl` 中的额外标识。
- `systemctl show` 只选择 OpenSSH unit 的名称、加载、启用和当前激活状态。
- `sudo ufw status verbose` 读取 UFW 当前运行状态、默认策略和规则摘要，不修改防火墙。
- 完整 journal 不进入文件；需要解释 failed unit 或 warning 时回到 [[systemd 服务与日志基础#9. 综合检查：失败状态、启用关系与本次启动日志]] 在现场读取。

`tee` 成功只表示数据写入完成，不表示每条状态都正常。仍要阅读 failed unit、OpenSSH、UFW 和网络等部分。

### 5.4 用完成标记区分完整文件与部分文件

只有整个采集管道成功并且权限调整成功后，代码才会追加：

```text
[capture-result]
status=complete
```

如果某条查询或 `tee` 失败，子 Shell 会在追加完成标记前停止。此前已写入的部分文件可能保留在已打印的路径中；它可以帮助定位失败，但不能改名后冒充完整基线。

`chmod 0600` 明确把文件限制为仅所有者可读写，`stat` 则显示最终模式、所有者、组和路径。这是文件访问边界的证据，不是内容已经脱敏的证据。

## 6. 检查完整性、权限和内容

把完整示例打印的路径原样输入下面代码。它只接受 `$HOME/system-baselines` 直属目录下的普通 `.txt` 文件，不接受子目录、符号链接或其他位置。

**执行位置：Ubuntu Server（当前普通用户会话，任意目录；只读检查）**

```bash
(
printf '请输入刚才输出的 baseline_file 路径：'
IFS= read -r baseline_file

case "$baseline_file" in
  "$HOME"/system-baselines/*/*|"$HOME"/system-baselines/)
    printf '%s\n' '停止：只接受基线目录直属的文件。' >&2
    exit 1
    ;;
  "$HOME"/system-baselines/*.txt)
    ;;
  *)
    printf '%s\n' '停止：路径不属于本文的基线目录或文件类型。' >&2
    exit 1
    ;;
esac

if test -L "$baseline_file" || ! test -f "$baseline_file"; then
  printf '%s\n' '停止：目标不是可接受的普通文件。' >&2
  exit 1
fi

stat -c 'mode=%A owner=%U group=%G path=%n' \
  "$HOME/system-baselines" "$baseline_file"

if ! tail -n 2 -- "$baseline_file" | grep -qx 'status=complete'; then
  printf '%s\n' '停止：没有找到完成标记，应按部分文件处理。' >&2
  exit 1
fi

complete_count="$(grep -xc 'status=complete' "$baseline_file")"
if test "$complete_count" -ne 1; then
  printf '停止：完成标记数量异常：%s\n' "$complete_count" >&2
  exit 1
fi

sed -n '1,160p' -- "$baseline_file"
)
```

检查结果应满足：

- `mode` 为 `-rw-------`，目录模式为 `drwx------`。
- 文件末尾存在唯一的 `status=complete`。
- 预期章节均有内容，没有命令错误混入正常数据。
- failed unit、OpenSSH 与 UFW 内容和第 10 节已经解释的状态一致。
- 人工检查后确认没有意外加入凭据、完整日志或业务数据。

若文件超过 160 行，`sed` 只显示前 160 行；还应按章节继续检查剩余内容，不能把“前 160 行正常”当作整份文件已审查。

## 7. 在变更前后比较两份基线

先分别完成并检查两份基线，例如 `before-os-upgrade` 和 `after-os-upgrade`。下面代码只读取两个文件并生成 unified diff（统一差异格式）：

**执行位置：Ubuntu Server（当前普通用户会话，任意目录；只读比较）**

```bash
(
printf '请输入变更前基线的完整路径：'
IFS= read -r baseline_before
printf '请输入变更后基线的完整路径：'
IFS= read -r baseline_after

if ! test -f "$baseline_before" || test -L "$baseline_before" ||
   ! test -f "$baseline_after" || test -L "$baseline_after"; then
  printf '%s\n' '停止：两个输入都必须是普通文件且不能是符号链接。' >&2
  exit 1
fi

diff -u -- "$baseline_before" "$baseline_after"
diff_status=$?

case "$diff_status" in
  0) printf '%s\n' '两份文件内容相同。' ;;
  1) printf '%s\n' '已显示差异；请逐项解释预期变化与异常变化。' ;;
  *) printf 'diff 执行失败，退出状态：%s\n' "$diff_status" >&2; exit "$diff_status" ;;
esac
)
```

`diff` 返回 `1` 表示找到了差异，不是执行错误；大于 `1` 才表示比较本身失败。常见差异需要分类判断。

| 差异 | 通常如何判断 |
| --- | --- |
| `captured_at`、`purpose` | 每次采集都应变化 |
| 内存、磁盘使用量 | 通常会变化，关注幅度和是否接近容量边界 |
| DHCP 地址、默认路由或 DNS | 可能因网络环境变化；必须确认仍符合预期路径 |
| 内核、Ubuntu 版本 | 只有执行相应升级后才应变化 |
| failed unit | 新增项必须定位；消失项也要确认是修复而非清除状态 |
| enabled unit 文件 | 安装、卸载或 enable/disable 后可能变化，其他变化需要解释 |
| OpenSSH 或 UFW | 必须与已批准的远程入口和防火墙变更一致 |

基线用于提出问题，不替代实际连接测试。例如 UFW 规则看起来相同，也不能替代从真实客户端建立一条新 SSH 会话。

## 8. 失败与恢复

### 文件存在但没有完成标记

这通常表示某条采集命令、权限调整或 `tee` 失败。保留终端错误和文件路径，找到最后一个完整章节，再检查对应命令。修正原因后重新采集新文件；不要手工补写 `status=complete`。

### 提示命令或 unit 不存在

先用 `/etc/os-release` 和 `command -V` 确认目标系统与命令，再检查 OpenSSH、UFW 和 systemd 前置步骤。若目标主机本来不使用这些组件，应设计匹配该主机的采集清单，而不是用 `|| true` 隐藏缺失项。

### UFW 查询失败

不要为了生成基线而跳过防火墙状态。返回 [[Linux 主机防火墙与 UFW 基础]] 核对安装、权限和运行状态；只有当前主机明确不使用 UFW 时，才在新的适用范围说明中移除该采集项。

### 发现不应保存或分享的内容

先保持目录和文件权限位仅授予所有者，停止同步、提交或发送。若需要对外提供证据，应从原文件中选择必要字段生成单独的脱敏副本并再次人工审查，不要把原基线直接变成公开材料。

### 需要恢复系统

状态基线只能说明先前状态，不能恢复软件包、配置、源码或数据。虚拟机回退和完整副本见 [[UTM 虚拟机快照、备份与恢复]]；源码与工作区恢复见 [[Git 仓库跨机器迁移与工作区保留]]。数据库和应用数据还需要各自可验证的备份方案。

## 完成标准

- [ ] 能说明状态基线、备份、快照、监控和日志的职责差异。
- [ ] 能为一次采集写出明确用途，并保留独立文件而不是覆盖旧记录。
- [ ] 能确认目录为 `0700`、文件为 `0600`，并识别 `status=complete` 的有限含义。
- [ ] 能说明采集了什么、没有采集什么，以及为什么基线仍不能公开。
- [ ] 能正确解释 `diff` 的 `0`、`1` 和大于 `1` 三类退出状态。
- [ ] 能把差异分成预期变化、需要解释的变化和比较失败。

## 相关笔记

- [[Ubuntu Server 初始化与基础安全]]
- [[systemd 服务与日志基础]]
- [[Shell 路径、变量、引用与展开]]
- [[Shell 标准流、管道、重定向与退出状态]]
- [[Shell 脚本阅读基础]]
- [[Linux 网络接口、IP 地址、路由与 DNS 基础]]
- [[OpenSSH 密钥登录、服务端配置与排查]]
- [[Linux 主机防火墙与 UFW 基础]]
- [[UTM 虚拟机快照、备份与恢复]]

## 官方参考资料

以下资料于 **2026-08-02** 核对：

- [GNU Bash：The Set Builtin](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html)
- [GNU Coreutils：mktemp](https://www.gnu.org/software/coreutils/manual/html_node/mktemp-invocation.html)
- [GNU Coreutils：tee](https://www.gnu.org/software/coreutils/manual/html_node/tee-invocation.html)
- [GNU Coreutils：chmod](https://www.gnu.org/software/coreutils/manual/html_node/chmod-invocation.html)
- [GNU Coreutils：stat](https://www.gnu.org/software/coreutils/manual/html_node/stat-invocation.html)
- [GNU Diffutils：Invoking diff](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html)
- [systemd：systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [systemd：journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)

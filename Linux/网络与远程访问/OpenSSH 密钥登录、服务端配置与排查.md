---
title: OpenSSH 密钥登录、服务端配置与排查
aliases:
  - OpenSSH 连接、密钥与主机指纹
  - SSH 连接、密钥交换与身份验证原理
  - SSH 连接与身份验证原理
  - SSH 密钥交换与认证流程
  - SSH 主机身份与用户身份
  - 从 macOS 使用 SSH 连接 Linux 虚拟机
  - macOS SSH 登录 Ubuntu 虚拟机
  - OpenSSH 客户端与 sshd 实操
  - Linux 虚拟机密钥登录
tags:
  - Linux
  - Linux/网络
  - Linux/远程访问
  - SSH
  - OpenSSH
created: 2026-07-16T00:28:30
updated: 2026-07-25T14:29:00
---

SSH 是一套加密远程访问协议，OpenSSH 是其常用实现，通常由客户端 `ssh` 与服务端 `sshd` 配合工作。本篇以传统公钥登录为主线，完成服务端准备、网络入口核对、主机指纹验证、用户密钥登记、客户端配置、认证收紧和故障排查。

本篇只解释会直接影响操作与排障的原理，不要求先掌握密码学或 SSH 协议消息。客户端 `ssh` 的通用命令骨架见 [[SSH 客户端命令基础]]；软件包、服务、网络、防火墙和权限分别由 [[APT 软件包管理基础]]、[[systemd 服务与日志基础]]、[[Linux 网络接口、IP 地址、路由与 DNS 基础]]、[[Linux 主机防火墙与 UFW 基础]] 与 [[Linux 用户、用户组、sudo 与文件权限]] 负责。

> [!info] 核对日期
> 本文于 **2026-07-19** 核对 Ubuntu Server 与 OpenBSD OpenSSH 手册，于 **2026-07-20** 核对 iproute2 的 `ss(8)`、OpenBSD 与 Ubuntu 的 `nc(1)` 手册，并于 **2026-07-21** 补充核对 OpenBSD 的 `ssh-keyscan(1)` 手册。修改认证策略前，应以目标主机和客户端上的 `man sshd_config`、`man ss`、`man nc` 及由 systemd 管理的实际服务状态为最终依据。

## 本篇掌握目标

- **必须熟练**：能区分主机身份与用户登录权限、`known_hosts` 与 `authorized_keys`；能安全完成主机指纹核对、用户公钥登记和独立新会话验证。
- **理解会查**：能读取 `sshd -T`、`systemctl`、`ss`、`nc`、`ssh -G` 与 `ssh -vvv` 的关键信息，并按网络、主机身份、用户认证与远程会话的层次定位问题。
- **认识即可**：知道 SSH 会自动协商算法并为当前连接建立加密通道；知道 `ssh-keyscan` 只负责收集主机公钥；能看出认证收紧脚本的备份、校验、重载与失败回滚顺序。

## 1. 先建立够用的 SSH 连接模型

日常使用不需要推导密钥交换算法，但必须知道每个阶段在验证什么：

```text
客户端能够连接服务端的 TCP 端口
  → SSH 在建立加密通道的过程中，由客户端核对服务端的主机身份
  → 服务端验证客户端是否有权登录目标 Linux 用户
  → 服务端创建受该 Linux 用户权限约束的远程会话
```

第一步只说明网络路径可达，不能证明对端是预期主机，也不能证明当前用户可以登录。后面三步分别对应主机身份、用户登录权限和 Linux 会话权限，排障时不要把它们混在一起。

SSH 使用两套用途不同的长期密钥：

| 验证对象 | 回答的问题 | 私钥保存位置 | 验证方的公钥或信任记录 |
| --- | --- | --- | --- |
| 主机身份 | 我连接到的是不是预期主机 | 服务端 `/etc/ssh/ssh_host_*_key` | 客户端 `~/.ssh/known_hosts` |
| 用户登录权限 | 当前客户端是否有权登录目标 Linux 用户 | 客户端的用户私钥 | 服务端目标用户的 `~/.ssh/authorized_keys` |

密钥对包含私钥和公钥。公钥可以交给验证方，私钥始终留在持有者一侧；SSH 通过签名证明对方持有对应私钥，不会把主机私钥或用户私钥发送给对方。`known_hosts` 记录客户端信任的主机公钥；`authorized_keys` 记录允许登录某个 Linux 用户的用户公钥。

主机指纹是主机公钥的短标识，便于人工比较，并不是另一把密钥。SSH 还会在每次连接时自动协商算法并生成临时的会话密钥，用它保护后续认证、命令和终端数据。它与使用 `ssh-keygen` 创建的用户身份文件不是同一类密钥，日常使用时无需追踪具体协议消息。

> [!danger] 两类长期私钥都不能外泄
> 主机私钥只由服务端使用，用户私钥只由客户端、SSH Agent 或硬件认证器使用。需要分发或登记时使用对应公钥；不要把任何私钥放进 Git、笔记、聊天或共享目录，也不要使用 `sudo ssh-keygen` 为普通客户端用户创建密钥。

实践主线按以下顺序推进：

```text
准备并核对 sshd
  → 取得服务端地址并验证 TCP 端口
  → 通过独立可信入口核对主机指纹
  → 创建用户密钥并登记公钥
  → 从新终端验证独立登录
  → 配置客户端别名
  → 在保留恢复入口的前提下收紧认证
```

> [!danger] 不要把“当前会话仍可用”当成配置成功
> 修改 `sshd` 配置时保留控制台和旧会话，每次都从新终端验证能重新连接。只有独立新会话成功，才说明新的登录路径可用。

## 2. 在服务端准备 sshd

Ubuntu 22.10 及更新版本默认使用 systemd socket activation。`ssh.socket` 先监听 SSH 端口，收到连接时再激活 `ssh.service`。因此，在尚未收到连接时看到 `ssh.socket` 为 `active`、`ssh.service` 为 `inactive` 可能是正常状态，不应为了让服务常驻而直接改变软件包的默认激活方式。

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
(
if ! sudo apt update || ! sudo apt install openssh-server; then
  printf '%s\n' '停止：软件包索引刷新或 openssh-server 安装失败。' >&2
  exit 1
fi
# 服务已运行时可直接检查当前配置；否则由 systemd 先准备运行时目录，再执行启动前配置检查。
if systemctl is-active --quiet ssh.service; then
  if ! sudo sshd -t; then
    printf '%s\n' '停止：SSH 配置检查失败。' >&2
    exit 1
  fi
else
  if ! sudo systemctl start ssh.service; then
    printf '%s\n' '停止：ssh.service 启动失败。' >&2
    exit 1
  fi
fi
if ! systemctl status ssh.socket ssh.service --no-pager; then
  printf '%s\n' '停止：SSH socket 或 service 状态不符合预期。' >&2
  exit 1
fi
if ! effective_config="$(sudo sshd -T)"; then
  printf '%s\n' '停止：无法读取 SSH 有效配置。' >&2
  exit 1
fi
if ! grep '^port ' <<<"$effective_config"; then
  printf '%s\n' '停止：有效配置中没有找到监听端口。' >&2
  exit 1
fi
sudo ss -lntp
)
```

本文核对的 Ubuntu 24.04 `ssh.service` 会通过 `RuntimeDirectory=sshd` 让 systemd 在启动时创建 `/run/sshd`，再由 `ExecStartPre=/usr/sbin/sshd -t` 检查配置。服务未运行时，`systemctl start` 成功就表明启动前配置检查已经通过；服务已运行时，它的运行时目录已经存在，可直接使用 `sshd -t` 检查当前配置。`systemctl start` 只改变当前运行状态，不会将 `ssh.service` 设为开机启用；下次启动仍可由已启用的 `ssh.socket` 按需激活。

预期 `ssh.socket` 处于 `active (listening)`，执行上述检查后，`ssh.service` 处于 `active (running)`。先使用 `sshd -T` 确认实际生效的端口，再在 `ss` 输出中查找对应的监听套接字。新安装的默认端口通常是 22，但不要将它视为所有主机的固定值。上述检查在 Ubuntu Server 控制台执行，此时尚未建立 SSH 会话，也不需要停止服务。后续若通过 SSH 远程维护，应先确认控制台或其他独立管理入口可用；如果当前 SSH 会话是唯一可用入口，不要停止 `ssh.service`，以免会话中断后无法重新连接。

服务端查看当前有效配置：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
sudo sshd -T | grep -E '^(port|listenaddress|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

`sshd -T` 比只阅读某一个配置文件可靠，因为 Ubuntu 可能通过 `Include` 加载 `/etc/ssh/sshd_config.d/*.conf`。

监听状态与主机防火墙是不同层次：`sshd` 或 `ssh.socket` 正常监听，不代表 UFW 已允许外部连接；UFW 允许端口，也不代表 SSH 主机指纹或用户认证正确。启用防火墙前应同时比较有效配置、`ss` 实际监听和 UFW 规则，详见 [[Linux 主机防火墙与 UFW 基础#6. 先比较 SSH 配置、监听状态与 UFW profile]]。

### 2.1 怎样阅读 `sudo ss -lntp`

`ss` 读取 Linux 内核当前的套接字状态。这里的组合短选项可以拆成 `-l`（只看监听套接字）、`-n`（保留数字地址和端口）、`-t`（只看 TCP）与 `-p`（显示持有套接字的进程）。`sudo` 用于尽量完整地读取其他用户的进程信息；这条命令本身只读取状态，不会启动服务或修改网络。

阅读输出时先看三处：

1. `State` 是否为 `LISTEN`。
2. `Local Address:Port` 是否包含 `sshd -T` 显示的有效端口，以及它绑定的是回环地址、某个具体地址还是所有本地地址。
3. `Process` 由谁持有。Ubuntu 使用 `ssh.socket` 时可能显示 `systemd`，因此不能只搜索 `sshd` 进程名。

`0.0.0.0:22` 表示示例端口绑定所有本地 IPv4 地址，`127.0.0.1:22` 只允许从本机 IPv4 回环入口访问；`[::]:22` 是 IPv6 通配绑定，不能脱离系统的 IPv4/IPv6 套接字设置推断它是否也接收 IPv4。端口以当前 `sshd -T` 输出为准。端口、监听地址、输出字段和其他 `ss` 骨架详见 [[Linux 端口、监听套接字与 ss 命令基础]]。

## 3. 取得地址并验证端口

在服务端读取当前地址，不把一次 DHCP 地址当作永久事实。网络接口、地址前缀和默认路由的读取方法见 [[Linux 网络接口、IP 地址、路由与 DNS 基础]]。

**执行位置：Linux 服务端（控制台，任意目录）**

```bash
hostnamectl --static
ip -brief address
ip route
```

从客户端输入目标地址并测试：

**执行位置：SSH 客户端（任意目录）**

```bash
(
printf '请输入 Linux 主机当前可达的地址或名称：'
IFS= read -r SSH_HOST
case "$SSH_HOST" in
  ''|-*)
    printf '%s\n' '停止：主机地址或名称为空，或以连字符开头。' >&2
    exit 1
    ;;
esac
nc -vz -w 5 "$SSH_HOST" 22
)
```

### 3.1 怎样阅读 `nc -vz -w 5 "$SSH_HOST" 22`

`nc` 又称 Netcat，这里从实际 SSH 客户端尝试连接目标 TCP 端口。`-v` 输出详细结果，`-z` 只做连接探测而不发送 SSH 应用数据，`-w 5` 将本文核对实现的等待限制为 5 秒；`"$SSH_HOST"` 是前面已经校验的目标名称或地址，`22` 是本次测试的 TCP 目标端口。这条命令不需要 `sudo`，也不会修改服务端配置或防火墙，但会实际产生一次网络连接尝试。

连接成功只表示从当前客户端到本次解析得到的目标地址和 TCP 端口完成了连接，不能证明对端一定是 OpenSSH、主机指纹可信或用户密钥有效。明确拒绝常见于没有监听入口、绑定地址不匹配或某层主动拒绝；超时可能涉及地址、路由、丢弃型防火墙、上游策略或目标离线。客户端现象只能提供排查方向，不能单独锁定根因，详见 [[TCP 端口连通性测试与 nc 命令基础]]。

如果服务端不是 22 端口，应从可信配置中取得实际端口，并在 `nc` 与 `ssh -p` 中显式指定。不同 Netcat 实现的低频选项可能不同，应在实际客户端使用 `command -V nc`、`nc -h` 和 `man nc` 核对。

## 4. 首次连接前独立核对主机指纹

客户端首次连接某个目标时，`known_hosts` 中还没有可信主机公钥，因此不能只根据当前网络连接中显示的指纹判断对端身份。应通过服务端控制台、云平台控制台或管理员提供的可信记录取得参考指纹，再与客户端提示比较。若两份指纹都来自同一条可能被攻击的网络路径，它们不能彼此证明可信。

本节从服务端控制台读取主机公钥指纹，再与客户端提示比较。以下以 Ed25519 主机密钥为例：`/etc/ssh/ssh_host_ed25519_key` 是主机私钥，对应的 `.pub` 文件是主机公钥；下面只读取公钥指纹，不读取或复制私钥。

从服务端控制台读取 Ed25519 主机公钥指纹：

**执行位置：Linux 服务端（控制台，任意目录）**

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

`ssh-keygen` 不仅能生成密钥，也能查看已有密钥的信息。这里的 `-l` 表示显示指纹，`-f` 指定要读取的文件，两者合写为 `-lf`。这条命令只读取公钥，不会生成或修改密钥；`sudo` 只用于以系统权限读取 `/etc/ssh` 中的文件，与使用 `sudo ssh-keygen` 为普通用户创建密钥不是一回事。典型输出会包含密钥位数、`SHA256:...` 指纹、注释和末尾的 `(ED25519)` 类型，人工比较时重点核对指纹和密钥类型。

再从客户端发起首次连接：

**执行位置：SSH 客户端（任意目录）**

```bash
(
printf '请输入 Linux 登录用户名：'
IFS= read -r SSH_USER
printf '请输入 Linux 主机当前可达的地址或名称：'
IFS= read -r SSH_HOST
case "$SSH_USER" in
  ''|-*|*[!A-Za-z0-9_-]*)
    printf '%s\n' '停止：Linux 用户名格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac
case "$SSH_HOST" in
  ''|-*)
    printf '%s\n' '停止：主机地址或名称为空，或以连字符开头。' >&2
    exit 1
    ;;
esac
ssh "$SSH_USER@$SSH_HOST"
)
```

客户端首次连接提示会同时显示主机密钥类型和指纹。只有提示类型为 `ED25519` 时，才能将它的 `SHA256:...` 指纹与上述结果比较；如果提示的是 RSA 或 ECDSA，应改为读取对应的 `/etc/ssh/ssh_host_rsa_key.pub` 或 `/etc/ssh/ssh_host_ecdsa_key.pub`，不能跨类型比较。类型与指纹都一致后才接受，接受的主机公钥会写入客户端 `~/.ssh/known_hosts`。

### 4.1 怎样阅读 `ssh "$SSH_USER@$SSH_HOST"`

`ssh` 是运行在当前客户端上的 OpenSSH 客户端命令，常用骨架是 `ssh [选项] [远程用户@]目标主机 [远程命令]`。这里本地 Shell 会把双引号中的变量展开成一个类似 `linux-user@server.example.com` 的目标参数；`@` 前是远程 Linux 用户名，后面是主机名或地址。由于没有提供远程命令，认证成功后会进入交互式远程 Shell。

这条命令不需要 `sudo`，它会读取当前客户端用户的 `~/.ssh/config`、`~/.ssh/known_hosts`、身份文件和 SSH Agent。公钥认证只在客户端使用私钥完成签名，不会把私钥发送到服务端；使用 `exit` 或 `Ctrl-D` 可以结束远程 Shell 并返回本地终端。交互式登录、远程命令、标准输入和常用选项详见 [[SSH 客户端命令基础]]。

### 4.2 `ssh-keyscan`：只收集主机公钥，不验证主机身份（认识即可）

`ssh-keyscan` 是 OpenSSH 提供的主机公钥收集工具，常用于为多台已知主机收集可供 `known_hosts` 使用的记录。它不是本文首次连接主线的必需命令；在批量准备或核对主机公钥时，可能会看到如下命令骨架：

```text
ssh-keyscan [-T 超时秒数] [-p SSH端口] [-t 密钥类型] 目标主机
```

`-T` 指定连接或读取等待超时，`-p` 指定目标 SSH 端口，`-t` 限定要收集的主机密钥类型，例如 `ed25519`。默认输出包含主机名、密钥类型和公钥，可作为 `known_hosts` 记录的格式来源，但不代表这些公钥已经通过可信通道验证。

`ssh-keyscan` 只能收集当前网络响应者提供的公钥，不能独立证明响应者就是预期主机。如果攻击者能够拦截网络流量，就可能用自己的主机公钥替换真实结果。因此，未经控制台或其他可信通道核对的输出，不应直接作为可信记录写入 `~/.ssh/known_hosts`。

## 5. 创建独立的用户密钥

这里创建的是客户端用户密钥，不是服务端主机密钥。登录 Linux 主机与访问 GitHub、GitLab 等代码平台属于不同授权场景，建议按用途使用独立密钥；Git SSH 详见 [[Git 凭据、SSH 与常见问题排查]]。

先选择用途明确的路径并避免覆盖。下面的 `test -e ... || test -L ...` 会把悬空符号链接也视为路径已占用，避免 `ssh-keygen` 写入任何已有目录项；条件检查与符号链接边界见 [[Shell 脚本阅读基础#6. 使用 test 表达条件|test 条件判断]]。

**执行位置：SSH 客户端（任意目录）**

```bash
(
KEY_PATH="$HOME/.ssh/id_ed25519_linux_host"
if ! mkdir -p "$HOME/.ssh" || ! chmod 700 "$HOME/.ssh"; then
  printf '%s\n' '停止：无法准备客户端 .ssh 目录。' >&2
  exit 1
fi

if test -e "$KEY_PATH" || test -L "$KEY_PATH" ||
   test -e "$KEY_PATH.pub" || test -L "$KEY_PATH.pub"; then
  printf '停止：密钥路径已存在，请先确认用途并选择新文件名：%s\n' "$KEY_PATH" >&2
  exit 1
fi

if ssh-keygen -t ed25519 -a 64 -f "$KEY_PATH" -C 'linux-host-access' &&
   chmod 600 "$KEY_PATH" &&
   chmod 644 "$KEY_PATH.pub"; then
  ssh-keygen -lf "$KEY_PATH.pub"
else
  printf '%s\n' '密钥创建或权限设置失败，请检查上述两个目标路径后再重试。' >&2
  exit 1
fi
)
```

最外层圆括号让路径冲突时的 `exit` 只停止这个代码块，不会结束当前客户端登录 Shell。

建议为私钥设置口令。`-a 64` 增加私钥口令派生轮次。生成与保存过程不会把私钥发给服务端；后续登记时传输公钥，认证时传输公钥和由私钥生成的签名。

## 6. 将公钥加入 authorized_keys

下面只通过标准输入发送公钥，并让远端以严格权限创建目录：

**执行位置：SSH 客户端（任意目录）**

```bash
(
KEY_PATH="$HOME/.ssh/id_ed25519_linux_host"
printf '请输入 Linux 登录用户名：'
IFS= read -r SSH_USER
printf '请输入 Linux 主机当前可达的地址或名称：'
IFS= read -r SSH_HOST

if ! test -f "$KEY_PATH.pub" || test -L "$KEY_PATH.pub"; then
  printf '停止：公钥不是预期的普通文件：%s\n' "$KEY_PATH.pub" >&2
  exit 1
fi
case "$SSH_USER" in
  ''|-*|*[!A-Za-z0-9_-]*)
    printf '%s\n' '停止：Linux 用户名格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac
case "$SSH_HOST" in
  ''|-*)
    printf '%s\n' '停止：主机地址或名称为空，或以连字符开头。' >&2
    exit 1
    ;;
esac

ssh "$SSH_USER@$SSH_HOST" \
  'umask 077; mkdir -p "$HOME/.ssh" && cat >> "$HOME/.ssh/authorized_keys"' \
  < "$KEY_PATH.pub"
)
```

目标参数后的单引号字符串是远程命令：单引号阻止本地 Shell 展开其中的 `$HOME`，远端 Shell 收到后再按远端用户家目录解释。`< "$KEY_PATH.pub"` 由本地 Shell 处理，只把本地公钥内容送入 `ssh` 的标准输入，再由远端 `cat` 追加到 `authorized_keys`；对应私钥没有被传输。这一步需要密码或其他已经可用的认证方式，命令边界详见 [[SSH 客户端命令基础#4. 交互式登录与单次远程命令]]。

随后在服务端核对：

**执行位置：Linux 服务端（当前登录用户家目录）**

```bash
chmod 700 "$HOME/.ssh"
chmod 600 "$HOME/.ssh/authorized_keys"
stat -c 'mode=%A owner=%U group=%G path=%n' \
  "$HOME/.ssh" "$HOME/.ssh/authorized_keys"
```

如果误追加重复公钥，先备份 `authorized_keys`，再只删除能确认重复的完整行。不要删除用途未知、可能仍被其他客户端使用的公钥。

## 7. 用新会话验证密钥

保留当前可用会话，另开客户端终端：

**执行位置：SSH 客户端（新终端，任意目录）**

```bash
(
KEY_PATH="$HOME/.ssh/id_ed25519_linux_host"
printf '请输入 Linux 登录用户名：'
IFS= read -r SSH_USER
printf '请输入 Linux 主机当前可达的地址或名称：'
IFS= read -r SSH_HOST

if ! test -f "$KEY_PATH" || test -L "$KEY_PATH"; then
  printf '停止：私钥不是预期的普通文件：%s\n' "$KEY_PATH" >&2
  exit 1
fi
case "$SSH_USER" in
  ''|-*|*[!A-Za-z0-9_-]*)
    printf '%s\n' '停止：Linux 用户名格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac
case "$SSH_HOST" in
  ''|-*)
    printf '%s\n' '停止：主机地址或名称为空，或以连字符开头。' >&2
    exit 1
    ;;
esac

ssh -o IdentitiesOnly=yes -i "$KEY_PATH" "$SSH_USER@$SSH_HOST"
)
```

`-i "$KEY_PATH"` 指定本次公钥认证使用的身份文件；`-o` 临时提供一个客户端配置项，`IdentitiesOnly=yes` 限制候选身份，避免 SSH Agent 中其他密钥干扰。这些选项只控制客户端如何选择身份，不会把私钥传给服务端，详见 [[SSH 客户端命令基础#6. 高频选项按问题记忆]]。

登录后验证身份和连接：

**执行位置：Linux 服务端（新 SSH 会话）**

```bash
whoami
id
hostnamectl --static
printf 'SSH_CONNECTION=%s\n' "$SSH_CONNECTION"
pwd
```

只有新的独立会话成功，才能认为密钥登录可用。

## 8. 使用客户端 ~/.ssh/config

客户端配置把动态地址、用户名和密钥路径集中在一个别名下。通用骨架与配置读取边界见 [[SSH 客户端命令基础#7. 使用 ~/.ssh/config 别名]]；以下是本次登录流程使用的结构示例，`HostName` 和 `User` 必须替换为实际值：

```sshconfig
Host linux-host
    HostName linux-host.example.internal
    User linux-user
    IdentityFile ~/.ssh/id_ed25519_linux_host
    IdentitiesOnly yes
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

保存到客户端 `~/.ssh/config` 后：

**执行位置：SSH 客户端（任意目录）**

```bash
chmod 700 "$HOME/.ssh"
chmod 600 "$HOME/.ssh/config"
ssh -G linux-host | grep -E '^(hostname|user|identityfile|identitiesonly) '
ssh linux-host
```

`ssh -G` 显示合并后的客户端配置，适合排查多个 `Host` 块、通配符和 `Include` 的优先级。别名稳定而地址可变时，只需更新 `HostName`。

## 9. Fail-closed 收紧服务端认证

本节使用的 `if`、函数、`trap`、重定向和退出状态见 [[Shell 脚本阅读基础]] 与 [[Shell 标准流、管道、重定向与退出状态]]。这里应把整段当作一个有前置条件和回滚的完整操作，不要抽出中间命令单独执行。

只有满足以下条件后，才考虑关闭密码登录：

- 控制台仍可用。
- 至少两个独立的新密钥会话已经成功。
- 已确认正确 Linux 用户拥有可用公钥。
- 当前旧会话保持打开。

以下脚本使用独立配置片段，先备份，再校验语法与有效值；任一步失败都会尝试恢复原状态。整段由最外层圆括号放进一个子 Shell，所以其中的 `trap` 和 `exit` 不会残留或结束当前登录 Shell：

此时至少一个 SSH 会话已经通过验证，`ssh.service` 正在运行且它的运行时目录已经存在，因此脚本可以直接使用 `sshd -t` 和 `sshd -T`。

**执行位置：Linux 服务端（已验证的 SSH 会话）**

```bash
(
  config_file=/etc/ssh/sshd_config.d/00-local-hardening.conf
  backup_file="${config_file}.before-hardening"
  absence_marker="${config_file}.absent-before-hardening"
  had_original=no

  if ! candidate_file="$(mktemp /tmp/ssh-hardening.XXXXXX)"; then
    printf '%s\n' '停止：无法创建候选配置文件。' >&2
    exit 1
  fi
  readonly candidate_file

  cleanup() {
    rm -f -- "$candidate_file"
  }
  trap cleanup EXIT

  if ! sudo -v; then
    printf '%s\n' '停止：无法取得 sudo 授权。' >&2
    exit 1
  fi

  if sudo test -e "$backup_file" || sudo test -e "$absence_marker"; then
    printf '%s\n' '停止：变更前状态记录已存在，请先核对后再操作。' >&2
    printf '备份路径：%s\n缺失标记：%s\n' \
      "$backup_file" "$absence_marker" >&2
    exit 1
  elif ! sudo test ! -e "$backup_file" ||
       ! sudo test ! -e "$absence_marker"; then
    printf '%s\n' '停止：无法确认变更前状态记录。' >&2
    exit 1
  fi

  if sudo test -e "$config_file"; then
    if ! sudo cp -a -- "$config_file" "$backup_file"; then
      printf '%s\n' '停止：无法备份原配置。' >&2
      exit 1
    fi
    had_original=yes
  elif ! sudo test ! -e "$config_file"; then
    printf '停止：无法确认配置路径状态：%s\n' "$config_file" >&2
    exit 1
  elif ! sudo install -o root -g root -m 0600 /dev/null "$absence_marker"; then
    printf '%s\n' '停止：无法记录原配置不存在。' >&2
    exit 1
  fi

  if ! sudo sshd -t; then
    printf '%s\n' '停止：变更前的 SSH 配置检查未通过。' >&2
    exit 1
  fi

  if ! cat >"$candidate_file" <<'EOF'
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
  then
    printf '%s\n' '停止：无法写入候选配置。' >&2
    exit 1
  fi

  restore_original() {
    if test "$had_original" = yes; then
      sudo cp -a -- "$backup_file" "$config_file"
    else
      sudo rm -f -- "$config_file"
    fi
  }

  effective_config=''
  if sudo install -o root -g root -m 0644 "$candidate_file" "$config_file" &&
     sudo sshd -t &&
     effective_config="$(sudo sshd -T)" &&
     grep -qxF 'pubkeyauthentication yes' <<<"$effective_config" &&
     grep -qxF 'passwordauthentication no' <<<"$effective_config" &&
     grep -qxF 'kbdinteractiveauthentication no' <<<"$effective_config" &&
     grep -qxF 'permitrootlogin no' <<<"$effective_config" &&
     sudo systemctl reload ssh.service; then
    printf '%s\n' '配置已应用；请保持旧会话并从新终端验证登录。'
  else
    printf '%s\n' '变更失败，正在恢复原配置。' >&2
    if ! restore_original; then
      printf '%s\n' '紧急：自动恢复写入失败，请保持当前会话并使用控制台处理。' >&2
      exit 1
    fi
    if ! sudo sshd -t; then
      printf '%s\n' '紧急：恢复后的配置检查失败，请保持当前会话并使用控制台处理。' >&2
      exit 1
    fi
    if ! sudo systemctl reload ssh.service; then
      printf '%s\n' '紧急：原配置已写回，但 reload 失败，请保持当前会话并使用控制台处理。' >&2
      exit 1
    fi
    printf '%s\n' '原配置已恢复，收紧操作未生效。' >&2
    exit 1
  fi
)
```

原片段存在时，脚本把它复制到固定备份；原片段不存在时，则创建一个独立的“原本不存在”标记。后续再次收紧或回滚时，只要这两种状态记录缺失、同时存在或彼此矛盾，脚本就会停止，不会猜测原状态。

脚本成功只说明配置与 reload 通过，不能证明客户端仍能登录。保持旧会话，从新终端用别名再次连接。

需要回滚时，通过控制台或仍可用的旧会话执行：

**执行位置：Linux 服务端（控制台或仍可用会话）**

```bash
(
  config_file=/etc/ssh/sshd_config.d/00-local-hardening.conf
  backup_file="${config_file}.before-hardening"
  absence_marker="${config_file}.absent-before-hardening"
  rollback_copy="${config_file}.rollback-$(date +%Y%m%d%H%M%S)-$$"

  if ! sudo -v; then
    printf '%s\n' '停止：无法取得 sudo 授权。' >&2
    exit 1
  fi
  if ! sudo test -e "$config_file"; then
    printf '停止：当前配置不存在：%s\n' "$config_file" >&2
    exit 1
  fi

  if sudo test -e "$backup_file" && sudo test ! -e "$absence_marker"; then
    rollback_mode=restore
  elif sudo test ! -e "$backup_file" && sudo test -e "$absence_marker"; then
    rollback_mode=remove
  else
    printf '%s\n' '停止：备份与缺失标记不符合预期，无法判断原状态。' >&2
    exit 1
  fi

  if sudo test -e "$rollback_copy" ||
     ! sudo cp -a -- "$config_file" "$rollback_copy"; then
    printf '%s\n' '停止：无法创建回滚前副本。' >&2
    exit 1
  fi

  restore_pre_rollback() {
    sudo cp -a -- "$rollback_copy" "$config_file" &&
      sudo sshd -t &&
      sudo systemctl reload ssh.service
  }

  apply_ok=no
  case "$rollback_mode" in
    restore)
      if sudo cp -a -- "$backup_file" "$config_file"; then
        apply_ok=yes
      fi
      ;;
    remove)
      if sudo rm -- "$config_file"; then
        apply_ok=yes
      fi
      ;;
  esac

  if test "$apply_ok" = yes &&
     sudo sshd -t &&
     sudo systemctl reload ssh.service; then
    printf '回滚完成；回滚前副本保留在：%s\n' "$rollback_copy"
  else
    printf '%s\n' '回滚失败，正在恢复回滚前配置。' >&2
    if ! restore_pre_rollback; then
      printf '紧急：自动恢复失败，请保持当前会话并通过控制台处理；副本：%s\n' \
        "$rollback_copy" >&2
    fi
    exit 1
  fi
)
```

再次从新会话验证。回滚前副本和变更前状态记录会继续保留，确认系统稳定后再逐个核对；不要用通配符批量删除 `/etc/ssh/sshd_config.d/` 中的文件。

## 10. 主机指纹变化

主机重装、主机密钥重新生成或地址被另一台机器复用时，SSH 会警告主机身份变化。不要设置 `StrictHostKeyChecking no` 绕过。

先通过控制台重新读取服务端指纹，确认变化合理，再只移除对应旧记录：

**执行位置：SSH 客户端（任意目录）**

```bash
(
printf '请输入已经通过可信通道核对的主机地址或名称：'
IFS= read -r SSH_HOST
case "$SSH_HOST" in
  ''|-*)
    printf '%s\n' '停止：主机地址或名称为空，或以连字符开头。' >&2
    exit 1
    ;;
esac
if ! test -f "$HOME/.ssh/known_hosts"; then
  printf '%s\n' '停止：known_hosts 不存在或不是普通文件。' >&2
  exit 1
fi
if ! known_hosts_backup="$(mktemp "$HOME/.ssh/known_hosts.backup.XXXXXX")"; then
  printf '%s\n' '停止：无法创建 known_hosts 备份路径。' >&2
  exit 1
fi
if ! cp -- "$HOME/.ssh/known_hosts" "$known_hosts_backup"; then
  rm -f -- "$known_hosts_backup"
  printf '%s\n' '停止：known_hosts 备份失败，未移除任何记录。' >&2
  exit 1
fi
printf 'known_hosts_backup=%s\n' "$known_hosts_backup"
ssh-keygen -F "$SSH_HOST"
ssh-keygen -R "$SSH_HOST"
)
```

下次连接时重新比较新指纹。不要删除整个 `known_hosts`。

## 11. 排查顺序

| 现象 | 优先检查 | 典型命令 |
| --- | --- | --- |
| 连接超时 | 地址、路由、防火墙 | `ip route`、`ufw status`、[[Linux 主机防火墙与 UFW 基础#11. 启用后无法连接时如何恢复|UFW 分层排查]] |
| `Connection refused` | SSH socket 或服务是否监听 | `systemctl status ssh.socket ssh.service`、`ss -lntp` |
| `Permission denied (publickey)` | 用户名、客户端密钥、目录权限 | `ssh -vvv`、`stat ~/.ssh` |
| 主机密钥警告 | 是否重装或地址复用 | 控制台 `ssh-keygen -lf` |
| 登录后断开 | 用户 Shell、HOME 权限、服务日志 | `getent passwd`、`journalctl -u ssh` |
| 终端与 IDE 行为不同 | 是否使用相同别名和配置 | `ssh -G linux-host` |

客户端详细调试：

**执行位置：SSH 客户端（任意目录）**

```bash
ssh -vvv linux-host
```

三个 `-v` 将客户端调试信息提高到最详细的常规级别，并实际尝试连接；输出可能包含用户名、地址、配置路径和公钥指纹，分享前应脱敏。调试选项与排查顺序见 [[SSH 客户端命令基础#9. 按顺序排查和自助查询]]。

### 服务端配置与启动检查

服务端先检查激活单元，再按服务的当前状态检查配置或通过 systemd 启动：

**执行位置：Linux 服务端（控制台）**

```bash
systemctl show ssh.service \
  --property=RuntimeDirectory \
  --property=RuntimeDirectoryMode \
  --property=ExecStartPre
systemctl status ssh.socket ssh.service --no-pager || true
if systemctl is-active --quiet ssh.service; then
  sudo sshd -t
else
  sudo systemctl start ssh.service
fi
sudo journalctl -u ssh.service -b -n 100 --no-pager
```

若上述检查分支成功完成，配置检查已经通过，再读取当前有效配置：

```bash
sudo sshd -T | grep -E '^(port|pubkeyauthentication|passwordauthentication|permitrootlogin) '
```

### `sshd -t` 提示缺少 `/run/sshd`

如果在 `ssh.service` 尚未启动时直接执行 `sudo sshd -t`，可能看到 `Missing privilege separation directory: /run/sshd`。这表示手工命令绕过了负责创建 `RuntimeDirectory` 的 systemd 单元，不足以证明 `sshd_config` 存在语法错误。应先按上述顺序启动服务并查看日志，不要在未核对软件包机制前自行添加 tmpfiles 规则或将手工建目录当作持久修复。

## 完成检查

- [ ] 能区分主机身份与用户登录权限，以及 `known_hosts` 与 `authorized_keys` 的位置和作用。
- [ ] 知道主机私钥与用户私钥都不会通过 SSH 发送给对方。
- [ ] 服务端启动前配置检查通过，SSH socket 或服务正常监听。
- [ ] 已根据 `sshd -T`、`ss` 和实际客户端探测确认有效地址与端口。
- [ ] 首次连接前通过独立可信通道核对了主机指纹。
- [ ] 知道 `ssh-keyscan` 收集的主机公钥仍需通过可信通道核对。
- [ ] 能读懂 `ssh [选项] [用户@]主机 [远程命令]`，并区分交互式登录与单次远程命令。
- [ ] 用户公钥位于正确目标用户的 `authorized_keys`，对应私钥没有被传输。
- [ ] 新开终端可以独立使用密钥登录。
- [ ] `~/.ssh/config` 的别名可通过 `ssh -G` 核对。
- [ ] 收紧认证时保留了控制台、旧会话、备份和回滚路径。
- [ ] 能区分 SSH 监听、UFW 放行和用户认证三个层次。
- [ ] 能解释 `nc` 探测成功只证明当前 TCP 连接成立，不能代替 SSH 身份验证。
- [ ] 遇到连接或认证失败时，能按地址、端口、主机身份、用户认证、会话创建的顺序排查。

## 官方参考资料

- [Ubuntu Server：OpenSSH Server](https://documentation.ubuntu.com/server/how-to/security/openssh-server/)
- [Ubuntu 24.04 LTS 发布说明：OpenSSH 的 systemd socket activation](https://documentation.ubuntu.com/release-notes/24.04/#openssh)
- [IETF RFC 4252：SSH 用户认证协议](https://www.rfc-editor.org/rfc/rfc4252)
- [IETF RFC 4253：SSH 传输层协议](https://www.rfc-editor.org/rfc/rfc4253)
- [OpenBSD：ssh 客户端手册](https://man.openbsd.org/ssh.1)
- [OpenBSD：sshd 服务端手册](https://man.openbsd.org/sshd.8)
- [OpenBSD：ssh-keygen 手册](https://man.openbsd.org/ssh-keygen.1)
- [OpenBSD：ssh-keyscan 手册](https://man.openbsd.org/ssh-keyscan.1)
- [OpenBSD：ssh_config 手册](https://man.openbsd.org/ssh_config)
- [OpenBSD：sshd_config 手册](https://man.openbsd.org/sshd_config)
- [iproute2：`ss(8)` 手册](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenBSD：`nc(1)` 手册](https://man.openbsd.org/nc.1)
- [Ubuntu 24.04：`netcat-openbsd` 提供的 `nc(1)` 手册](https://manpages.ubuntu.com/manpages/noble/man1/nc_openbsd.1.html)

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
updated: 2026-08-21T14:54:41
---

SSH 是一套加密远程访问协议，OpenSSH 是其常用实现，通常由客户端 `ssh` 与服务端 `sshd` 配合工作。本篇以传统公钥登录为主线，完成服务端准备、网络入口核对、主机指纹验证、用户密钥登记、客户端配置、认证收紧和故障排查。

本篇只保留完成操作所需的最小原理、执行顺序、成功判据和恢复边界，不在流程中逐项讲解命令。`ssh`、`ssh-keygen`、`ssh-keyscan` 与 `sshd` 见 [[OpenSSH 常用命令基础]]；软件包、服务、网络、防火墙和权限分别由 [[APT 软件包管理基础]]、[[systemd 服务与日志基础]]、[[Linux 网络接口、IP 地址、路由与 DNS 基础]]、[[Linux 主机防火墙与 UFW 基础]] 与 [[Linux 用户、用户组、sudo 与文件权限]] 负责。

> [!info] 核对日期
> 本文于 **2026-07-26** 核对 Ubuntu Server 的 OpenSSH 配置流程与 Ubuntu 24.04 的 systemd socket activation。具体命令以 [[OpenSSH 常用命令基础]] 及目标主机上的实际手册为准；修改认证策略前，还必须核对由 systemd 管理的实际服务状态。

## 本篇掌握目标

- **必须熟练**：能区分主机身份与用户登录权限、`known_hosts` 与 `authorized_keys`；能安全完成主机指纹核对、用户公钥登记和独立新会话验证。
- **理解会查**：能根据服务状态、有效端口、监听结果、客户端探测和认证现象选择下一层检查，并按网络、主机身份、用户认证与远程会话的层次定位问题；能按“记录基线 → 写入配置片段 → 校验 → reload → 新会话验证 → 回滚”的顺序收紧认证。
- **认识即可**：知道 SSH 会自动协商算法并为当前连接建立加密通道；知道使用 `Match`、双因素认证、集中式认证或配置管理工具时，不能直接套用本文的简化收紧流程。

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
  → （可选）在保留恢复入口的前提下关闭密码登录
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

预期 `ssh.socket` 处于 `active (listening)`，执行上述检查后，`ssh.service` 处于 `active (running)`。同时确认有效配置中存在明确端口，并在 `ss` 输出中找到对应监听套接字；任一检查失败都先停止，不继续尝试客户端连接。`sshd` 检查模式、systemd 运行时目录和 socket activation 的边界见 [[OpenSSH 常用命令基础#5. 使用 sshd 检查服务端配置]] 与 [[systemd 服务与日志基础]]。

上述步骤在 Ubuntu Server 控制台执行，此时不需要停止服务。后续若通过 SSH 远程维护，应先确认控制台或其他独立管理入口可用；如果当前 SSH 会话是唯一可用入口，不要停止 `ssh.service`，以免会话中断后无法重新连接。

服务端查看当前有效配置：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
sudo sshd -T | grep -E '^(port|listenaddress|pubkeyauthentication|authorizedkeysfile|strictmodes|passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

确认端口、监听地址和各项认证配置符合预期。第 6 节会写入 `$HOME/.ssh/authorized_keys`，因此 `authorizedkeysfile` 必须包含 `.ssh/authorized_keys`；使用 `Match` 时，还要按实际连接上下文核对。详见 [[OpenSSH 常用命令基础#5.1 区分 sshd -t 与 sshd -T]] 与 [[OpenSSH 常用命令基础#5.2 Match 条件需要连接上下文]]。

监听状态与主机防火墙是不同层次：`sshd` 或 `ssh.socket` 正常监听，不代表 UFW 已允许外部连接；UFW 允许端口，也不代表 SSH 主机指纹或用户认证正确。启用防火墙前应同时比较有效配置、`ss` 实际监听和 UFW 规则，详见 [[Linux 主机防火墙与 UFW 基础#6.1 确认 SSH 管理入口并选择规则写法]]。

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

以上代码以 TCP 22 为例。只有当前客户端到已经核对的实际 SSH 端口明确连接成功，才继续核对主机身份；成功不能证明对端一定是 OpenSSH、主机指纹可信或用户密钥有效。明确拒绝、超时和不同 `nc` 实现的处理见 [[TCP 端口连通性测试与 nc 命令基础]]；如果 `sshd -T` 显示的实际端口不是 22，应在 `nc` 与后续 `ssh -p` 中使用同一个端口。

## 4. 首次连接前独立核对主机指纹

客户端首次连接某个目标时，`known_hosts` 中还没有可信主机公钥，因此不能只根据当前网络连接中显示的指纹判断对端身份。应通过服务端控制台、云平台控制台或管理员提供的可信记录取得参考指纹，再与客户端提示比较。若两份指纹都来自同一条可能被攻击的网络路径，它们不能彼此证明可信。

本节从服务端控制台读取主机公钥指纹，再与客户端提示比较。以下以 Ed25519 主机密钥为例：`/etc/ssh/ssh_host_ed25519_key` 是主机私钥，对应的 `.pub` 文件是主机公钥；下面只读取公钥指纹，不读取或复制私钥。

从服务端控制台读取 Ed25519 主机公钥指纹：

**执行位置：Linux 服务端（控制台，任意目录）**

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

记录输出中的 `SHA256:...` 指纹和 `ED25519` 类型；这一步只读取服务端主机公钥。`ssh-keygen` 的读取模式与输出字段见 [[OpenSSH 常用命令基础#3.2 读取公钥指纹]]。

如果命令提示 `.pub` 文件不存在，先不要继续首次连接。仅缺少公钥副本不等于对应主机私钥也缺失，也不能据此判断 `sshd` 一定无法连接；应先按 [[#11.3.1 服务端缺少可用主机密钥]] 区分实际状态，恢复可独立核对的参考指纹后再返回本节。

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

客户端首次连接提示会同时显示主机密钥类型和指纹。只有提示类型为 `ED25519` 时，才能将它的 `SHA256:...` 指纹与上述结果比较；如果提示的是 RSA 或 ECDSA，应改为读取对应的服务端主机公钥，不能跨类型比较。类型与指纹都一致后才接受，接受的主机公钥会写入客户端 `~/.ssh/known_hosts`。连接命令的执行边界见 [[OpenSSH 常用命令基础#2.3 本地 Shell、ssh 与远端 Shell 的边界]]。

不要把同一网络路径上的 `ssh-keyscan` 输出当作独立可信依据；它的用途和信任边界见 [[OpenSSH 常用命令基础#4. 使用 ssh-keyscan 收集主机公钥（认识即可）]]。

## 5. 创建独立的用户密钥

这里创建的是客户端用户密钥，不是服务端主机密钥。登录 Linux 主机与访问 GitHub、GitLab 等代码平台属于不同授权场景，建议按用途使用独立密钥；Git SSH 详见 [[Git 凭据、SSH 与常见问题排查]]。

先选择用途明确的路径并避免覆盖。下面的代码块只在私钥路径和 `.pub` 路径都未被文件、目录或符号链接占用时继续；任一路径已存在都停止，由使用者核对后另选名称。路径占用判断由 `test` 完成，相关语法与符号链接边界见 [[Shell 脚本阅读基础#6. 使用 test 表达条件|test 条件判断]]。

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

成功后应同时得到私钥和同名 `.pub` 公钥，并能读取公钥指纹。为私钥设置口令，只保留在可信客户端；创建模式、参数和文件影响见 [[OpenSSH 常用命令基础#3.1 创建用户密钥]]。

## 6. 将公钥加入 authorized_keys

下面先显示待登记公钥的指纹，再通过标准输入发送其内容。远端以目标用户身份设置目录和文件权限后追加公钥：

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
if ! public_key_info="$(ssh-keygen -lf "$KEY_PATH.pub")"; then
  printf '停止：无法读取待登记公钥的指纹。\n' >&2
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

printf '准备登记的客户端公钥：%s\n' "$public_key_info"
if ! ssh "$SSH_USER@$SSH_HOST" \
  'umask 077
   mkdir -p "$HOME/.ssh" &&
   chmod 700 "$HOME/.ssh" &&
   touch "$HOME/.ssh/authorized_keys" &&
   chmod 600 "$HOME/.ssh/authorized_keys" &&
   cat >> "$HOME/.ssh/authorized_keys"' \
  < "$KEY_PATH.pub"; then
  printf '%s\n' '停止：公钥追加或远端权限设置失败。' >&2
  exit 1
fi
)
```

这一步需要密码或其他已有认证方式。标准输入只发送公钥文件内容，私钥仍留在客户端。记下密钥类型和 `SHA256:...` 指纹，用于核对服务端记录。详见 [[OpenSSH 常用命令基础#2.3 本地 Shell、ssh 与远端 Shell 的边界]] 与 [[OpenSSH 常用命令基础#2.4 标准输入、输出与退出状态]]。

随后以目标用户核对账户、有效配置、权限和公钥指纹：

**执行位置：Linux 服务端（以刚才的目标登录用户执行，任意目录）**

```bash
id -un
sudo sshd -T |
  grep -E '^(pubkeyauthentication|authorizedkeysfile|strictmodes) '
stat -c 'mode=%A owner=%U group=%G path=%n' \
  "$HOME" "$HOME/.ssh" "$HOME/.ssh/authorized_keys"
ssh-keygen -lf "$HOME/.ssh/authorized_keys"
```

`id -un` 输出当前进程的有效用户名，参数含义见 [[Linux 用户、用户组、sudo 与文件权限#2.3 用 id 与 getent 分别观察两层状态|id -un 参数含义]]；结果应与刚才的 `SSH_USER` 一致，否则当前 `$HOME` 属于错误用户。有效配置应显示 `pubkeyauthentication yes`、`authorizedkeysfile` 包含 `.ssh/authorized_keys`、`strictmodes yes`；有 `Match` 时需按实际连接上下文核对。

`$HOME/.ssh` 和 `authorized_keys` 应分别为 `700` 和 `600`，并归目标用户所有；用户家目录不应允许组或其他用户写入。`ssh-keygen` 可能输出多行指纹，其中必须有一行的密钥类型和 `SHA256:...` 指纹与客户端一致。注释不参与认证。如果找不到匹配指纹，不要继续收紧认证。详见 [[OpenSSH 常用命令基础#3.2 读取公钥指纹]] 与 [[Linux 用户、用户组、sudo 与文件权限]]。

指纹一致只能证明公钥已写入服务端，实际认证仍需通过第 7 节的新会话验证。

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

ssh -F /dev/null \
  -o IdentitiesOnly=yes \
  -o PreferredAuthentications=publickey \
  -i "$KEY_PATH" \
  "$SSH_USER@$SSH_HOST"
)
```

`-F /dev/null` 忽略用户级和系统级客户端配置，`IdentitiesOnly=yes` 限制候选密钥，`PreferredAuthentications=publickey` 只尝试公钥认证。因此，新会话成功才能证明指定密钥可用，而不是回退到密码认证。私钥口令用于在客户端解锁私钥，不是服务端登录密码。

`-F /dev/null` 也会忽略 `~/.ssh/config` 中的端口、跳板机和代理设置。非默认端口需显式增加 `-p`；跳板或代理场景不属于本节的直连验证。详见 [[OpenSSH 常用命令基础#2.5 高频选项按问题记忆]]。失败时保持原有会话和控制台可用，不要继续收紧认证。

登录后验证身份和连接：

**执行位置：Linux 服务端（新 SSH 会话）**

```bash
whoami
id
hostnamectl --static
printf 'SSH_CONNECTION=%s\n' "$SSH_CONNECTION"
pwd
```

只有上述限定为指定身份和公钥认证的独立新会话成功，才能认为该密钥登录可用。

## 8. 使用客户端 ~/.ssh/config

客户端配置把动态地址、用户名和密钥路径集中在一个别名下。以下是本次登录流程使用的结构示例，`HostName` 和 `User` 必须替换为实际值；字段和配置读取边界见 [[OpenSSH 常用命令基础#2.6 使用 ~/.ssh/config 别名]]。

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

确认输出中的 `hostname`、`user`、`identityfile` 和 `identitiesonly` 都符合预期后，再使用别名连接。别名稳定而地址可变时，只需更新 `HostName`。

## 9. 可选：确认密钥登录可用后关闭密码登录

完成第 7 节后，密钥登录已经可以独立工作。关闭密码登录是额外的认证收紧步骤，不是理解或使用密钥登录的前置条件。本节只处理一条便于新手验证的主线：在没有同名既有配置的练习主机上，新建一个独立配置片段；不使用自动备份和自动回滚脚本。

整个过程按以下顺序推进：

```text
确认适用范围
  → 记录变更前有效值
  → 新建独立配置片段
  → 检查语法与有效值
  → reload SSH 服务
  → 从新终端验证密钥登录
```

> [!danger] 任何检查失败都先停止
> reload 前检查失败时，不执行 reload；reload 后新会话失败时，不关闭控制台、基准会话或变更前复测会话。旧会话仍然可用，只代表原连接尚未断开，不能证明新认证配置正确。

### 9.1 先判断本节是否适用

本节只适用于以下条件全部满足的场景：

- Ubuntu Server 控制台仍可用。
- 当前有一条已经验证的 SSH **基准会话**保持打开。
- 从另一个客户端终端再次使用密钥登录成功，形成**变更前复测会话**。
- `/etc/ssh/sshd_config.d/00-local-hardening.conf` 原本不存在。
- 当前没有无法解释的 `Match`、`AuthenticationMethods`、双因素认证、集中式认证或配置管理工具。

先观察配置片段和条件认证设置：

**执行位置：Linux 服务端（基准会话，任意目录）**

```bash
sudo ls -la /etc/ssh/sshd_config.d
sudo grep -RnisE '^[[:space:]]*(Match|AuthenticationMethods)[[:space:]]' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d
```

第一条命令的输出中如果已经存在 `00-local-hardening.conf`，立即停止，不要覆盖。第二条命令没有输出，表示在这两个标准位置没有找到已启用的 `Match` 或 `AuthenticationMethods`；只要出现匹配结果、读取错误，或者你不能解释现有片段的来源和用途，就不继续套用本节。

如果登录过程已经要求一次性验证码、硬件令牌或其他第二因素，也应停止。`KbdInteractiveAuthentication` 可能承载这类认证，不能为了关闭密码而直接禁用。

### 9.2 看懂准备修改的三项配置

本次只修改三项认证设置：

| 配置 | 本次值 | 作用 |
| --- | --- | --- |
| `PubkeyAuthentication` | `yes` | 明确保留公钥认证 |
| `PasswordAuthentication` | `no` | 关闭 SSH 的普通密码认证 |
| `KbdInteractiveAuthentication` | `no` | 关闭键盘交互认证，避免继续通过交互提示完成密码类认证 |

本次不同时修改 `PermitRootLogin`。关闭普通用户的密码认证与禁止 root 远程登录是两个不同决策，一次只验证一个变化更容易判断结果。root 登录策略放在 [[#9.8 可选：单独禁止 root 远程登录]]。

### 9.3 记录变更前基线

先确认服务运行、现有配置语法正确，再读取变更前的有效值：

**执行位置：Linux 服务端（基准会话，任意目录）**

```bash
systemctl is-active ssh.service
sudo sshd -t
sudo sshd -T | grep -E '^(pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

预期第一条命令输出 `active`，`sshd -t` 成功时没有输出，最后一条命令显示当前四项有效值。记录这些值，回滚后需要与它们比较。任一命令失败都先按第 11 节排查，不修改配置。

如果前三项已经分别是 `yes`、`no`、`no`，说明目标状态已经生效，不需要再创建重复片段。否则，从另一个客户端终端完成变更前复测并保持会话：

**执行位置：SSH 客户端（新终端，任意目录）**

```bash
ssh linux-host
```

若没有使用第 8 节的别名，则重复第 7 节已经验证成功的限定身份连接命令。变更前复测失败时，不继续关闭密码登录。

### 9.4 新建独立配置片段

Ubuntu 默认在主配置开头加载 `/etc/ssh/sshd_config.d/*.conf`，OpenSSH 对多数配置采用先读取到的值；通配匹配到的文件按字典顺序处理。因此，文件名中的 `00-` 用于让本地值尽早出现，不是随意编号。最终是否生效仍必须以 `sshd -T` 为准。

> [!warning] 风险：系统配置变化
> 只有第 9.1 节已经确认目标文件原本不存在时，才执行下面的编辑命令。`sudoedit` 的权限边界见 [[Linux 用户、用户组、sudo 与文件权限#3.3 sudo 不会自动提升 Shell 处理的重定向]]。

**执行位置：Linux 服务端（基准会话，任意目录）**

```bash
sudoedit /etc/ssh/sshd_config.d/00-local-hardening.conf
```

编辑器打开后只写入以下三行，然后保存并退出：

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

重新读取文件和元数据，确认内容完整，所有者为 `root`，并且普通用户不能写入：

```bash
sudo sed -n '1,20p' /etc/ssh/sshd_config.d/00-local-hardening.conf
sudo stat -c 'mode=%A owner=%U group=%G path=%n' \
  /etc/ssh/sshd_config.d/00-local-hardening.conf
```

内容、所有者或权限不符合预期时先停止，不要 reload。文件权限处理见 [[Linux 用户、用户组、sudo 与文件权限]]。

### 9.5 reload 前检查语法和有效值

先检查所有已加载配置的语法：

```bash
sudo sshd -t
```

成功时没有输出。然后读取有效认证配置：

```bash
sudo sshd -T | grep -E '^(pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication) '
```

预期结果是：

```text
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

如果语法检查失败或有效值不一致，不执行 reload。因为第 9.1 节已经确认该片段原本不存在，所以可以删除刚创建的文件并恢复变更前状态：

```bash
sudo rm -- /etc/ssh/sshd_config.d/00-local-hardening.conf
sudo sshd -t
sudo sshd -T | grep -E '^(pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

确认输出重新等于第 9.3 节记录的基线；如果文件并非本节新建，绝不能使用这组删除命令。

### 9.6 reload 并验证新会话

只有第 9.5 节的两类检查都通过后，才让认证配置作用于后续连接：

> [!warning] 风险：系统运行状态变化
> 保持控制台、基准会话和变更前复测会话，不停止或重启 SSH 服务。

```bash
sudo systemctl reload ssh.service
```

reload 成功后再读取服务状态和有效值：

```bash
systemctl is-active ssh.service
sudo sshd -T | grep -E '^(pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication) '
```

确认服务仍为 `active`，三项有效值仍符合预期。随后不要复用已有连接，从客户端再开一个终端建立**变更后验证会话**：

**执行位置：SSH 客户端（另一个新终端，任意目录）**

```bash
ssh linux-host
```

登录成功后执行 `whoami`、`hostnamectl --static` 和 `printf 'SSH_CONNECTION=%s\n' "$SSH_CONNECTION"`，确认目标用户、主机和连接都正确。登录失败时保持已有入口，并立即按 [[#9.7 需要撤销时如何回滚]] 恢复；只有这条新会话成功，才能认为“关闭密码后，密钥登录仍然可用”。

### 9.7 需要撤销时如何回滚

本节开始前已经确认配置片段不存在，因此回滚就是删除本节新增的文件。通过控制台或仍可用的基准会话执行：

**执行位置：Linux 服务端（控制台或基准会话，任意目录）**

```bash
sudo rm -- /etc/ssh/sshd_config.d/00-local-hardening.conf
sudo sshd -t
```

`sshd -t` 成功且没有输出时，才 reload 并读取恢复后的有效值：

```bash
sudo systemctl reload ssh.service
sudo sshd -T | grep -E '^(pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

确认有效值回到第 9.3 节记录的基线，再从客户端建立一条全新会话。语法检查失败时不要 reload；删除目标不是本节新建的文件时也不要继续。不要用通配符清理 `/etc/ssh/sshd_config.d/`。

### 9.8 可选：单独禁止 root 远程登录

先读取当前有效策略：

```bash
sudo sshd -T | grep '^permitrootlogin '
```

`prohibit-password` 表示 root 不能使用密码或键盘交互认证，但仍可能使用公钥；`no` 表示禁止 root 通过 SSH 登录。只有前面已经由本节创建配置片段，并且确认没有 root 密钥登录、远程备份、自动化任务或应急流程依赖这一入口时，才考虑在该片段中增加：

```text
PermitRootLogin no
```

增加后必须重新完成第 9.5 节的语法与有效值检查、第 9.6 节的 reload 和新会话验证。无法确认依赖时保持现状，不把它与关闭普通用户密码登录捆绑执行。

## 10. 安全处理主机身份变化警告

客户端出现 `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` 时，只能确定“当前对端提供的主机公钥与本地信任记录不同”。主机重装、主机密钥重新生成或旧主机被永久替换都可能导致这个现象，但连错目标或网络中间人也可能导致相同警告。

> [!danger] 先停止连接，不要先删除记录
> 未通过独立可信通道确认变化原因和新指纹前，不要接受新密钥，不要执行 `ssh-keygen -R`，也不要设置 `StrictHostKeyChecking no` 绕过警告。

### 10.1 先确认变化原因和新指纹

通过服务端控制台、云平台控制台或管理员保存的可信记录，先确认这次重装、密钥恢复或主机替换确实是预期变更。然后在控制台读取当前配置的主机密钥路径和指纹：

**执行位置：Linux 服务端（控制台，任意目录）**

```bash
sudo sshd -T | grep '^hostkey '
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key
```

第二条以默认 Ed25519 主机私钥为例；它只读取指纹，不输出私钥内容。应根据第一条命令和客户端警告中的密钥类型，改为实际对应的 `HostKey` 路径。如果 `sshd -T` 失败，先按 [[#11.3.1 服务端缺少可用主机密钥]] 排查，不要猜测路径后删除客户端记录。

只有控制台指纹的密钥类型和 `SHA256:...` 都与客户端当前收到的新指纹一致，且变化原因已经确认，才继续。两份指纹都来自当前 SSH 网络连接时，不构成独立验证。

如果同一地址和端口仍会在多台同时存活的主机之间轮换，不要反复删除信任记录。应先为每台主机提供稳定名称，或由管理员设计独立的 `HostKeyAlias` 与主机身份管理方案。

### 10.2 确定 known_hosts 实际使用的目标标识

先用发生警告时的同一个 SSH 目标和同一组连接选项读取客户端有效配置。以第 8 节别名为例：

**执行位置：SSH 客户端（任意目录）**

```bash
ssh -G linux-host |
  grep -E '^(hostname|port|hostkeyalias|userknownhostsfile) '
```

`hostname` 和 `port` 表示实际目标。如果显示了 `hostkeyalias`，`known_hosts` 使用该值代替主机名查找身份；没有这一行时忽略它。`userknownhostsfile` 表示客户端可能查询的用户级记录文件，下一节只处理本篇主线使用的 `~/.ssh/known_hosts`；警告指向其他文件时不要修改默认文件。

默认端口 22 通常使用主机部分本身作为查询标识；非默认端口通常使用 `[主机]:端口`，例如 `[linux-host.example.internal]:2222`。SSH 警告中如果给出了记录文件、行号和 `ssh-keygen -R` 目标，可以用它们反向核对，但仍然只能在第 10.1 节已经独立确认身份变化后使用。

### 10.3 查看、备份并移除该目标标识下的旧记录

本节只处理上一节确认的精确目标标识，以及本篇主线使用的 `~/.ssh/known_hosts`。以下命令需要在同一个客户端终端中依次执行：先查看影响范围，确认没有特殊记录，再创建备份，最后移除旧记录。任一步不符合预期都先停止。

**第一步：设置目标标识和记录文件**

**执行位置：SSH 客户端（任意目录）**

默认端口 22 可以这样填写：

```bash
KNOWN_HOSTS_TARGET='linux-host.example.internal'
KNOWN_HOSTS_FILE="$HOME/.ssh/known_hosts"
```

非默认端口则将第一行改为对应的 `[主机]:端口`，例如：

```bash
KNOWN_HOSTS_TARGET='[linux-host.example.internal]:2222'
KNOWN_HOSTS_FILE="$HOME/.ssh/known_hosts"
```

选择其中一组执行。只修改 `KNOWN_HOSTS_TARGET` 中的示例值，不要改变记录文件路径，也不要填入多个目标、通配符、空白字符或额外选项。

**第二步：查看旧记录及其指纹**

先确认记录文件是普通文件，再查看该目标的原始记录和指纹：

```bash
ls -l "$KNOWN_HOSTS_FILE"
ssh-keygen -F "$KNOWN_HOSTS_TARGET" -f "$KNOWN_HOSTS_FILE"
ssh-keygen -F "$KNOWN_HOSTS_TARGET" \
  -f "$KNOWN_HOSTS_FILE" |
  ssh-keygen -lf -
```

这些命令只读取信息，不会修改文件。`ls -l` 的输出应当表示普通文件，而不是带有 `->` 的符号链接。后两条命令应当找到上一节确认的精确目标；同一目标存在多种主机密钥类型时，可能显示多条记录，需要检查完整输出。

> [!warning] 本流程的停止条件
> 出现以下任一情况时，不执行后续移除命令：
>
> - SSH 警告指向其他记录文件；
> - 记录文件不存在、不是普通文件或是符号链接；
> - 查询没有结果，或结果不属于预期目标；
> - 记录使用 `@cert-authority`、`@revoked` 或通配模式。
>
> 这些情况的影响范围不能由本篇主线安全判定。

**第三步：创建独立备份**

```bash
KNOWN_HOSTS_BACKUP="$HOME/.ssh/known_hosts.backup.$(date +%Y%m%d-%H%M%S)"
cp -p "$KNOWN_HOSTS_FILE" "$KNOWN_HOSTS_BACKUP"
ls -l "$KNOWN_HOSTS_FILE" "$KNOWN_HOSTS_BACKUP"
```

`$(date +%Y%m%d-%H%M%S)` 在备份文件名中加入当前时间，避免覆盖以前的备份。最后一条命令应当同时显示原文件和备份文件，并且两者大小相同；如果复制失败、备份不存在或大小不同，先停止，不执行下一步。

**第四步：移除并确认旧记录已经不存在**

```bash
ssh-keygen -R "$KNOWN_HOSTS_TARGET" -f "$KNOWN_HOSTS_FILE"
ssh-keygen -F "$KNOWN_HOSTS_TARGET" -f "$KNOWN_HOSTS_FILE"
```

`ssh-keygen -R` 会移除这个精确目标标识下的全部匹配密钥，不是只移除某一种密钥类型。第二条命令此时应当没有匹配输出；如果移除失败或仍能查到旧记录，先停止，不要继续连接。不要删除整个 `known_hosts`。查询与移除模式见 [[OpenSSH 常用命令基础#3.3 查询和移除 known_hosts 记录]]。

> [!tip] 删除结果异常时恢复
> 在尚未重新连接、客户端还没有写入新记录之前，可以从刚才的独立备份恢复，然后重新查询：
>
> ```bash
> cp -p "$KNOWN_HOSTS_BACKUP" "$KNOWN_HOSTS_FILE"
> ssh-keygen -F "$KNOWN_HOSTS_TARGET" -f "$KNOWN_HOSTS_FILE"
> ```
>
> 如果已经重新连接并写入了新记录，不要直接用旧备份覆盖当前文件，应先比较两份文件再决定如何恢复。

查询不到旧记录只表示旧信任记录已经移除，还不能证明当前服务器可信。接下来仍须在第 10.4 节重新连接，并再次核对新主机指纹。

### 10.4 重新连接并完成恢复验证

使用与发生警告时相同的目标和连接选项重新连接。以第 8 节别名为例：

**执行位置：SSH 客户端（新终端，任意目录）**

```bash
ssh linux-host
```

客户端再次显示主机密钥类型和指纹时，必须与第 10.1 节从独立可信通道取得的新指纹再比较一次；两者一致才接受。登录后执行：

**执行位置：Linux 服务端（新 SSH 会话）**

```bash
whoami
hostnamectl --static
printf 'SSH_CONNECTION=%s\n' "$SSH_CONNECTION"
```

最后回到客户端，将下面的示例值替换为第 10.3 节实际使用的目标标识，确认新记录的类型和指纹：

```bash
KNOWN_HOSTS_TARGET='linux-host.example.internal'
ssh-keygen -F "$KNOWN_HOSTS_TARGET" \
  -f "$HOME/.ssh/known_hosts" |
  ssh-keygen -lf -
```

只有新主机指纹已经独立核对、新记录已写入，且新会话完成用户认证和身份检查，恢复才算完成。

## 11. 按连接阶段分层排查

排查的目标不是把所有命令都执行一遍，而是找到连接流程中第一个没有成功的阶段。客户端有效配置决定实际连接目标，因此排查时先在第 1 节的四个连接阶段前增加一道入口检查：

```text
客户端有效配置
  → 目标地址与 TCP 端口
  → SSH 传输与主机身份
  → 用户认证
  → 远程会话创建
```

前一层成功只是进入下一层的条件，不代表后续阶段也成功。无法远程连接时，服务端检查应使用控制台或其他独立管理入口，不要为了排查而关闭仍可用的基准会话。

### 11.1 先确定有效目标和失败阶段

先读取客户端实际采用的目标、用户、端口和身份文件：

**执行位置：SSH 客户端（任意目录）**

```bash
ssh -G linux-host |
  grep -E '^(hostname|user|port|identityfile|identitiesonly|proxycommand|proxyjump|hostkeyalias|userknownhostsfile) '
```

别名、直接主机名和 IDE 里填写的目标可能解析为不同配置。这一步不建立 SSH 网络连接，但可信配置中的 `Match exec` 可能执行本地命令，具体边界见 [[OpenSSH 常用命令基础#2.6 使用 ~/.ssh/config 别名]]。

配置正确后再进行一次带详细日志的实际连接：

```bash
ssh -vvv linux-host
```

`ssh -vvv` 的最后成功阶段决定下一步：TCP 尚未建立时进入第 11.2 节；TCP 已建立但 SSH 握手或主机身份失败时进入第 11.3 节；出现 `Permission denied (publickey)` 时进入第 11.4 节；已经显示认证成功却无法建立 Shell 或会话时进入第 11.5 节。输出可能包含用户名、地址、配置路径和公钥指纹，分享前应脱敏；客户端命令顺序见 [[OpenSSH 常用命令基础#2.8 按顺序排查]]。

### 11.2 排查目标地址与 TCP 端口

先查看第 11.1 节的有效配置中是否存在非 `none` 的 `proxyjump` 或 `proxycommand`。如果 SSH 需要经过跳板机或代理，从当前客户端直接执行 `nc` 不能复现 SSH 的实际网络路径，应分别检查客户端到跳板机、跳板机到目标的路径；该场景不套用下面的直连示例。

直连场景使用第 11.1 节读到的有效 `hostname` 和 `port` 进行有边界的单目标探测。以默认端口为例：

**执行位置：SSH 客户端（任意目录）**

```bash
nc -vz -w 5 linux-host.example.internal 22
```

超时通常应先检查目标名称或地址、客户端路由、中间网络和防火墙丢弃；`Connection refused` 表示对端已经明确拒绝当前端口，应重新确认端口是否正确、是否存在监听套接字，以及防火墙是否显式拒绝。两种结果都不能单凭现象确定是 SSH 配置问题。

在服务端控制台核对本机地址、路由、激活单元、实际监听和 UFW：

**执行位置：Linux 服务端（控制台，任意目录）**

```bash
ip -brief address
ip route
systemctl status ssh.socket ssh.service --no-pager
sudo ss -lntp
sudo ufw status verbose
```

`ss` 只证明本机存在监听，UFW 允许规则也只证明主机防火墙的当前政策。只有客户端对实际地址和端口的 `nc` 探测成功，才说明这条 TCP 路径在当时可达。详见 [[Linux 网络接口、IP 地址、路由与 DNS 基础]]、[[Linux 端口、监听套接字与 ss 命令基础]]、[[TCP 端口连通性测试与 nc 命令基础]] 与 [[Linux 主机防火墙与 UFW 基础#7.3 按层次定位]]。

### 11.3 排查 SSH 传输、服务启动与主机身份

TCP 端口可达只说明当前端口接受了连接，不能确定对端就是可用的 OpenSSH 服务。如果 `ssh -vvv` 显示 TCP 已连接但 SSH 版本交换、密钥协商或主机身份阶段中断，先查看服务状态和日志，不要直接把原因定为缺少主机密钥。

**执行位置：Linux 服务端（控制台，任意目录）**

```bash
systemctl show ssh.service \
  --property=RuntimeDirectory \
  --property=RuntimeDirectoryMode \
  --property=ExecStartPre
systemctl status ssh.socket ssh.service --no-pager
sudo journalctl -u ssh.service -b -n 100 --no-pager
```

先根据 unit 状态确认当前是由 `ssh.socket` 还是 `ssh.service` 承担入口。默认 socket activation 下，尚未收到连接时 `ssh.service` 未运行不一定是故障；已经发生握手失败时，日志才是判断激活失败原因的主要依据。

服务已经运行时直接检查配置和主机密钥健全性；服务未运行时通过 systemd 启动，让 unit 先准备运行时目录：

```bash
if systemctl is-active --quiet ssh.service; then
  sudo sshd -t
else
  sudo systemctl start ssh.service
fi
```

上述分支成功后，再读取有效配置并核对实际监听：

```bash
sudo sshd -T |
  grep -E '^(port|listenaddress|hostkey) '
sudo ss -lntp
```

成功判据是实际激活方式下的入口单元正常、有效配置可读，且 `ss` 中存在对应监听，不是机械要求 `ssh.socket` 和 `ssh.service` 在所有系统上保持同一状态。若检查提示缺少 `/run/sshd`，按 [[OpenSSH 常用命令基础#5.4 sshd -t 提示缺少 /run/sshd]] 核对 systemd 运行时目录，不要直接手工创建目录或添加未经确认的 tmpfiles 规则。

如果客户端已经显示主机身份变化警告，转到 [[#10. 安全处理主机身份变化警告]]；如果 `sshd -t`、systemd 启动结果或日志指向没有可用主机密钥，再进入下一分支。

#### 11.3.1 服务端缺少可用主机密钥

主机身份核对发生在用户认证之前。如果所有已配置的主机私钥都缺失、权限不安全或无法读取，`sshd` 不能向客户端证明服务端身份，会在进入用户认证前退出。这不是 `authorized_keys`、客户端用户私钥或登录密码的问题。在 socket activation 下，`ssh.socket` 仍可能使 TCP 端口表现为可达，但随后激活的 `ssh.service` 启动失败，因此不能把一次 `nc` 成功当成 SSH 握手可用。

先通过控制台检查日志、显式 `HostKey` 配置和默认主机密钥文件：

**执行位置：Linux 服务端（控制台，任意目录）**

```bash
sudo journalctl -u ssh.service -b -n 100 --no-pager
sudo grep -RnisE '^[[:space:]]*HostKey[[:space:]]+' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d
sudo find /etc/ssh -maxdepth 1 \
  \( -name 'ssh_host_*_key' -o -name 'ssh_host_*_key.pub' \) \
  -exec stat -c 'mode=%A owner=%U group=%G path=%n' {} +
```

`grep` 没有匹配结果可能表示正在使用 OpenSSH 默认路径；读取错误则必须单独处理，不能当成“没有配置”。如果显式 `HostKey` 指向 `/etc/ssh` 以外或非默认名称，必须对该实际路径另行执行 `stat`；上面的 `find` 只显示默认目录和命名范围，不能证明自定义密钥一定存在。当 `sshd -T` 可以成功输出时，应以其 `hostkey` 结果核对实际有效路径。

结合前面的 `sshd -t` 或 systemd 启动结果区分三种状态：

| 文件状态 | 对连接的影响 | 下一步 |
| --- | --- | --- |
| 只有某个 `.pub` 公钥副本缺失，对应私钥仍可用 | `sshd` 通常仍能使用私钥建立连接，但直接读取该 `.pub` 指纹的命令会失败 | 不要重新生成私钥；可用 `sudo ssh-keygen -lf 对应私钥路径` 只读取指纹，并按主机的备份或管理方式恢复公钥副本 |
| 主机私钥存在，但所有者、权限或可读性不符合要求 | `sshd` 会拒绝使用不安全或无法读取的私钥，服务可能降级使用其他密钥或直接启动失败 | 根据日志、可信备份和同系统的受管默认值恢复所有者与权限；不要用 `ssh-keygen -A` 规避既有文件的权限问题 |
| 某一种默认主机私钥缺失，仍有其他可用主机私钥 | 服务可能继续使用其余类型，但可用的主机密钥算法会减少 | 先确认缺失原因；确需补齐默认类型时再使用 `ssh-keygen -A` |
| 没有任何可用主机私钥 | `sshd` 配置检查或服务启动失败，客户端无法完成 SSH 握手 | 优先恢复原主机密钥；无法恢复且接受身份变化时才生成新密钥 |

对于已经被客户端信任的既有服务器，应先从可信备份、系统镜像或既定密钥管理流程恢复原主机私钥及对应公钥，以保持主机身份不变。不要把“文件不存在”直接当作授权重新生成；先确认它不是挂载失败、权限错误、自定义 `HostKey` 路径或意外删除。

只有新服务器尚未建立主机身份，或者已经确认原密钥无法恢复并接受身份变化时，才补齐默认路径中缺失的主机密钥：

```bash
sudo ssh-keygen -A
```

`ssh-keygen -A` 成功后，服务已经运行时先检查配置，只有 `sshd -t` 成功且没有输出才 reload：

```bash
sudo sshd -t &&
  sudo systemctl reload ssh.service
```

服务未运行时通过 systemd 启动：

```bash
sudo systemctl start ssh.service
```

不要在 `sshd -t` 失败后继续 reload。启动或 reload 后根据实际激活方式核对 unit、监听和日志：

```bash
systemctl status ssh.socket ssh.service --no-pager
sudo ss -lntp
sudo journalctl -u ssh.service -b -n 100 --no-pager
```

`ssh-keygen -A` 只为默认路径中尚不存在的默认类型生成主机密钥，不覆盖已经存在的默认主机密钥；它不会恢复丢失的原身份，也不能修复指向其他路径的自定义 `HostKey`。命令模式与影响范围见 [[OpenSSH 常用命令基础#3.4 补齐缺失的默认主机密钥（服务端恢复）]]。

服务恢复后，先在控制台重新读取实际提供类型的主机指纹。恢复的是原密钥时，指纹应与原可信记录一致；生成的是新密钥时，按 [[#10. 安全处理主机身份变化警告]] 完成独立指纹核对、客户端旧记录处理和新会话验证。

### 11.4 排查用户公钥认证

出现 `Permission denied (publickey)` 时，TCP 和主机身份阶段通常已经通过，不应回到 UFW 或 `known_hosts` 开始猜测。先在客户端确认实际用户、身份文件和被提供的公钥：

**执行位置：SSH 客户端（任意目录）**

```bash
ssh -G linux-host |
  grep -E '^(hostname|user|port|identityfile|identitiesonly|preferredauthentications) '
ssh-keygen -lf "$HOME/.ssh/id_ed25519_linux_host.pub"
ssh -vvv linux-host
```

客户端公钥指纹应与服务端授权记录中的对应指纹一致。`ssh -vvv` 中要区分“客户端发现了密钥”、“客户端向服务端提供了该公钥”与“服务端接受了该公钥”，不能只看本地文件存在。

服务端存在 `Match` 时，普通 `sshd -T` 不足以代表这次连接的实际配置。使用真实连接上下文替换下面的示例值：

**执行位置：Linux 服务端（控制台或基准会话，任意目录）**

```bash
sudo sshd -T -C \
  'user=linux-user,addr=192.0.2.10,host=client.example.internal,laddr=192.0.2.20,lport=22' |
  grep -E '^(pubkeyauthentication|authorizedkeysfile|authorizedkeyscommand|strictmodes|authenticationmethods|allowusers|denyusers|allowgroups|denygroups) '
```

`user` 是目标 Linux 用户，`addr` 和 `host` 分别是客户端源地址和用于匹配的源主机名，`laddr` 和 `lport` 是服务端本地地址和 SSH 端口。无法确认这些值时，不要用猜测值解释 `Match` 结果。

使用 `getent` 确认目标用户的实际 HOME，再以该用户身份检查本篇默认授权路径。下面的 `linux-user` 必须替换为实际目标用户：

```bash
getent passwd linux-user
sudo -H -u linux-user sh -c '
  id -un
  printf "HOME=%s\n" "$HOME"
  stat -c "mode=%A owner=%U group=%G path=%n" \
    "$HOME" "$HOME/.ssh" "$HOME/.ssh/authorized_keys"
  ssh-keygen -lf "$HOME/.ssh/authorized_keys"
'
```

如果有效 `authorizedkeysfile` 不是 `.ssh/authorized_keys`，应先展开其 `%h`、`%u` 等令牌并检查实际路径，不要继续检查默认文件。如果使用由外部命令提供授权密钥的 `AuthorizedKeysCommand`、集中式身份或 SSH 证书，也应转入对应的授权来源，不盲目修改用户文件。匹配的 `authorized_keys` 行如果携带 `from=`、`expiry-time=` 或 `cert-authority` 等选项，还必须核对来源地址、有效期和密钥用途；不要为了快速复测而删除用途未知的限制。

默认主线中，`$HOME/.ssh` 和 `authorized_keys` 应分别为 `700` 和 `600`，并归目标用户所有；HOME 不应允许组或其他用户写入。还应在服务端打开实时日志，然后从客户端重试一次：

```bash
sudo journalctl -fu ssh.service
```

按 `Ctrl-C` 结束跟踪。日志可能包含用户名和客户端地址，分享前应脱敏。最后按第 7 节的限定身份方式建立新会话；只有预期用户、预期私钥和公钥认证独立成功，才证明该密钥登录路径已恢复。

### 11.5 排查认证成功后的会话创建

如果 `ssh -vvv` 已经显示用户认证成功，但登录后立即断开、无法进入 Shell 或只有某类远程命令失败，不要再更换用户密钥。此时应检查账户记录、HOME、登录 Shell、会话限制和服务日志：

**执行位置：Linux 服务端（控制台或基准会话，任意目录）**

```bash
getent passwd linux-user
getent shells
sudo test -e /etc/nologin && sudo sed -n '1,40p' /etc/nologin
sudo journalctl -u ssh.service -b -n 100 --no-pager
```

`getent passwd` 输出的第六、第七个字段分别是 HOME 和登录 Shell。确认 HOME 存在且目标用户可访问，Shell 路径存在、可执行且符合预期；`/etc/nologin` 存在时会限制普通用户登录。如果有效配置中存在强制执行指定命令的 `ForceCommand`、限制会话文件系统根目录的 `ChrootDirectory`、禁止分配终端的 `PermitTTY no`，或 `authorized_keys` 行内的 `command=` 与 `restrict` 限制，还要根据实际会话类型检查这些配置。

### 11.6 排查终端与 IDE 的配置差异

如果终端使用某个别名可以成功登录，而 IDE 失败，已经证明该终端实际使用的网络、主机身份和用户认证路径可用。下一步是比较 IDE 与 `ssh -G linux-host` 的实际目标，而不是重新修改服务端。

优先核对 IDE 使用的 SSH 程序、配置文件、`Host` 别名、`HostName`、`User`、`Port`、`IdentityFile`、`IdentitiesOnly`、跳板机或代理设置，以及 IDE 是否使用了独立密钥库。只有两端的有效参数一致后，它们的连接结果才具有可比性。

### 11.7 按现象快速定位

| 第一个失败阶段 | 典型现象 | 优先检查 | 执行位置 |
| --- | --- | --- | --- |
| 客户端有效配置 | 终端与 IDE 结果不同，或连到错误目标 | `ssh -G`、别名、用户、端口、身份文件和代理 | 客户端 |
| 目标地址与 TCP 路径 | 连接超时 | 名称或地址、路由、中间网络、UFW | 客户端与服务端控制台 |
| 目标端口与监听 | `Connection refused` | 有效端口、unit 状态、`ss -lntp`、防火墙显式拒绝 | 客户端与服务端控制台 |
| SSH 传输与服务启动 | TCP 可达但 SSH 握手中断，或 `ssh.service` 启动失败 | `ssh -vvv`、`sshd -t`、unit 状态与服务日志 | 客户端与服务端控制台 |
| 主机身份 | 主机身份变化警告 | 变更原因、独立可信指纹与 [[#10. 安全处理主机身份变化警告\|第 10 节恢复流程]] | 服务端控制台与客户端 |
| 用户认证 | `Permission denied (publickey)` | 实际用户和身份、`sshd -T -C`、有效授权来源、指纹、权限和日志 | 客户端与服务端控制台 |
| 远程会话创建 | 认证成功后立即断开或无法创建 Shell | 账户记录、HOME、登录 Shell、会话限制和服务日志 | 服务端控制台 |

## 完成检查

- [ ] 能区分主机身份与用户登录权限，以及 `known_hosts` 与 `authorized_keys` 的位置和作用。
- [ ] 知道主机私钥与用户私钥都不会通过 SSH 发送给对方。
- [ ] 服务端启动前配置检查通过，SSH socket 或服务正常监听。
- [ ] 已根据 `sshd -T`、`ss` 和实际客户端探测确认有效地址与端口。
- [ ] 首次连接前通过独立可信通道核对了主机指纹。
- [ ] 用户公钥位于正确目标用户的有效 `authorized_keys` 路径，服务端指纹与客户端公钥一致，对应私钥没有被传输。
- [ ] 新开终端可以在不回退到密码或其他客户端身份的情况下，独立使用指定密钥登录。
- [ ] `~/.ssh/config` 的别名可通过 `ssh -G` 核对。
- [ ] 可选收紧认证时记录了变更前基线，保留了控制台、基准会话和复测会话，并完成语法检查、有效值检查、reload、新会话验证与回滚验证。
- [ ] 能区分关闭普通用户密码认证与禁止 root 远程登录；遇到 `Match`、双因素认证、集中式认证或配置管理工具时知道停止套用简化流程。
- [ ] 能区分 SSH 监听、UFW 放行和用户认证三个层次。
- [ ] 客户端 TCP 探测成功后，仍独立完成了主机身份与用户身份验证。
- [ ] 主机身份变化时，能先通过独立可信通道确认原因和新指纹，再核对实际 `known_hosts` 目标标识、备份并移除该目标的旧记录，最后用新会话复验。
- [ ] 能区分公钥副本缺失、部分主机私钥缺失与没有任何可用主机私钥，并优先恢复既有主机身份。
- [ ] 遇到连接或认证失败时，能按客户端有效配置、地址与 TCP 端口、SSH 传输与主机身份、用户认证、会话创建的顺序排查。

## 官方参考资料

- [Ubuntu Server：OpenSSH Server](https://documentation.ubuntu.com/server/how-to/security/openssh-server/)
- [Ubuntu 24.04 LTS 发布说明：OpenSSH 的 systemd socket activation](https://documentation.ubuntu.com/release-notes/24.04/#openssh)
- [Ubuntu 24.04：`ssh_config(5)` 手册](https://manpages.ubuntu.com/manpages/noble/man5/ssh_config.5.html)
- [Ubuntu 24.04：`ssh-keygen(1)` 手册](https://manpages.ubuntu.com/manpages/noble/man1/ssh-keygen.1.html)
- [Ubuntu 24.04：`sshd(8)` 手册](https://manpages.ubuntu.com/manpages/noble/man8/sshd.8.html)
- [IETF RFC 4252：SSH 用户认证协议](https://www.rfc-editor.org/rfc/rfc4252)
- [IETF RFC 4253：SSH 传输层协议](https://www.rfc-editor.org/rfc/rfc4253)
- [OpenBSD：sshd_config 手册](https://man.openbsd.org/sshd_config)

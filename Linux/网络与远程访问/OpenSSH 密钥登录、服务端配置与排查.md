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
updated: 2026-07-25T16:22:15
---

SSH 是一套加密远程访问协议，OpenSSH 是其常用实现，通常由客户端 `ssh` 与服务端 `sshd` 配合工作。本篇以传统公钥登录为主线，完成服务端准备、网络入口核对、主机指纹验证、用户密钥登记、客户端配置、认证收紧和故障排查。

本篇只保留完成操作所需的最小原理、执行顺序、成功判据和恢复边界，不在流程中逐项讲解命令。`ssh`、`ssh-keygen`、`ssh-keyscan` 与 `sshd` 见 [[OpenSSH 常用命令基础]]；软件包、服务、网络、防火墙和权限分别由 [[APT 软件包管理基础]]、[[systemd 服务与日志基础]]、[[Linux 网络接口、IP 地址、路由与 DNS 基础]]、[[Linux 主机防火墙与 UFW 基础]] 与 [[Linux 用户、用户组、sudo 与文件权限]] 负责。

> [!info] 核对日期
> 本文于 **2026-07-25** 核对 Ubuntu Server 的 OpenSSH 配置流程与 Ubuntu 24.04 的 systemd socket activation。具体命令以 [[OpenSSH 常用命令基础]] 及目标主机上的实际手册为准；修改认证策略前，还必须核对由 systemd 管理的实际服务状态。

## 本篇掌握目标

- **必须熟练**：能区分主机身份与用户登录权限、`known_hosts` 与 `authorized_keys`；能安全完成主机指纹核对、用户公钥登记和独立新会话验证。
- **理解会查**：能根据服务状态、有效端口、监听结果、客户端探测和认证现象选择下一层检查，并按网络、主机身份、用户认证与远程会话的层次定位问题。
- **认识即可**：知道 SSH 会自动协商算法并为当前连接建立加密通道；能看出认证收紧脚本的备份、校验、重载与失败回滚顺序。

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

预期 `ssh.socket` 处于 `active (listening)`，执行上述检查后，`ssh.service` 处于 `active (running)`。同时确认有效配置中存在明确端口，并在 `ss` 输出中找到对应监听套接字；任一检查失败都先停止，不继续尝试客户端连接。`sshd` 检查模式、systemd 运行时目录和 socket activation 的边界见 [[OpenSSH 常用命令基础#5. 使用 sshd 检查服务端配置]] 与 [[systemd 服务与日志基础]]。

上述步骤在 Ubuntu Server 控制台执行，此时不需要停止服务。后续若通过 SSH 远程维护，应先确认控制台或其他独立管理入口可用；如果当前 SSH 会话是唯一可用入口，不要停止 `ssh.service`，以免会话中断后无法重新连接。

服务端查看当前有效配置：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
sudo sshd -T | grep -E '^(port|listenaddress|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

确认输出中的端口、监听地址和认证策略符合预期；命令模式与 `Include` 解析见 [[OpenSSH 常用命令基础#5.1 区分 sshd -t 与 sshd -T]]。

监听状态与主机防火墙是不同层次：`sshd` 或 `ssh.socket` 正常监听，不代表 UFW 已允许外部连接；UFW 允许端口，也不代表 SSH 主机指纹或用户认证正确。启用防火墙前应同时比较有效配置、`ss` 实际监听和 UFW 规则，详见 [[Linux 主机防火墙与 UFW 基础#6. 先比较 SSH 配置、监听状态与 UFW profile]]。

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
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

记录输出中的 `SHA256:...` 指纹和 `ED25519` 类型；这一步只读取服务端主机公钥。`ssh-keygen` 的读取模式与输出字段见 [[OpenSSH 常用命令基础#3.2 读取公钥指纹]]。

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

客户端首次连接提示会同时显示主机密钥类型和指纹。只有提示类型为 `ED25519` 时，才能将它的 `SHA256:...` 指纹与上述结果比较；如果提示的是 RSA 或 ECDSA，应改为读取对应的服务端主机公钥，不能跨类型比较。类型与指纹都一致后才接受，接受的主机公钥会写入客户端 `~/.ssh/known_hosts`。连接命令的执行边界见 [[OpenSSH 常用命令基础#2.2 怎样阅读 ssh "$SSH_USER@$SSH_HOST"]]。

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

这一步需要密码或其他已经可用的认证方式。成功时只把本地公钥追加到目标用户的 `authorized_keys`，对应私钥不会离开客户端；远程命令和标准输入的执行边界见 [[OpenSSH 常用命令基础#2.3 交互式登录与单次远程命令]] 与 [[OpenSSH 常用命令基础#2.4 标准输入、输出与退出状态]]。

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

成功时应使用指定密钥建立一个新的独立会话；失败时保持原有会话和控制台可用，不要继续收紧认证。身份选择与候选密钥范围见 [[OpenSSH 常用命令基础#2.5 高频选项按问题记忆]]。

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

## 9. Fail-closed 收紧服务端认证

这里应把整段当作一个有前置条件和回滚的完整操作，不要抽出中间命令单独执行。Shell 控制结构见 [[Shell 脚本阅读基础]] 与 [[Shell 标准流、管道、重定向与退出状态]]，服务端检查与 reload 边界见 [[OpenSSH 常用命令基础#5.3 检查配置不等于让配置生效]]。

只有满足以下条件后，才考虑关闭密码登录：

- 控制台仍可用。
- 至少两个独立的新密钥会话已经成功。
- 已确认正确 Linux 用户拥有可用公钥。
- 当前旧会话保持打开。

以下脚本使用独立配置片段，先备份，再校验语法与有效值；任一步失败都会尝试恢复原状态。必须从第一行到最后一行完整执行：

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

代码块先创建独立备份，再显示并移除这个目标的旧记录。确认备份路径已经输出后，下次连接重新比较新指纹；不要删除整个 `known_hosts`。查询与移除模式见 [[OpenSSH 常用命令基础#3.3 查询和移除 known_hosts 记录]]。

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

该命令会实际尝试连接，输出可能包含用户名、地址、配置路径和公钥指纹，分享前应脱敏；客户端排查顺序见 [[OpenSSH 常用命令基础#2.8 按顺序排查]]。

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

若检查提示缺少 `/run/sshd`，按 [[OpenSSH 常用命令基础#5.4 sshd -t 提示缺少 /run/sshd]] 核对 systemd 运行时目录，不要直接手工创建目录或添加未经确认的 tmpfiles 规则。

## 完成检查

- [ ] 能区分主机身份与用户登录权限，以及 `known_hosts` 与 `authorized_keys` 的位置和作用。
- [ ] 知道主机私钥与用户私钥都不会通过 SSH 发送给对方。
- [ ] 服务端启动前配置检查通过，SSH socket 或服务正常监听。
- [ ] 已根据 `sshd -T`、`ss` 和实际客户端探测确认有效地址与端口。
- [ ] 首次连接前通过独立可信通道核对了主机指纹。
- [ ] 用户公钥位于正确目标用户的 `authorized_keys`，对应私钥没有被传输。
- [ ] 新开终端可以独立使用密钥登录。
- [ ] `~/.ssh/config` 的别名可通过 `ssh -G` 核对。
- [ ] 收紧认证时保留了控制台、旧会话、备份和回滚路径。
- [ ] 能区分 SSH 监听、UFW 放行和用户认证三个层次。
- [ ] 客户端 TCP 探测成功后，仍独立完成了主机身份与用户身份验证。
- [ ] 遇到连接或认证失败时，能按地址、端口、主机身份、用户认证、会话创建的顺序排查。

## 官方参考资料

- [Ubuntu Server：OpenSSH Server](https://documentation.ubuntu.com/server/how-to/security/openssh-server/)
- [Ubuntu 24.04 LTS 发布说明：OpenSSH 的 systemd socket activation](https://documentation.ubuntu.com/release-notes/24.04/#openssh)
- [IETF RFC 4252：SSH 用户认证协议](https://www.rfc-editor.org/rfc/rfc4252)
- [IETF RFC 4253：SSH 传输层协议](https://www.rfc-editor.org/rfc/rfc4253)
- [OpenBSD：sshd_config 手册](https://man.openbsd.org/sshd_config)

---
title: SSH 客户端命令基础
aliases:
  - ssh 客户端命令基础
  - OpenSSH ssh 命令基础
  - SSH 远程登录与命令执行
tags:
  - Linux
  - Linux/网络
  - Linux/命令行
  - Linux/远程访问
  - SSH
  - OpenSSH
created: 2026-07-21T00:43:10
updated: 2026-07-21T00:43:10
---

`ssh` 是 OpenSSH 的客户端命令，在当前客户端上启动，用于连接运行 `sshd` 的远程主机。它既可以创建交互式远程 Shell，也可以只执行一条远程命令，然后把远端的输出和退出状态带回本地。

本篇只解释 `ssh` 客户端的常用命令骨架、本地与远端的执行边界、身份选择、客户端配置和基础调试。服务端安装、主机指纹、用户密钥、`authorized_keys` 与认证策略见 [[OpenSSH 连接、密钥与主机指纹]]；连接前的 TCP 探测见 [[TCP 端口连通性测试与 nc 命令基础]]；变量、引号和重定向见 [[Shell 路径、变量、引用与展开]] 与 [[Shell 标准流、管道、重定向与退出状态]]。

> [!abstract] 本篇掌握目标
> - **必须熟练**：能读懂 `ssh [选项] [用户@]主机 [远程命令]`，区分交互式登录与单次远程命令，并判断一段内容由本地 Shell 还是远端 Shell 解释。
> - **理解会查**：知道 `-p`、`-i`、`-o`、`-G` 和 `-v` 分别解决什么问题，忘记细节时会使用 `man ssh`、`man ssh_config` 和 `ssh -G`。
> - **认识即可**：端口转发、跳板机、连接复用、Agent 转发和复杂算法选项；有明确需求时再查对应手册。

> [!info] 核对日期与适用范围
> 本文于 **2026-07-21** 核对 OpenBSD OpenSSH 的 `ssh(1)`、`ssh_config(5)` 手册与 Ubuntu 24.04 对应手册。不同客户端版本和配置可能影响默认用户名、端口、身份文件及算法选择，应以实际客户端的 `ssh -V`、`man ssh`、`man ssh_config` 和 `ssh -G` 输出为准。

## 1. 先分清本地客户端与远端服务端

一条 SSH 命令会经过四个执行者：

| 执行者 | 所在位置 | 负责什么 |
| --- | --- | --- |
| 本地 Shell | SSH 客户端 | 先处理变量、引号、重定向和命令行参数 |
| `ssh` | SSH 客户端 | 建立连接、验证主机、完成用户认证并维护加密会话 |
| `sshd` | SSH 服务端 | 接受连接，按服务端策略验证用户并创建会话 |
| 远端 Shell | SSH 服务端 | 启动交互式 Shell，或解释传给远端的命令 |

`ssh` 不会安装或启动 `sshd`，不会修改服务端防火墙，也不能代替 TCP 端口探测。连接能够建立，也不等于主机指纹和用户认证一定正确。

## 2. 记住一个基础骨架

```text
ssh [选项] [远程用户@]目标主机 [远程命令]
```

- **目标主机**可以是主机名、IP 地址或 `~/.ssh/config` 中的别名。
- **远程用户**可以写在 `@` 前，也可以由客户端配置中的 `User` 提供；两处都没有指定时，通常使用当前本地用户名。
- **远程命令**省略时，认证成功后进入交互式远程 Shell；提供时，只执行该命令，完成后返回本地。
- **选项**用于覆盖或补充端口、身份文件和客户端配置。不要背完整选项表，按实际问题查询即可。

默认目标端口通常是 22，但命令行 `-p` 或客户端配置中的 `Port` 可以改变它。网络测试和 `ssh` 必须使用同一个实际端口。

## 3. 怎样阅读 `ssh "$SSH_USER@$SSH_HOST"`

```bash
ssh "$SSH_USER@$SSH_HOST"
```

本地 Shell 会先展开双引号中的变量，例如把它变成一个 `linux-user@server.example.com` 参数。`@` 只是远程用户名与目标主机的分隔符；双引号保证展开结果作为一个整体交给 `ssh`。实际脚本仍应像 [[OpenSSH 连接、密钥与主机指纹]] 那样先拒绝空值和以连字符开头的目标，避免把输入误认成选项。

这条命令没有提供远程命令，因此成功后进入交互式远程 Shell。典型过程是：

1. 读取命令行和客户端配置，确定目标、端口、用户名及候选身份。
2. 建立到服务端 `sshd` 的 TCP 连接。
3. 验证服务端主机身份。
4. 使用服务端允许的方式验证用户身份。
5. 创建受远端 Linux 用户权限约束的 Shell。

这条命令不需要 `sudo`。它以当前本地用户身份读取该用户的 `~/.ssh/config`、`~/.ssh/known_hosts`、身份文件和 SSH Agent；SSH Agent 是可以在本地保存可用身份并代为签名的进程。首次接受主机公钥时可能更新本地 `known_hosts`；进入远端后执行的命令则会在远端产生相应影响。公钥认证只让客户端使用私钥完成签名，不会把私钥传给服务端。

在远端执行 `exit` 或按 `Ctrl-D` 可以结束远程 Shell，随后回到本地终端。操作前先观察 Shell 提示符、`hostnamectl --static` 和 `pwd`，不要把本地路径与远端路径混为一谈。

## 4. 交互式登录与单次远程命令

下面两种形式的目标相同，但成功结果不同：

```bash
ssh linux-host
ssh linux-host 'hostnamectl --static'
```

第一条进入交互式 Shell；第二条让远端 Shell 执行 `hostnamectl --static`，输出返回本地后连接结束。

本地 Shell 总是先解释整条命令。远程命令使用单引号时，其中的 `$HOME` 等变量不会在本地展开，而是原样交给远端 Shell：

```bash
ssh linux-host 'printf "remote_home=%s\n" "$HOME"'
```

如果改用双引号包围整段远程命令，本地 Shell 可能先展开其中的变量。除非确实需要把本地值传入远端，否则优先使用单引号明确边界；不要把未经校验的输入直接拼进远程命令字符串。

## 5. 标准输入、输出与退出状态

`ssh` 会把本地标准输入送入远端会话，并把远端标准输出和标准错误带回本地。因此下面的骨架表示“让远端命令读取本地文件内容”：

```text
ssh 目标主机 '远程命令' < 本地文件
```

重定向符 `<` 由本地 Shell 处理，本地文件不会在远端按同一路径打开；只有文件内容通过 SSH 会话进入远端命令的标准输入。[[OpenSSH 连接、密钥与主机指纹]] 使用这一边界把用户公钥送给远端 `cat`，并不会传输对应私钥。

| 通道 | 数据方向 |
| --- | --- |
| 标准输入 | 本地终端或文件 → `ssh` → 远端 Shell 或命令 |
| 标准输出 | 远端命令 → `ssh` → 本地标准输出 |
| 标准错误 | 远端命令 → `ssh` → 本地标准错误 |

执行单次远程命令时，`ssh` 通常返回远程命令的退出状态；连接、认证或协议发生错误时返回 255。因此自动化脚本必须检查退出状态，不能只凭是否出现输出判断成功。

## 6. 高频选项按问题记忆

| 我想解决的问题 | 常用骨架 | 作用 |
| --- | --- | --- |
| 连接非默认端口 | `ssh -p "$SSH_PORT" "$SSH_USER@$SSH_HOST"` | 用本次命令指定远端 SSH 端口 |
| 指定身份文件 | `ssh -i "$KEY_PATH" "$SSH_USER@$SSH_HOST"` | 选择用于公钥认证的身份文件 |
| 限制候选身份 | `ssh -o IdentitiesOnly=yes -i "$KEY_PATH" "$SSH_USER@$SSH_HOST"` | 避免 SSH Agent 中其他身份干扰 |
| 查看有效客户端配置 | `ssh -G linux-host` | 解析 `Host`、`Match` 和配置后输出结果，不建立 SSH 连接 |
| 增加调试信息 | `ssh -vvv linux-host` | 输出最详细的常规连接、配置和认证诊断信息 |

`-o` 用于临时提供一个 `ssh_config` 配置项。`IdentitiesOnly=yes` 表示只使用默认、配置文件或命令行明确指定的身份，而不是把 SSH Agent 提供的所有身份都作为候选。`-vvv` 是三个 `-v`，详细程度最高为三级；它会实际尝试连接，输出可能包含用户名、地址、配置路径和公钥指纹，分享前应脱敏。

## 7. 使用 `~/.ssh/config` 别名

频繁使用的目标可以写入当前客户端用户的 `~/.ssh/config`：

```sshconfig
Host linux-host
    HostName server.example.com
    User linux-user
    Port 22
    IdentityFile ~/.ssh/id_ed25519_linux_host
    IdentitiesOnly yes
```

`Host` 定义本地别名，`HostName` 才是实际目标；`User`、`Port` 和 `IdentityFile` 分别提供远程用户名、端口和身份文件。保存后先读取合并结果，再连接：

```bash
chmod 700 "$HOME/.ssh"
chmod 600 "$HOME/.ssh/config"
ssh -G linux-host | grep -E '^(hostname|user|port|identityfile|identitiesonly) '
ssh linux-host
```

`ssh -G` 解析并输出有效配置，不建立 SSH 连接，适合确认命令行、用户配置、系统配置、通配符与 `Include` 合并后的结果。不要仅凭肉眼阅读某一个 `Host` 块推断最终值。

## 8. 先判断影响范围

| 命令或动作 | 主要影响 |
| --- | --- |
| `ssh -G linux-host` | 解析并输出客户端有效配置，不建立 SSH 连接 |
| `ssh linux-host` | 发起网络连接；首次确认主机后可能修改本地 `known_hosts` |
| 交互式远程 Shell | 后续命令以远端 Linux 用户权限生效 |
| `ssh linux-host '远程命令'` | 直接在远端执行命令，应先判断其读写范围 |
| `ssh -vvv linux-host` | 发起连接并输出可能需要脱敏的调试信息 |

不要使用 `sudo ssh` 解决普通用户连接问题，否则读取的可能是 root 的客户端配置、`known_hosts` 和身份文件。不要使用 `StrictHostKeyChecking=no` 跳过身份核对，也不要把私钥内容拼进命令、标准输入或远程命令。

## 9. 按顺序排查和自助查询

遇到连接问题时，先区分层次：

1. 用 `ssh -G` 核对目标、用户名、端口和身份文件。
2. 用 [[TCP 端口连通性测试与 nc 命令基础|nc]] 检查当前客户端到目标端口的 TCP 路径。
3. 按 [[OpenSSH 连接、密钥与主机指纹]] 核对主机指纹、用户公钥和服务端策略。
4. 最后使用 `ssh -vvv` 观察客户端实际采用了哪些配置和认证步骤。

本机查询入口：

```bash
command -V ssh
ssh -V
man ssh
man ssh_config
```

`ssh -V` 显示客户端版本并退出。`man ssh` 负责命令行选项、连接流程和退出状态，`man ssh_config` 负责客户端配置项；具体行为以当前实际客户端为准。

## 10. 完成检查

- [ ] 能解释 `ssh [选项] [用户@]主机 [远程命令]` 的四个部分。
- [ ] 能区分交互式远程 Shell 与单次远程命令。
- [ ] 能判断变量、引号和重定向由本地还是远端 Shell 处理。
- [ ] 知道公钥认证不会把私钥发送给服务端。
- [ ] 会使用 `-p`、`-i`、`IdentitiesOnly`、`-G` 和 `-vvv` 解决对应问题。
- [ ] 会先查看有效配置和网络路径，再排查主机指纹与用户认证。

## 相关笔记

- [[OpenSSH 连接、密钥与主机指纹]]
- [[TCP 端口连通性测试与 nc 命令基础]]
- [[Linux 端口、监听套接字与 ss 命令基础]]
- [[使用 Tailscale 访问 Linux 主机]]
- [[Shell 命令结构、类型与帮助系统]]
- [[Shell 路径、变量、引用与展开]]
- [[Shell 标准流、管道、重定向与退出状态]]

## 官方参考资料

- [OpenBSD：`ssh(1)` 手册](https://man.openbsd.org/ssh.1)
- [OpenBSD：`ssh_config(5)` 手册](https://man.openbsd.org/ssh_config.5)
- [Ubuntu 24.04：`ssh(1)` 手册](https://manpages.ubuntu.com/manpages/noble/man1/ssh.1.html)
- [Ubuntu 24.04：`ssh_config(5)` 手册](https://manpages.ubuntu.com/manpages/noble/man5/ssh_config.5.html)

---
title: OpenSSH 常用命令基础
aliases:
  - SSH 客户端命令基础
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
updated: 2026-07-26T19:40:18
---

OpenSSH 不是一条命令，而是一组分工不同的程序。本篇围绕密钥登录与服务端排查主线，集中解释 `ssh`、`ssh-keygen`、`ssh-keyscan` 和 `sshd` 的常用方式、选项边界与影响范围。

完整的密钥登录顺序、安全判据和回滚流程见 [[OpenSSH 密钥登录、服务端配置与排查]]；网络、服务管理、防火墙、文件权限和 Shell 语法见第 6 节的分层导航。文件传输、SSH Agent 管理、端口转发、跳板机、连接复用和算法调整不在本篇范围内，遇到实际需求时再进入对应专题。

> [!abstract] 本篇掌握目标
> - **必须熟练**：能区分 `ssh`、`ssh-keygen`、`ssh-keyscan` 与 `sshd` 的职责及客户端、服务端配置边界；能读懂 `ssh [选项] [用户@]主机 [远程命令]`；能区分 `sshd -t` 与 `sshd -T`。
> - **理解会查**：能根据问题选择 `ssh` 的端口、身份、配置和调试选项；能先识别 `ssh-keygen` 的工作模式，再判断它是只读还是会修改文件；能区分配置解析与实际生效。
> - **认识即可**：`ssh-keyscan` 的批量收集与信任边界、`ssh-keygen -A` 的主机密钥恢复边界，以及 `sshd -T -C` 的连接条件模拟。

> [!info] 核对日期与适用范围
> 本文于 **2026-07-26** 核对 OpenBSD OpenSSH 的 `ssh(1)`、`ssh_config(5)`、`ssh-keygen(1)`、`ssh-keyscan(1)`、`sshd(8)` 与 `sshd_config(5)` 手册，并结合 Ubuntu 24.04 的 OpenSSH Server 运行方式编排。不同版本可能改变默认算法、身份文件和服务激活方式，应以实际主机上的版本、手册和有效配置为准。

## 1. 先按程序分清职责

| 程序 | 通常执行位置 | 主要回答的问题 | 主要影响 |
| --- | --- | --- | --- |
| `ssh` | 客户端 | 如何连接目标、验证主机、认证用户并创建会话 | 发起网络连接；可能更新客户端 `known_hosts`；远程命令会在服务端生效 |
| `ssh-keygen` | 客户端或服务端 | 如何创建、读取和管理密钥及主机记录 | 取决于工作模式；既可能只读，也可能创建或修改文件 |
| `ssh-keyscan` | 客户端 | 当前网络响应者提供了哪些 SSH 主机公钥 | 发起网络连接并输出公钥；不会自动建立信任 |
| `sshd` | 服务端 | 如何接受连接，以及服务端配置是否合法、最终如何解析 | `-t`、`-T` 只检查；直接启动或由 systemd 操作服务会改变运行状态 |

SSH 配置先按连接两端区分：

| 配置端 | 读取程序 | 主要原始配置 | 查看解析结果 |
| --- | --- | --- | --- |
| 客户端 | `ssh` | 当前用户的 `~/.ssh/config`、系统级 `/etc/ssh/ssh_config` 及它们 `Include` 的片段 | `ssh -G 目标主机` |
| 服务端 | `sshd` | `/etc/ssh/sshd_config` 及它 `Include` 的片段 | `sudo sshd -T`；有 `Match` 时按需要增加 `-C` 上下文 |

`~/.ssh/config` 用于控制当前本地用户如何向外连接；客户端 `known_hosts` 记录受信任的主机身份，服务端目标用户的 `authorized_keys` 记录允许登录的用户公钥。它们可能都出现在 `.ssh` 目录中，但不是同一类配置入口。

排查时还要区分三层：

```text
原始配置文件
  → ssh -G / sshd -T 的解析结果
  → 实际连接、监听状态和新会话验证
```

只阅读某个文件，看不到其他配置来源和内置默认值；只查看解析结果，也不能证明运行中的服务已重读配置或真实连接已经成功。

## 2. 使用 `ssh` 连接和执行远程命令

### 2.1 记住基础骨架

```text
ssh [选项] [远程用户@]目标主机 [远程命令]
```

- **目标主机**可以是主机名、IP 地址或 `~/.ssh/config` 中的别名。
- **远程用户**可以写在 `@` 前，也可以由客户端配置中的 `User` 提供；两处都没有指定时，通常使用当前本地用户名。
- **远程命令**省略时，认证成功后进入交互式远程 Shell；提供时，只执行该命令，完成后返回本地。
- **选项**用于覆盖或补充端口、身份文件和客户端配置。不要背完整选项表，按实际问题查询即可。

默认目标端口通常是 22，但命令行 `-p` 或客户端配置中的 `Port` 可以改变它。网络测试和 `ssh` 必须使用同一个实际端口。

### 2.2 交互式登录与单次远程命令

```bash
ssh linux-host
ssh linux-host 'hostnamectl --static'
```

第一条进入交互式 Shell；第二条让远端 Shell 执行 `hostnamectl --static`，输出返回本地后连接结束。

省略远程命令时，典型连接过程是：

1. 读取命令行和客户端配置，确定目标、端口、用户名及候选身份。
2. 建立到服务端 `sshd` 的 TCP 连接。
3. 验证服务端主机身份。
4. 使用服务端允许的方式验证用户身份。
5. 创建受远端 Linux 用户权限约束的 Shell。

普通用户连接不需要 `sudo`。`ssh` 会读取当前本地用户的 `~/.ssh/config`、`~/.ssh/known_hosts`、身份文件和 SSH Agent。首次接受主机公钥时可能更新本地 `known_hosts`；进入远端后执行的命令则以远端 Linux 用户权限生效。公钥认证只让客户端使用私钥完成签名，不会把私钥传给服务端。

在远端执行 `exit` 或按 `Ctrl-D` 可以结束远程 Shell。操作前先观察 Shell 提示符、`hostnamectl --static` 和 `pwd`，不要把本地路径与远端路径混为一谈。

### 2.3 本地 Shell、`ssh` 与远端 Shell 的边界

一条 `ssh` 命令通常会经过四个执行者：

| 执行者 | 所在位置 | 负责什么 |
| --- | --- | --- |
| 本地 Shell | SSH 客户端 | 先处理变量、引号、重定向和命令行参数 |
| `ssh` | SSH 客户端 | 建立连接、验证主机、完成用户认证并维护加密会话 |
| `sshd` | SSH 服务端 | 接受连接，按服务端策略验证用户并创建会话 |
| 远端 Shell | SSH 服务端 | 启动交互式 Shell，或解释传给远端的命令 |

先看带变量的交互式连接：

```bash
ssh "$SSH_USER@$SSH_HOST"
```

本地 Shell 会先展开双引号中的变量，例如把它变成一个 `linux-user@server.example.com` 参数。`@` 是远程用户名与目标主机的分隔符，双引号保证展开结果作为一个整体交给 `ssh`。脚本在调用前还应拒绝空值和以连字符开头的目标，避免把输入误认成选项。这条命令没有提供远程命令，因此成功后进入交互式远程 Shell。

本地 Shell 总是先解释整条命令。远程命令使用单引号时，其中的 `$HOME` 等变量不会在本地展开，而是原样交给远端 Shell：

```bash
ssh linux-host 'printf "remote_home=%s\n" "$HOME"'
```

如果改用双引号包围整段远程命令，本地 Shell 可能先展开其中的变量。除非确实需要把本地值传入远端，否则优先使用单引号明确边界；不要把未经校验的输入直接拼进远程命令字符串。

### 2.4 标准输入、输出与退出状态

`ssh` 会把本地标准输入送入远端会话，并把远端标准输出和标准错误带回本地：

```text
ssh 目标主机 '远程命令' < 本地文件
```

重定向符 `<` 由本地 Shell 处理，本地文件不会在远端按同一路径打开；只有文件内容通过 SSH 会话进入远端命令的标准输入。[[OpenSSH 密钥登录、服务端配置与排查]] 使用这一边界把用户公钥送给远端 `cat`，并不会传输对应私钥。

| 通道 | 数据方向 |
| --- | --- |
| 标准输入 | 本地终端或文件 → `ssh` → 远端 Shell 或命令 |
| 标准输出 | 远端命令 → `ssh` → 本地标准输出 |
| 标准错误 | 远端命令 → `ssh` → 本地标准错误 |

执行单次远程命令时，`ssh` 通常返回远程命令的退出状态；连接、认证或协议发生错误时返回 255。自动化脚本必须检查退出状态，不能只凭是否出现输出判断成功。

### 2.5 高频选项按问题记忆

| 我想解决的问题 | 常用骨架 | 作用 |
| --- | --- | --- |
| 连接非默认端口 | `ssh -p "$SSH_PORT" "$SSH_USER@$SSH_HOST"` | 用本次命令指定远端 SSH 端口 |
| 指定身份文件 | `ssh -i "$KEY_PATH" "$SSH_USER@$SSH_HOST"` | 选择用于公钥认证的身份文件 |
| 限制候选身份 | `ssh -o IdentitiesOnly=yes -i "$KEY_PATH" "$SSH_USER@$SSH_HOST"` | 避免 SSH Agent 中其他身份干扰 |
| 只尝试公钥认证 | `ssh -o PreferredAuthentications=publickey "$SSH_USER@$SSH_HOST"` | 不回退到键盘交互或密码认证 |
| 隔离验证指定密钥 | `ssh -F /dev/null -o IdentitiesOnly=yes -o PreferredAuthentications=publickey -i "$KEY_PATH" "$SSH_USER@$SSH_HOST"` | 不读取现有客户端配置，只用命令行提供的直连参数验证该密钥 |
| 查看有效客户端配置 | `ssh -G linux-host` | 解析 `Host`、`Match` 和配置后输出结果，不建立 SSH 连接 |
| 增加调试信息 | `ssh -vvv linux-host` | 实际尝试连接并输出最详细的常规诊断信息 |

`-o` 用于临时提供一个 `ssh_config` 配置项。`IdentitiesOnly=yes` 表示只使用默认、配置文件或命令行明确指定的身份，而不是把 SSH Agent 提供的所有身份都作为候选；它本身不禁止密码认证，也不会排除配置文件中的其他 `IdentityFile`。`PreferredAuthentications=publickey` 把本次客户端尝试范围限制为公钥认证，但不负责选择具体密钥。

`-F` 指定替代的用户级配置文件，并忽略系统级配置。`-F /dev/null` 在 macOS 和 Linux 上提供空配置，因此非默认端口必须显式指定，已有的跳板和代理设置也不会生效。它只用于隔离的直连验证。

`-vvv` 是三个 `-v`，常规详细程度最高为三级；输出可能包含用户名、地址、配置路径和公钥指纹，分享前应脱敏。

### 2.6 使用 `~/.ssh/config` 别名

`ssh` 按以下顺序取得客户端配置：

1. 当前 `ssh` 命令行上的选项。
2. 当前本地用户的 `~/.ssh/config`。
3. 系统级 `/etc/ssh/ssh_config`。
4. 尚未取值参数的 OpenSSH 内置默认值。

除手册明确说明的例外外，每个参数首次取得的值生效，不是后读取的文件覆盖前面的值。因此，用户配置中通常先写更具体的 `Host`，再在后面写 `Host *` 通用默认值。`IdentityFile` 等可以累积多个值的参数属于例外，所以最终仍应以 `ssh -G` 输出为准。

先查看当前两级原始配置：

```bash
less "$HOME/.ssh/config"
less /etc/ssh/ssh_config
```

如果第一条命令提示文件不存在，只表示当前用户尚未创建用户级客户端配置；系统级配置和内置默认值仍然存在。两个文件都可以使用 `Include` 加载其他片段；`less` 只能展示当前文件的原始内容，不会展开 `Include` 或补全默认值。

频繁使用的目标可以写入当前客户端用户的 `~/.ssh/config`：

```sshconfig
Host linux-host
    HostName server.example.com
    User linux-user
    Port 22
    IdentityFile ~/.ssh/id_ed25519_linux_host
    IdentitiesOnly yes
```

`Host` 定义本地别名，`HostName` 才是实际目标；`User`、`Port` 和 `IdentityFile` 分别提供远程用户名、端口和身份文件。保存后先读取完整解析结果或筛选关键字段，再连接：

```bash
chmod 700 "$HOME/.ssh"
chmod 600 "$HOME/.ssh/config"
ssh -G linux-host | less
ssh -G linux-host | grep -E '^(hostname|user|port|identityfile|identitiesonly) '
ssh linux-host
```

`ssh -G` 在计算 `Host`、`Match`、`Include` 和内置默认值后输出本次客户端的有效配置，然后退出，不建立 SSH 连接。参数既可以是 `Host` 别名，也可以是普通主机名，例如 `ssh -G github.com | less`。不要仅凭肉眼阅读某一个 `Host` 块推断最终值。

`ssh -G` 输出可能包含用户名、目标地址、本地路径和代理命令，对外分享前应先脱敏。它不建立 SSH 连接，但配置中的 `Match exec` 会在判断条件时通过当前用户的 Shell 执行指定本地命令；因此只对可信配置使用，不能把“不建立连接”理解为绝对没有本地影响。

### 2.7 先判断影响范围

| 命令或动作 | 主要影响 |
| --- | --- |
| `ssh -G linux-host` | 解析并输出客户端有效配置，不建立 SSH 连接；`Match exec` 可能执行本地命令 |
| `ssh linux-host` | 发起网络连接；首次确认主机后可能修改本地 `known_hosts` |
| 交互式远程 Shell | 后续命令以远端 Linux 用户权限生效 |
| `ssh linux-host '远程命令'` | 直接在远端执行命令，应先判断其读写范围 |
| `ssh -vvv linux-host` | 发起连接并输出可能需要脱敏的调试信息 |

不要使用 `sudo ssh` 解决普通用户连接问题，否则读取的可能是 root 的客户端配置、`known_hosts` 和身份文件。不要使用 `StrictHostKeyChecking=no` 跳过身份核对，也不要把私钥内容拼进命令、标准输入或远程命令。

### 2.8 按顺序排查

1. 用 `ssh -G` 核对目标、用户名、端口和身份文件。
2. 如果目标名称、地址或路由尚不明确，先按 [[Linux 网络接口、IP 地址、路由与 DNS 基础]] 核对。
3. 用 [[TCP 端口连通性测试与 nc 命令基础|nc]] 检查当前客户端到目标端口的 TCP 路径。
4. 按 [[OpenSSH 密钥登录、服务端配置与排查]] 核对主机指纹、用户公钥和服务端策略。
5. 最后使用 `ssh -vvv` 观察客户端实际采用了哪些配置和认证步骤。

## 3. 根据工作模式阅读 `ssh-keygen`

`ssh-keygen` 不只用于“生成密钥”。必须先看决定工作模式的选项，再解释后面的 `-f` 等参数；同一个 `-f` 在不同模式下可能表示输出密钥路径、输入密钥文件或 `known_hosts` 文件。

### 3.1 创建用户密钥

```text
ssh-keygen -t ed25519 -a 64 -f 密钥路径 -C 用途注释
```

| 部分 | 作用 |
| --- | --- |
| `-t ed25519` | 创建 Ed25519 类型的密钥对 |
| `-a 64` | 保存私钥时使用 64 轮口令派生；轮数越高，验证口令越慢，抵抗私钥被盗后的口令暴力猜测能力越强 |
| `-f 密钥路径` | 指定私钥输出路径；公钥写入同名 `.pub` 文件 |
| `-C 用途注释` | 写入便于识别用途的注释，不承担认证作用 |

创建模式会写入两个文件，不能把它当作只读检查。执行前应确认私钥和 `.pub` 路径都未占用，并为用户私钥设置口令；完整的防覆盖流程见 [[OpenSSH 密钥登录、服务端配置与排查#5. 创建独立的用户密钥]]。

### 3.2 读取公钥指纹

```bash
ssh-keygen -lf "$HOME/.ssh/id_ed25519_linux_host.pub"
```

上面是在客户端读取示例用户公钥。在服务端读取系统主机公钥时使用：

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

这里的工作模式由 `-l` 决定：读取指定密钥并显示指纹；`-f` 指定输入文件。读取 `.pub` 文件不会创建或修改密钥。典型输出包含密钥位数、`SHA256:...` 指纹、注释和密钥类型，人工核对时必须同时比较指纹和类型。

第二条命令在服务端读取系统主机公钥。主机公钥通常允许普通用户读取；只有实际权限不足时才在核对路径后增加 `sudo`。使用 `sudo` 读取系统公钥，与使用 `sudo ssh-keygen` 创建普通用户密钥不是一回事。

### 3.3 查询和移除 `known_hosts` 记录

```bash
ssh-keygen -F "$SSH_HOST"
ssh-keygen -R "$SSH_HOST"
```

这里假定 `SSH_HOST` 已保存经过核对的目标名称或地址。`-F` 在客户端默认的 `known_hosts` 中查找目标记录，是只读查询；它能够查找经过哈希处理的主机名。`-R` 会移除属于该目标的全部记录，是文件修改操作。非默认端口在 `known_hosts` 中通常使用 `[主机]:端口` 形式，查询和移除时也应使用相同目标形式。

只有通过可信通道确认主机密钥变化合理后，才应先备份 `known_hosts`、再执行 `-R`。不要通过删除整个文件来绕过单个目标的主机密钥警告；完整恢复流程见 [[OpenSSH 密钥登录、服务端配置与排查#10. 安全处理主机身份变化警告]]。

### 3.4 补齐缺失的默认主机密钥（服务端恢复）

```bash
sudo ssh-keygen -A
```

这里的工作模式由 `-A` 决定：在服务端为默认路径中尚不存在的默认密钥类型生成主机密钥。它与第 3.1 节创建客户端用户密钥不是同一用途，也不应在客户端执行。

`-A` 不覆盖已经存在的默认主机密钥，但每把新生成的密钥都会产生新的指纹；它无法找回已经丢失的原密钥，也不会处理 `sshd_config` 中指向其他位置的自定义 `HostKey`。对于已经被客户端信任的服务器，应优先恢复原主机密钥；只有新服务器或确认原密钥无法恢复并接受相应指纹变化时，才使用该命令。完整的诊断、服务启动、指纹核对和客户端恢复流程见 [[OpenSSH 密钥登录、服务端配置与排查#11.3.1 服务端缺少可用主机密钥]]。

## 4. 使用 `ssh-keyscan` 收集主机公钥（认识即可）

```text
ssh-keyscan [-T 超时秒数] [-p SSH端口] [-t 密钥类型] 目标主机
```

`ssh-keyscan` 会连接当前网络中的目标并收集其提供的 SSH 主机公钥，不需要登录权限。默认输出可以作为 `known_hosts` 记录的格式来源；`-T` 限制连接或读取等待时间，`-p` 指定目标端口，`-t` 限定要收集的密钥类型，例如 `ed25519`。

这条命令只把结果写到标准输出；只有显式重定向或通过其他命令写入文件时才会修改记录。它会实际产生网络连接，也支持多目标和网段输入，因此只应对有权访问、已经确认范围的目标使用。

> [!danger] 收集不等于验证
> `ssh-keyscan` 无法证明收到的公钥属于预期主机。攻击者若能拦截当前网络流量，也可能返回自己的公钥。结果必须通过控制台、云平台控制台、管理员记录或其他独立可信通道核对，不能未经验证就直接当作可信记录写入 `known_hosts`。

## 5. 使用 `sshd` 检查服务端配置

`sshd` 是服务端守护程序。日常服务启动、停止和 reload 应交给目标系统的服务管理器；这里只直接使用 `sshd` 的检查模式，不手工启动第二个守护进程。

### 5.1 区分 `sshd -t` 与 `sshd -T`

先阅读服务端主配置的原始内容：

```bash
less /etc/ssh/sshd_config
```

Ubuntu 的主配置默认可以通过 `Include /etc/ssh/sshd_config.d/*.conf` 加载模块化配置片段。因此，只阅读 `/etc/ssh/sshd_config` 不能代表最终结果；还应按文件中的 `Include` 路径核对已加载片段。主配置通常允许普通用户读取；只有实际权限不足时才在核对路径后使用 `sudo`。

再进入 `sshd` 的检查模式：

```bash
sudo sshd -t
sudo sshd -T | less
```

| 命令 | 回答的问题 | 输出与影响 |
| --- | --- | --- |
| `sshd -t` | 配置文件与主机密钥是否通过基本有效性检查 | 成功时通常无输出；只检查，不启动或 reload 服务 |
| `sshd -T` | 配置能否通过检查，并最终解析出哪些有效值 | 将有效配置写到标准输出；`less` 只负责分页；不启动或 reload 服务 |

`sshd -T` 输出的是当前文件在本次检查中的解析结果。如果文件刚被修改但运行中的服务尚未 reload，该输出可能与当前服务实际使用的配置不同。修改前后应先运行 `sshd -t`，再用 `sshd -T` 查看完整输出或筛选关键项，并按第 5.3 节完成 reload 和新会话验证。

```bash
sudo sshd -T | grep -E '^(port|addressfamily|listenaddress|pubkeyauthentication|authorizedkeysfile|strictmodes|passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

`authorizedkeysfile` 指出 `sshd` 会从哪些路径读取用户授权公钥，相对路径以目标用户家目录为基准；`strictmodes` 指出 `sshd` 是否在认证前检查用户文件与家目录的权限和归属。`grep` 只负责从已经生成的有效配置中筛选行，不改变 `sshd` 的解析结果。

### 5.2 `Match` 条件需要连接上下文

不带连接条件的 `sshd -T` 适合读取全局有效值；如果配置使用 `Match`，某个用户、来源地址或本地端口的最终结果可能不同。需要模拟具体连接时，可以使用扩展测试模式的 `-C` 提供上下文：

```text
sudo sshd -T -C user=用户,host=来源主机名,addr=来源地址,laddr=服务端地址,lport=服务端端口
```

这些值必须来自实际连接场景，不能为了让输出符合预期而随意编造。没有使用 `Match` 或尚未遇到条件配置时，认识这一能力即可。

### 5.3 检查配置不等于让配置生效

`sshd -t` 与 `sshd -T` 都不会让正在运行的服务重读配置，也不会更换 systemd 已经打开的监听套接字。执行生效操作前，必须先判断变更影响的是 `sshd` 进程行为，还是由 `ssh.socket` 管理的监听入口。

#### 认证与会话类配置

`PubkeyAuthentication`、`AuthorizedKeysFile`、`PasswordAuthentication` 等选项由 `sshd` 读取。在 `ssh.service` 已运行且支持 reload 时，安全变更顺序是：

```text
保留控制台和旧会话
  → 备份原配置
  → 写入候选配置
  → sshd -t 检查
  → sshd -T 核对有效值
  → systemctl reload ssh.service
  → 从新终端验证独立登录
```

`systemctl reload ssh.service` 属于 systemd 操作，不是 `sshd` 的命令行选项。reload 成功也只说明服务接受了重载请求，不能代替新会话验证。如果 `ssh.service` 尚未运行或实际单元不支持 reload，应先核对 `ssh.socket`、`ssh.service` 和日志，不要盲目改用 restart。

#### 监听端口与地址配置

Ubuntu 24.04 默认使用 systemd socket activation。`Port`、`AddressFamily` 和 `ListenAddress` 会由 systemd generator 转换为 `ssh.socket` 的监听配置；只重启或 reload `ssh.service` 不会让新入口生效。

变更前先核对实际激活方式：

```bash
systemctl status ssh.socket ssh.service --no-pager
systemctl cat ssh.socket
```

只有确认当前由 `ssh.socket` 管理监听时，才按以下边界处理：

```text
保留控制台和旧会话
  → 先准备并允许新的防火墙管理路径
  → 备份并写入候选配置
  → sshd -t 检查
  → sshd -T 核对目标端口和地址
  → systemctl daemon-reload
  → systemctl restart ssh.socket
  → 核对单元状态与实际监听套接字
  → 从新终端验证新入口
```

`systemctl daemon-reload` 会重新运行 generator，`systemctl restart ssh.socket` 才会重建监听套接字。之后应同时检查 `systemctl status ssh.socket ssh.service`、[[Linux 端口、监听套接字与 ss 命令基础|`ss` 实际监听]]、[[Linux 主机防火墙与 UFW 基础|UFW 规则]]与独立新连接。如果同一次还修改了认证类配置，并且 `ssh.service` 已运行，还要完成前一分支的 reload 和新会话验证。非 socket activation 系统应以实际服务管理方式为准，不要机械重启不存在或未激活的 `ssh.socket`。

### 5.4 `sshd -t` 提示缺少 `/run/sshd`

Ubuntu 的 `ssh.service` 可以通过 `RuntimeDirectory=sshd` 让 systemd 在启动服务时创建 `/run/sshd`。若服务尚未启动就直接运行 `sudo sshd -t`，可能看到 `Missing privilege separation directory: /run/sshd`。

这表示手工检查绕过了 systemd 准备运行时目录的步骤，不足以证明 `sshd_config` 存在语法错误。应先查看 `ssh.socket`、`ssh.service`、unit 定义和日志，再通过 systemd 启动服务；不要把手工创建运行时目录或自行添加 tmpfiles 规则当作未经核对的持久修复。

## 6. 不属于 OpenSSH 命令族的相邻工具

| 要回答的问题 | 对应工具与笔记 |
| --- | --- |
| Ubuntu 是否已安装 OpenSSH 软件包 | [[APT 软件包管理基础]] |
| 目标名称、IP 地址、路由或 DNS 是否正确 | [[Linux 网络接口、IP 地址、路由与 DNS 基础]] |
| 服务端是否存在监听套接字 | [[Linux 端口、监听套接字与 ss 命令基础]] |
| 当前客户端能否连接目标 TCP 端口 | [[TCP 端口连通性测试与 nc 命令基础]] |
| SSH unit 当前状态、reload 和日志 | [[systemd 服务与日志基础]] |
| UFW 是否允许实际 SSH 端口 | [[Linux 主机防火墙与 UFW 基础]] |
| `chmod`、文件所有者与目录权限 | [[Linux 用户、用户组、sudo 与文件权限]] |
| 变量、引号、重定向、条件、函数和 `trap` | [[Shell 路径、变量、引用与展开]]、[[Shell 标准流、管道、重定向与退出状态]]、[[Shell 脚本阅读基础]] |

不要因为这些命令出现在 SSH 操作流程中，就把它们误认为 OpenSSH 自身的选项或配置层。

## 7. 查询本机版本与手册

在 SSH 客户端查询客户端程序：

```bash
command -V ssh
command -V ssh-keygen
command -V ssh-keyscan
ssh -V
man ssh
man ssh_config
man ssh-keygen
man ssh-keyscan
```

在 SSH 服务端查询守护程序和配置：

```bash
command -V sshd
man sshd
man sshd_config
```

`ssh -V` 显示客户端版本并退出。客户端使用本机的 `man ssh` 与 `man ssh_config`；服务端配置应以目标服务器的 `man sshd`、`man sshd_config` 和实际 systemd 单元为准。

## 完成检查

### 必须熟练

- [ ] 能按执行位置和职责区分 `ssh`、`ssh-keygen`、`ssh-keyscan` 与 `sshd`。
- [ ] 能区分当前用户的 `~/.ssh/config`、系统级 `/etc/ssh/ssh_config` 与服务端 `/etc/ssh/sshd_config`，并知道用户级配置不存在不等于 SSH 没有配置。
- [ ] 能区分阅读原始配置、查看 `ssh -G` / `sshd -T` 解析结果与验证实际连接或服务状态。
- [ ] 能解释 `ssh [选项] [用户@]主机 [远程命令]` 的四个部分。
- [ ] 能判断远程命令中的变量、引号和重定向由本地还是远端 Shell 处理。
- [ ] 能区分 `sshd -t`、`sshd -T`、配置生效操作与独立新会话验证。

### 理解会查

- [ ] 会使用 `-p`、`-i`、`IdentitiesOnly`、`-G` 和 `-vvv` 解决对应问题。
- [ ] 能区分“选择哪些密钥”与“允许哪种认证方式”，并知道 `-F /dev/null` 只用于隔离的直连验证。
- [ ] 能先识别 `ssh-keygen` 的工作模式，再判断它是只读还是会写文件。
- [ ] 能区分认证类配置的 `ssh.service` reload，与 Ubuntu 24.04 socket activation 下监听配置的 generator、`ssh.socket` 和实际监听验证。
- [ ] 会把网络、端口、systemd、UFW、权限和 Shell 语法问题送回对应专题排查。

### 认识即可

- [ ] 知道 `ssh-keygen -A` 只补齐缺失的默认主机密钥，不能恢复既有主机身份或自定义路径中的密钥。
- [ ] 知道 `ssh-keyscan` 只能收集主机公钥，不能独立验证主机身份。
- [ ] 知道只有提供实际连接上下文时，`sshd -T -C` 才能用来模拟对应的 `Match` 条件。

## 相关笔记

- [[OpenSSH 密钥登录、服务端配置与排查]]
- [[APT 软件包管理基础]]
- [[Linux 网络接口、IP 地址、路由与 DNS 基础]]
- [[TCP 端口连通性测试与 nc 命令基础]]
- [[Linux 端口、监听套接字与 ss 命令基础]]
- [[Linux 主机防火墙与 UFW 基础]]
- [[systemd 服务与日志基础]]
- [[使用 Tailscale 访问 Linux 主机]]
- [[Shell 命令结构、类型与帮助系统]]
- [[Shell 路径、变量、引用与展开]]
- [[Shell 标准流、管道、重定向与退出状态]]
- [[Shell 脚本阅读基础]]

## 官方参考资料

- [OpenBSD：`ssh(1)` 手册](https://man.openbsd.org/ssh.1)
- [OpenBSD：`ssh_config(5)` 手册](https://man.openbsd.org/ssh_config.5)
- [OpenBSD：`ssh-keygen(1)` 手册](https://man.openbsd.org/ssh-keygen.1)
- [OpenBSD：`ssh-keyscan(1)` 手册](https://man.openbsd.org/ssh-keyscan.1)
- [OpenBSD：`sshd(8)` 手册](https://man.openbsd.org/sshd.8)
- [OpenBSD：`sshd_config(5)` 手册](https://man.openbsd.org/sshd_config.5)
- [Ubuntu Server：OpenSSH Server](https://documentation.ubuntu.com/server/how-to/security/openssh-server/)
- [Ubuntu 24.04 LTS 发布说明：OpenSSH 的 systemd socket activation](https://documentation.ubuntu.com/release-notes/24.04/#openssh)
- [Ubuntu Launchpad：修改 `Port` 等配置后重新生成并重启 `ssh.socket`](https://bugs.launchpad.net/bugs/2069041)
- [Ubuntu 24.04：`ssh(1)` 手册](https://manpages.ubuntu.com/manpages/noble/man1/ssh.1.html)
- [Ubuntu 24.04：`ssh_config(5)` 手册](https://manpages.ubuntu.com/manpages/noble/man5/ssh_config.5.html)
- [Ubuntu 24.04：`ssh-keygen(1)` 手册](https://manpages.ubuntu.com/manpages/noble/man1/ssh-keygen.1.html)
- [Ubuntu 24.04：`sshd(8)` 手册](https://manpages.ubuntu.com/manpages/noble/man8/sshd.8.html)

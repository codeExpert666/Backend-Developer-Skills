---
title: Linux 网络接口、IP 地址、路由与 DNS 基础
aliases:
  - Linux 主机网络基础
  - Linux 网络配置基础
  - Linux 网卡与 IP 地址基础
tags:
  - Linux
  - Linux/网络
  - 网络/基础
  - TCP/IP
created: 2026-07-18T19:52:31
updated: 2026-08-23T15:20:43
---

本文从一台 Linux 主机的视角，解释网络接口、MAC 地址、IP 地址与子网前缀、DHCP、默认路由和 DNS 之间的关系。目标是能读懂 Ubuntu Server 安装器与常用验证命令，并在修改配置前先定位出问题所在层。

本文前半部分先读取 Linux 当前正在使用的网络状态，最后再介绍 Ubuntu 如何通过 Netplan 保存并应用下次启动仍要使用的网络配置。“当前状态”与“期望配置”是两件不同的事。

操作示例以 Ubuntu Server 开发主机常见的 DHCPv4 和 IPv4 为主，同时补充读取命令中最常见的 IPv6 地址与路由。本文不展开完整的 IPv6 地址规划、自动配置或静态配置流程。

虚拟机如何接入宿主或局域网，见 [[虚拟机网络模式与可达性]]；安装器中的具体选择，见 [[使用 UTM 创建 Ubuntu Server 虚拟机]]；应用如何绑定端口并接受连接，见 [[Linux 端口、监听套接字与 ss 命令基础]]；数据到达 Linux 主机后如何受入站规则控制，见 [[Linux 主机防火墙与 UFW 基础]]。

> [!abstract] 本篇掌握目标
> - **必须熟练**：按接口、地址、路由、DNS 和应用连接分层读取现状，能看懂 `ip -brief address`、IPv4 路由与基本 DNS 查询结果。
> - **理解会查**：区分 DHCP 与静态配置、常见 IPv4 与 IPv6 地址、当前生效状态与 Netplan 期望配置，修改前保留控制台恢复路径。
> - **认识即可**：bond、不同网络后端、IPv6 自动配置和复杂 Netplan 配置；没有明确需求时不必记忆配置语法。
>
> 命令行拆解、选项与参数查询见 [[Linux 命令行学习路线与命令地图]] 和 [[Shell 命令结构、类型与帮助系统]]；组合验证代码块中的标准流与退出状态见 [[Shell 标准流、管道、重定向与退出状态]]。

> [!info] 资料核对日期
> 本文于 **2026-07-26** 核对 systemd 网络接口与 DNS 资料、IANA IPv6 地址登记、Netplan、curl 和 UTM 官方资料。网卡名、Netplan 后端、DNS 解析服务和系统预装组件可能随 Ubuntu 版本与安装方式不同，应以目标主机的实际输出为准。

## 完成标准

完成本文后，应能够：

- 区分 UTM 网络模式、Linux 网络接口和应用服务三个层次。
- 读懂 `enp0s1`、`192.168.64.2/24`、`::1/128`、`fe80::`、MAC 地址和 `default via` 一类字段。
- 解释 DHCP 与静态配置的适用边界。
- 分层验证接口、地址、路由、DNS 和 HTTPS。
- 区分网络可达、端口监听、主机防火墙允许和应用认证。
- 解释 Netplan、网络后端、Linux 内核、DNS 解析服务和查询命令之间的关系，并在修改配置前保留控制台恢复路径。

## 1. 先区分不同层次

一次网络访问依赖多个层次。某一层正常，不代表后续各层也一定正常。

| 层次 | 常见对象 | 要回答的问题 |
| --- | --- | --- |
| 虚拟化网络 | UTM Shared Network、Bridged、虚拟网卡 | 虚拟网卡被接到哪个网络？ |
| Linux 网络接口 | `enp0s1`、`ens3`、`eth0` | 内核是否识别到网卡，接口是否启用？ |
| 链路层标识 | MAC 地址 | 当前网卡在本地链路上如何被识别？ |
| 网络层地址 | IPv4、IPv6、子网前缀 | 主机使用什么地址，哪些目标在本地子网？ |
| 路由 | 直连路由、默认路由、网关 | 数据包应从哪个接口发出，下一跳是谁？ |
| 名称解析 | DNS 服务器、搜索域 | 域名如何转换成 IP 地址？ |
| 应用与访问控制 | SSH、HTTPS、监听端口、[[Linux 主机防火墙与 UFW 基础\|主机防火墙]] | 目标服务是否存在，连接是否被允许？ |

例如，Ubuntu 安装器显示 `DHCPv4` 地址，只能证明已识别接口并取得了地址。这不能单独证明默认路由、DNS、HTTPS 或 SSH 已经可用。

## 2. 网络接口、来源与名称

Linux 将实体网卡、虚拟网卡和软件创建的逻辑网络设备统一表示为网络接口。常见名称包括：

- `lo`：回环接口，用于本机通信，不是对外网卡。
- `enp0s1`、`ens3`：systemd/udev 根据设备类型与位置生成的可预测以太网接口名。`en` 表示以太网，后续部分与设备路径或插槽信息有关。
- `eth0`：传统命名，某些镜像、虚拟环境或自定义配置仍可能使用。
- `tailscale0`、`docker0`：软件创建的虚拟接口。
- `bond0`：将多个接口组合后形成的逻辑接口，名称可以由配置决定。

接口名不能单独说明它背后是实体硬件还是虚拟设备。例如，UTM 会向 Linux 客户机呈现 VirtIO 虚拟网卡，Ubuntu 安装器可能将它显示为 `Virtio network device`，Linux 仍会为它创建 `enp0s1` 一类普通网络接口。具体虚拟机接入方式由 UTM 决定，见 [[虚拟机网络模式与可达性]]。

bond 是将多个网络接口组成一个逻辑接口的机制，可用于冗余或特定的流量分配策略。它需要多块接口、明确的 bond 模式，有时还需要上游网络配合。只有一块虚拟网卡的开发虚拟机不应为了完成安装而创建 bond。

不要假设所有主机都使用 `enp0s1`。先读取当前系统的实际名称。

**执行位置：Linux 主机（任意目录）**

```bash
ip -brief link
ip -brief address
```

`ip -brief link` 用于确认接口是否存在并读取当前运行状态；`ip -brief address` 在此基础上列出已分配的 IPv4 和 IPv6 地址。`UP` 表示接口当前处于已启动状态，不等于已经获得地址、具备可用路由或能够访问互联网。

## 3. MAC 地址、IP 地址与主机名不是一回事

这三类标识解决不同问题。

| 标识 | 用途 | 是否可能变化 |
| --- | --- | --- |
| MAC 地址 | 识别本地链路上的网络接口 | 替换或重建虚拟网卡、手动设置时可变 |
| IP 地址 | 让主机在 IP 网络中发送和接收数据 | DHCP 续租、切换网络或修改配置时可变 |
| 主机名 | 便于人和系统识别主机 | 可修改；不保证在其他设备上自动解析 |

MAC 地址不能直接代替 IP 地址用于 SSH。主机名也只有在 DNS、mDNS、`/etc/hosts` 或其他名称服务能解析它时，才能作为网络目标。

## 4. IP 地址、子网前缀与常见 IPv6 输出

先用 IPv4 示例理解地址和前缀。`192.168.64.2/24` 由两部分组成：

- `192.168.64.2`：分配给接口的 IPv4 地址。
- `/24`：子网前缀长度，表示前 24 位用于识别网络。在这个示例中，直连子网通常表示为 `192.168.64.0/24`。

这是用于学习字段含义的示例，不是要在所有 UTM 虚拟机中写死的配置。两台主机的地址看似位于同一子网，也不代表中间一定没有虚拟化隔离、防火墙或其他访问限制。

`ip -brief address` 还可能在同一个接口上显示一个或多个 IPv6 地址。初次阅读时先识别：

- `127.0.0.1/8` 和 `::1/128`：分别是 IPv4 与 IPv6 回环地址，只用于本机通信。
- `fe80::` 开头的地址：属于 `fe80::/10` IPv6 链路本地地址范围，只在所在链路内使用。接口输出中常见 `/64` 前缀长度；仅看到链路本地地址不能证明主机已经具备 IPv6 互联网访问能力。

IPv4 与 IPv6 都使用前缀长度表示网络范围，但地址分配和自动配置机制并不完全相同。本文只要求能识别上述常见输出。

## 5. 网络参数从哪里来：DHCP 与静态配置

接口使用的地址、子网前缀、默认路由和 DNS 信息需要来自自动配置或人工配置。本文以 Ubuntu 安装器常见的 DHCPv4 为主。

DHCPv4 通常会向客户端提供：

- IP 地址和子网前缀。
- 默认路由或网关。
- DNS 服务器与可选的搜索域。
- 租约期限与其他网络参数。

IPv6 还可能通过 SLAAC、DHCPv6 或静态配置取得地址和其他参数。看到 IPv6 地址时不应直接套用 DHCPv4 的判断，但本文不展开这些机制的配置方法。

对 UTM 的普通开发虚拟机，先用 DHCPv4 完成安装、系统更新和宿主到客户机的 SSH 验证。DHCP 地址可能在重建网卡、网络环境变化或租约变化后不同，因此不应把一次安装器输出当作永久地址。

只有在下列信息明确时，才评估静态地址：

- 所在网络允许使用的地址范围。
- 子网前缀、默认网关和 DNS。
- 所选地址不会与 DHCP 池或其他设备冲突。
- 固定地址确实能解决具体的访问或服务发现需求。

在客户机内设置静态地址，不会自动改变 UTM Shared Network、桥接或端口转发的可达性边界。对外访问需要同时核对虚拟化网络模式、目标服务和防火墙。

## 6. 默认路由与网关

路由表告诉 Linux 如何为目标地址选择出口。

**执行位置：Linux 主机（任意目录）**

```bash
ip route
ip -6 route
```

`ip route` 默认读取 IPv4 路由；`ip -6 route` 读取 IPv6 路由。下面继续使用 IPv4 输出解释共同的路由判断方法。

下列输出只是阅读示例：

```text
default via 192.168.64.1 dev enp0s1
192.168.64.0/24 dev enp0s1 proto kernel scope link src 192.168.64.2
```

- `default`：没有更具体路由可匹配时使用的默认路由。
- `via 192.168.64.1`：示例中的下一跳网关，实际地址以当前输出为准。
- `dev enp0s1`：数据包从该接口发出。
- `scope link`：该子网与接口直接相连，不需要先经过其他网关。

主机有 IP 地址但没有可用的默认路由时，仍可访问直连子网以及已有更具体路由覆盖的网络；其他没有路由匹配的目标无法正常访问。

## 7. DNS 名称解析

DNS 将 `archive.ubuntu.com` 一类域名解析为 IP 地址。它与路由是不同的层次：可以出现“能访问某个 IP，但不能解析域名”的情况。

在常见 Ubuntu 系统中，网络后端可以从 DHCP 或静态配置取得 DNS 服务器等信息，再将这些信息交给系统使用的名称解析服务。`systemd-resolved` 是常见的 DNS 解析服务；它与名称相似的 `systemd-networkd` 职责不同：前者处理名称解析，后者可以作为网络后端管理接口、地址和路由。

**执行位置：Ubuntu Server（任意目录）**

```bash
resolvectl status
getent ahosts archive.ubuntu.com
```

- `resolvectl status` 用于查询 `systemd-resolved` 当前生效的全局及各接口 DNS 设置。
- `getent ahosts` 通过系统实际的名称服务配置做解析，比只查某个配置文件更接近应用的行为。

某些最小化镜像可能没有运行 `systemd-resolved`，因而不能使用 `resolvectl`。这不等于 DNS 一定异常：仍可使用 `getent ahosts` 验证应用实际使用的名称服务路径，再识别系统采用的解析组件。不要为了让示例命令存在而安装或启动不属于当前系统架构的服务。

## 8. 一组只读的分层验证

先使用只读命令收集现状，再决定是否修改配置。

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
printf '%s\n' '--- interfaces ---'
ip -brief link
ip -brief address

printf '%s\n' '--- routes ---'
ip route
ip -6 route

printf '%s\n' '--- DNS ---'
resolvectl status
getent ahosts archive.ubuntu.com

printf '%s\n' '--- HTTPS ---'
curl -I --max-time 15 https://archive.ubuntu.com/
```

按以下顺序判断：

1. 是否存在预期接口。
2. 接口是否启用并拥有预期的 IPv4 或 IPv6 地址。
3. 是否有相应地址族的直连路由；目标不在直连子网时，是否存在匹配的具体路由或默认路由。
4. DNS 服务器是否已配置，域名是否能解析。
5. HTTPS 是否能建立连接并完成证书校验。

`curl -I` 能收到 HTTP 响应即可证明网络路径基本成立，状态码不一定必须是 `200`。单独使用 `ping` 不能覆盖 DNS、TLS 或目标服务，而且中间网络可能禁止 ICMP，因此不应把 ping 失败直接等同于断网。

`curl` 会读取 `http_proxy`、`HTTPS_PROXY` 和 `ALL_PROXY` 等代理环境变量。命令成功说明当前应用访问路径可用，但这条路径可能经过代理；如果目标是验证直接访问，还需要单独核对当前代理配置。

这里的“出站代理”是指 `curl` 等客户端进程先连接代理服务，再由代理转发对外请求。代理不会让“完成一次 HTTPS 请求”反过来证明原始直连路由可用；当前 Shell、`sudo` 启动的命令和 systemd 后台服务也可能读取不同配置。如需做直连与代理对照，再进入 [[Linux 开发环境出站代理配置与分层排查]]；本篇继续只负责接口、地址、路由和 DNS 基础。

## 9. Ubuntu 如何保存并应用网络配置：Netplan

前面使用的 `ip -brief address`、`ip route` 和 `resolvectl status` 主要用于查看当前已经生效的网络状态。但 Ubuntu 还需要描述：下次开机时，某块接口应该继续使用 DHCP，还是应该配置静态地址、路由和 DNS。

**Netplan 是 Ubuntu 用来描述并应用这些“期望网络配置”的抽象配置层。** 持久配置通常保存在 YAML 文件中。Ubuntu Server 安装器中选择 DHCP 后，安装器一般会生成对应的 Netplan 配置，让已安装系统重启后仍会为该接口请求 DHCP 地址。

### 9.1 期望配置与当前状态不是一回事

期望配置描述接口“应该怎样配置”，当前状态描述接口“此刻实际怎样运行”。配置尚未应用、网络后端应用失败，或者 DHCP 在运行时提供了具体租约时，两者可能并不完全相同。

因此，排查时需要对比 Netplan 当前读取到的期望配置与 `ip`、`resolvectl` 等命令显示的实际状态，而不是只看某一边。

### 9.2 Netplan、网络后端与其他组件的关系

这里的“网络后端”不是 Web 开发中的后端服务，而是 Linux 中实际负责管理网络接口的软件组件。Netplan 读取并合并 YAML 期望配置，再将它转换成网络后端能够理解的配置；Netplan 本身不作为持续接管接口的网络管理服务。

常见网络后端包括 `systemd-networkd` 和 `NetworkManager`。它们负责启用接口、请求 DHCP，并应用地址、路由等配置。Netplan 中的 `renderer` 用来选择由哪个后端处理配置，可以针对整个配置、某类设备或具体设备指定。Ubuntu Server 常见 `systemd-networkd`，桌面系统常见 `NetworkManager`，但应以目标主机的实际配置为准。

一台主机可以由不同后端分别管理不同接口，这本身不一定错误；需要避免的是多个网络管理组件同时接管同一个接口。

可以把配置从文件到实际生效的过程理解为：

```text
Netplan YAML 期望配置
→ Netplan 合并并生成后端配置
→ systemd-networkd 或 NetworkManager 管理接口
→ Linux 内核维护接口、地址和路由等运行时状态
→ DNS 解析服务维护名称解析状态
→ ip、resolvectl 和 getent 从不同位置读取结果
```

| 对象 | 负责什么 | 不负责什么 |
| --- | --- | --- |
| UTM | 向客户机提供虚拟网卡，并选择 Shared Network 或 Bridged 等接入方式 | 不编辑 Ubuntu 内部的 Netplan 配置 |
| Netplan | 合并 YAML 中的 DHCP、静态地址、路由和 DNS 等期望配置，并为所选网络后端生成配置 | 不持续接管接口，也不是网卡、DHCP 服务器、路由器或 DNS 服务器 |
| `systemd-networkd` 或 `NetworkManager` | 作为网络后端管理接口，将地址、路由等配置应用到系统，并按各自机制提供 DNS 设置 | 不决定 UTM 使用哪种网络模式 |
| Linux 内核 | 维护当前接口、地址和路由等运行时状态 | 不保存下次启动的 Netplan 期望配置，也不负责 DNS 名称解析 |
| `systemd-resolved`（系统使用时） | 接收 DNS 设置并提供名称解析、缓存以及查询状态 | 不管理接口的 IP 地址或内核路由 |
| `ip`、`resolvectl`、`getent` | 分别读取内核网络状态、`systemd-resolved` 状态或系统名称服务结果 | 这些查询用法不会保存下次启动要使用的配置 |

### 9.3 先读取，不要立即修改

先查看最常见的持久配置目录和 Netplan 合并后的期望配置：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
ls -l /etc/netplan
sudo netplan get
```

这两条命令只用于读取。`/etc/netplan` 通常保存安装器或管理员写入的持久配置，但 Netplan 还会按规则读取并合并 `/lib/netplan`、`/etc/netplan` 和 `/run/netplan` 中的 YAML；`/run` 下的内容不保证重启后仍存在。

因此，`sudo netplan get` 回答的是“Netplan 当前读取到的合并期望配置”，不是“某一个 YAML 文件写了什么”，也不是“当前接口实际上怎样运行”。常见 YAML 文件名可能由安装器或 cloud-init 生成，不要假设所有 Ubuntu 主机使用同一个文件名。

### 9.4 Netplan 能管理的范围

Netplan 可以描述 DHCP、静态地址、路由、DNS、bond 和桥接等 Ubuntu 客户机内的配置，但它不能：

- 为 UTM 选择 Shared Network 或 Bridged 模式。
- 创建或修复 UTM 中本来就不存在的虚拟网卡。
- 代替上游网络中的 DHCP 服务器、网关或 DNS 服务器。
- 自动开放 SSH 端口或修改 Linux 防火墙。

这就是本节所说的“边界”：Netplan 只负责描述 Ubuntu 客户机内应该如何配置网络，不负责整条网络路径上的其他系统。

### 9.5 修改配置时保留回退路径

修改 `/etc/netplan/*.yaml` 后应用配置，可能立即中断网络。操作前应保留 UTM 控制台或其他带外恢复入口；远程主机优先使用带超时回退的 `sudo netplan try`，不要在唯一的 SSH 会话中盲目运行 `netplan apply`。

本文不提供一组可直接照抄的静态地址、bond 或桥接配置。这些配置依赖实际网卡名、地址规划、默认网关、DNS 和上游网络，应在需求明确后单独设计和验证。

## 按现象定位问题

| 现象 | 优先检查 | 不应立即做什么 |
| --- | --- | --- |
| 没有预期网络接口 | UTM 虚拟网卡是否启用，Linux 是否识别设备 | 不要先填静态 IP |
| 接口存在但没有预期地址 | 接口状态、DHCP 和网络模式 | 不要猜测网关或 DNS |
| 只看到 `fe80::` 链路本地地址 | IPv6 自动配置、地址作用域和 IPv6 路由 | 不要据此断定 IPv6 互联网可用 |
| 目标不在直连子网且没有匹配路由 | 路由表、DHCP 租约、Netplan 与网络后端 | 不要将 DNS 当成根因 |
| Netplan 期望配置与 `ip` 当前状态不一致 | 合并后的配置、`renderer`、网络后端状态与日志 | 不要同时修改多个 YAML 或反复盲目运行 `netplan apply` |
| 能访问 IP 但不能解析域名 | `resolvectl status`、`getent ahosts` | 不要关闭 TLS 校验 |
| DNS 正常但 HTTPS 失败 | 系统时间、代理、防火墙和目标服务 | 不要随意替换为不明软件源 |
| 客户机能出网，宿主或局域网设备不能连入 | UTM 网络模式、服务监听地址和防火墙 | 不要只通过改静态 IP 解决 |

需要检查网络后端的服务状态和日志时，见 [[systemd 服务与日志基础]]。

## 后续阅读

- 创建与安装虚拟机：[[使用 UTM 创建 Ubuntu Server 虚拟机]]
- 选择虚拟机网络模式：[[虚拟机网络模式与可达性]]
- 完成新系统基线：[[Ubuntu Server 初始化与基础安全]]
- 读取端口与监听状态：[[Linux 端口、监听套接字与 ss 命令基础]]
- 从实际客户端测试 TCP 端口：[[TCP 端口连通性测试与 nc 命令基础]]
- 理解 SSH 连接并验证 OpenSSH 服务与登录：[[OpenSSH 密钥登录、服务端配置与排查]]
- 理解主机入站规则：[[Linux 主机防火墙与 UFW 基础]]
- 验证开发工具直连与出站代理：[[Linux 开发环境出站代理配置与分层排查]]
- 跨局域网扩展访问：[[使用 Tailscale 访问 Linux 主机]]

## 官方参考资料

以下资料于 **2026-07-26** 核对：

- [systemd：网络设备命名方案](https://www.freedesktop.org/software/systemd/man/latest/systemd.net-naming-scheme.html)
- [systemd：resolvectl](https://www.freedesktop.org/software/systemd/man/latest/resolvectl.html)
- [IANA：IPv6 特殊用途地址登记表](https://www.iana.org/assignments/iana-ipv6-special-registry/iana-ipv6-special-registry.xhtml)
- [Netplan：配置结构、设备与网络后端](https://netplan.readthedocs.io/en/stable/structure-id/)
- [Netplan：命令行接口](https://netplan.readthedocs.io/en/stable/cli/)
- [Netplan：DHCP、静态地址、DNS 与 bond 示例](https://netplan.readthedocs.io/en/stable/examples/)
- [Netplan：安全试用配置](https://netplan.readthedocs.io/en/stable/netplan-try/)
- [curl：命令行手册](https://curl.se/docs/manpage.html)
- [UTM：Apple 虚拟机网络设置](https://docs.getutm.app/settings-apple/devices/network/)
- [UTM：QEMU 虚拟机网络设置](https://docs.getutm.app/settings-qemu/devices/network/network/)

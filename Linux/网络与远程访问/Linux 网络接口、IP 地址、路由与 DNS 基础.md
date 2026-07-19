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
updated: 2026-07-20T00:49:15
---

本文从一台 Linux 主机的视角，解释网络接口、MAC 地址、IP 地址与子网前缀、DHCP、默认路由和 DNS 之间的关系。目标是能读懂 Ubuntu Server 安装器与常用验证命令，并在修改配置前先定位出问题所在层。

本文前半部分先读取 Linux 当前正在使用的网络状态，最后再介绍 Ubuntu 如何通过 Netplan 保存下次启动仍要使用的网络配置。“当前状态”与“持久配置”是两件不同的事。

虚拟机如何接入宿主或局域网，见 [[虚拟机网络模式与可达性]]；安装器中的具体选择，见 [[使用 UTM 创建 Ubuntu Server 虚拟机]]。

> [!abstract] 本篇掌握目标
> - **必须熟练**：按接口、地址、路由、DNS 和应用连接分层读取现状，能看懂 `ip -brief address`、`ip route` 与基本 DNS 查询结果。
> - **理解会查**：区分 DHCP 与静态地址、当前生效状态与 Netplan 持久配置，修改前保留控制台恢复路径。
> - **认识即可**：bond、多网络后端和复杂 Netplan 配置；没有明确需求时不必记忆配置语法。
>
> 命令行拆解、选项与参数查询见 [[Linux 命令行学习路线与命令地图]] 和 [[Shell 命令结构、类型与帮助系统]]；组合验证代码块中的标准流与退出状态见 [[Shell 标准流、管道、重定向与退出状态]]。

> [!info] 资料核对日期
> 本文于 **2026-07-18** 核对 systemd 网络接口命名、Netplan 和 UTM 官方资料。网卡名、Netplan 后端和系统预装组件可能随 Ubuntu 版本与安装方式不同，应以目标主机的实际输出为准。

## 完成标准

完成本文后，应能够：

- 区分 UTM 网络模式、Linux 网络接口和应用服务三个层次。
- 读懂 `enp0s1`、`192.168.64.2/24`、MAC 地址和 `default via` 一类字段。
- 解释 DHCP 与静态地址的适用边界。
- 分层验证接口、地址、路由、DNS 和 HTTPS。
- 解释 Netplan、网络后端和 `ip` 命令之间的关系，并在修改配置前保留控制台恢复路径。

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
| 应用与访问控制 | SSH、HTTPS、监听端口、防火墙 | 目标服务是否存在，连接是否被允许？ |

例如，Ubuntu 安装器显示 `DHCPv4` 地址，只能证明已识别接口并取得了地址。这不能单独证明默认路由、DNS、HTTPS 或 SSH 已经可用。

## 2. 网络接口与名称

Linux 将每块实体或虚拟网卡表示为一个网络接口。常见名称包括：

- `lo`：回环接口，用于本机通信，不是对外网卡。
- `enp0s1`、`ens3`：systemd/udev 根据设备类型与位置生成的可预测以太网接口名。`en` 表示以太网，后续部分与设备路径或插槽信息有关。
- `eth0`：传统命名，某些镜像、虚拟环境或自定义配置仍可能使用。
- `tailscale0`、`docker0`：软件创建的虚拟接口。

不要假设所有主机都使用 `enp0s1`。先读取当前系统的实际名称。

**执行位置：Linux 主机（任意目录）**

```bash
ip -brief link
ip -brief address
```

`ip -brief link` 用于确认接口是否存在与启用；`ip -brief address` 在此基础上列出已分配的 IPv4 和 IPv6 地址。`UP` 表示接口处于启用状态，不等于已经能访问互联网。

## 3. MAC 地址、IP 地址与主机名

这三类标识解决不同问题。

| 标识 | 用途 | 是否可能变化 |
| --- | --- | --- |
| MAC 地址 | 识别本地链路上的网络接口 | 替换或重建虚拟网卡、手动设置时可变 |
| IP 地址 | 让主机在 IP 网络中发送和接收数据 | DHCP 续租、切换网络或修改配置时可变 |
| 主机名 | 便于人和系统识别主机 | 可修改；不保证在其他设备上自动解析 |

MAC 地址不能直接代替 IP 地址用于 SSH。主机名也只有在 DNS、mDNS、`/etc/hosts` 或其他名称服务能解析它时，才能作为网络目标。

## 4. IP 地址与子网前缀

`192.168.64.2/24` 由两部分组成：

- `192.168.64.2`：分配给接口的 IPv4 地址。
- `/24`：子网前缀长度，表示前 24 位用于识别网络。在这个示例中，直连子网通常表示为 `192.168.64.0/24`。

这是用于学习字段含义的示例，不是要在所有 UTM 虚拟机中写死的配置。两台主机的地址看似位于同一子网，也不代表中间一定没有虚拟化隔离、防火墙或其他访问限制。

## 5. DHCP 与静态地址

DHCP 通常会向客户端提供：

- IP 地址和子网前缀。
- 默认路由或网关。
- DNS 服务器与可选的搜索域。
- 租约期限与其他网络参数。

对 UTM 的普通开发虚拟机，先用 DHCP 完成安装、系统更新和宿主到客户机的 SSH 验证。DHCP 地址可能在重建网卡、网络环境变化或租约变化后不同，因此不应把一次安装器输出当作永久地址。

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
```

下列输出只是阅读示例：

```text
default via 192.168.64.1 dev enp0s1
192.168.64.0/24 dev enp0s1 proto kernel scope link src 192.168.64.2
```

- `default`：没有更具体路由可匹配时使用的默认路由。
- `via 192.168.64.1`：示例中的下一跳网关，实际地址以当前输出为准。
- `dev enp0s1`：数据包从该接口发出。
- `scope link`：该子网与接口直接相连，不需要先经过其他网关。

主机有 IP 地址但没有可用的默认路由时，往往只能访问直连子网，不能正常访问其他网络。

## 7. DNS 名称解析

DNS 将 `archive.ubuntu.com` 一类域名解析为 IP 地址。它与路由是不同的层次：可以出现“能访问某个 IP，但不能解析域名”的情况。

**执行位置：Ubuntu Server（任意目录）**

```bash
resolvectl status
getent ahosts archive.ubuntu.com
```

- `resolvectl status` 用于查看当前接口的 DNS 服务器和域名路由信息。
- `getent ahosts` 通过系统实际的名称服务配置做解析，比只查某个配置文件更接近应用的行为。

某些最小化镜像可能没有运行 `systemd-resolved`，因而没有 `resolvectl`。这时仍可以使用 `getent ahosts`，并根据实际的网络管理组件检查 DNS，不要为了照抄命令而同时启用多个网络后端。

## 8. 安装器中的 VirtIO 网卡与 bond

UTM 会向 Linux 客户机呈现一块虚拟网卡。Ubuntu 安装器可能将它显示为 `Virtio network device`，并附带硬件厂商字符串。这只是客户机识别到的虚拟硬件信息，不代表 Ubuntu 发行版或 UTM 配置错误。

bond 是将多个网络接口组成一个逻辑接口的机制，可用于冗余或特定的流量分配策略。它需要多块接口、明确的 bond 模式，有时还需要上游网络配合。只有一块虚拟网卡的开发虚拟机不应为了完成安装而创建 bond。

## 9. 一组只读的分层验证

先使用只读命令收集现状，再决定是否修改配置。

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
printf '%s\n' '--- interfaces ---'
ip -brief link
ip -brief address

printf '%s\n' '--- routes ---'
ip route

printf '%s\n' '--- DNS ---'
resolvectl status
getent ahosts archive.ubuntu.com

printf '%s\n' '--- HTTPS ---'
curl -I --max-time 15 https://archive.ubuntu.com/
```

按以下顺序判断：

1. 是否存在预期接口。
2. 接口是否启用并拥有预期地址。
3. 是否有直连路由和默认路由。
4. DNS 服务器是否已配置，域名是否能解析。
5. HTTPS 是否能建立连接并完成证书校验。

`curl -I` 能收到 HTTP 响应即可证明网络路径基本成立，状态码不一定必须是 `200`。单独使用 `ping` 不能覆盖 DNS、TLS 或目标服务，而且中间网络可能禁止 ICMP，因此不应把 ping 失败直接等同于断网。

## 10. Ubuntu 如何保存网络配置：Netplan

前面使用的 `ip -brief address`、`ip route` 和 `resolvectl status` 主要用于查看当前已经生效的网络状态。但 Linux 还需要知道：下次开机时，某块网卡应该继续使用 DHCP，还是应该配置静态地址。

**Netplan 就是 Ubuntu 用来保存这些“期望网络配置”的配置层。** Ubuntu Server 安装器中选择 DHCP 后，安装器通常会生成对应的 Netplan 配置，让已安装系统重启后仍会为该接口请求 DHCP 地址。

### 10.1 Netplan 与其他组件的关系

可以把配置从文件到实际生效的过程理解为：

`/etc/netplan/*.yaml` 期望配置 → Netplan 转换配置 → 网络后端应用配置 → Linux 内核保存当前状态 → `ip` 等命令读取结果

| 对象 | 负责什么 | 不负责什么 |
| --- | --- | --- |
| UTM | 向客户机提供虚拟网卡，并选择 Shared Network 或 Bridged 等接入方式 | 不编辑 Ubuntu 内部的 Netplan 配置 |
| Netplan | 读取 YAML 中的 DHCP、静态地址、路由和 DNS 等期望配置，并转换给网络后端 | 不是网卡、DHCP 服务器、路由器或 DNS 服务器 |
| `systemd-networkd` 或 NetworkManager | 作为网络后端，把期望配置实际应用到接口和内核 | 不决定 UTM 使用哪种网络模式 |
| `ip -brief address`、`ip route`、`resolvectl status` | 读取已经生效的接口、地址、路由和 DNS 状态 | 这些查询用法不会保存下次启动要使用的配置 |

因此，`sudo netplan get` 回答的是“Ubuntu 已保存了什么期望配置”，`ip address` 和 `ip route` 回答的是“当前实际生效了什么”。排查时需要对比这两者，而不是只看某一边。

### 10.2 先读取，不要立即修改

先查看 Netplan 配置文件和合并后的期望配置：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
ls -l /etc/netplan
sudo netplan get
```

这两条命令只用于读取。常见 YAML 文件的具体名称可能由安装器或 cloud-init 生成，不要假设所有 Ubuntu 主机使用同一个文件名。

### 10.3 Netplan 能管理的范围

Netplan 可以描述 DHCP、静态地址、路由、DNS、bond 和桥接等 Ubuntu 客户机内的配置，但它不能：

- 为 UTM 选择 Shared Network 或 Bridged 模式。
- 创建或修复 UTM 中本来就不存在的虚拟网卡。
- 代替上游网络中的 DHCP 服务器、网关或 DNS 服务器。
- 自动开放 SSH 端口或修改 Linux 防火墙。

这就是本节所说的“边界”：Netplan 只负责描述 Ubuntu 客户机内应该如何配置网络，不负责整条网络路径上的其他系统。

### 10.4 修改配置时保留回退路径

修改 `/etc/netplan/*.yaml` 后应用配置，可能立即中断网络。操作前应保留 UTM 控制台或其他带外恢复入口；远程主机优先使用带超时回退的 `sudo netplan try`，不要在唯一的 SSH 会话中盲目运行 `netplan apply`。

本文不提供一组可直接照抄的静态地址、bond 或桥接配置。这些配置依赖实际网卡名、地址规划、默认网关、DNS 和上游网络，应在需求明确后单独设计和验证。

## 常见问题

| 现象 | 优先检查 | 不应立即做什么 |
| --- | --- | --- |
| 没有预期网络接口 | UTM 虚拟网卡是否启用，Linux 是否识别设备 | 不要先填静态 IP |
| 接口存在但没有地址 | 接口状态、DHCP 和网络模式 | 不要猜测网关或 DNS |
| 有地址但没有默认路由 | DHCP 租约、Netplan 与网络后端 | 不要将 DNS 当成根因 |
| 能访问 IP 但不能解析域名 | `resolvectl status`、`getent ahosts` | 不要关闭 TLS 校验 |
| DNS 正常但 HTTPS 失败 | 系统时间、代理、防火墙和目标服务 | 不要随意替换为不明软件源 |
| 客户机能出网，宿主或局域网设备不能连入 | UTM 网络模式、服务监听地址和防火墙 | 不要只通过改静态 IP 解决 |

## 后续阅读

- 创建与安装虚拟机：[[使用 UTM 创建 Ubuntu Server 虚拟机]]
- 选择虚拟机网络模式：[[虚拟机网络模式与可达性]]
- 完成新系统基线：[[Ubuntu Server 初始化与基础安全]]
- 验证 SSH 服务与身份：[[OpenSSH 连接、密钥与主机指纹]]
- 跨局域网扩展访问：[[使用 Tailscale 访问 Linux 主机]]

## 官方参考资料

以下资料于 **2026-07-18** 核对：

- [systemd：网络设备命名方案](https://www.freedesktop.org/software/systemd/man/latest/systemd.net-naming-scheme.html)
- [Netplan：DHCP、静态地址、DNS 与 bond 示例](https://netplan.readthedocs.io/en/stable/examples/)
- [Netplan：安全试用配置](https://netplan.readthedocs.io/en/stable/netplan-try/)
- [UTM：Apple 虚拟机网络设置](https://docs.getutm.app/settings-apple/devices/network/)
- [UTM：QEMU 虚拟机网络设置](https://docs.getutm.app/settings-qemu/devices/network/network/)

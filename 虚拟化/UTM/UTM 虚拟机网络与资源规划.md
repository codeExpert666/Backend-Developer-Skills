---
title: UTM 虚拟机网络与资源规划
aliases:
  - UTM vCPU 内存与磁盘规划
  - UTM Shared Network 与桥接网络
tags:
  - 虚拟化
  - UTM
  - 虚拟机
  - 网络
  - 资源规划
created: 2026-07-17T00:48:00
updated: 2026-07-17T01:12:07
---

本文解释如何为 UTM 虚拟机选择 vCPU、内存、稀疏磁盘和网络模式。重点不是给出一个永久固定配置，而是建立“从宿主资源与工作负载出发，运行后再观察调整”的方法。

开始前先理解 [[虚拟机、客户机与 CPU 架构]]。

## 资源规划原则

虚拟机资源来自宿主机。分配给 VM 的上限过高，不会创造新资源，反而可能让宿主机、IDE、浏览器和虚拟机同时争抢。

### vCPU

主要受以下工作负载影响：

- Go 并行编译和测试。
- Maven 编译、测试和静态分析。
- Docker BuildKit 构建。
- 数据库、缓存和 IDE 远程服务。

选择方法：

1. 给宿主机保留调度余量。
2. 从中等数量开始，不追求把所有核心都分给 VM。
3. 观察构建期间 CPU 饱和度和宿主交互延迟。
4. 只有任务稳定受 CPU 限制时再增加。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
nproc
lscpu
uptime
```

### 内存

内存需要同时覆盖：

- Ubuntu 内核、systemd 和基础服务。
- IDE Remote、语言服务器和文件索引。
- JVM heap、Maven 和测试进程。
- Go 编译缓存与测试进程。
- Docker daemon、容器和数据库缓存。

调整依据：

- 宿主机持续内存压力或大量 swap：降低 VM 内存或减少并发。
- VM 内频繁 OOM、swap 或 JVM 无法启动：在宿主仍有余量时增加内存。
- 数据库与应用同时运行：分别观察，而不是只看空闲时的数字。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
free -h
vmstat 1 5
```

### 稀疏磁盘

稀疏磁盘的“容量上限”与“当前实际占用”不是同一个数字：

- 客户机可以看到较大的逻辑磁盘。
- 宿主文件通常随写入逐步增长。
- 删除客户机文件不保证宿主文件立即缩小。
- 复制到不支持稀疏文件的介质时，副本可能明显膨胀。

磁盘需要覆盖：

- Ubuntu 系统和更新。
- 源码与 Git 对象。
- Go module/build cache。
- Maven 本地仓库和构建输出。
- Docker 镜像、构建缓存、容器日志和卷。
- 数据库文件与备份。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
df -hT
du -sh "$HOME/src" "$HOME/go" "$HOME/.m2" "$HOME/.cache" 2>/dev/null || true
docker system df 2>/dev/null || true
```

不要把磁盘紧张直接等同于“执行所有 prune”。先识别占用类型和可恢复性，特别是数据库卷。

## UTM 网络模式

| 模式 | 客户机如何接入 | 适合场景 | 主要边界 |
| --- | --- | --- | --- |
| Shared Network | 由宿主机路由，使用体验类似 NAT | 默认开发主线，客户机出网、宿主访问客户机 | 局域网其他设备未必能直接到达 |
| Bridged | 客户机像独立设备接入局域网 | 需要被 LAN 其他设备直接访问 | 暴露面更大，Wi-Fi 和网络策略可能限制 |
| 端口转发 | 宿主端口映射到客户机端口 | Shared 下无法直接到达某端口时 | 需要管理端口冲突和监听地址 |
| Tailscale | 在节点之间建立受控覆盖网络 | 跨局域网或外出访问 | 依赖账号、设备授权和访问控制 |

主线选择原则：

1. 先用 Shared Network 完成客户机联网和宿主 SSH。
2. 只有明确需要 LAN 其他设备访问时才评估桥接。
3. 外出访问优先把 Tailscale 作为独立网络层，不同时改桥接、端口转发和防火墙。
4. 每次只改变一层，保留可回退路径。

## 网络分层排查

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
ip -brief address
ip route
getent hosts archive.ubuntu.com
curl -I https://archive.ubuntu.com/
```

**执行位置：宿主机（任意目录）**

```bash
printf '客户机地址或可解析名称: '
IFS= read -r VM_HOST
ping -c 3 "$VM_HOST"
nc -vz "$VM_HOST" 22
```

判断顺序：

1. 客户机是否有地址。
2. 是否有默认路由。
3. DNS 是否解析。
4. HTTPS 是否工作。
5. 宿主机是否能到达客户机。
6. 目标服务是否监听。
7. 客户机防火墙是否放行。

## 共享目录边界

UTM 共享目录适合临时导入和导出，不应默认成为长期 Linux 工作区。跨操作系统共享可能在以下方面存在差异：

- Unix 所有者和权限位。
- 可执行位和符号链接。
- 文件锁与大小写语义。
- Unix socket。
- `inotify` 文件监听。
- 大量小文件性能。

项目长期位置见 [[Linux 开发工作区与本地文件系统规划]]。

## 如何记录项目案例

技术笔记不把某组数字写成普遍答案。具体项目应在阶段基线中记录：

- 宿主机资源。
- 候选 vCPU、内存和磁盘。
- 预计运行的构建、数据库和容器。
- 选择理由。
- 运行后观察结果。
- 调整和回滚方法。

EventHub 当前案例见 [[EventHub 第 1 阶段环境与版本基线]]。

## 完成标准

- [ ] 能解释 vCPU、内存和磁盘配置依据。
- [ ] 知道稀疏磁盘上限不等于立即占用。
- [ ] 能区分 Shared、Bridged、端口转发和 Tailscale。
- [ ] 能按网络层次排查，而不是同时改多项配置。
- [ ] 不把共享目录当作默认长期工作区。

## 官方参考资料

以下 UTM 资料于 **2026-07-16** 核对：

- [UTM：Apple 虚拟机网络设置](https://docs.getutm.app/settings-apple/devices/network/)
- [UTM：Apple 虚拟机 Drive](https://docs.getutm.app/settings-apple/drive/)
- [UTM：QEMU 网络设置](https://docs.getutm.app/settings-qemu/devices/network/network/)
- [UTM：QEMU Drive](https://docs.getutm.app/settings-qemu/drive/drive/)

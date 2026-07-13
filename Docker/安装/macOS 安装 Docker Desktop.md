---
title: macOS 安装 Docker Desktop
aliases:
  - Mac 安装 Docker Desktop
  - macOS Docker Desktop 安装
tags:
  - Docker
  - Docker/安装
  - Docker/macOS
  - Docker/Docker-Desktop
created: 2026-07-13T22:43:39
updated: 2026-07-13T22:54:50
---

在 macOS 上进行后端开发时，推荐使用 Docker Desktop。它为 Docker CLI、Docker Compose、镜像缓存、端口映射和桌面设置提供完整的开发环境；若需要了解平台选择，请先阅读 [[Docker 安装概览]]。

Linux 容器不能直接在 macOS 内核运行。Docker Desktop 会使用 Linux 虚拟化环境来承载 Docker Engine 和容器，因此容器中的 `uname -s` 通常显示 Linux，而不是 Darwin。不要尝试在 macOS 上用 `systemctl` 管理 Docker，也不要把 Docker Desktop 当作 macOS 原生容器运行时。

## 适用范围与选择建议

本文适用于 Apple Silicon 与 Intel Mac 上的日常开发、调试和本地集成测试。根据芯片选择对应的 Docker Desktop 安装包：

| Mac 类型 | 检查方式 | 应选择的安装包 | 镜像注意事项 |
| --- | --- | --- | --- |
| Apple Silicon | `uname -m` 输出通常为 `arm64` | Apple silicon 版本 | 优先使用 `linux/arm64` 或 multi-arch 镜像 |
| Intel Mac | `uname -m` 输出通常为 `x86_64` | Intel chip 版本 | 通常使用 `linux/amd64` 镜像 |

Apple Silicon 可运行部分 `linux/amd64` 镜像，但这会涉及架构模拟，构建和运行性能可能下降。对数据库、消息队列等长时间运行的本地依赖，优先选择项目维护的 multi-arch 镜像或原生 `arm64` 镜像。

## 前置条件

先确认系统版本、芯片与资源：

```bash
sw_vers
uname -m
sysctl -n hw.memsize
```

Docker Desktop 官方要求使用受支持的 macOS 版本并至少具备 4 GB 内存；支持范围会随 macOS 大版本发布而变化，应以安装页为准。若团队使用 MDM、受管设备、企业代理或统一 Docker Desktop 设置，先按组织说明安装，不要用个人配置覆盖受管策略。

在 Apple Silicon 上，如项目仍依赖 Darwin/AMD64 辅助工具，可按需要安装 Rosetta 2：

```bash
softwareupdate --install-rosetta
```

Rosetta 2 并不是 Docker Desktop 启动的前提条件；是否需要取决于工具链和镜像架构。

## 安装步骤

### 1. 下载与安装正确架构的 Docker Desktop

从 Docker 官方 macOS 安装页下载与芯片匹配的 `Docker.dmg`。打开磁盘镜像后，将 Docker 图标拖入 Applications 文件夹；默认安装位置为 `/Applications/Docker.app`。

不要从来源不明的镜像站、网盘或重新打包的安装程序获取 Docker Desktop。企业若要求校验下载文件，应使用官方发布页提供的校验信息和组织的软件下载流程。

### 2. 首次启动

在 Applications 中打开 Docker.app，按界面完成许可协议和必要的权限确认。等待菜单栏中的 Docker 状态变为 Engine 正在运行后，再打开终端执行 Docker 命令。

首次启动时不必立刻调整所有设置。先完成最小验证，再根据实际项目的镜像大小、构建并发和数据库负载调整资源，能避免把宿主机内存全部分配给 Docker Linux VM。

## 配置与权限

### 资源设置

在 Docker Desktop 的 Settings 中查看 Resources。资源上限作用于承载容器的 Linux VM，而不是直接作用于 macOS 进程。

| 设置项 | 何时调整 | 建议 |
| --- | --- | --- |
| CPU | 多阶段构建或并行编译持续受限 | 从少量增量开始，避免抢占 IDE 与数据库资源 |
| Memory | 数据库、消息队列或构建发生 OOM | 留出足够内存给 macOS 与日常开发工具 |
| Disk usage / disk image location | 镜像、卷或 BuildKit 缓存持续增长 | 选择可靠磁盘位置；迁移前先备份重要卷 |
| File sharing | 项目需要挂载宿主目录 | 只共享必要目录，并留意大量小文件的 I/O 开销 |

可以用 `docker system df` 观察镜像、容器和构建缓存占用。清理命令会删除资源，执行 `docker system prune` 或带 `--volumes` 的变体前，先确认没有需要保留的本地数据。

### Docker context 与 CLI

Docker Desktop 正常启动后，CLI 通常自动使用默认 context。先确认连接目标，不要在不知情时切换到远端 Docker 主机：

```bash
docker context ls
docker context show
docker version
```

若 `docker version` 只有 Client 信息或提示不能连接 daemon，优先确认 Docker Desktop 是否已经完成启动，而不是重新安装 CLI。

### 代理、镜像拉取与企业网络

Docker Desktop 的代理应在应用内配置，而不是照搬 Linux `daemon.json`。进入 Docker Desktop Settings → Resources → Proxies，按组织要求选择 System proxy 或 Manual configuration；镜像拉取使用的 Containers proxy 也应与策略一致。

遇到镜像拉取、DNS 或企业网络问题时，可按以下顺序定位：

1. 在 Docker Desktop Dashboard 确认 Engine 正在运行，并打开 Settings → Resources → Proxies 检查代理模式和例外域名。
2. 在 Settings 的 Network 相关页面检查 DNS、网络模式或内部网段是否与 VPN、办公网冲突。
3. 在终端运行 `docker pull hello-world`，再运行项目实际的 `docker compose pull`，分别确认通用与项目镜像路径。
4. 仍无法定位时，从 Dashboard 的 Troubleshoot / diagnostics 入口收集诊断信息，并交给网络或终端支持团队。

不要把代理账号、密码或内部 CA 私自写入项目仓库。需要信任企业根证书时，应使用组织批准的证书分发与 Docker Desktop 配置方式。

## 验证

在终端执行：

```bash
docker version
docker context ls
docker info
docker run --rm hello-world
docker compose version
docker compose ls
```

`docker run --rm hello-world` 验证 Docker Engine 能拉取并运行容器，`docker compose version` 验证 Compose v2 插件可用。对已有项目，再进入包含 `compose.yaml` 或 `docker-compose.yml` 的目录验证配置：

```bash
docker compose config --quiet
docker compose pull
```

后两条命令不会启动项目服务，但会分别检查 Compose 配置和镜像拉取路径。

## 中国大陆网络环境提示

> [!tip] 在中国大陆网络环境下
> Docker Desktop 下载和 Docker Hub 镜像拉取可能受网络路径、代理或 DNS 影响。优先使用组织内部镜像仓库、受信任代理或当时可用的官方服务；不要在团队文档中固化来源不明的第三方镜像加速地址。
>
> 配置后至少验证 `docker pull hello-world`、一个项目真实镜像和 `docker compose pull`。若浏览器可以访问网页但拉取失败，重点检查 Docker Desktop 的 Proxies 配置、Containers proxy、企业证书和 DNS，而不是只检查 shell 环境变量。

## 常见问题

### `Cannot connect to the Docker daemon`

确认 Docker Desktop 已启动并显示 Engine 正在运行，然后执行：

```bash
docker context show
docker version
```

若 context 指向非默认远端主机，先确认这是有意配置；不要为了消除报错而随意暴露本地 daemon 的 TCP 端口。

### Apple Silicon 上镜像架构不兼容或运行缓慢

先检查镜像是否提供 `linux/arm64` 或 multi-arch manifest。临时指定 `--platform linux/amd64` 可以帮助兼容旧镜像，但不应成为长期默认；优先升级镜像或构建多架构制品。

### 挂载目录慢、构建卡住或 Docker 占用过多资源

检查 Resources 的 CPU、内存和磁盘限制，确认项目代码位于本地可靠磁盘而非网络同步目录，并用 `docker system df` 查看缓存。不要在不了解影响时直接删除卷，因为本地数据库数据通常存放在卷中。

## 官方参考资料

- [Docker：在 Mac 安装 Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/)
- [Docker：Docker Desktop 设置与资源配置](https://docs.docker.com/desktop/settings-and-maintenance/settings/)
- [Docker：Mac 上 Docker Desktop 的虚拟机管理器](https://docs.docker.com/desktop/features/vmm/)
- [Docker：Docker Desktop 网络与代理设置](https://docs.docker.com/desktop/features/networking/networking-how-tos/)

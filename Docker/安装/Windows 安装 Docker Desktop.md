---
title: Windows 安装 Docker Desktop
aliases:
  - Windows Docker Desktop 安装
  - Windows 使用 WSL 2 安装 Docker
tags:
  - Docker
  - Docker/安装
  - Docker/Windows
  - Docker/Docker-Desktop
  - Docker/WSL
created: 2026-07-13T22:43:39
updated: 2026-07-13T22:54:50
---

本文面向 Windows 主机上的常规后端开发。默认方案是 Docker Desktop 使用 WSL 2 后端，并把 Docker CLI 集成到需要的 WSL 发行版中。它适合运行 Java、Go、Node.js、Python 等后端服务依赖的 Linux 容器；完整平台选择见 [[Docker 安装概览]]。

如果你确实需要在某个 WSL 发行版内自行管理 Docker 守护进程，请改走 [[WSL 2 中安装 Docker Engine]]。这不是 Docker Desktop integration 的“加强版”，而是另一条独立路线；同一个 WSL 发行版中不要同时配置两套守护进程。

## 适用范围与选择建议

| 需求 | 推荐方案 | 原因 |
| --- | --- | --- |
| Windows 上的日常后端开发 | Docker Desktop + WSL 2 后端 | 管理、升级、Compose、网络与 WSL 集成较完整 |
| 在 WSL 终端内使用 Docker | 开启 Docker Desktop 的 WSL integration | WSL 中的 CLI 连接 Desktop 管理的 Engine，不需另装 `dockerd` |
| 依赖 Windows 内核或 Windows 基础镜像 | Docker Desktop 的 Windows Containers 模式 | 仅在明确需要 Windows Containers 时选择 |
| 必须独立控制 WSL 中的守护进程 | WSL 2 内独立 Docker Engine | 需停止使用 Desktop integration，见 [[WSL 2 中安装 Docker Engine]] |

> [!important] 默认聚焦 Linux Containers
> 多数后端项目使用 Linux 镜像和 Linux 容器。Docker Desktop 的默认模式应保持 Linux Containers；切换到 Windows Containers 后，CLI 连接的是另一套 Windows daemon，镜像、容器和设置也不会与 Linux Containers 混用。

## 前置条件

### 硬件虚拟化与受支持的 Windows

Docker Desktop 的 WSL 2 后端需要 64 位处理器、SLAT、足够内存和 BIOS/UEFI 中启用的硬件虚拟化。Windows 版本、版本号、edition 与 Docker Desktop 支持范围会更新，应在下载前核对官方安装页。

在 PowerShell 中先检查基本状态：

```powershell
systeminfo | Select-String -Pattern 'Virtualization|Hyper-V'
wsl --status
wsl --version
wsl -l -v
```

若硬件虚拟化显示未启用，先在固件设置中开启对应选项，再继续安装。企业电脑若由 IT 管理 BIOS、WSL 或虚拟化策略，应先申请相应权限。

### 安装或更新 WSL 2

在以管理员身份运行的 PowerShell 中，尚未安装 WSL 时可执行：

```powershell
wsl --install
```

按提示重启 Windows，并完成 Ubuntu 或所选发行版的首次用户创建。然后更新 WSL、设置新发行版默认使用 WSL 2，并确认现有发行版版本：

```powershell
wsl --update
wsl --set-default-version 2
wsl -l -v
```

若现有发行版仍显示 Version 1，替换实际名称后升级：

```powershell
wsl --set-version <发行版名称> 2
```

Docker Desktop 当前文档要求至少使用 WSL 2.1.5，并建议保持 WSL 为最新版本。

## 安装步骤

### 1. 安装 Docker Desktop

从 Docker 官方 Windows 安装页下载 `Docker Desktop Installer.exe`。安装程序会询问安装模式和后端；常规 Linux 容器开发选择 WSL 2 后端。安装范围、管理员权限与是否需要 Windows Containers 应遵从组织设备管理策略。

安装完成后，从 Start 菜单启动 Docker Desktop。首次启动时接受必要的许可协议，并等待 Docker Engine 就绪。

> [!note] `docker-users` 组不是 Linux Containers 的常规前置条件
> Docker 官方说明中，使用 WSL 2 后端运行 Linux Containers 时通常不需要将用户加入 `docker-users`。该组与 all-users 安装、Hyper-V VM 管理或 Windows Containers 相关，并能带来高权限能力；不要为了“避免权限报错”盲目添加。

### 2. 打开 WSL integration

在 Docker Desktop 中完成以下设置：

1. 打开 Settings，确认使用 WSL 2 based engine；在支持 WSL 2 的环境中它通常已是默认设置。
2. 进入 Resources → WSL Integration。
3. 确认目标发行版显示为 Version 2，并启用该发行版的 integration。
4. 选择 Apply / Restart（若界面要求），然后重新打开该 WSL 终端。

Docker Desktop 会在自己的 `docker-desktop` WSL 发行版中运行相关组件。启用 integration 后，普通 WSL 发行版获得 Docker CLI 访问能力；不要在其中再执行 Ubuntu Docker Engine 的安装步骤。

## 配置与权限

### 在 PowerShell 与 WSL 中的使用边界

同一套 Docker Desktop Engine 可以从 Windows PowerShell 与已集成的 WSL 终端调用。开发代码建议放在 WSL 的 Linux 文件系统中，再从 WSL 终端运行 Docker 命令，以获得更自然的 Linux 开发体验和较好的文件挂载性能。

不要通过在 WSL 中安装独立 `docker-ce`、`dockerd` 或另一套 Compose 来“修复”集成问题。先在 Docker Desktop 的 WSL Integration 页面检查分发版开关；仍不通时，检查 WSL 是否为 Version 2、Docker Desktop 是否处于 Linux Containers 模式，以及当前终端是否打开了正确发行版。

### Linux Containers 与 Windows Containers

| 类型 | 内核与镜像 | 常见用途 | 选择提醒 |
| --- | --- | --- | --- |
| Linux Containers | Linux 内核与 Linux 基础镜像 | 大多数后端服务、数据库、消息队列、CI 镜像 | 默认选择 |
| Windows Containers | Windows 内核与 Windows 基础镜像 | 依赖 Windows API、IIS 或特定 Windows 运行时 | 需满足 Windows edition、安装模式和镜像版本要求 |

可从 Docker Desktop 菜单切换 Linux Containers 或 Windows Containers。切换会改变 CLI 对接的 daemon，不会把 Linux 镜像自动转换成 Windows 镜像；项目没有明确要求时保持 Linux Containers。

### 资源、代理与企业网络

WSL 2 使用动态资源分配。遇到构建后内存长期不回收、镜像磁盘膨胀或 VPN/DNS 异常时，优先在 Docker Desktop Settings 中检查 Resources、WSL Integration、Proxies 和 Network，而不是改写 WSL 发行版内的 `daemon.json`。

代理配置应由组织提供。在 Docker Desktop Settings → Resources → Proxies 中选择系统代理或手工配置，并为内部域名、`localhost` 与私有网段正确配置 bypass / no-proxy 规则。不要把代理凭据提交到项目仓库。

## 验证

先在 PowerShell 验证 Docker Desktop 的 CLI 和 Engine：

```powershell
docker version
docker context ls
docker run --rm hello-world
docker compose version
```

再打开已启用 integration 的 WSL 发行版，执行同样的基本验证：

```bash
docker version
docker info
docker run --rm hello-world
docker compose version
```

两边都应能看到 Docker Server 信息，并且 `hello-world` 能成功运行。对真实项目，在 WSL 项目目录执行：

```bash
docker compose config --quiet
docker compose pull
```

如果 PowerShell 可用而 WSL 不可用，问题通常在 WSL 版本或 integration，而不是镜像本身。

## 中国大陆网络环境提示

> [!tip] 在中国大陆网络环境下
> Docker Desktop 下载、Docker Hub 拉取、企业镜像仓库和 DNS 可能受到网络路径影响。优先使用组织内部镜像仓库、受信任代理或当时可用的官方服务；不要硬编码或推荐来源不明的第三方镜像加速地址。
>
> 验证时分别运行 `docker pull hello-world`、项目的 `docker compose pull`，并在 Docker Desktop 设置中确认 Proxies 与 Network。若 Windows 浏览器能访问、但 Docker 拉取失败，重点检查 Desktop 的 Containers proxy、企业根证书和 VPN/DNS 策略。

## 常见问题

### WSL integration 页面找不到发行版

先运行 `wsl -l -v`，确认目标发行版存在且 Version 为 2。若 Docker Desktop 当前处于 Windows Containers 模式，切换回 Linux Containers 后再查看 WSL Integration。

### Docker Desktop 启动失败或持续提示需要 WSL

依次检查硬件虚拟化、`wsl --version`、`wsl --status` 和 Windows 更新状态。不要同时启用多个来源不明的虚拟化或网络优化工具；企业设备应附上 Docker Desktop diagnostics 和 WSL 版本信息交给 IT 支持。

### 为什么 WSL 里出现两个 Docker？

这是 Docker Desktop integration 与独立 Docker Engine 同时存在的典型症状。选择一个方案：常规开发保留 Docker Desktop 并卸载 WSL 内独立 Engine/CLI；若必须独立维护 WSL daemon，则关闭 Desktop integration，按 [[WSL 2 中安装 Docker Engine]] 整理环境。

## 官方参考资料

- [Docker：在 Windows 安装 Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/)
- [Docker：Docker Desktop 的 WSL 2 后端](https://docs.docker.com/desktop/features/wsl/)
- [Docker：Docker Desktop 设置与 WSL integration](https://docs.docker.com/desktop/settings-and-maintenance/settings/)
- [Microsoft：安装 WSL](https://learn.microsoft.com/windows/wsl/install)

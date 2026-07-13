---
title: WSL 2 中安装 Docker Engine
aliases:
  - WSL 2 独立安装 Docker Engine
  - WSL Ubuntu 安装 Docker
tags:
  - Docker
  - Docker/安装
  - Docker/Windows
  - Docker/WSL
  - Docker/Engine
created: 2026-07-13T22:43:39
updated: 2026-07-13T22:54:50
---

先阅读 [[Windows 安装 Docker Desktop]]。对大多数 Windows 开发者，Docker Desktop 的 WSL integration 是默认且更省心的路线：它在 Docker Desktop 管理的运行时中提供 Engine，并让 WSL 终端使用同一套 Docker CLI。本文只面向确实需要在某个 WSL 2 发行版内`独立`运行 Docker Engine 的情况。

独立 Engine 路线与 Docker Desktop WSL integration 二选一。不要在同一个 WSL 发行版中既启用 Desktop integration，又安装 `docker-ce` 和启动自己的 `dockerd`；这会造成 Docker socket、context、镜像缓存、端口发布和服务状态相互混淆。

## 适用范围与选择建议

| 你的目标 | 建议 | 原因 |
| --- | --- | --- |
| Windows 上的普通本地开发 | 使用 [[Windows 安装 Docker Desktop]] | Docker Desktop 统一管理升级、资源、代理、Compose 与 WSL 集成 |
| 需要 GUI、Docker Desktop 功能或多发行版方便共享 CLI | 使用 Docker Desktop + WSL integration | 不维护 WSL 内第二套 daemon |
| 需要完全由 Linux 发行版控制 Docker 服务、配置和生命周期 | 本文的独立 Docker Engine | 服务、镜像与配置都在该发行版内管理 |
| 需要生产级 Linux 宿主机 | 使用真实 Linux 服务器或 VM | WSL 生命周期、网络和 Windows 更新不适合作为常规生产宿主 |

> [!warning] 先清理冲突路线，再安装
> Docker 官方在介绍 Desktop WSL 2 后端时明确要求，启用 integration 前应卸载 WSL 发行版中直接安装的 Docker Engine 或 CLI。反过来选择独立 Engine 时，也应关闭目标发行版的 Docker Desktop integration，并确认终端不会继续连接 Docker Desktop 的 daemon。不要依赖“刚好能运行”的偶然状态。

## 前置条件

### 1. 确认 WSL 2 与目标发行版

在 Windows PowerShell 中检查 WSL 版本和发行版模式：

```powershell
wsl --version
wsl --status
wsl -l -v
```

目标发行版必须显示 Version 为 2。若尚未安装 WSL，可在管理员 PowerShell 中安装 Ubuntu；已有其他发行版时，将名称替换为实际值：

```powershell
wsl --install -d Ubuntu
wsl --update
wsl --set-default-version 2
wsl --set-version <发行版名称> 2
```

最后一条只在现有发行版仍为 WSL 1 时需要执行。完成后，首次进入发行版时创建 Linux 用户与密码：

```powershell
wsl -d Ubuntu
```

### 2. 选择并隔离 Docker 运行时

在 Docker Desktop 中进入 Settings → Resources → WSL Integration，关闭目标发行版的 integration。若不再使用 Docker Desktop，可再按组织策略停止或卸载它；关键是目标 WSL 发行版只连接自己将要安装的 `dockerd`。

进入目标 WSL 发行版后，先确认当前命令不会仍然连到 Docker Desktop：

```bash
command -v docker || true
docker context ls 2>/dev/null || true
docker version 2>/dev/null || true
```

如果显示 Docker Desktop 的 Server 信息，先回到 Desktop 配置中处理 integration，再继续。不要一边保留 Desktop socket，一边继续下面的安装命令。

### 3. 启用并验证 systemd

Docker Engine 需要由 systemd 管理服务。当前通过 `wsl --install` 安装的 Ubuntu 通常已默认使用 systemd，但仍应验证：

```bash
ps -p 1 -o comm=
systemctl --version
```

若第一个命令不是 `systemd`，先确保 WSL 版本至少符合 Microsoft 的 systemd 支持要求，然后在 WSL 发行版中编辑 `/etc/wsl.conf`：

```ini
[boot]
systemd=true
```

保存后，在 Windows PowerShell 中关闭全部 WSL 实例，再重新进入发行版：

```powershell
wsl.exe --shutdown
wsl.exe -d Ubuntu
```

重新验证：

```bash
ps -p 1 -o comm=
systemctl status --no-pager
```

`systemctl status` 在 WSL 中可能显示部分非关键单元未启动或 `degraded`，不能仅凭这一点判断 Docker 安装失败；重点是 systemd 已作为 PID 1，可管理 `docker.service`。

## 安装步骤

以下步骤在目标 WSL 发行版内执行。Ubuntu 路线与 [[Ubuntu 安装 Docker]] 相同，使用 Docker 官方 apt 仓库；这里完整列出，避免把 Windows 与 Linux 侧命令混在一起。

### 1. 卸载冲突旧包

```bash
sudo apt remove docker.io docker-compose docker-compose-v2 docker-doc podman-docker
```

此命令不应被当作数据清理。若之前运行过独立 Docker，先评估 `/var/lib/docker` 中的卷、镜像、容器和项目数据。

### 2. 配置 Docker 官方 apt 仓库

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
apt-cache policy docker-ce
```

### 3. 安装 Docker Engine 与 Compose 插件

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

安装后不需要额外安装旧的 `docker-compose` v1。当前推荐使用 `docker compose` 子命令。

## 配置与权限

### 启动与管理服务

使用 systemd 启动 Docker 与 containerd，并配置在该 WSL 发行版启动时自动启动：

```bash
sudo systemctl enable --now docker.service
sudo systemctl enable --now containerd.service
sudo systemctl status docker.service --no-pager
```

日常管理命令如下：

```bash
sudo systemctl restart docker.service
sudo systemctl stop docker.service
sudo systemctl start docker.service
sudo journalctl -u docker.service -n 100 --no-pager
```

> [!note] WSL 的“开机启动”含义
> `systemctl enable` 表示 Docker 会在该发行版启动后由 systemd 启动，并不表示 Windows 登录时 WSL 一定长期驻留。WSL 在没有活动进程时可能停止；需要使用 Docker 时先进入发行版或由受控的 Windows 自动化任务启动它。不要用无意义的无限循环进程强行保活。

### 配置当前 Linux 用户权限

Docker socket 默认仅 root 可访问。将当前用户加入 `docker` 组后可以免 `sudo` 使用 Docker，但这会授予接近 root 的权限：

```bash
getent group docker >/dev/null || sudo groupadd docker
sudo usermod -aG docker "$USER"
newgrp docker
id
```

最可靠的生效方式是退出并重新进入 WSL。不要使用 `chmod 666 /var/run/docker.sock`；它会让该发行版中的任何用户取得 Docker 高权限接口。

### 代理与 Docker daemon

独立 WSL Engine 的镜像拉取由本发行版内的 `dockerd` 发起，因此代理配置应写给该 daemon，不是 Docker Desktop。使用组织提供的代理地址、证书和例外域名，并按 Docker 官方 daemon proxy 文档配置 `/etc/docker/daemon.json` 或 systemd drop-in；变更后执行：

```bash
sudo systemctl restart docker.service
sudo systemctl status docker.service --no-pager
```

不要把代理凭据提交到项目仓库，也不要同时让 Docker Desktop 与独立 Engine 对同一发行版注入不同的代理设置。

## 从 Windows 访问容器端口

### Windows 主机本地访问

在 WSL 2 的默认 NAT 网络模式下，Windows 主机通常可以通过 `localhost` 访问 WSL 中发布的端口。可做一次短暂验证：

```bash
docker run -d --rm --name port-check -p 8080:80 nginx:alpine
curl http://localhost:8080
docker rm -f port-check
```

然后在 Windows 浏览器或 PowerShell 中访问 `http://localhost:8080`。如果项目端口由 Compose 发布，同样优先从 Windows 本机 `localhost` 验证。

### 允许局域网访问时的额外限制

WSL 2 默认使用 NAT，WSL 的 IP 地址会随重启变化。要从局域网访问，除了容器端口映射，还需要考虑 Windows 防火墙、应用绑定地址、企业安全策略和可能需要更新的端口转发规则。只有确有需求且经批准时，才在管理员 PowerShell 中配置类似规则：

```powershell
$wslIp = (wsl.exe -d Ubuntu hostname -I).Trim().Split(' ')[0]
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=8080 connectaddress=$wslIp
New-NetFirewallRule -DisplayName 'WSL Docker 8080' -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8080
```

`listenaddress=0.0.0.0` 会监听所有 IPv4 接口，扩大了暴露面；应限制来源、防火墙范围和端口，并在 WSL 重启后重新核对 `$wslIp`。生产服务不应依赖这种临时开发型端口转发。

从 WSL 访问 Windows 主机服务时，默认 NAT 模式可先查询 Windows 主机在 WSL 看来使用的 IP：

```bash
ip route show | grep -i default | awk '{ print $3 }'
```

Windows 11 的 mirrored networking 改变了网络行为，必要时按 Microsoft 文档评估，而不是同时叠加多个 portproxy 规则。

## 验证

在 WSL 发行版中验证服务、权限、Engine 和 Compose：

```bash
systemctl is-active docker.service
docker version
docker info
docker run --rm hello-world
docker compose version
```

若这些命令均成功，说明当前 WSL 发行版已通过自己的 systemd 服务运行 Docker Engine。再用 `docker context ls` 确认没有误切回 Docker Desktop 或远端 daemon。

## 中国大陆网络环境提示

> [!tip] 在中国大陆网络环境下
> Docker 官方 apt 仓库、Docker Hub、企业镜像仓库和 DNS 可能存在不同的可达性。优先使用组织内部镜像仓库、受信任代理或当时可用的官方服务；不要写死或推荐来源不明的第三方镜像加速地址。
>
> 应分别验证 `curl -I https://download.docker.com/linux/ubuntu/`、`docker pull hello-world` 与项目的 `docker compose pull`。若 apt 可用而镜像拉取失败，问题通常在 `dockerd` 的代理、证书、DNS 或仓库访问策略，而不是 WSL 安装本身。

## 常见问题

### `System has not been booted with systemd as init system`

检查 `ps -p 1 -o comm=`。若不是 `systemd`，确认 `/etc/wsl.conf` 的 `[boot]` / `systemd=true` 配置已保存，然后在 Windows 执行 `wsl.exe --shutdown` 并重新进入发行版。

### `Cannot connect to the Docker daemon`

按顺序检查：

```bash
systemctl is-active docker.service
sudo systemctl status docker.service --no-pager
id
docker context ls
```

服务未启动时看 journal；服务正常但权限不足时重新登录或执行 `newgrp docker`；context 指向 Docker Desktop 时先完成运行时隔离。

### Windows 可以访问、局域网设备却无法访问端口

先确认容器发布端口、Windows 本机 `localhost` 访问、Windows 防火墙和应用监听地址。WSL 2 NAT 下局域网访问并非默认保证；如使用 `portproxy`，还需注意 WSL IP 变化和安全组/终端防护策略。

## 官方参考资料

- [Microsoft：安装 WSL](https://learn.microsoft.com/windows/wsl/install)
- [Microsoft：在 WSL 中使用 systemd 管理 Linux 服务](https://learn.microsoft.com/windows/wsl/systemd)
- [Microsoft：访问 WSL 中的网络应用](https://learn.microsoft.com/windows/wsl/networking)
- [Docker：在 Ubuntu 安装 Docker Engine](https://docs.docker.com/engine/install/ubuntu/)
- [Docker：Linux Docker Engine 安装后步骤](https://docs.docker.com/engine/install/linux-postinstall/)
- [Docker：Docker Desktop 的 WSL 2 后端](https://docs.docker.com/desktop/features/wsl/)

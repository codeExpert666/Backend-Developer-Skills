---
title: Ubuntu 安装 Docker
aliases:
  - Ubuntu 安装 Docker Engine
  - Ubuntu Docker Engine 安装
tags:
  - Docker
  - Docker/安装
  - Docker/Linux
  - Docker/Ubuntu
  - Docker/Engine
created: 2026-07-13T22:43:39
updated: 2026-07-20T21:49:36
---

本文用于在 Ubuntu LTS 上安装 Docker Engine，适合本地 Linux 开发机、CI Runner 和服务器。若宿主机是 macOS 或 Windows，请先回到 [[Docker 安装概览]] 选择 Docker Desktop 路线；若 Ubuntu 是 WSL 2 中的发行版，先阅读 [[WSL 2 中安装 Docker Engine]]，确认自己没有同时使用 Docker Desktop integration。

安装主线采用 Docker 官方 apt 仓库，而不是 Ubuntu 自带的 `docker.io` 包或一次性 convenience script。这样可以通过 apt 明确查看、选择和升级 Docker Engine、Buildx 与 Compose 插件。

## 适用范围与选择建议

本文以 Docker 官方仍支持的 Ubuntu LTS 为前提。仓库配置会从 `/etc/os-release` 读取当前系统代号，因此不要手工把示例中的代号固定为某一个版本；在服务器上执行前，仍应核对 [Docker 的 Ubuntu 支持范围](https://docs.docker.com/engine/install/ubuntu/)。

| 场景 | 建议 | 重点关注 |
| --- | --- | --- |
| 个人 Ubuntu 开发机 | 安装官方 apt 仓库中的最新稳定版 | 非 root 权限、Compose、磁盘空间 |
| CI Runner | 按 CI 镜像或主机基线固定版本 | 升级节奏、并发构建、缓存清理 |
| 生产服务器 | 先走变更流程，再按可回滚版本升级 | 最小权限、日志、监控、防火墙、镜像来源 |

> [!warning] 不要把 Docker Engine 安装当作无状态操作
> `/var/lib/docker`、卷、镜像、容器和网络都可能承载本地或生产数据。下面“卸载冲突旧包”只移除可能冲突的软件包，不应顺手删除数据目录；需要清理数据时，先备份并按单独的变更计划执行。

## 前置条件

确认当前系统是受支持的 Ubuntu LTS、使用 systemd，并检查体系结构：

```bash
. /etc/os-release
printf '%s\n' "$PRETTY_NAME"
printf '%s\n' "$VERSION_CODENAME"
uname -m
ps -p 1 -o comm=
```

以下步骤需要可使用 `sudo` 的账号以及到 Docker 官方软件仓库的网络访问。Docker 官方 apt 仓库支持常见的 `amd64` 与 `arm64` 等体系结构；实际可用性以当前官方仓库为准。

如果机器已经运行其他容器运行时、业务容器或 Kubernetes 组件，先确认它们是否依赖现有 `containerd`、iptables 规则或镜像数据。不要在未评估的生产主机上直接替换运行时。

## 安装步骤

### 1. 卸载可能冲突的旧包

Docker 官方要求先移除可能冲突的发行版包。即使下列包没有安装，这条命令通常也可以安全执行：

```bash
sudo apt remove docker.io docker-compose docker-compose-v2 docker-doc podman-docker
```

这一步不会自动删除 `/var/lib/docker` 中已有数据。若 apt 提示还存在其他本地安装的 Docker 二进制文件或手工配置，应先盘点其来源和依赖关系。

### 2. 配置 Docker 官方 apt 仓库

按官方方式安装证书与 curl、保存仓库签名密钥，再写入 deb822 格式的软件源：

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
```

检查 apt 是否能看到 Docker 官方候选版本：

```bash
apt-cache policy docker-ce
apt list --all-versions docker-ce
```

个人开发机通常安装当前稳定版即可；生产环境应先记录候选版本、兼容性和回滚方案，必要时使用官方文档中的指定版本安装方式。

### 3. 安装 Docker Engine、Buildx 与 Compose 插件

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

这里的 `docker-compose-plugin` 提供的是 `docker compose` 子命令。新项目应优先使用该形式，不要把已停止维护的独立 `docker-compose` v1 当作默认方案。

## 配置与权限

### 启动服务并确认开机启动

在 Debian/Ubuntu 上，安装后 Docker 服务通常会自动启动并配置为开机启动。下面命令可显式确保 `docker` 与 `containerd` 现在运行且被 systemd 启用：

```bash
sudo systemctl enable --now docker.service
sudo systemctl enable --now containerd.service
sudo systemctl status docker --no-pager
sudo systemctl is-enabled docker.service
```

停止、重启或查看日志时，使用 systemd，而不是手工后台运行 `dockerd`：

```bash
sudo systemctl restart docker
sudo journalctl -u docker.service -n 100 --no-pager
```

### 为个人账号配置非 root 使用权限

默认 Docker Unix socket 由 root 管理。将用户加入 `docker` 组后可免 `sudo` 使用 Docker，但该组拥有接近 root 的主机控制能力，不应无差别授予。

```bash
getent group docker >/dev/null || sudo groupadd docker
sudo usermod -aG docker "$USER"
newgrp docker
```

重新登录是最可靠的生效方式；在远程 SSH 会话或虚拟机中，可能还需要重新建立会话或重启虚拟机。若此前用 `sudo docker` 运行过命令，导致 `~/.docker` 属主错误，可按 Docker 官方 post-install 文档修复该目录权限。

> [!important] 服务器上的权限边界
> 在个人开发机上，将单个开发者加入 `docker` 组较常见；在生产服务器上，这相当于授予高权限宿主机控制能力。优先采用最小账号集合、受审计的 sudo 命令、Rootless mode 或组织既有的部署平台，不要为了“免 sudo”扩大权限范围。

### 代理与守护进程配置

独立 Docker Engine 的镜像拉取由 `dockerd` 执行。需要企业代理时，优先使用 Docker 官方支持的 `/etc/docker/daemon.json` 或 systemd drop-in 配置，并在变更后重启服务。一个不含真实凭据的结构示例如下：

```json
{
  "proxies": {
    "http-proxy": "http://proxy.example.com:3128",
    "https-proxy": "http://proxy.example.com:3128",
    "no-proxy": "localhost,127.0.0.1,.example.internal"
  }
}
```

代理地址、证书和 `no-proxy` 范围应由组织网络策略提供。不要把账号密码写进共享仓库、shell 历史或镜像构建参数。

## 验证

先以当前用户验证守护进程、Compose 插件和容器运行：

```bash
docker version
docker info
docker run --rm hello-world
docker compose version
```

再检查服务与 socket 权限：

```bash
systemctl is-active docker.service
ls -l /var/run/docker.sock
id
```

预期是 `docker.service` 为 `active`，`docker run --rm hello-world` 能输出欢迎信息，且当前账号的组列表包含 `docker`（若选择了非 root 方式）。

## 中国大陆网络环境提示

> [!tip] 在中国大陆网络环境下
> 若 Docker 官方 apt 仓库、Docker Hub 或企业镜像仓库访问不稳定，优先向组织确认内部镜像仓库、受信任代理和 DNS 方案。不要写死或推荐来源不明的第三方镜像加速地址。
>
> 应分层验证：先运行 `curl -I https://download.docker.com/linux/ubuntu/` 检查软件仓库路径；再运行 `docker pull hello-world` 检查守护进程的镜像拉取路径；最后拉取项目真实镜像或执行 `docker compose pull`。HTTP `401 Unauthorized` 有时也表示镜像仓库的网络与 TLS 已可达，需结合认证状态判断。

## 常见问题

### `permission denied while trying to connect to the Docker daemon socket`

先确认服务正在运行，再确认当前登录会话已重新加载 `docker` 组：

```bash
systemctl is-active docker.service
id
newgrp docker
```

不要用 `chmod 666 /var/run/docker.sock` 规避问题；它会把 Docker 高权限接口开放给所有本机用户。

### Docker 服务没有启动

先查看状态和最近日志：

```bash
sudo systemctl status docker --no-pager
sudo journalctl -u docker.service -n 100 --no-pager
```

常见原因包括代理配置格式错误、磁盘不足、已有守护进程手工占用 socket，或底层网络/防火墙规则冲突。先修正根因，再重启服务。

### 发布端口后 UFW 规则没有生效

Docker 官方文档明确提示，Docker 发布的容器端口可能绕过 `ufw` 或 `firewalld` 规则。生产服务器应同时审查 Docker 的端口发布、云安全组、宿主机防火墙和应用认证；不要仅凭 `ufw status` 判断服务没有对外暴露。主机防火墙的基础观察和规则模型见 [[Linux 主机防火墙与 UFW 基础#12. 与 Tailscale、Docker 和上游防火墙的边界]]，Docker 的底层规则仍以本篇所列官方文档为准。

### 如何升级 Docker Engine

先在测试环境验证新版本，再在维护窗口中查看 apt 候选版本、执行安装并回归验证。不要把不受控的系统全量升级和 Docker 运行时升级混为一次变更：

```bash
apt list --all-versions docker-ce
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
docker version
docker compose version
```

## 官方参考资料

- [Docker：在 Ubuntu 安装 Docker Engine](https://docs.docker.com/engine/install/ubuntu/)
- [Docker：Linux Docker Engine 安装后步骤](https://docs.docker.com/engine/install/linux-postinstall/)
- [Docker：配置 Docker daemon 代理](https://docs.docker.com/engine/daemon/proxy/)
- [Docker：Docker 与 UFW](https://docs.docker.com/engine/network/packet-filtering-firewalls/)

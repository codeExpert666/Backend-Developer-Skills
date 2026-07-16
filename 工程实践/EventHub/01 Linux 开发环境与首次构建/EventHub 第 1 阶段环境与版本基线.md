---
title: EventHub 第 1 阶段环境与版本基线
aliases:
  - EventHub Linux 开发环境版本矩阵
  - EventHub 第 1 阶段资源基线
tags:
  - 工程实践
  - EventHub
  - Linux
  - 开发环境
  - 版本基线
created: 2026-07-17T00:48:00
updated: 2026-07-17T01:13:33
---

本文记录 EventHub 第 1 阶段采用的具体环境决策。它不是通用安装教程；资源如何估算见 [[UTM 虚拟机网络与资源规划]]，工具安装和系统原理通过对应技术笔记学习。

> [!info] 核对时间与状态
> 本基线中的宿主机、仓库和工具事实核对于 **2026-07-16**。它是开始实施前的决策依据，不代表 UTM 虚拟机已经创建，也不代表项目已经在 Linux 中构建成功。执行前必须重新运行本文的核对命令。

## 访问路径

日常主路径：

```text
Mac mini 终端或 IDE
  -> 传统 OpenSSH
  -> Mac mini 内运行的 Ubuntu Server 虚拟机
```

外出可选路径：

```text
MacBook Air
  -> Tailscale 网络
  -> Ubuntu 虚拟机上的传统 OpenSSH
```

Mac mini 连接内部虚拟机仍是真实的 SSH 客户端—服务端流程，可以练习主机指纹、用户密钥、`known_hosts`、`authorized_keys`、`sshd`、远程 Shell 和 Linux 权限。它暂时不覆盖公网、跨地域、运营商 NAT 和云安全组，这些属于后续阶段。

## 宿主机事实与资源决策

| 项目 | 2026-07-16 只读事实或候选值 | 决策含义 | 重新核对 |
| --- | --- | --- | --- |
| CPU | Apple M4，ARM64 | Ubuntu 客户机采用 ARM64，优先虚拟化而非 AMD64 模拟 | `uname -m`、`system_profiler` |
| 内存 | 24GB | 候选分配 12GB，仍为 macOS、IDE 和浏览器保留余量 | `system_profiler`、Activity Monitor |
| 根卷可用空间 | 约 347GiB | 候选稀疏磁盘上限 180GB，但创建前必须重查 | `df -h /` |
| UTM | 当时未安装 | 第 1 阶段必须实际安装和创建后才能验收 | 检查 `/Applications/UTM.app` |
| OrbStack | 已安装，版本 2.2.1 | 只用于说明边界，不代替完整 Ubuntu Server VM 主线 | `orb version` |
| 候选 vCPU | 6 | 支撑并行 Go/Maven 构建和容器，同时避免占满宿主核心 | VM 内 `nproc` |
| 候选内存 | 12GB | 支撑 JDK、Go、IDE 远程进程和少量容器 | VM 内 `free -h` |
| 候选磁盘 | 180GB 稀疏上限 | 为源码、依赖缓存、镜像、卷和后续阶段留余量 | VM 内 `df -hT`，宿主机持续观察实际占用 |

**执行位置：macOS 宿主机（任意目录，只读）**

```bash
uname -m
system_profiler SPHardwareDataType | sed -n '/Chip:/p;/Total Number of Cores:/p;/Memory:/p'
df -h /
test -d /Applications/UTM.app && printf 'UTM installed\n' || printf 'UTM not installed\n'
test -d /Applications/OrbStack.app && printf 'OrbStack installed\n' || printf 'OrbStack not installed\n'
```

这里的 `6 vCPU、12GB、180GB` 只是本阶段起点。若宿主机频繁交换内存，应降低 VM 内存或构建并发；若 Docker 镜像、Maven 缓存和数据库卷增长明显，应在审计数据后扩容，而不是把初始值视为永久标准。

## 客户机与网络基线

| 项目 | 本阶段选择 | 原因 |
| --- | --- | --- |
| 客户机 | Ubuntu Server 24.04 LTS ARM64 | 与 Apple Silicon 架构一致，适合完整 Linux 服务和 systemd 学习 |
| UTM 模式 | Virtualize | 同架构硬件虚拟化比 AMD64 模拟更适合日常开发 |
| 网络主线 | UTM Shared Network | 宿主机与客户机互通，配置变量少 |
| SSH | 传统 OpenSSH | 直接学习 SSH 客户端、`sshd`、密钥和主机身份 |
| Tailscale | 可选 | 只在需要 MacBook Air 外出访问时增加，不阻塞第 1 阶段 |
| 项目目录 | `$HOME/src/eventhub-go`、`$HOME/src/eventhub` | 使用 Linux 本地文件系统，不沿用 macOS 绝对路径或长期运行在共享挂载 |

## 项目工具版本矩阵

版本选择优先级：

1. 当前 checkout 中已跟踪的项目声明和 Wrapper。
2. CI 与 Dockerfile 中的构建基线。
3. 团队批准的支持和安全策略。
4. 没有项目约束时才考虑当前受支持版本。

| 工具 | 2026-07-16 项目事实 | 本阶段策略 | 重新核对 |
| --- | --- | --- | --- |
| Git | 项目未固定精确版本 | 使用 Ubuntu 支持版本并验证实际行为 | README、CI、`git --version` |
| Go | `go.mod` 声明 `1.24.0`；README、Dockerfile 与 CI 一致 | 安装 Go 1.24.0 Linux ARM64 | `go.mod`、README、Dockerfile、CI |
| JDK | Java POM 与 README 要求 21 | 安装 JDK 21 | `pom.xml`、README、`java -version` |
| Maven | 远端基线要求 Maven 3.9+；本地工作区曾存在未跟踪 Wrapper | fresh clone 先按当前 revision 的真实文件选择全局 Maven或 Wrapper | `git ls-files`、README、Wrapper properties |
| Docker | 项目使用 Dockerfile、Compose 和 Testcontainers | 安装受支持的 Engine、Buildx 与 Compose v2 plugin | Docker 官方支持范围、项目 CI |
| Node.js | Go CI 使用 Node 24 运行 OpenAPI lint | 只有完整复现该门禁时安装 Node 24 | 当前 `.github/workflows/ci.yml` |

**执行位置：macOS 宿主机（Go 项目根目录，只读）**

```bash
awk '$1 == "go" { print "go=" $2; exit }' go.mod
awk '$1 == "toolchain" { print "toolchain=" $2; exit }' go.mod
grep -nE 'go-version-file|setup-node|node-version|^FROM golang:' \
  README.md .github/workflows/*.yml Dockerfile 2>/dev/null || true
```

**执行位置：macOS 宿主机（Java 项目根目录，只读）**

```bash
grep -nE 'java\.version|maven\.compiler\.release|Maven 3' pom.xml README.md

for FILE_NAME in mvnw .mvn/wrapper/maven-wrapper.properties Makefile .github/workflows/ci.yml; do
  if git ls-files --error-unmatch "$FILE_NAME" >/dev/null 2>&1; then
    printf 'tracked    %s\n' "$FILE_NAME"
  elif [ -e "$FILE_NAME" ]; then
    printf 'local-only %s\n' "$FILE_NAME"
  else
    printf 'absent     %s\n' "$FILE_NAME"
  fi
done
```

文件存在不等于被 Git 跟踪。任何 `local-only` 工程入口都不会随 fresh clone 出现，必须先提交、生成可审计迁移材料或选择受控工作区复制。

## 工具链验证目标

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
git --version
go version
go env GOOS GOARCH
java -version
javac -version
mvn -version 2>/dev/null || true
docker version
docker compose version
docker info
```

第 1 阶段不是“命令能输出版本”就完成。最终还必须由 [[EventHub 仓库迁移与首次质量门禁]] 证明两个仓库能够在 Linux 本地文件系统中按当前 revision 完整验证。

## 相关知识笔记

- 虚拟化概念：[[虚拟机、客户机与 CPU 架构]]
- UTM 资源与网络：[[UTM 虚拟机网络与资源规划]]
- 创建虚拟机：[[使用 UTM 创建 Ubuntu Server 虚拟机]]
- Ubuntu 初始化：[[Ubuntu Server 初始化与基础安全]]
- SSH：[[OpenSSH 连接、密钥与主机指纹]]
- Tailscale：[[使用 Tailscale 访问 Linux 主机]]
- Linux 工作区：[[Linux 开发工作区与本地文件系统规划]]
- Go：[[Ubuntu 安装 Go]]

---
title: Linux 后端开发目录与工具链规划
aliases:
  - Linux 后端开发工作区规划
  - Ubuntu 后端工具链规划
  - Linux 项目目录与工具链
tags:
  - Linux
  - Linux/开发环境
  - Linux/目录规划
  - 后端开发
  - 开发工具链
created: 2026-07-16T00:31:00
updated: 2026-07-16T01:17:59
---

本文负责把 Linux 开发机中的源码目录、Git、Go、JDK、Maven Wrapper、Docker Engine、Compose plugin 和少量项目专用工具串成一条清晰主线。它不重复已有安装教程，而是回答三个更容易被忽略的问题：项目应放在哪里、每个工具解决哪一层问题、怎样证明当前 Shell 与 Docker daemon 都处于可构建状态。

本阶段的日常路径是 Mac mini 终端或 IDE 通过 SSH 进入 Ubuntu 虚拟机，然后在 **Ubuntu 的本地文件系统**中编辑、构建和测试。MacBook Air 经 Tailscale 远程访问只是可选扩展，不改变 Linux 内部的目录与工具链安排。

## 第 1 阶段阅读位置

本组 Linux 主线建议按下列顺序阅读。本文处于“系统与远程访问已经可用，准备迁移项目”之间。

| 顺序 | 笔记 | 解决的问题 |
| --- | --- | --- |
| 1 | [[Linux 后端开发虚拟机搭建概览]] | 明确阶段目标、主路径、完成清单和后续边界 |
| 2 | [[使用 UTM 创建 Ubuntu Server 开发虚拟机]] | 创建与宿主架构匹配的 Ubuntu Server 虚拟机 |
| 3 | [[Ubuntu Server 开发机初始化与安全基线]] | 建立用户、权限、时区、更新、服务和基础安全状态 |
| 4 | [[从 macOS 使用 SSH 连接 Linux 虚拟机]] | 建立稳定的 SSH 客户端—服务端访问路径 |
| 5 | [[使用 Tailscale 远程访问 Linux 开发机]] | 按需扩展 MacBook Air 外出访问，不作为必需条件 |
| 6 | [[Linux 后端开发目录与工具链规划]] | 规划本地源码目录，安装并验证开发工具角色 |
| 7 | [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]] | 迁移两仓库并执行真实构建、测试和质量门禁 |
| 8 | [[Linux 开发虚拟机备份恢复与常见问题]] | 建立快照、源码备份、恢复演练和故障处理基线 |

## 目标与完成标准

| 检查项 | 应达到的状态 | 典型验证 |
| --- | --- | --- |
| 源码目录 | 两个仓库位于 `$HOME/src`，属于当前用户 | `findmnt -T`、`stat`、`git status` |
| Git | 命令、提交身份与远程访问方式清楚 | `git --version`、`git config --list --show-origin` |
| Go | 版本满足 `go.mod`，OS/架构为 Linux ARM64 | `go version`、`go env GOOS GOARCH` |
| Java | `java` 与 `javac` 为项目要求的 JDK | `java -version`、`javac -version` |
| Maven | 先确认 Wrapper 是否被 Git 跟踪；有则优先，没有则按项目声明安装 Maven | `git ls-files`、`./mvnw -version` 或 `mvn -version` |
| Docker CLI 与 daemon | Client、Server、服务状态都正常 | `docker version`、`systemctl is-active docker` |
| Compose plugin | 使用 `docker compose` 而非旧 `docker-compose` v1 | `docker compose version` |
| 项目附加工具 | 只按真实门禁安装，例如 Go OpenAPI lint 所需 Node | `node --version`、`npx --version` |
| 权限 | 构建和 Git 操作不依赖 `sudo` | `id`、`find ... -not -user "$USER"` |

## 1. 为什么项目放在 Linux 本地文件系统

虚拟机能看见 macOS 共享目录，并不意味着共享目录适合长期开发。共享机制需要在两个不同操作系统和文件系统语义之间做映射，可能在权限位、所有者、大小写敏感、符号链接、文件锁、Unix socket、`inotify` 文件监听、可执行位和大量小文件性能上存在差异。

后端项目会频繁创建或读取：

- Git 的 `.git/objects`、索引和锁文件。
- Go module / build 缓存与大量小文件。
- Maven 本地仓库、编译输出和 Wrapper 文件。
- Docker bind mount、socket、数据库数据与生成代码。
- IDE 索引、语言服务器临时文件和测试报告。

因此主线是：

```text
macOS 宿主机
  └─ 终端 / IDE / SSH 客户端
       └─ SSH
            └─ Ubuntu 虚拟机本地磁盘
                 └─ $HOME/src
                      ├─ eventhub-go
                      └─ eventhub
```

UTM 共享目录只适合作为受控导入、导出或临时交换区。iCloud 同步目录也留在 macOS 一侧，不把它当作 Linux 活跃工作区。项目最终应通过 Git 远程、bundle、patch 或经过核对的复制进入 `$HOME/src`；具体选择见 [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]]。

### 本地磁盘与共享挂载的判断方法

不要根据目录名猜测。让 Linux 显示该路径实际位于哪个挂载点和文件系统。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
findmnt -T "$HOME"
findmnt -T "$HOME/src" 2>/dev/null || true
df -hT "$HOME"
stat -f -c 'filesystem_type=%T path=%n' "$HOME"
```

本地 Ubuntu 安装常见文件系统是 ext4，但实际结果取决于安装方式；不要把设备名或文件系统类型写死。若 `$HOME/src` 显示为 UTM 共享相关的网络/FUSE/virtiofs/9p 挂载，应重新选择客户机本地目录。

## 2. 建立 `$HOME/src` 工作区

`$HOME/src` 的优点是当前用户天然拥有、路径简短、与语言无关，也不需要为日常 Git 和构建操作使用 `sudo`。它不是 Linux 标准强制目录，而是本组笔记的个人开发机约定。

不要把个人仓库放到 `/root`、`/usr/local/src`、`/opt` 或 `/srv`：这些目录更偏向系统管理、共享源代码或部署数据，通常需要 root 权限。也不必沿用 macOS 的绝对路径；客户机拥有独立的 HOME 和目录树。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
install -d -m 0750 "$HOME/src"
stat -c 'owner=%U group=%G mode=%a path=%n' "$HOME/src"
test -r "$HOME/src"
test -w "$HOME/src"
test -x "$HOME/src"
findmnt -T "$HOME/src"
```

预期所有者是当前用户，且读、写、进入目录三个测试都成功。`0750` 表示当前用户可读写进入、同组用户可读进入、其他用户无权限；个人机器也可以按需要改成更严格的 `0700`。

### 推荐目录结构

```text
$HOME/
├── src/
│   ├── eventhub-go/          Go 项目仓库
│   └── eventhub/             Java 项目仓库
├── go/
│   ├── bin/                  go install 的个人工具，默认位置
│   └── pkg/mod/              Go module 缓存，按实际 go env 为准
├── .cache/go-build/          Go 构建缓存，按实际 go env 为准
├── .m2/
│   ├── repository/           Maven 本地依赖仓库
│   └── wrapper/              Maven Wrapper 下载内容
├── .docker/                  当前用户的 Docker CLI 配置
├── .gitconfig                当前用户的 Git 全局配置
└── .ssh/                     SSH 客户端配置、密钥和 known_hosts
```

`$HOME/go` 是默认 GOPATH，不代表现代 Go Modules 项目必须放回 `$HOME/go/src`。项目源码统一放 `$HOME/src`，语言缓存由各工具管理，能避免把源码、依赖缓存和生成物混在一个目录中。

系统级工具和服务通常在 HOME 之外：

| 位置 | 典型内容 | 谁管理 |
| --- | --- | --- |
| `/usr/local/go` | Go 官方工具链 | root 安装，普通用户只读执行 |
| `/usr/lib/jvm` | Ubuntu 安装的 JDK | APT / root |
| `/usr/bin/git`、`/usr/bin/docker` | 系统命令入口 | APT / root |
| `/var/run/docker.sock` | Docker CLI 与 daemon 通信的 Unix socket | systemd / Docker |
| `/var/lib/docker` | rootful Docker Engine 常见默认数据目录 | 以 `docker info` 的 `Docker Root Dir` 为准；Rootless 或自定义 `data-root` 会不同，不要手工修改 |

## 3. 每个工具负责什么

| 工具 | 它解决的问题 | 它不等于什么 |
| --- | --- | --- |
| Git | 保存提交历史、分支和远程协作状态 | 不等于 GitHub，也不自动备份未提交文件 |
| Go 工具链 | 编译、测试、格式化和管理 Go module | 不等于 Docker 中的 Go 构建镜像 |
| JDK | 提供 JVM、`java`、`javac` 等开发工具 | 只装 JRE 不能完成 Java 编译 |
| Maven Wrapper | 在 Wrapper 文件被仓库跟踪时，让项目使用固定的 Maven 版本 | 不会替你安装 JDK；本地未跟踪的 Wrapper 也不会随 clone 出现 |
| Docker CLI | 解析 `docker` 命令并向 daemon 发请求 | CLI 存在不代表 daemon 已启动 |
| Docker daemon | 构建镜像、拉取镜像、运行容器和管理卷/网络 | 不是当前 Shell 中的普通前台进程 |
| Compose plugin | 让 Docker CLI 读取 Compose 文件并编排多服务 | 不是另一套 Docker daemon |
| Node.js / npx | 执行特定前端或规范检查工具 | 不是 EventHub Go 服务的运行时前置条件 |

### 两种 SSH 场景不要混为一件事

Mac mini SSH 登录 Ubuntu 时：Mac 是 SSH 客户端，Ubuntu 的 `sshd` 是服务端。进入 Ubuntu 后，如果使用 `git@github.com:...` clone 或 push，则 **Ubuntu 又成为连接代码托管平台的 SSH 客户端**。

两条连接使用相同的 SSH 协议思想，但服务端、主机指纹、`known_hosts` 记录、密钥授权和用途不同。默认不要把 Mac 的私钥复制进虚拟机；可以为 Linux 开发机创建独立凭据并在代码平台登记。Git 认证与配置详见 [[Git 凭据、SSH 与常见问题排查]]，登录 Linux 的流程见 [[从 macOS 使用 SSH 连接 Linux 虚拟机]]。

## 4. 安装顺序：先事实源，后工具

推荐顺序不是按“哪个语言更重要”，而是按依赖关系与可验证性排列。

### 第一步：完成系统基线和通用工具

先完成 [[Ubuntu Server 开发机初始化与安全基线]]。随后安装后端构建常用的基础命令；不要把系统全量升级混入单个工具安装任务。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
sudo apt update
sudo apt install -y ca-certificates curl make build-essential unzip rsync
```

- `ca-certificates`、`curl`：安全下载官方文件。
- `make`：执行 Go、Java 仓库提供的 Makefile 入口。
- `build-essential`：提供编译器等基础能力，Go race/CGO 或本地原生依赖可能需要。
- `unzip`：Maven Wrapper 解压 Maven 发行包时需要。
- `rsync`：需要受控复制本地工作区时使用。

### 第二步：安装和配置 Git

按以下顺序复用现有笔记，不在本文复制整套命令：

1. [[Git 安装与初始配置概览]]：理解安装、配置、远程认证边界。
2. [[Ubuntu 从零安装 Git]]：通过 Ubuntu APT 安装并核对来源。
3. [[Git 常用配置与本地验证]]：配置提交身份、默认行为和作用域。
4. [[Git 凭据、SSH 与常见问题排查]]：按需配置代码托管平台认证。

Git 安装后即可 clone 一个干净基线，从仓库声明中读取后续版本约束。若本地 Mac 仓库有未推送提交或未提交修改，先按迁移专题选择 bundle、patch 或受控复制；不要假设 clone 会带上这些内容。

### 第三步：按 `go.mod` 安装 Go

阅读 [[Ubuntu 安装 Go]]，按项目 `go.mod`、CI 和 Dockerfile 选择版本。本次 EventHub Go 案例在 2026-07-16 声明 Go 1.24.0，并运行在 Ubuntu ARM64，因此示例使用官方 `linux-arm64` 归档。

### 第四步：按 `pom.xml` 与 CI 安装 JDK

阅读：

- [[Java 与 Maven 环境搭建概览]]
- [[Ubuntu 安装 Java 与 Maven]]
- [[Java 版本管理与环境变量配置]]
- [[Maven 常用配置与仓库管理]]
- [[Java 与 Maven 环境排障与维护]]

本次 EventHub Java 案例在 2026-07-16 的 `pom.xml` 与仓库 README 都要求 JDK 21。以后重新搭建时仍要再次读取当前 checkout，不把 JDK 21 当作所有 Java 项目的通用答案。

### 第五步：先判断 Maven Wrapper 是否真属于仓库

Maven Wrapper 由 `mvnw`、`mvnw.cmd` 和 `.mvn/wrapper/` 共同组成。只有这些文件被 Git 跟踪，fresh clone 才能稳定获得 Wrapper；不能仅因为某台开发机看得到 `mvnw`，就把它当作远端仓库固定入口。

先检查当前 checkout 的跟踪状态，再选择命令：

**执行位置：Ubuntu 虚拟机（Java 项目根目录）**

```bash
cd "$HOME/src/eventhub"
test -f pom.xml

if git ls-files --error-unmatch \
  mvnw \
  .mvn/wrapper/maven-wrapper.properties \
  >/dev/null 2>&1; then
  test -x ./mvnw
  grep -n 'distributionUrl=' .mvn/wrapper/maven-wrapper.properties
  ./mvnw -version
else
  printf '%s\n' '当前 checkout 没有完整、已跟踪的 Maven Wrapper；按 README/pom.xml 使用全局 Maven。'
  mvn -version
fi
```

当前远端可 clone 基线的 README 要求 Java 21 与 Maven 3.9+；因此没有已跟踪 Wrapper 时，应按 [[Ubuntu 安装 Java 与 Maven]] 安装满足该约束的 Maven。若某个未来分支完整跟踪 Wrapper，则以该分支的 `distributionUrl` 与校验配置为准，并优先使用 `./mvnw`。

无论走哪条路线，都要检查输出中的 Java version / Java home 是否指向 JDK 21。若已跟踪的 `mvnw` 在正常 Git clone 后丢失可执行位，应先检查 Git 记录的 mode 和当前文件系统，而不是立即用另一套 Maven 绕过。

### 第六步：安装 Docker Engine 与 Compose plugin

先阅读 [[Docker 安装概览]] 理解平台选择，再按 [[Ubuntu 安装 Docker]] 使用 Docker 官方 APT 仓库安装 Engine、CLI、Buildx 与 Compose plugin。

安装 `docker` 命令只是 Client 层完成。还必须验证 daemon、socket、镜像拉取和真实容器：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
docker version
docker info
systemctl is-active docker.service
systemctl is-enabled docker.service
docker compose version
docker run --rm hello-world
```

`docker version` 应同时出现 Client 和 Server；如果只有 Client，通常是 daemon 未启动、当前 context 不正确或 socket 权限不足。

> [!warning] `docker` 组接近 root 权限
> 能访问 Docker daemon 的用户通常可以挂载宿主目录、启动特权容器并影响主机。个人开发虚拟机可以评估后把唯一开发账号加入 `docker` 组，但这不是普通、低风险权限。共享机或生产机应采用最小授权、受审计 sudo、Rootless mode 或组织平台。不要用 `chmod 666 /var/run/docker.sock` 解决权限报错。

加入组后，最可靠的生效方法是退出并新建 SSH 会话，再执行 `id` 和 `docker info`。不要只看 `usermod` 退出码就认为现有会话已经获得新组。

### 第七步：只在完整复现 Go CI 时补 Node.js

EventHub Go 的服务运行、`go build` 和大部分测试不依赖 Node.js。当前 Go CI 使用 Node 24，仅为了通过 `npx` 运行固定版本 Redocly CLI，也就是 `make openapi-lint`。

因此：

- 只学习 Go 或运行服务时，不必为了“工具齐全”先装 Node。
- 要在 Linux 上完整复现 Go 仓库当前 CI 时，需要 Node 24、npm 与 npx。
- 版本仍应以 `.github/workflows/ci.yml` 和 Makefile 为准；CI 调整后不要继续照抄 Node 24。
- 使用 Node 官方发行版或组织批准的版本管理方案，不执行来源不明的安装脚本。

**执行位置：Ubuntu 虚拟机（任意目录，已按批准方案安装 Node 后）**

```bash
node --version
npm --version
npx --version
```

Node 的完整安装和多版本管理不属于本文主线。官方版本与下载入口见文末参考资料。

## 5. 本次 EventHub 工具版本矩阵

此表是 **2026-07-16 的实操核对记录**，不是永久固定要求。重新搭建或切换分支时必须再次读取仓库。

| 工具 | 当前项目事实源 | 本次案例要求或策略 |
| --- | --- | --- |
| Git | README、团队流程、Ubuntu 支持包 | 项目未固定精确版本；使用 Ubuntu 受支持版本并验证行为 |
| Go | `eventhub-go/go.mod`、README、Dockerfile、Go CI | `go.mod` 为 1.24.0；本案例安装 Go 1.24.0 Linux ARM64 |
| JDK | `eventhub/pom.xml`、README | JDK 21 |
| Maven | README，以及当前分支中已被 Git 跟踪的 Wrapper 文件 | 当前远端基线要求 Maven 3.9+；若分支完整跟踪 Wrapper，则以 Wrapper 为准 |
| Docker / Compose | Dockerfile、Compose、CI 与官方支持范围 | 安装受支持的 Engine 与 Compose v2 plugin；项目未把 daemon 固定成一个永久版本 |
| Node.js | `eventhub-go/.github/workflows/ci.yml` | Node 24，仅用于完整复现 OpenAPI lint 门禁 |

版本选择优先级：

1. 当前 checkout 中已跟踪的项目声明与 Wrapper。
2. CI 与容器构建基线。
3. 组织批准的安全和支持策略。
4. 没有项目约束时，才考虑当前受支持稳定版本。

## 6. 原生运行与容器运行的边界

| 方式 | 典型用途 | 优点 | 需要留意 |
| --- | --- | --- | --- |
| Linux 原生 Go/JDK | 编辑、编译、单元测试、IDE/语言服务器 | 启动快、调试直接、缓存复用方便 | 本机版本必须满足项目约束 |
| Maven Wrapper 原生进程 | Java 编译、测试、打包和插件门禁 | 只有 Wrapper 文件已被当前 revision 跟踪时，Maven 版本才由仓库固定 | 仍依赖本机正确 JDK 与网络；fresh clone 后要先确认 `mvnw` 与 `.mvn/` 确实存在 |
| Docker 容器 | MySQL、Redis、应用镜像、隔离集成环境 | 服务依赖可复现、容易重建 | daemon 权限、镜像网络、卷数据和磁盘占用 |
| Docker Compose | 一次编排多个容器及依赖顺序 | 接近项目定义的整套环境 | 静态配置通过不代表运行一定成功 |

二者不是互斥路线。典型后端开发循环是：代码和语言工具链原生运行，MySQL/Redis 等依赖运行在容器中，最后再构建应用镜像。Go 测试还可能通过 Testcontainers 调用 Docker daemon；这说明“原生执行 `go test`”也可能间接启动容器。

不要把项目源码、数据库数据卷和 Docker daemon 数据目录混为一体：

- 项目源码由 Git 管理并另做源码备份。
- Compose named volume 由 Docker 管理，需要独立数据库备份策略。
- rootful Engine 常把 `/var/lib/docker` 用作 daemon 数据目录，但 Rootless 或自定义 `data-root` 会不同；以 `docker info` 为准，且不应手工复制其中单个文件或把它当作 Git 工作区。
- 虚拟机快照能帮助整体回退，但不替代 Git 远程和数据库逻辑备份。

## 7. 统一验证当前工具链

### 7.1 系统级命令和 daemon

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
printf 'user=%s\nhome=%s\nshell=%s\n' "$USER" "$HOME" "$SHELL"
uname -a
dpkg --print-architecture

command -v git
command -v go
command -v java
command -v javac
command -v mvn || true
command -v docker

git --version
go version
go env GOOS GOARCH GOROOT GOPATH GOTOOLCHAIN
java -version
javac -version
mvn -version 2>/dev/null || true
docker version
docker compose version
systemctl is-active docker.service
```

某一命令失败时先停在该层排查，不要继续运行项目门禁。例如 `docker compose version` 成功但 `docker version` 没有 Server，说明 Compose plugin 已安装，却仍无法运行容器。

### 7.2 Go 项目约束

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
cd "$HOME/src/eventhub-go"
test -f go.mod
awk '$1 == "go" { print "go directive=" $2; exit }' go.mod
go version
go env GOOS GOARCH GOMOD
git status --short --branch
```

### 7.3 Java 项目约束与 Wrapper

**执行位置：Ubuntu 虚拟机（Java 项目根目录）**

```bash
cd "$HOME/src/eventhub"
test -f pom.xml
grep -n 'java.version' pom.xml

if git ls-files --error-unmatch \
  mvnw \
  .mvn/wrapper/maven-wrapper.properties \
  >/dev/null 2>&1; then
  grep -n 'distributionUrl=' .mvn/wrapper/maven-wrapper.properties
  ./mvnw -version
else
  mvn -version
fi

git status --short --branch
```

### 7.4 可选 Node 门禁依赖

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
if command -v node >/dev/null 2>&1; then
  node --version
  npm --version
  npx --version
else
  printf '%s\n' 'Node.js 未安装；只有复现 Go OpenAPI lint 门禁时才需要。'
fi
```

完成这些版本和服务验证后，再进入 [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]] 执行构建、测试、lint、生成物和 Compose 静态门禁。不要把“命令能输出版本”误写成“项目已经构建成功”。

## 8. 从共享目录受控导入时怎么做

优先使用 Git clone。只有需要带入未推送提交、未提交修改或尚未纳入 Git 的文件时，才考虑 bundle、patch 或 `rsync`。复制前后都要检查 Git 状态，不覆盖一个已存在且内容未知的 `$HOME/src/eventhub-go`。

下面只做范围盘点与 dry-run，不在工具链规划篇直接执行真实复制。先把 `SHARED_IMPORT_DIR` 改为 UTM 实际共享挂载路径；变量形式避免在命令中放裸占位符。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
SHARED_IMPORT_DIR="$HOME/utm-share"
PROJECT_NAME='eventhub-go'
SOURCE_REPO="$SHARED_IMPORT_DIR/$PROJECT_NAME"
TARGET_REPO="$HOME/src/$PROJECT_NAME"

if ! test -d "$SOURCE_REPO/.git"; then
  printf '停止：源目录不是可识别的 Git 工作区：%s\n' "$SOURCE_REPO" >&2
elif test -e "$TARGET_REPO"; then
  printf '停止：目标目录已经存在：%s\n' "$TARGET_REPO" >&2
else
  git -C "$SOURCE_REPO" status --short --branch
  git -C "$SOURCE_REPO" status --short --ignored
  git -C "$SOURCE_REPO" ls-files --others --exclude-standard
  git -C "$SOURCE_REPO" ls-files --others --ignored --exclude-standard
  rsync -a --no-owner --no-group --dry-run --itemize-changes \
    "$SOURCE_REPO/" "$TARGET_REPO/"
fi
```

逐项审查 ignored、untracked 与 dry-run 输出，排除密钥、令牌、数据库转储和缓存；确认目标层级正确后，再按 [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]] 的完整流程冻结源仓库写入、执行真实复制并比较两端状态。`rsync -a` 加上源目录末尾 `/` 会复制目录内容，包括 `.git` 和普通隐藏文件；它不是“自动解决冲突”的工具。共享文件系统若不能表达 Unix 可执行位，复制到本地后还要核对 `mvnw`、脚本和 Git 记录的 mode。

## 9. 权限与磁盘维护

### 日常构建不要使用 sudo

Git 仓库、Go 缓存、Maven 本地仓库应由当前用户管理。检查 `$HOME/src` 中是否出现其他所有者：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
find "$HOME/src" -xdev -not -user "$USER" -printf '%u:%g %m %p\n' | sed -n '1,80p'
```

有输出时，先查清来源，并保存目标当前的 owner、group 与 mode。不要把整棵源码目录递归改属主；只对已经逐项确认、确实由误用 `sudo` 创建的单个文件或目录处理。下面的 guard 只允许选择 `$HOME/src` 下的一个明确路径，并在修改前后打印状态：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
SOURCE_ROOT=$(realpath -e -- "$HOME/src") || exit 1
printf '要修复的单个绝对路径: '
IFS= read -r AFFECTED_PATH

if [ -L "$AFFECTED_PATH" ]; then
  printf '停止：目标本身是符号链接，不执行 sudo chown。\n' >&2
elif ! CANONICAL_PATH=$(realpath -e -- "$AFFECTED_PATH"); then
  printf '停止：目标不存在或无法解析。\n' >&2
else
  case "$CANONICAL_PATH" in
    "$SOURCE_ROOT"/*)
      stat -c 'before owner=%U:%G uid=%u gid=%g mode=%a path=%n' "$CANONICAL_PATH"
      sudo chown -- "$USER":"$(id -gn)" "$CANONICAL_PATH"
      stat -c 'after  owner=%U:%G uid=%u gid=%g mode=%a path=%n' "$CANONICAL_PATH"
      ;;
    *)
      printf '停止：规范化后的路径不在 %s/ 内。\n' "$SOURCE_ROOT" >&2
      ;;
  esac
fi
```

> [!warning] 不要用 `chown -R` 掩盖根因
> 上述 guard 使用 `realpath` 消解 `..` 和父目录中的符号链接，并拒绝把最终符号链接本身交给 `chown`；规范化结果必须仍是 `$HOME/src` 的子项。绝不能把目标改成 `$HOME/src` 本身、`$HOME`、`/usr/local`、Docker 数据目录或根目录。若一整棵目录都异常，应先停止产生 root 文件的命令、保存所有权清单并制定单独恢复方案；共享机上还要由管理员确认。误改时按修改前 `stat` 记录的数字 UID/GID 恢复，而不是猜测。

### 分层查看磁盘占用

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
df -hT "$HOME"
du -sh "$HOME/src" "$HOME/go" "$HOME/.m2" "$HOME/.cache/go-build" 2>/dev/null || true
docker system df
```

源码、依赖缓存、构建缓存、Docker 镜像和卷的清理方式不同。不要看到磁盘紧张就执行网上复制来的 `docker system prune -a --volumes`、`go clean -modcache` 或删除整个 `~/.m2/repository`；先确认占用者、是否可重新下载、是否包含数据库卷，再选择可恢复的最小清理。

## 10. 常见问题与恢复

### 命令存在，但版本不是刚安装的版本

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
type -a git go java javac docker 2>/dev/null || true
readlink -f "$(command -v go)" 2>/dev/null || true
readlink -f "$(command -v java)" 2>/dev/null || true
printf '%s\n' "$PATH" | tr ':' '\n'
```

PATH 中靠前的命令获胜。先识别 APT、官方 tarball、SDK 或版本管理器各自来源，再调整；不要直接删除 `/usr/bin` 文件。

### Docker CLI 正常，Server 连接失败

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
docker context show
docker context ls
systemctl status docker.service --no-pager
ls -l /var/run/docker.sock
id
sudo journalctl -u docker.service -n 100 --no-pager
```

常见原因是服务未启动、当前会话尚未加载 `docker` 组、连接到错误 context、代理配置错误或磁盘不足。不要用放宽 socket 到所有用户的方式绕过。

### Maven Wrapper 下载失败

先确认 JDK、当前 Maven 入口与仓库配置，再排查 DNS、代理、证书和 Maven Central：

**执行位置：Ubuntu 虚拟机（Java 项目根目录）**

```bash
cd "$HOME/src/eventhub"
java -version

if git ls-files --error-unmatch .mvn/wrapper/maven-wrapper.properties >/dev/null 2>&1; then
  grep -n 'distributionUrl=' .mvn/wrapper/maven-wrapper.properties
else
  mvn -version
fi

getent hosts repo.maven.apache.org
curl -I https://repo.maven.apache.org/maven2/
```

如果当前分支已完整跟踪 Wrapper，不要用全局 Maven 掩盖 Wrapper 下载或校验失败；如果没有 Wrapper，也不要凭空创建一个来改写本阶段范围。详细处理见 [[Maven 常用配置与仓库管理]] 和 [[Java 与 Maven 环境排障与维护]]。

### 误把仓库放进共享目录

先停止 IDE、构建和容器 bind mount；记录 `git status --short --branch`、远程、当前提交与本地未跟踪文件。然后按迁移专题复制到一个**新的** `$HOME/src` 目标目录，验证完整后再把旧共享目录改名留存。不要在没有比较和备份的情况下直接删除旧目录。

### 想撤销本阶段目录规划

目录规划本身不要求删除数据。安全撤销顺序是：

1. 停止项目进程和使用该路径的容器。
2. 核对两个仓库的 Git 状态，处理未提交和未推送内容。
3. 将 `$HOME/src` 改名为带时间戳的停用目录，而非立即删除。
4. 在新位置 clone 或恢复，完成构建验证。
5. 确认备份后再单独决定旧目录处理方式。

## 11. 本文不负责的内容

到这里仍然只是在建立 Linux **开发环境**：

- 不部署云服务器或公网服务。
- 不建立正式 CI/CD、制品仓库或发布审批。
- 不设计生产监控、告警、日志平台和回滚编排。
- 不引入 Kubernetes。
- 不把开发机 Docker Compose 直接视为生产部署方案。

两项目真实构建与质量门禁属于下一篇 [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]]；快照、源码备份和恢复演练属于 [[Linux 开发虚拟机备份恢复与常见问题]]。完成这两篇后，第 1 阶段才算闭环。

## 官方与项目参考资料

以下易变化资料于 **2026-07-16** 核对；执行安装时应重新检查官方支持范围和当前项目声明。

- [Go：Managing source code](https://go.dev/doc/modules/managing-source)
- [Go：Download and install](https://go.dev/doc/install)
- [Git：Installing Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Apache Maven：Maven Wrapper](https://maven.apache.org/wrapper/)
- [Docker：Docker overview](https://docs.docker.com/get-started/docker-overview/)
- [Docker：Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Docker：Linux post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/)
- [Node.js：官方下载页](https://nodejs.org/en/download)
- [UTM：Linux guest support](https://docs.getutm.app/guest-support/linux/)
- EventHub Go 项目自身：`go.mod`、`README.md`、`Makefile`、`.github/workflows/ci.yml`、`Dockerfile`、`docker-compose.yml`
- EventHub Java 项目自身：`pom.xml`、`mvnw`、`.mvn/wrapper/maven-wrapper.properties`、`README.md`、`Makefile`、`.github/workflows/ci.yml`

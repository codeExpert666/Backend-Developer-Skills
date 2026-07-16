---
title: Linux 开发工作区与本地文件系统规划
aliases:
  - Linux 后端开发目录与工具链规划
  - Linux 后端开发工作区规划
  - Ubuntu 后端工具链规划
  - Linux 项目目录与工具链
tags:
  - Linux
  - Linux/文件与目录
  - Linux/开发环境
  - 后端开发
  - 文件系统
created: 2026-07-16T00:31:00
updated: 2026-07-17T01:12:07
---

本文解决三个问题：Linux 开发项目长期放在哪里、为什么不直接在宿主机共享目录中开发，以及怎样把源码、工具链、缓存、容器数据和备份分开管理。

这里的原则适用于虚拟机、物理 Linux 开发机和远程服务器。某个项目的目录名、精确工具版本和质量门禁属于工程实践基线，不应写成所有 Linux 环境的固定答案。

## 1. 首选 Linux 本地文件系统

虚拟机能访问宿主机共享目录，不代表共享目录适合长期承载活跃仓库。共享机制需要映射两套操作系统和文件系统语义，可能在以下方面出现差异：

- 文件所有者、用户组和权限位。
- 大小写敏感性与文件名规范。
- 符号链接、硬链接和 Unix socket。
- Git 索引、锁文件和可执行位。
- `inotify` 等文件监听机制。
- 大量小文件、依赖缓存和编译输出性能。
- Docker bind mount 和数据库数据一致性。

后端项目会频繁读写 `.git/objects`、Go module 缓存、Maven 本地仓库、编译产物、测试报告、容器构建上下文和 IDE 索引。主工作区因此应位于 Linux 自己管理的本地文件系统。

```text
宿主机终端或 IDE
    -> SSH
    -> Linux 主机
         -> 本地文件系统中的 $HOME/src
```

共享目录适合：

- 临时导入安装包、归档或补丁。
- 把构建报告、日志或导出文件传回宿主机。
- 迁移前后的受控交换。

共享目录不适合：

- 长期活跃 Git 工作树。
- 数据库数据目录。
- Docker volume 的替代目录。
- 需要稳定文件监听的源码树。
- 唯一备份副本。

## 2. 不根据目录名猜测挂载类型

让 Linux 显示目标路径真实位于哪个挂载点。

**执行位置：Linux 开发机（任意目录）**

```bash
findmnt --target "$HOME"
df -hT "$HOME"
stat -f -c 'filesystem_type=%T path=%n' "$HOME"
```

准备工作区后继续检查：

**执行位置：Linux 开发机（任意目录）**

```bash
findmnt --target "$HOME/src"
df -hT "$HOME/src"
namei -l "$HOME/src"
```

本地 Ubuntu 常见 ext4，但不能把文件系统类型或设备名写死。判断依据是挂载来源和你实际采用的虚拟磁盘布局，而不是目录恰好叫 `src`。

## 3. 建立用户级源码根目录

个人开发机可以统一使用 `$HOME/src`：

- 当前用户天然拥有主目录。
- Git、构建和测试不需要 `sudo`。
- 路径与编程语言无关。
- 源码与系统软件、部署目录保持边界。

这是一项开发约定，不是 Linux 强制标准。多人共享机器应结合组织的用户、组和权限模型重新规划。

**执行位置：Linux 开发机（任意目录）**

```bash
install -d -m 0750 "$HOME/src"
stat -c 'owner=%U group=%G mode=%a path=%n' "$HOME/src"
test -r "$HOME/src"
test -w "$HOME/src"
test -x "$HOME/src"
findmnt --target "$HOME/src"
```

`0750` 表示当前用户可读、写、进入，同组用户可读、进入，其他用户无权限。仅个人使用且无需同组访问时，可以选择 `0700`。

不要把个人活跃仓库默认放到：

| 路径 | 不适合作为个人工作区的原因 |
| --- | --- |
| `/root` | 只属于 root，日常开发会被迫提权 |
| `/usr/local/src` | 更偏向系统管理员维护的本地源码 |
| `/opt` | 常用于可选软件安装 |
| `/srv` | 常用于服务对外提供的数据 |
| `/tmp` | 生命周期和清理策略不适合作为长期源码 |

## 4. 使用变量表达项目目录

**执行位置：Linux 开发机（任意目录）**

```bash
printf '请输入项目目录名：'
IFS= read -r PROJECT_NAME

case "$PROJECT_NAME" in
  ''|*/*|.|..)
    printf '%s\n' '停止：目录名不能为空、不能包含斜杠，也不能是点目录。' >&2
    exit 1
    ;;
esac

PROJECT_DIR="$HOME/src/$PROJECT_NAME"

if [ -e "$PROJECT_DIR" ]; then
  printf '停止：目标已经存在：%s\n' "$PROJECT_DIR" >&2
  exit 1
fi

install -d -m 0750 "$PROJECT_DIR"
printf 'project_dir=%s\n' "$PROJECT_DIR"
```

若下一步准备 `git clone`，通常只创建 `$HOME/src`，让 Git 创建最终项目目录即可。上面的命令更适合创建非 Git 工作目录或迁移暂存目录。

推荐的通用结构：

```text
$HOME/
├── src/
│   ├── project-a/
│   └── project-b/
├── go/
│   ├── bin/
│   └── pkg/mod/
├── .cache/
│   └── go-build/
├── .m2/
│   ├── repository/
│   └── wrapper/
├── .docker/
├── .gitconfig
└── .ssh/
```

现代 Go Modules 项目不要求位于 `$HOME/go/src`。源码可以统一放 `$HOME/src`，`GOPATH` 和各种缓存目录继续由工具链管理。

## 5. 区分源码、工具链、缓存和服务数据

| 类型 | 常见位置 | 管理原则 |
| --- | --- | --- |
| 项目源码 | `$HOME/src/$PROJECT_NAME` | Git 与当前用户管理 |
| Go 工具链 | `/usr/local/go` 或发行版目录 | 系统级安装，普通用户只读执行 |
| Go module 与构建缓存 | `go env GOMODCACHE GOCACHE` | 可再生，但清理前先评估下载成本 |
| JDK | `/usr/lib/jvm` 等 | APT 或批准的版本管理方式 |
| Maven 本地仓库 | `$HOME/.m2/repository` | 用户缓存，不等于源码备份 |
| Docker CLI 配置 | `$HOME/.docker` | 可能包含 context 或认证信息，应保护权限 |
| Docker daemon 数据 | `docker info` 显示的 Docker Root Dir | 由 daemon 管理，不手工移动单个子目录 |
| 应用数据库 | 项目明确的数据目录或 volume | 需要独立逻辑备份与恢复方法 |

检查真实位置：

**执行位置：Linux 开发机（任意目录）**

```bash
go env GOPATH GOMODCACHE GOCACHE 2>/dev/null || true
mvn -version 2>/dev/null || true
docker info --format '{{.DockerRootDir}}' 2>/dev/null || true
```

缓存可以重新生成，但会消耗网络和时间；数据库与用户上传文件通常不可简单重建。两者不能采用相同清理策略。

## 6. 工具角色与边界

| 工具 | 负责什么 | 不代表什么 |
| --- | --- | --- |
| Git | 提交历史、分支和远程协作 | 不自动保存未提交或被忽略文件 |
| Go 工具链 | Go 编译、测试、格式化和 module | 不等于容器中的 Go 构建阶段 |
| JDK | JVM、`java`、`javac` 等 | 只装 JRE 不足以编译 |
| Maven / Wrapper | Java 依赖与构建生命周期 | Wrapper 不会替你安装 JDK |
| Docker CLI | 向 Docker daemon 发请求 | CLI 存在不代表 daemon 正常 |
| Docker daemon | 镜像、容器、网络和 volume | 不是当前 Shell 的普通前台进程 |
| Compose plugin | 解析并编排多服务配置 | 配置解析成功不等于服务健康 |

具体安装由各专题负责：

- Git：[[Git 安装与初始配置概览]]、[[Ubuntu 从零安装 Git]]。
- Go：[[Ubuntu 安装 Go]]。
- Java：[[Java 与 Maven 环境搭建概览]]、[[Ubuntu 安装 Java 与 Maven]]。
- Docker：[[Docker 安装概览]]、[[Ubuntu 安装 Docker]]。

项目约束永远优先于通用笔记中的示例。先读取仓库的 README、CI、Makefile、`go.mod`、`pom.xml` 和 Wrapper，再决定版本与入口。

## 7. 原生运行与容器运行

原生构建与容器构建验证的边界不同：

| 方式 | 主要验证 | 不能单独证明 |
| --- | --- | --- |
| Linux 原生构建 | 本地语言工具链、源码、依赖解析 | Dockerfile、容器网络和镜像架构 |
| Docker 镜像构建 | Dockerfile、构建上下文、基础镜像 | 服务已启动、数据库可用 |
| Compose 静态解析 | 配置语法、变量展开、服务定义 | daemon 正常、容器健康、端口可用 |
| Compose 实际启动 | 多服务运行、网络、依赖和健康检查 | 生产安全、容量和可恢复性 |

开发环境可以同时保留原生工具链和容器能力。不要因为镜像能构建，就跳过项目要求的本地 Go/JDK 版本核对；也不要因为原生测试通过，就宣称容器发布链路已经成立。

## 8. 迁入项目时保持可追溯

项目进入 `$HOME/src` 的常见方式：

1. fresh clone 远端已保存的提交。
2. 用 Git bundle 搬运未推送提交。
3. 用 patch 搬运可审查修改。
4. 用受控 `rsync` 迁移完整工作现场。

完整选择和恢复方法见 [[Git 仓库跨机器迁移与工作区保留]]。

共享目录只作为交换区时，建议：

- 先在目标端创建全新暂存目录。
- 迁移前冻结源工作区或生成不可变归档。
- 不在第一次复制时使用 `--delete`。
- 复制后核对 Git 状态、SHA、隐藏文件和权限。
- 验证通过后才切换为正式目录。
- 保留源目录直到目标完成构建和备份。

EventHub 第 1 阶段的实际目录、版本和迁移选择见 [[EventHub 第 1 阶段环境与版本基线]] 与 [[EventHub 仓库迁移与首次质量门禁]]。

## 9. 日常权限原则

- Git、Go、Maven 和项目构建不使用 `sudo`。
- 系统包、`/usr/local` 工具链和 systemd 服务由管理员权限管理。
- 不用 `chmod -R 777` 解决工作区权限问题。
- 不对未知目录执行递归 `chown`。
- 先用 `namei -l`、`stat` 和 `findmnt` 找到真正阻塞的目录层级。

**执行位置：Linux 开发机（任意目录）**

```bash
namei -l "$HOME/src"
stat -c '%U:%G %a %n' "$HOME" "$HOME/src"
find "$HOME/src" -xdev -not -user "$USER" -print
```

若发现异常所有者，先判断文件是否来自共享挂载、容器或曾经误用 `sudo`，再按 [[Linux 用户、用户组、sudo 与文件权限]] 做最小范围修复。

## 10. 磁盘与缓存维护

**执行位置：Linux 开发机（任意目录）**

```bash
df -hT "$HOME"
du -sh "$HOME/src" 2>/dev/null || true
du -sh "$HOME/.m2" 2>/dev/null || true
du -sh "$HOME/go" 2>/dev/null || true
du -sh "$HOME/.cache" 2>/dev/null || true
docker system df 2>/dev/null || true
```

先识别增长来源，再决定清理：

- 源码目录异常大：检查构建产物、日志、数据库和未忽略文件。
- Maven/Go 缓存大：评估重新下载成本后使用工具提供的清理方法。
- Docker 占用大：先区分镜像、容器、构建缓存和 volume。
- VM 宿主磁盘紧张：同时检查客户机内部占用和稀疏磁盘实际增长。

> [!warning] 不要为了腾空间盲目运行全局清理
> Docker volume、数据库目录和被忽略文件可能包含不可重建数据。清理前先确认内容、备份和恢复方法。

## 11. 常见问题与恢复

| 现象 | 优先检查 | 处理 |
| --- | --- | --- |
| 文件修改监听不稳定 | 项目是否位于共享挂载 | 迁移到 Linux 本地暂存目录并重新验证 |
| Git 报所有权异常 | `stat`、`namei`、复制方式 | 修正明确目标的所有者，不放宽全局权限 |
| 命令存在但版本不对 | `type -a`、`PATH`、Shell 启动文件 | 按工具专题修正路径优先级 |
| Docker CLI 正常但不能运行容器 | daemon、context、socket 权限 | 阅读 [[Ubuntu 安装 Docker]]，不要改 socket 为全局可写 |
| 磁盘持续增长 | 缓存、镜像、volume、数据库、日志 | 分层审计，不删除未知数据 |
| 想撤销目录规划 | 是否已有未推送或未提交内容 | 先按 Git 迁移方法建立可恢复副本，再移动或归档 |

## 完成标准

- [ ] 活跃仓库位于 Linux 本地文件系统。
- [ ] 当前用户可读写工作区，日常构建不依赖 `sudo`。
- [ ] 能区分源码、工具链、缓存、Docker 数据和数据库。
- [ ] 能解释原生构建、镜像构建、Compose 静态解析和实际启动的边界。
- [ ] 共享目录只作为受控交换区。
- [ ] 项目迁移后核对了远程、分支、SHA、状态和质量门禁。
- [ ] 清楚 VM 快照、Git 备份和数据库备份不是同一保护层。

## 官方参考资料

- [Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)
- [util-linux：findmnt 手册源文件](https://github.com/util-linux/util-linux/blob/master/misc-utils/findmnt.8.adoc)
- [GNU Coreutils：Directory Setuid and Setgid](https://www.gnu.org/software/coreutils/manual/html_node/Directory-Setuid-and-Setgid.html)
- [Go：Managing source code](https://go.dev/doc/modules/managing-source)
- [Docker：Storage overview](https://docs.docker.com/engine/storage/)

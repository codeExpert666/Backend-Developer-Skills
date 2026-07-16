---
title: Linux 开发虚拟机备份恢复与常见问题
aliases:
  - Ubuntu 开发虚拟机快照与恢复
  - UTM 开发虚拟机备份基线
  - Linux 开发环境故障排查
tags:
  - Linux
  - Linux/开发环境
  - Linux/备份恢复
  - Linux/排障
  - UTM
created: 2026-07-16T00:28:30
updated: 2026-07-16T01:17:59
---

本文在第 1 阶段末尾建立可恢复基线，并汇总从 UTM 启动、Ubuntu 网络、SSH、工具链到项目门禁的常见故障。目标不是“点过一次快照按钮”，而是知道不同备份层保护什么、怎样验证备份真的能恢复，以及恢复副本为什么不能未经处理就与原虚拟机同时上线。

先完成 [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]]，并确认 [[Linux 后端开发虚拟机搭建概览]] 的必需检查项已经由真实命令验证。本文不会把尚未安装的 UTM 或尚未创建的虚拟机写成已备份成功。

## 快照、虚拟机副本、源码与数据库备份不是一回事

| 保护层 | 能快速恢复什么 | 不能替代什么 | 主要风险 |
| --- | --- | --- | --- |
| UTM 快照（若当前后端与版本支持） | 虚拟磁盘和设备状态的某个时间点 | 独立源码远端、异机备份、数据库逻辑备份 | 常依赖原虚拟磁盘；底层损坏时可能一起丢失 |
| 关机后的完整 `.utm` 副本或 UTM Duplicate | 整台客户机的系统、配置和虚拟磁盘 | Git 远端、密钥轮换、数据库跨版本迁移 | 体积大；复制稀疏文件到其他文件系统时可能膨胀 |
| Git 远端或 Git bundle | 已提交的源码历史与 refs | 未提交文件、未跟踪文件、数据库、系统配置 | 未推送提交不会出现在远端；bundle 也不包含工作区修改 |
| 受控工作区复制 | 未提交、未跟踪和 `.git` 等本地状态 | 长期版本历史策略与异地容灾 | 容易复制缓存、密钥或本地敏感文件，必须先审计 |
| 数据库逻辑备份 | 表结构和业务数据 | OS、Docker 卷元数据、源码 | 必须与数据库版本、字符集和恢复命令匹配 |
| 密钥与恢复码的安全保管 | 重新获得账号或服务访问权限 | 虚拟机与源码备份 | 不应与公开笔记或普通源码备份放在一起 |

> [!important] 快照不是备份的同义词
> 快照适合在升级或实验前快速回退；独立副本能抵御一部分原文件损坏；Git 远端保护已提交源码；数据库导出保护可迁移的数据。可靠基线至少要覆盖虚拟机、源码和未来持久化数据三个层次。

## 先确认 UTM 实际提供什么能力

UTM 的功能会随版本、QEMU 后端和 Apple Virtualization 后端变化。执行时应在 UTM 当前官方文档和虚拟机菜单中核对：

1. 当前虚拟机采用哪个 backend。
2. 当前版本是否为该 backend 提供可命名、可恢复的快照。
3. Duplicate、Clone、Export 或在 Finder 中复制 `.utm` 包分别如何工作。
4. 复制目标文件系统是否保留稀疏特性，以及需要多少真实空间。

本次笔记创建时 UTM 尚未安装，因此不能声称某个快照按钮当前一定存在。无论是否支持快照，**完整关机后的独立虚拟机副本** 都应作为第 1 阶段的最低 VM 级基线。

## 建立基线前冻结状态

### 1. 记录客户机状态

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
baseline_dir="$HOME/ops-baseline"
mkdir -p "$baseline_dir"
baseline_file="$baseline_dir/stage1-$(date +%Y%m%dT%H%M%S).txt"
{
  date --iso-8601=seconds
  hostnamectl
  uname -a
  printf '\nFilesystem:\n'
  df -h
  printf '\nMemory:\n'
  free -h
  printf '\nNetwork:\n'
  ip -brief address
  printf '\nTime:\n'
  timedatectl
  printf '\nServices:\n'
  systemctl is-active ssh docker 2>&1 || true
  printf '\nToolchain:\n'
  git --version 2>&1 || true
  go version 2>&1 || true
  java -version 2>&1 || true
  docker version 2>&1 || true
  docker compose version 2>&1 || true
} | tee "$baseline_file"
printf 'baseline=%s\n' "$baseline_file"
```

这份记录不应包含密码、令牌、私钥、完整环境变量或内部 URL。它只是恢复后的对照，不是备份本身。

### 2. 确认源码状态

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
for repo in "$HOME/src/eventhub-go" "$HOME/src/eventhub"; do
  printf '\nrepository=%s\n' "$repo"
  git -C "$repo" status --short --branch
  git -C "$repo" remote -v
  git -C "$repo" rev-parse HEAD
done
```

若存在未提交或未推送内容，先按 [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]] 选择提交推送、bundle、patch 或受控复制。不要让唯一副本只存在于即将备份的虚拟磁盘里。

### 3. 停止写入并正常关机

构建命令已经结束后，先检查容器：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
sync
sudo shutdown -h now
```

如果有需要保留的数据库容器，应先按项目文档做逻辑备份并正常停止 Compose 栈，再关机。`sync` 不能替代应用一致性处理；它只要求内核尽快把缓存写入磁盘。

在 UTM 中等待状态明确变为 **Stopped**。不要在客户机仍运行、暂停或写磁盘时直接从 Finder 复制 `.utm` 包，否则只能得到崩溃一致性甚至损坏的副本。

## 创建三层第 1 阶段基线

### 第一层：UTM 快照或实验回退点

如果当前 UTM 版本与后端明确支持快照，可创建带日期和用途的快照，例如：

```text
stage1-toolchains-and-first-build-verified-20260716
```

记录它基于哪个 UTM 版本、backend 和客户机状态。快照应在重大升级或破坏性实验前创建，实验结束后按容量和恢复需求清理；不要无限累积而从不测试。

### 第二层：关机状态的独立虚拟机副本

在 UTM 界面使用当前版本明确提供的 Duplicate、Clone 或 Export；如果采用 Finder 复制，先用 UTM 的 Reveal in Finder 定位真实 `.utm` 包，再复制到容量足够、受备份保护的位置。

复制前后在 macOS 检查空间：

**执行位置：macOS 宿主机（任意目录，只读检查）**

```bash
df -h /
diskutil info / | grep -E 'File System Personality|Free Space'
```

稀疏虚拟磁盘的“最大容量”与“当前实际占用”不同。APFS 上的稀疏文件不会立即占满标称上限，但复制到不支持稀疏或 clone 特性的文件系统时可能接近逻辑大小。复制完成后不要马上删除原 VM。

建议为副本同时记录：

- 创建日期、UTM 版本与 backend。
- 原虚拟机名称、Ubuntu 版本和架构。
- 虚拟 CPU、内存、磁盘上限和网络模式。
- 两个仓库的 commit SHA 与工作区是否干净。
- 是否包含 Tailscale 节点状态或本地数据库。
- 恢复验证日期与结果。

### 第三层：独立源码备份

已经推送的提交应通过代码托管远端再次核对。未推送但已提交的历史可额外创建 Git bundle：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
backup_dir="$HOME/backups/git"
mkdir -p "$backup_dir"
git -C "$HOME/src/eventhub-go" bundle create \
  "$backup_dir/eventhub-go-$(date +%Y%m%d).bundle" --all
git -C "$HOME/src/eventhub" bundle create \
  "$backup_dir/eventhub-$(date +%Y%m%d).bundle" --all
git -C "$HOME/src/eventhub-go" bundle verify \
  "$backup_dir/eventhub-go-$(date +%Y%m%d).bundle"
git -C "$HOME/src/eventhub" bundle verify \
  "$backup_dir/eventhub-$(date +%Y%m%d).bundle"
```

bundle 不包含未提交和未跟踪文件。这些内容若必须保留，应先明确清单，再用 patch 或受控复制单独保存。备份仍放在同一虚拟磁盘只能防误操作，不能防整个 VM 文件丢失；最终副本要离开该虚拟磁盘并进入受控备份位置。

## 数据库与 Docker 卷的后续基线

第 1 阶段可以只做首次构建与临时容器验证；一旦开始在 MySQL、Redis 或其他卷中保存不可重建的数据，就必须增加独立备份。

| 数据类型 | 首选保护方式 | 恢复验证 |
| --- | --- | --- |
| 可由 Compose 重建的空容器与网络 | 保留已跟踪 Compose 和镜像版本 | 在空环境 `docker compose config` 并重新创建 |
| Maven、Go、Docker 构建缓存 | 通常不备份 | 允许删除后重新下载或构建 |
| MySQL 业务数据 | 数据库逻辑备份或经过设计的物理备份 | 恢复到隔离实例并核对表、行数和字符集 |
| Redis 仅作缓存 | 通常从源数据重建 | 明确它确实不是唯一数据源 |
| Docker named volume | 先识别数据格式与一致性要求 | 不只复制 `/var/lib/docker`，应按应用恢复流程验证 |

> [!danger] 不要把清理命令当成排障命令
> `docker compose down -v` 会删除 Compose 命名卷；删除 `/var/lib/docker` 或 `/var/lib/containerd` 会破坏容器数据。只有确认数据可重建或备份已恢复验证后，才按明确清理计划执行。

## 恢复演练：不要覆盖唯一可用副本

### 1. 先隔离原虚拟机

恢复副本前关闭原 VM。完整克隆会复制主机名、SSH 主机密钥、Tailscale 节点状态、机器 ID 和可能的静态地址；两个副本同时接入同一网络会造成身份冲突。

如果恢复副本只是为了替换损坏的原 VM，应保持原 VM 关闭，再以同一身份接管。如果要让两台 VM 长期并行运行，应先隔离网络，再为副本重新生成独立主机名、机器身份、SSH host keys 和 Tailscale 节点身份；这属于独立克隆设计，不是本阶段恢复主线。

### 2. 导入或打开副本

按当前 UTM 官方说明导入 `.utm` 包或使用 Duplicate 副本。不要移动、重命名或删除原 VM，直到副本完成下面全部验证。

### 3. 在 UTM 控制台验证客户机

**执行位置：恢复后的 Ubuntu 虚拟机（UTM 控制台）**

```bash
hostnamectl
uname -m
df -h
free -h
ip -brief address
timedatectl
systemctl --failed
systemctl is-active ssh docker
```

预期架构、磁盘、内存和关键服务符合基线，`systemctl --failed` 没有未解释的失败单元。

### 4. 重新核对 SSH 主机身份

完整 VM 副本通常保留原 SSH host key。若它是原 VM 的替代恢复，这能保持客户端信任；若副本被作为新机器长期使用，就不能与原机共享同一主机身份。

从 UTM 控制台读取指纹，再按 [[从 macOS 使用 SSH 连接 Linux 虚拟机]] 比较。不要因为恢复后出现主机密钥警告就删除整个 `known_hosts`。

### 5. 恢复并验证源码

**执行位置：恢复后的 Ubuntu 虚拟机（任意目录）**

```bash
for repo in "$HOME/src/eventhub-go" "$HOME/src/eventhub"; do
  git -C "$repo" status --short --branch
  git -C "$repo" remote -v
  git -C "$repo" rev-parse HEAD
  git -C "$repo" fsck --full
done
```

如果 VM 副本没有源码，可从远端 clone，或从 bundle 恢复到新目录：

**执行位置：恢复后的 Ubuntu 虚拟机（`$HOME/src` 的父目录）**

```bash
mkdir -p "$HOME/src"
printf '待恢复的 EventHub Go bundle 绝对路径: '
IFS= read -r BUNDLE_PATH
RESTORE_DIR="$HOME/src/eventhub-go-restored"
if ! test -f "$BUNDLE_PATH"; then
  printf '停止：bundle 文件不存在：%s\n' "$BUNDLE_PATH" >&2
elif test -e "$RESTORE_DIR"; then
  printf '停止：恢复目录已经存在：%s\n' "$RESTORE_DIR" >&2
else
  git clone "$BUNDLE_PATH" "$RESTORE_DIR"
  git -C "$RESTORE_DIR" log -1 --oneline
fi
```

只有 bundle 存在且目标目录尚不存在时才会 clone。若目标目录已经存在，先换一个新的恢复目录；不要覆盖现有仓库。

### 6. 重跑第 1 阶段门禁

恢复验证不能停在“能开机”。按 [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]] 重新核对版本、Docker daemon、Compose 静态配置、Go 编译测试和 Java 构建测试，并记录 commit SHA 与退出码。

## 常见问题速查

| 现象 | 最可能的层次 | 第一组检查 |
| --- | --- | --- |
| 安装后进入 EFI Shell | ISO、可移动驱动器或启动顺序 | UTM 中确认 ISO 挂载；安装完成后卸载 ISO |
| VM 启动很慢、宿主机卡顿 | vCPU、内存或磁盘压力 | macOS Activity Monitor；Ubuntu `free -h`、`df -h` |
| Ubuntu 没有地址 | UTM 网卡、DHCP、客户机接口 | `ip -brief address`、`ip route` |
| 能 ping IP，不能访问域名 | DNS | `resolvectl status`、`getent hosts ubuntu.com` |
| APT、Git、Maven、Go 都出现 TLS 异常 | 时间、DNS、代理或证书 | `timedatectl`、`env | grep -i proxy` |
| SSH 超时 | 网络或防火墙 | `nc -vz`、`ufw status` |
| SSH 拒绝连接 | `sshd` 未运行 | `systemctl status ssh`、`ss -ltnp` |
| SSH 主机密钥变化 | VM 重装、克隆或地址复用 | 从 UTM 控制台重新核对指纹 |
| 二进制报 `Exec format error` | ARM64 与 AMD64 不匹配 | `uname -m`、`file`、`docker image inspect` |
| Docker permission denied | daemon、socket 或用户组 | `systemctl status docker`、`id`、`ls -l /var/run/docker.sock` |
| `go test ./...` 很快但集成测试没有执行 | Docker provider 不健康导致跳过 | `docker info`，再运行指定测试并观察 `SKIP` |
| Java 能运行但 Maven 版本或 JDK 不对 | Wrapper、`JAVA_HOME`、PATH | `java -version`、`./mvnw -version` 或 `mvn -version` |
| 共享目录中权限或文件监听异常 | macOS 与 Linux 文件语义差异 | 把仓库移到 `$HOME/src` 后复测 |
| Docker 拉取后磁盘骤减 | 镜像、构建缓存或卷增长 | `docker system df -v`；先识别再清理 |

## 分层排障命令

### macOS 宿主机

**执行位置：macOS 宿主机（任意目录，只读检查）**

```bash
uname -m
df -h /
ping -c 3 "$(route -n get default | awk '/gateway:/{print $2}')"
```

这组命令只确认宿主架构、磁盘和默认网关，不证明 VM 正常。

### Ubuntu 基础系统

**执行位置：Ubuntu 虚拟机（UTM 控制台）**

```bash
uname -m
ip -brief address
ip route
resolvectl status
getent hosts ubuntu.com
timedatectl
df -h
free -h
systemctl --failed
journalctl -p warning -b --no-pager
```

`journalctl -p warning` 可能包含无害告警；应结合时间、服务和失败现象判断，不要看到一条 warning 就重装系统。

### SSH

**执行位置：Ubuntu 虚拟机（UTM 控制台）**

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(port|pubkeyauthentication|passwordauthentication|permitrootlogin) '
sudo journalctl -u ssh.service -n 100 --no-pager
```

### Docker

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
systemctl is-active docker
docker version
docker info
docker system df -v
```

不要在排障第一步运行 prune、删除卷或删除 Docker 数据目录。

## 何时回滚，何时继续修复

| 情况 | 建议 |
| --- | --- |
| 刚修改 `sshd` 后失去远程连接 | 立即用 UTM 控制台恢复配置，不必回滚整个 VM |
| 工具链 PATH 写错 | 恢复 Shell 配置备份并新开会话 |
| APT 安装单个包失败 | 先看 APT 错误、网络和锁，不要直接恢复 VM |
| 大规模系统升级后多个服务异常 | 若已记录升级且有已验证快照，可评估回滚 |
| 虚拟磁盘损坏或 VM 无法启动 | 从独立关机副本恢复，再从源码与数据库备份补齐 |
| 项目测试失败 | 保留失败证据，按项目代码与依赖排查；不要用 VM 快照掩盖真实测试失败 |

回滚只是恢复到旧状态，不会解释故障原因。恢复后要记录触发条件、丢失的变更、重新验证结果和防止重现的方法。

## 第 1 阶段停止点

完成本篇后，本阶段可以结束。下面内容不要继续塞进当前虚拟机搭建任务：

- 云服务器购买、VPC、公网 IP、云安全组和域名。
- 正式 CI/CD、制品仓库、部署凭据和环境审批。
- 生产日志、指标、链路追踪、告警和值班。
- 灰度发布、蓝绿发布、数据库生产迁移和业务回滚。
- Kubernetes、Ingress、Operator 或多节点容器编排。

这些主题需要在第 1 阶段基线稳定后分别设计。开发 VM 的 Compose 能运行，只说明本地开发能力成立，不代表生产部署方案成立。

## 完成标准

- [ ] 已确认当前 UTM 版本与 backend 的真实快照或复制能力。
- [ ] 已在正常关机状态创建独立 VM 副本，并记录位置、日期与配置。
- [ ] 已确认稀疏磁盘副本的实际占用与备份介质容量。
- [ ] 已把已提交源码推送或保存为经过验证的 bundle。
- [ ] 已单独处理必须保留的未提交和未跟踪内容。
- [ ] 已明确 Docker 卷与未来数据库数据需要独立备份。
- [ ] 已执行一次不覆盖原 VM 的恢复核对，并验证启动、网络、SSH、源码和门禁。
- [ ] 已避免让具有相同 SSH、Tailscale 和机器身份的克隆同时上线。
- [ ] 已记录故障排查与恢复结果，且记录中不含秘密信息。

## 官方参考资料

以下涉及 UTM、systemd、Docker、Git 与 Tailscale 的资料于 **2026-07-16** 核对；执行恢复前仍要按当时实际 UTM backend、应用版本和数据类型重新确认能力边界。

- [UTM：Ubuntu 安装指南](https://docs.getutm.app/guides/ubuntu/)
- [UTM：Drive 与 APFS 稀疏文件](https://docs.getutm.app/settings-apple/drive/)
- [UTM：QEMU Drive](https://docs.getutm.app/settings-qemu/drive/drive/)
- [UTM：脚本参考中的 duplicate 与 import](https://docs.getutm.app/scripting/reference/)
- [systemd：systemctl 官方手册](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [Ubuntu：journalctl 手册](https://manpages.ubuntu.com/manpages/noble/en/man1/journalctl.1.html)
- [Git：git-bundle](https://git-scm.com/docs/git-bundle)
- [Git：检查仓库完整性](https://git-scm.com/docs/git-fsck)
- [Docker：备份、恢复或迁移数据卷](https://docs.docker.com/engine/storage/volumes/#back-up-restore-or-migrate-data-volumes)
- [Tailscale：节点密钥](https://tailscale.com/docs/concepts/node-keys)

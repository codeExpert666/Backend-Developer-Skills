---
title: UTM 虚拟机快照、备份与恢复
aliases:
  - Linux 开发虚拟机备份恢复与常见问题
  - UTM 虚拟机备份恢复
tags:
  - 虚拟化
  - UTM
  - 备份恢复
  - Linux/运维
created: 2026-07-16T00:28:30
updated: 2026-07-18T14:22:18
---

本文说明如何为 UTM 中的 Linux 虚拟机建立可恢复基线。重点不是“拥有一个快照”，而是区分快速回退点、关机状态的完整 VM 副本、源码备份、数据库备份和身份恢复，并通过隔离恢复演练证明备份可用。

> [!info] 资料核对日期
> 本文涉及 UTM Actions、控制方式和磁盘操作的信息于 **2026-07-18** 根据官方文档核对。UTM 的快照、Export、Duplicate 或复制能力会随版本、虚拟化后端和虚拟磁盘格式变化，操作前应在当前版本中重新确认。

## 1. 先建立正确的备份模型

不同保护层解决的问题不同。

| 保护层 | 主要用途 | 不能替代什么 | 典型风险 |
| --- | --- | --- | --- |
| UTM 快照或回退点 | 升级、配置实验前快速回退 | 独立 VM 副本、源码远端、数据库逻辑备份 | 可能依赖原虚拟磁盘，底层损坏时一起丢失 |
| 关机状态的完整 VM 副本 | 恢复客户机系统、配置和虚拟磁盘 | Git 历史治理、数据库跨版本迁移 | 文件大，复制到其他文件系统时可能膨胀 |
| Git 远端或 Git bundle | 已提交的源码历史和 refs | 未提交、未跟踪文件、数据库和系统配置 | 未推送提交不在远端；bundle 不含工作区修改 |
| 受控工作区复制 | 未提交、未跟踪和本地 Git 状态 | 长期版本历史和异地容灾 | 可能误带密钥、缓存或本地敏感文件 |
| 数据库逻辑或物理备份 | 可恢复的业务数据 | OS、源码和容器定义 | 必须与一致性、版本和恢复流程匹配 |
| 密钥与恢复码保管 | 恢复账号或服务访问权 | VM、源码和数据库备份 | 不应与普通源码或公开笔记放在一起 |

> [!important] VM 快照不等于源码和数据库备份
> 快照能回退磁盘状态，但不能代替已提交源码的独立远端，也不能自动保证数据库应用一致性。可靠基线至少要分别覆盖 VM、源码和不可重建数据。

源码存在未提交、未推送或未跟踪内容时，应先按 [[Git 仓库跨机器迁移与工作区保留]] 选择提交推送、bundle、patch 或受控复制，不要让唯一副本只存在于待备份的虚拟磁盘。

## 2. 明确恢复目标

建立备份前先回答两个问题：

- 最多能接受丢失多少时间内的变更，即恢复点目标。
- 故障后希望多快恢复到可工作的状态，即恢复时间目标。

开发虚拟机通常可以采用：

- 重要变更进入 Git 远端后再建立阶段性 VM 基线。
- 系统升级、Docker 变更或破坏性实验前建立快速回退点。
- 定期制作关机状态的独立 VM 副本。
- 一旦数据库中出现不可重建数据，单独建立数据库备份计划。

备份名称应表达时间和用途，例如：

```text
linux-dev-before-os-upgrade-20260717
linux-dev-toolchain-baseline-20260717
```

名称只是索引，还应记录 UTM 版本、虚拟化后端、客户机版本和恢复验证结果。

## 3. 建立基线前记录状态

### 3.1 记录 UTM 与 VM 配置

在 macOS 的 UTM 界面记录：

- UTM 版本和 build number。
- VM 名称、客户机架构和虚拟化后端。
- vCPU、内存、虚拟磁盘上限。
- 网络模式和共享目录设置。
- 可移动驱动器是否仍挂载 ISO。
- 当前版本实际提供的快照、Export、Create Copy 或其他动作。

不要根据旧截图猜测菜单。UTM 官方 Version 文档建议通过 **UTM → About UTM** 查看版本；Actions 文档列出的动作包括导出全部数据、移动 VM、创建包含全部数据的副本和仅复制配置，但实际可用项仍应以当前 VM 为准。

### 3.2 记录客户机状态

下面命令会在客户机内创建一份不含密码和令牌的基线记录。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
umask 077
BASELINE_DIR="$HOME/ops-baseline"
BASELINE_FILE="$BASELINE_DIR/vm-$(date +%Y%m%dT%H%M%S).txt"

mkdir -p "$BASELINE_DIR"

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
  ip route
  printf '\nTime:\n'
  timedatectl
  printf '\nFailed units:\n'
  systemctl --failed
} | tee "$BASELINE_FILE"

printf 'baseline=%s\n' "$BASELINE_FILE"
```

不要把完整环境变量、私钥、访问令牌、数据库密码或内部凭据写入基线文件。

### 3.3 核对源码状态

下面脚本只检查 `$HOME/src` 下具有 `.git` 目录的仓库。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
for REPO_DIR in "$HOME"/src/*; do
  if [ ! -d "$REPO_DIR/.git" ]; then
    continue
  fi

  printf '\nrepository=%s\n' "$REPO_DIR"
  git -C "$REPO_DIR" status --short --branch
  git -C "$REPO_DIR" remote -v
  git -C "$REPO_DIR" rev-parse HEAD
done
```

如果没有输出，说明该目录下没有被脚本识别的普通 Git 工作区；子模块、worktree 或裸仓库需要按各自结构单独检查。

## 4. 让磁盘状态进入一致点

完整 VM 副本应尽量在客户机正常关机后创建。关机前：

1. 停止正在写入的构建、编辑器远程进程和后台任务。
2. 正常停止应用服务和 Compose 栈。
3. 对数据库执行与其存储引擎匹配的备份或一致性流程。
4. 确认需要保留的源码变更已有独立副本。
5. 再让客户机正常关机。

> [!warning] 下面命令会关闭虚拟机
> 先确认已经停止写入，并保留 UTM 控制台访问方式。执行后当前 SSH 会话会断开，这是预期结果。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}' 2>/dev/null || true
sync
sudo systemctl poweroff
```

在 UTM 中等待 VM 状态明确变为已停止。不要把 UTM 的 Force Shut Down 或 Force Kill 当作日常关机方式；UTM 官方 Controls 文档明确提示这些操作会丢弃未保存数据，并有文件系统损坏风险。

`sync` 只要求内核尽快写回缓存，不能替代数据库事务一致性、应用停止或文件系统正常卸载。

## 5. 创建快速回退点

如果当前 UTM 版本、后端和虚拟磁盘格式为该 VM 提供快照或类似回退能力，可以在以下时机创建：

- 操作系统大版本或内核升级前。
- Docker、网络或安全配置大改前。
- 需要试验高风险命令前。
- 工具链和项目门禁刚完成验证时。

记录：

- 创建时间和目的。
- UTM 版本与后端。
- VM 当时是运行、暂停还是关机状态。
- 依赖的父快照或原虚拟磁盘。
- 预计删除时间和恢复验证结果。

不要无限累积快照。快照链会增加容量和维护复杂度，而且通常仍依赖原 VM 数据，因此它只适合快速回退，不是独立备份。

若当前版本没有可验证的快照能力，直接跳过这一层，使用关机状态的完整 VM 副本作为最低基线。

## 6. 创建关机状态的完整 VM 副本

### 6.1 优先使用 UTM 当前提供的动作

UTM 官方 Actions 文档说明，VM 动作菜单可以提供：

- Export virtual machine and all its data。
- Move virtual machine。
- Create a copy with all its data。
- Create a copy of configuration without data。
- Reveal the virtual machine package in Finder。

备份必须选择包含全部数据的 Export 或完整副本；“仅复制配置”不包含虚拟磁盘，不能恢复客户机。

由于菜单名称、文件格式和可用能力会变化，实际步骤应以当前版本为准。创建副本前保持 VM 关机，并确保目标位置有足够真实空间。

### 6.2 检查 VM 包和目标空间

如果通过 **Reveal in Finder** 找到 `.utm` 包，可以用下面命令只读检查逻辑占用和所在文件系统空间。

**执行位置：macOS 宿主机（任意目录）**

```bash
printf 'UTM 虚拟机包绝对路径: '
IFS= read -r VM_PACKAGE

if [ ! -d "$VM_PACKAGE" ]; then
  printf '停止：目录不存在：%s\n' "$VM_PACKAGE" >&2
  exit 1
fi

case "$VM_PACKAGE" in
  *.utm)
    du -sh "$VM_PACKAGE"
    df -h "$(dirname "$VM_PACKAGE")"
    ;;
  *)
    printf '停止：所选目录不是 .utm 包：%s\n' "$VM_PACKAGE" >&2
    exit 1
    ;;
esac
```

这里复制的是宿主机上的 `.utm` 包或虚拟磁盘镜像，不是 Ubuntu 客户机内的普通文件。虚拟磁盘的容量上限、镜像逻辑大小和宿主实际占用并不相同，完整模型见 [[虚拟磁盘的逻辑容量与实际占用]]。

`du` 显示的是源包当前分配的宿主空间，不能直接作为目标所需空间上限。复制路径可能产生三种不同结果：

- 目标文件系统创建 CoW 克隆，副本最初与源文件共享数据块。
- 复制工具在目标端重新创建稀疏空洞，副本只为有效数据分配空间。
- 目标文件系统或复制路径无法保留这些特性，把读取到的零区域写成真实磁盘块，导致副本明显膨胀。

对 raw 稀疏镜像，完全分配的副本可能接近镜像逻辑大小；QCOW2 等格式则取决于镜像自身布局。这个风险应写成“可能膨胀”，不能假定每次复制都会增长到虚拟磁盘容量上限。复制前应给目标卷保留额外余量，复制完成后再次检查目标卷空间，不要立即删除原 VM。

> [!warning] 回收空间不是无风险的复制前准备
> UTM 当前文档把 Resize、Reclaim Space 和 Compress 标记为实验性操作，并要求先备份虚拟机。Reclaim Space 的最佳结果还依赖客户机 TRIM、正常关机和当前磁盘格式。不要为了缩小待备份镜像，在没有独立恢复路径时先执行转换或回收。

### 6.3 让副本成为独立备份

完整副本至少应满足：

- 不与唯一原 VM 位于同一块故障域中。
- 有明确的创建日期、来源 VM 和 UTM 版本记录。
- 备份位置受访问控制和正常备份策略保护。
- 已完成恢复演练，而不只是“文件复制完成”。

如果副本仍在同一宿主机同一物理磁盘，它可以防部分误操作，但不能防宿主磁盘故障或设备丢失。

## 7. 单独保护源码与数据库

### 7.1 源码

源码至少应有一条独立于 VM 的恢复路径：

- 已提交并推送到受控 Git 远端。
- 未推送提交使用 Git bundle。
- 需要保留的工作区修改使用 patch 或受控复制。

具体选择、验证与恢复见 [[Git 仓库跨机器迁移与工作区保留]]。

### 7.2 数据库和容器卷

| 数据类型 | 推荐保护方式 | 恢复验证 |
| --- | --- | --- |
| 可由声明文件重建的空容器、网络 | 保存已跟踪的 Compose 和镜像约束 | 在空环境重新创建 |
| Go、Maven、Docker 构建缓存 | 通常不备份 | 删除后可重新下载或构建 |
| MySQL、PostgreSQL 等业务数据 | 数据库官方逻辑或物理备份 | 恢复到隔离实例并核对结构和数据 |
| Redis 仅作缓存 | 从真实数据源重建 | 先确认它不是唯一数据源 |
| Docker named volume | 按卷内应用的数据格式制定方案 | 不只复制 Docker 内部目录 |

> [!danger] 不要用清理命令验证备份
> `docker compose down -v` 会删除命名卷。删除 Docker 或数据库数据目录也可能不可恢复。只有在独立环境成功恢复并确认数据可重建后，才按明确计划清理。

## 8. 做一次隔离恢复演练

### 8.1 先决定是“替换恢复”还是“并行克隆”

| 目标 | 原 VM | 恢复副本身份 |
| --- | --- | --- |
| 替换损坏的原 VM | 保持关闭，不再同时上线 | 可以继续使用原主机身份 |
| 创建一台长期并行的新 VM | 原 VM 仍会继续运行 | 必须建立新的机器、SSH 和 Tailscale 身份 |

完整 VM 副本通常会复制：

- 主机名。
- `/etc/machine-id`。
- SSH host keys。
- Tailscale 节点状态。
- 可能存在的静态 IP、证书和本地服务身份。

两个具有相同身份的副本同时联网，可能引发地址、主机指纹、Tailscale 节点或应用身份冲突。

### 8.2 先隔离网络

恢复演练的安全顺序：

1. 保留原 VM 和原备份不动。
2. 关闭原 VM。
3. 通过当前 UTM 支持的导入或完整复制方式创建恢复实例。
4. 首次启动恢复实例前暂时断开其虚拟网卡。
5. 通过 UTM 控制台登录并检查身份。
6. 明确它是替换恢复还是新克隆后，再决定是否联网。

如果目标是长期并行克隆，应在隔离状态下重新规划主机名、machine ID、SSH host keys、Tailscale 节点和任何应用证书。不要只改主机名就认为身份已经完全独立。

### 8.3 在恢复实例中检查身份

**执行位置：恢复后的 Ubuntu 虚拟机（UTM 控制台，网络保持隔离）**

```bash
hostnamectl
cat /etc/machine-id

if [ -f /etc/ssh/ssh_host_ed25519_key.pub ]; then
  sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
fi

if command -v tailscale >/dev/null 2>&1; then
  sudo tailscale status
fi
```

如果是原 VM 的替换恢复，并且原 VM 保持永久下线，保留原 SSH host key 可以维持客户端信任。如果是新克隆，就不能与原 VM 长期共享同一 SSH 或 Tailscale 身份。SSH 身份处理见 [[OpenSSH 连接、密钥与主机指纹]]，Tailscale 节点处理见 [[使用 Tailscale 访问 Linux 主机]]。

### 8.4 验证系统和源码

**执行位置：恢复后的 Ubuntu 虚拟机（UTM 控制台，任意目录）**

```bash
printf '%s\n' 'System:'
uname -m
cat /etc/os-release
df -h
free -h
systemctl --failed

printf '\n%s\n' 'Repositories:'
for REPO_DIR in "$HOME"/src/*; do
  if [ ! -d "$REPO_DIR/.git" ]; then
    continue
  fi

  printf '\nrepository=%s\n' "$REPO_DIR"
  git -C "$REPO_DIR" status --short --branch
  git -C "$REPO_DIR" rev-parse HEAD
  git -C "$REPO_DIR" fsck --full
done
```

验证内容至少包括：

- 架构、Ubuntu 版本、磁盘和内存符合基线。
- 文件系统可以读取，`systemctl --failed` 没有未解释故障。
- 重要仓库存在，分支、commit SHA 和工作区状态符合记录。
- Git 对象检查通过。
- 数据库备份可以恢复到隔离实例。
- 确定身份策略后，网络、DNS、SSH 和必要服务能够恢复。

恢复验证不能只停在“VM 能开机”。应重跑该开发机所需的最小构建、测试和服务验证，并记录退出码。

## 9. 恢复失败时如何处理

| 现象 | 优先检查 | 处理原则 |
| --- | --- | --- |
| UTM 无法打开副本 | UTM 版本、包是否完整、导出格式 | 保留原文件，使用创建备份时相同或兼容版本重试 |
| 副本启动后进入 EFI Shell | 启动盘、可移动介质、启动顺序 | 检查虚拟磁盘是否随副本包含；不要重新安装覆盖 |
| 副本磁盘异常膨胀 | 目标文件系统不保留稀疏或 clone 特性 | 先确保空间，核对官方 Export 与磁盘格式说明 |
| SSH 报主机密钥变化 | 地址复用、重装或克隆身份变化 | 在控制台核对真实指纹，只移除对应旧记录 |
| Tailscale 节点表现异常 | 两个副本共享节点状态 | 立即隔离其中一个，决定替换恢复或新节点 |
| Git 仓库缺少修改 | 变更从未提交、打包或复制 | 从独立 patch、bundle 或工作区备份恢复 |
| 数据库能启动但数据不一致 | 备份时仍有写入或方法不匹配 | 回到数据库专用备份重新做隔离恢复 |

> [!warning] 恢复演练完成前不要删除原 VM 或唯一备份
> UTM 的 Remove 动作可能同时删除默认路径中的 VM 数据。只有恢复实例通过系统、源码、身份和数据验证，并且至少还有一份独立副本时，才能安排清理。

## 10. 基线检查清单

- [ ] 已记录 UTM 版本、后端、VM 配置和客户机版本。
- [ ] 已记录重要仓库的分支、commit SHA 和工作区状态。
- [ ] 已停止写入并让客户机正常关机。
- [ ] 当前版本支持时，已创建并说明快速回退点的用途。
- [ ] 已创建包含虚拟磁盘数据的关机状态 VM 副本。
- [ ] VM 副本不只是唯一原文件旁的一份同盘复制。
- [ ] 已为源码建立独立于 VM 的恢复路径。
- [ ] 已为不可重建数据库数据建立独立备份。
- [ ] 已在隔离条件下启动恢复实例。
- [ ] 已明确替换恢复与并行克隆的身份差异。
- [ ] 已检查 SSH、Tailscale 和 machine ID 冲突风险。
- [ ] 已重跑最小系统、源码、构建和数据恢复验证。
- [ ] 已记录恢复日期、结果和仍未覆盖的风险。

## 官方资料

- [UTM Documentation：Actions](https://docs.getutm.app/basics/actions/)
- [UTM Documentation：Controls](https://docs.getutm.app/basics/controls/)
- [UTM Documentation：Version](https://docs.getutm.app/advanced/version/)
- [UTM Documentation：Resize and Compress](https://docs.getutm.app/settings-qemu/drive/resize-and-compress/)
- [UTM Documentation：Ubuntu 指南](https://docs.getutm.app/guides/ubuntu/)

---
title: 使用 UTM 创建 Ubuntu Server 虚拟机
aliases:
  - 使用 UTM 创建 Ubuntu Server 开发虚拟机
  - UTM 安装 Ubuntu Server
  - Ubuntu Server 虚拟机搭建
tags:
  - 虚拟化
  - UTM
  - Ubuntu
  - Linux/安装
created: 2026-07-16T00:31:57
updated: 2026-07-19T14:59:36
---

本文是一篇可复用的 UTM 操作笔记：从选择虚拟化方式、校验 Ubuntu Server ISO 开始，完成虚拟机创建、系统安装、安装介质卸载和首次启动验证。它不规定唯一的 CPU、内存、磁盘、网络模式或 Ubuntu 版本；具体选择应先参考 [[虚拟机、客户机与 CPU 架构]]、[[UTM 虚拟机资源规划]] 和 [[虚拟机网络模式与可达性]]。

> [!info] 资料核对日期
> 本文涉及 UTM 界面、Ubuntu 镜像和校验流程的信息于 **2026-07-19** 根据官方文档核对。UTM 的按钮名称、虚拟化后端和 Ubuntu 点版本会变化，实际操作时应重新核对文末官方资料。

## 完成标准

完成本文后，应能够：

- 根据宿主机 CPU 架构选择正确的 Ubuntu Server ISO。
- 解释 UTM 中 **Virtualize** 与 **Emulate** 的差异，并选择合适路线。
- 验证 ISO 文件名、文件类型和 SHA-256 摘要。
- 在 UTM 中创建虚拟机并完成 Ubuntu Server 安装。
- 能读懂安装器的网络接口、DHCP 地址和 bond 选项，并判断何时可以继续。
- 能区分引导式与自定义存储布局、LVM 与 LUKS，并核对根文件系统和 VG 空闲空间。
- 安装结束后卸载 ISO，确认系统从虚拟磁盘启动。
- 能区分 cloud-init 正常完成、被 Ubuntu 安装器按设计禁用和真正的初始化异常。
- 在客户机内验证架构、系统版本、磁盘、网络、DNS、时区与时间同步状态。
- 遇到 EFI Shell、重复进入安装器、网络不可用等问题时，知道从哪一层排查。

## 1. 先确定使用 Virtualize 还是 Emulate

UTM 同时支持虚拟化和模拟，但二者解决的问题不同。

| 模式 | 宿主机与客户机架构 | 适用场景 | 代价 |
| --- | --- | --- | --- |
| Virtualize | 架构需要匹配 | 日常 Linux 开发机、构建和容器实验 | 性能接近原生，主线优先 |
| Emulate | 可以不同 | 必须运行另一种 CPU 架构的旧系统或软件 | 需要指令翻译，通常更慢、耗能更多 |

UTM 官方说明：虚拟化要求客户机架构与宿主机匹配。Apple Silicon 宿主机对应 `aarch64` / ARM64，Intel Mac 对应 `x86_64` / AMD64。如果架构不匹配，即使勾选了虚拟化，UTM 也不能按同架构虚拟化执行。

先在宿主机确认架构。

**执行位置：macOS 宿主机（任意目录）**

```bash
uname -m
sysctl -n hw.logicalcpu
sysctl -n hw.memsize
# 以 GB 为单位显示内存
sysctl -n hw.memsize | awk '{printf "%.0f GiB\n", $1 / 1024 / 1024 / 1024}'
```

常见结果：

- `arm64`：选择 Ubuntu Server ARM64 ISO，并走 **Virtualize → Linux**。
- `x86_64`：选择 Ubuntu Server AMD64 ISO，并走 **Virtualize → Linux**。

只有在明确需要跨架构系统时才选择 **Emulate**。后端开发工具和容器镜像如果支持宿主架构，通常没有必要先引入模拟层。

## 2. 选择 Ubuntu 版本与镜像

Ubuntu 版本不是“越新越好”。应按以下顺序决策：

1. 项目或团队已声明的操作系统约束。
2. 仍在安全维护期内的 Ubuntu Server LTS。
3. 所需 JDK、Go、Docker、数据库和其他原生依赖对该版本的支持情况。
4. 宿主机 CPU 架构。

从 Ubuntu 官方下载目录获取 Server 安装 ISO。文件名通常会包含：

- Ubuntu 版本。
- `live-server`。
- `arm64` 或 `amd64`。
- `.iso`。

不要只根据“64 位”判断架构，因为 ARM64 和 AMD64 都是 64 位指令集。

## 3. 校验 ISO 架构与完整性

下载 ISO 后，同时下载同一发布目录中的 `SHA256SUMS` 和 `SHA256SUMS.gpg`。先用下面脚本确认宿主机架构与 ISO 文件名相符，再计算实际摘要。

**执行位置：macOS 宿主机（任意目录）**

```bash
printf 'Ubuntu Server ISO 绝对路径: '
IFS= read -r ISO_FILE

if [ ! -f "$ISO_FILE" ]; then
  printf '停止：文件不存在：%s\n' "$ISO_FILE" >&2
  exit 1
fi

HOST_ARCH="$(uname -m)"
ISO_NAME="$(basename "$ISO_FILE")"

case "$HOST_ARCH" in
  arm64)
    EXPECTED_ARCH_TOKEN='arm64'
    ;;
  x86_64)
    EXPECTED_ARCH_TOKEN='amd64'
    ;;
  *)
    printf '停止：尚未为宿主机架构 %s 选择默认镜像。\n' "$HOST_ARCH" >&2
    exit 1
    ;;
esac

case "$ISO_NAME" in
  *"$EXPECTED_ARCH_TOKEN"*.iso)
    printf '架构文件名检查通过：host=%s iso=%s\n' "$HOST_ARCH" "$ISO_NAME"
    ;;
  *)
    printf '停止：ISO 文件名与宿主机架构不匹配：host=%s iso=%s\n' \
      "$HOST_ARCH" "$ISO_NAME" >&2
    exit 1
    ;;
esac

file "$ISO_FILE"
shasum -a 256 "$ISO_FILE"
```

预期结果：

- 文件名包含宿主机对应的架构标记。
- `file` 将文件识别为 ISO 9660 或可启动光盘映像。
- `shasum` 输出 64 位十六进制 SHA-256 摘要。

再把实际摘要与官方 `SHA256SUMS` 中同名文件的摘要比较。

**执行位置：macOS 宿主机（ISO 与 `SHA256SUMS` 位于同一目录）**

```bash
printf 'Ubuntu Server ISO 绝对路径: '
IFS= read -r ISO_FILE

ISO_DIR="$(dirname "$ISO_FILE")"
ISO_NAME="$(basename "$ISO_FILE")"
CHECKSUM_FILE="$ISO_DIR/SHA256SUMS"

if [ ! -f "$ISO_FILE" ] || [ ! -f "$CHECKSUM_FILE" ]; then
  printf '%s\n' '停止：ISO 或 SHA256SUMS 不存在。' >&2
  exit 1
fi

EXPECTED_HASH="$(
  awk -v name="$ISO_NAME" \
    '$2 == name || $2 == "*" name { print $1; exit }' \
    "$CHECKSUM_FILE"
)"
ACTUAL_HASH="$(shasum -a 256 "$ISO_FILE" | awk '{ print $1 }')"

printf 'expected=%s\nactual=%s\n' "$EXPECTED_HASH" "$ACTUAL_HASH"

if [ -n "$EXPECTED_HASH" ] && [ "$EXPECTED_HASH" = "$ACTUAL_HASH" ]; then
  printf '%s\n' 'SHA-256 校验通过。'
else
  printf '%s\n' '校验失败：不要启动该 ISO，请重新核对下载来源。' >&2
  exit 1
fi
```

如果 `EXPECTED_HASH` 为空，通常是 ISO 被重命名，或 ISO 与校验文件不属于同一发布目录。不要手工修改校验清单来“让它通过”。

SHA-256 对比能发现文件损坏或清单不一致；若要验证校验清单本身的真实性，还应按 Ubuntu 官方流程使用 `SHA256SUMS.gpg` 和 Ubuntu Image Signing Key 建立签名信任链。

## 4. 创建虚拟机

### 4.1 创建前记录决策

创建前先结合 [[UTM 虚拟机资源规划]] 和 [[虚拟机网络模式与可达性]] 明确：

- 客户机架构与 UTM 虚拟化后端。
- vCPU、内存和磁盘上限。
- 必要的访问方向，以及 Shared Network、Bridged Network 或其他网络模式。
- 是否需要共享目录。
- 虚拟机名称和用途。

资源值应根据宿主机容量和工作负载选择，不要把示例规格照抄成唯一答案。为宿主机保留足够 CPU、内存和磁盘空间，并考虑虚拟磁盘增长、Docker 镜像和后续备份占用。虚拟磁盘容量上限不等于创建时立即占用或预留的宿主空间，具体见 [[虚拟磁盘的逻辑容量与实际占用]]。

### 4.2 使用创建向导

UTM 不同版本的文字和顺序可能略有变化，核心步骤如下：

1. 打开 UTM，选择新建虚拟机。
2. 选择 **Virtualize**。
3. 选择 **Linux**。
4. 选择已经校验通过的 Ubuntu Server ISO。
5. 按资源规划填写 vCPU 和内存。
6. 设置虚拟磁盘容量上限。
7. 按网络规划选择网络模式。
8. 默认先不配置项目工作区共享目录。
9. 保存虚拟机配置，并在启动前再次核对架构、ISO 和磁盘。

开发仓库长期放在 Linux 本地文件系统更接近服务器真实语义。共享目录适合临时交换安装文件或导出物，不应在未验证权限、文件监听、符号链接和性能行为前，直接作为主要开发目录。

> [!tip] 创建后先不要急着启动
> 重新打开虚拟机设置，确认 ISO 架构、启动介质、CPU、内存、磁盘和网络与规划一致。此时修正配置的成本最低。

## 5. 安装 Ubuntu Server

启动虚拟机后进入 Ubuntu Server 安装器。安装界面会随 Ubuntu 版本变化，但应逐项理解以下选择。

### 5.1 安装器基础选项与网络

#### 语言、安装器更新与键盘

1. 选择能够稳定操作的语言。该选择通常也会影响已安装系统的默认语言环境。
2. 如果出现安装器更新提示，网络正常且没有固定安装器版本的要求时，可以先更新以获取已发布的修复；网络受限或需要使用 ISO 内置版本重现安装时，可以跳过。
3. 选择与实际键盘一致的布局和变体。如有需要，使用 **Identify keyboard** 识别布局，并在继续前确认字母、数字和密码中可能使用的符号输入正常。

#### 安装类型与第三方驱动

部分 Ubuntu Server 安装器会在键盘配置后显示 **Choose the type of installation**。这里选择的是写入虚拟磁盘的 Ubuntu Server 安装源，不是 UTM 的 Virtualize 或 Emulate 模式。

| 界面选项 | 代表什么 | 普通 UTM 开发虚拟机如何选择 |
| --- | --- | --- |
| `Ubuntu Server` | 标准 Server 安装源，包含适合日常管理的常用软件包 | 主线选择此项 |
| `Ubuntu Server (minimized)` | 缩小运行时体积，偏向很少有人登录的系统 | 只有明确需要最小化基础系统，并接受后续自行补齐工具时才选择 |
| `Search for third-party drivers` | 让安装器搜索可用的第三方或专有驱动 | 普通 UTM 虚拟硬件保持未勾选；特殊硬件直通且已确认驱动需求时再评估 |

交互式后端开发机使用 `Ubuntu Server` 即可，这不会因此安装 Ubuntu Desktop。保持第三方驱动搜索未勾选，然后选择 **Done**。如果当前 ISO 没有显示该页面，直接沿用安装器默认的 Server 安装源继续。

#### 网络配置与继续条件

Ubuntu Server 安装器会列出 Linux 已识别的网络接口。界面文字会随版本变化，但常见字段的意义不变。

| 界面内容 | 代表什么 | 当前主线如何处理 |
| --- | --- | --- |
| `enp0s1` 一类名称 | Linux 为以太网接口生成的名称；实际名称可能不同 | 确认至少出现一块预期网卡 |
| `eth` | 该接口是以太网类型 | 无需单独修改 |
| `DHCPv4 192.168.64.2/24` 一类行 | DHCP 已为接口分配 IPv4 地址和子网前缀 | 使用 Shared Network 时通常可以直接继续；不要照抄示例地址 |
| `Virtio network device` | UTM 向客户机提供的 VirtIO 虚拟网卡 | 这不代表安装了错误的 Linux 发行版 |
| 冒号分隔的十六进制数 | 虚拟网卡的 MAC 地址 | 只用于识别网卡，不是 SSH 连接地址 |
| `Create bond` | 将多块网卡合成一个逻辑接口 | 当前单网卡开发虚拟机不需要创建 |

使用预先规划的 **Shared Network** 时，如果安装器已列出预期接口，并在 `DHCPv4` 后显示一个地址，可以选择 **Done** 继续。这只表明 DHCP 分配已成功；安装完成后仍要验证默认路由、DNS 和 HTTPS。

如果接口没有获得地址，先回到 UTM 检查虚拟网卡是否启用、网络模式是否与规划一致，不要猜测静态地址、网关或 DNS。只有所在网络明确提供了固定参数，或实际需求不能由 DHCP 满足时，才评估手动配置。

安装器内显示的 DHCP 地址不应写成长期固定资产。网络接口、IP 与前缀、默认路由、DNS 和验证命令详见 [[Linux 网络接口、IP 地址、路由与 DNS 基础]]。

#### 代理与软件源

1. 在代理配置页面，没有明确且已验证的代理时保持留空，不要照抄来源不明或只能从宿主机访问的代理地址。
2. 在 Ubuntu 软件源页面，先使用安装器自动选择的 Ubuntu 归档镜像，等待可用性检查通过后再选择 **Done**。
3. 如果镜像检查失败，先检查网卡地址、默认路由、DNS、系统时间和代理。只有团队或所在网络明确提供了可信镜像时才替换，不要为了跳过诊断而填写不明镜像。

### 5.2 存储

Ubuntu Server 的引导式存储页面同时包含“如何生成布局”“是否使用 LVM”“是否使用 LUKS”三个不同决策。LVM 是位于磁盘或分区与文件系统之间的逻辑卷管理层，不是文件系统，也不是手动分区的反义词。完整存储分层、PV/VG/LV 和容量检查方法见 [[Linux 磁盘、分区、文件系统与 LVM 基础]]。

| 界面选项 | 代表什么 | 当前单用途开发虚拟机如何选择 |
| --- | --- | --- |
| `Use an entire disk` | 让安装器为所选磁盘生成整盘布局 | 选择此项，并确认目标是 UTM 提供的空虚拟磁盘 |
| `Set up this disk as an LVM group` | 在底层磁盘与文件系统之间加入 LVM | 本文主线保持勾选，为后续扩展保留空间池能力 |
| `Encrypt the LVM group with LUKS` | 增加磁盘加密与启动解锁要求 | 没有明确威胁模型、口令保管和恢复方案时不勾选 |
| `Custom storage layout` | 不套用引导式布局，由用户自行设计 | 多磁盘、既有数据或独立挂载点有特殊要求时再使用 |

传统分区与 LVM 都可以由安装器自动生成，也都可以在自定义布局中手动配置。这里选择整盘 LVM 只是当前新建开发虚拟机的主线，不是所有 Linux 系统的唯一正确方案。

> [!important] 虚拟磁盘容量不一定全部分给根文件系统
> Canonical 当前文档中的默认 `scaled` LVM 策略会在 VG 中保留空闲空间。对于约 20～200 GiB 的可用空间，根 LV 默认可能只使用约一半。因此，180 GB 虚拟磁盘安装完成后，`df -h /` 可能显示明显更小的根文件系统；剩余容量通常仍位于 VG 中，并没有丢失。具体以当前安装器下一页生成的摘要为准，安装后可按 [[Linux 磁盘、分区、文件系统与 LVM 基础#7. 安装后的只读检查]] 使用 `vgs` 和 `lvs` 核对。

选择引导式方案后，先进入存储摘要页面，确认：

- 安装目标确实是预期虚拟磁盘，容量与 UTM 中设置的磁盘上限相符。
- 启动分区、LVM PV、VG、根 LV、文件系统和 `/` 挂载点均已生成。
- 根 LV 的容量与 VG 保留空间符合预期。
- 没有误选安装 ISO、额外数据盘或需要保留的既有分区。
- 只有理解最终摘要后，才提交安装器的写入确认。

> [!warning] 分区确认会覆盖目标虚拟磁盘上的原有内容
> 新建空虚拟机通常没有需要保留的数据；重装现有虚拟机前必须先建立 [[UTM 虚拟机快照、备份与恢复]] 中的独立备份。

### 5.3 用户、主机名与 SSH

安装器会先显示 **Profile configuration**，再显示独立的 **SSH Setup**。前一页创建本地用户并设置主机名，后一页决定是否安装 SSH 服务端以及允许哪些初始认证方式。

#### Profile configuration

此页面创建安装后的默认普通管理员用户。该用户可以通过 `sudo` 临时取得管理权限，因此即使计划只用 SSH 公钥远程登录，仍需要设置本地密码。

| 界面字段 | 代表什么 | 如何填写 |
| --- | --- | --- |
| `Your name` | 用户的显示名称，不是登录名 | 填写便于识别且不包含隐私或凭据的名称 |
| `Your server's name` | Ubuntu 的系统主机名 | 使用简短、稳定且在当前网络中易于区分的名称；建议只使用小写字母、数字和连字符，例如 `project-dev-linux` |
| `Pick a username` | 控制台、SSH 和 `sudo` 使用的 Linux 登录名 | 使用简短、稳定的小写用户名，不要使用 `root` |
| `Choose a password` | 上述 Linux 用户的本地密码 | 使用唯一且足够强的密码；不要与项目、主机名或其他账号共用 |
| `Confirm your password` | 密码确认 | 再次输入完全相同的密码 |

UTM 中的虚拟机显示名称与 Ubuntu 主机名是两个独立设置。二者可以使用相同名称以便识别，但不能因为 UTM 窗口已经有名称，就假定安装器会自动填写主机名。

#### SSH Setup

下一页决定是否安装 OpenSSH 服务端。Ubuntu 默认安装不会主动开放 SSH 端口；需要从宿主机或其他客户端远程管理这台虚拟机时，勾选 **Install OpenSSH server**。

| 界面选项 | 代表什么 | 如何选择 |
| --- | --- | --- |
| `Install OpenSSH server` | 安装并启用 SSH 服务端 | 需要远程管理时勾选；只使用 UTM 控制台时可以暂不安装 |
| `Import SSH identity` | 从 GitHub 或 Launchpad 为当前 Linux 用户导入公钥 | 只有确认账号由自己控制，并已核对其中的公钥时才导入；否则保持 `No`，安装后再手动添加 |
| `Allow password authentication over SSH` | 是否允许使用该用户的密码远程登录 | 未导入密钥但需要首次远程登录时可暂时保留；已经导入并核对公钥时通常不需要启用 |

SSH 密码认证只控制远程登录，不会取消本地用户密码或 `sudo` 的密码要求。不要根据相似的 GitHub 或 Launchpad 用户名盲目导入密钥，也不要把私钥交给安装器或复制到服务端。

安装完成后，应先从控制台核对 SSH 服务状态、客户机地址和主机指纹，再建立并验证公钥登录。只有控制台仍可用且新的密钥会话已经成功时，才关闭 SSH 密码认证。

用户、`sudo`、权限和安全初始化在 [[Ubuntu Server 初始化与基础安全]] 中继续完成；SSH 客户端、服务端、密钥和主机指纹见 [[OpenSSH 连接、密钥与主机指纹]]。

### 5.4 Featured Server Snaps

如果已配置网络，安装器可能在 SSH 之后显示 **Featured Server Snaps**。这些是可选软件，不是 Ubuntu Server 完成安装的前置条件。

当前开发虚拟机主线不预选任何 Snap，直接选择 **Done**。系统首次启动并完成基础验证后，再按项目要求确定软件来源、版本、端口和数据持久化方式。

### 5.5 等待安装完成

等待安装器明确显示安装完成后再重启。安装期间不要强制关闭 UTM；强制关机会丢弃未写入数据，甚至破坏虚拟磁盘文件系统。

## 6. 卸载 ISO 并从虚拟磁盘启动

安装结束后的第一次重启可能因为 ISO 仍挂载而再次进入安装器，或停在黑屏。UTM 官方 Ubuntu 指南建议：若安装完成后未自动正常重启，可以关闭虚拟机、卸载安装 ISO，再重新启动。

操作顺序：

1. 确认安装器已经明确完成写入。
2. 关闭虚拟机窗口或让客户机正常停止。
3. 在 UTM 的可移动驱动器或 CD/DVD 菜单中卸载 ISO。
4. 检查启动设备仍包含系统虚拟磁盘。
5. 再次启动虚拟机。

判断是否成功：

- 启动过程不再进入 Ubuntu 安装器。
- 出现已安装系统的登录提示。
- 使用安装时创建的普通用户可以登录。

> [!info] 登录后出现 cloud-init 和 SSH 主机密钥日志通常不是故障
> 首次从虚拟磁盘启动时，cloud-init 可能仍在完成最后阶段，其控制台输出会在 shell 提示符出现后继续刷屏。若看到 `Cloud-init ... finished`，按一次 Enter 恢复提示符即可，无需重启或重装。`no authorized SSH keys fingerprints found for user ...` 表示该用户尚未配置 SSH 授权公钥；`SSH HOST KEY FINGERPRINTS` 和 `SSH HOST KEY KEYS` 区块显示的是服务端主机公钥及其指纹，不是用户私钥泄露。

如果仍进入安装器，先检查 ISO 是否确实卸载，再检查启动顺序。不要重复安装覆盖已经写入的系统。

## 7. 首次启动验证

登录客户机控制台后，先做只读验证，不要立即安装大量工具。先确认安装器安排的首次启动初始化已经完成，再检查系统、磁盘、网络和时间。

### 7.1 确认安装器首次启动初始化完成

Ubuntu Server live installer 会让 cloud-init 参与已安装系统的首次启动，并在完成后将其禁用，避免后续重启时重复套用安装器配置。因此，不能只把 `status: done` 当作成功标志；`status: disabled` 也可能是安装器预期的最终状态。

**执行位置：Ubuntu 虚拟机（UTM 控制台，任意目录）**

```bash
printf '%s\n' 'Cloud-init first-boot state:'
sudo cloud-init status --long

if sudo test -f /etc/cloud/cloud-init.disabled; then
  printf '\n%s\n' 'Cloud-init disable marker:'
  sudo sed -n '1,3p' /etc/cloud/cloud-init.disabled
fi

if sudo test -f /var/lib/cloud/instance/boot-finished; then
  printf '\n%s\n' 'Cloud-init first-boot initialization finished.'
else
  printf '\n%s\n' 'Warning: cloud-init boot-finished marker is missing.' >&2
fi
```

结果解读：

| 关键状态 | 含义 | 处理 |
| --- | --- | --- |
| `status: disabled`、`boot_status_code: disabled-by-marker-file`、禁用文件说明由 Ubuntu live installer 创建，且 `errors: []` | 安装器首次启动初始化已完成，随后按设计禁用 cloud-init | 视为正常成功，继续后续验证 |
| `status: done`，且没有错误 | 其他镜像或部署路径中的 cloud-init 已完成 | 视为正常成功 |
| `status: running` | cloud-init 仍在执行 | 等待后重新检查，不要在刷盘或配置阶段强制关机 |
| 状态或扩展状态显示错误、降级，或 `errors` 非空 | 初始化未完整成功 | 查看具体错误和 cloud-init 日志后再决定是否修复 |
| `status: disabled`，但禁用原因不是安装器标记文件 | 与本文主线不同 | 先查明禁用来源，不要盲目重新启用 |

同时满足以下条件时，可确认本文主线的首次启动初始化成功：

- `boot_status_code` 为 `disabled-by-marker-file`。
- `/etc/cloud/cloud-init.disabled` 说明该文件由 Ubuntu live installer 在首次启动后创建。
- `errors: []`、`recoverable_errors: {}`，且 `/var/lib/cloud/instance/boot-finished` 存在。

其他字段和提示应这样理解：

- `detail: DataSourceNone` 表示没有使用外部云平台元数据源，不表示网络未连接。
- 如果 `last_update` 显示 1970 年，不要用该字段判断客户机的实际时间；系统时间以后续的 `timedatectl` 为准。
- 非 root 用户直接执行命令时，可能看到针对 `99-installer.cfg` 或安装器网络配置的 `REDACTED ... insufficient permissions` 警告；这些文件本就限制普通用户读取。使用 `sudo cloud-init status --long` 检查，不要为消除警告而放宽文件权限。

> [!warning] 普通开发虚拟机不要重新启用 cloud-init
> 不要删除 `/etc/cloud/cloud-init.disabled`，也不要为了把状态变成 `done` 而执行 `cloud-init clean --machine-id`。这会让后续启动再次进入 cloud-init 初始化路径。只有在明确制作模板或黄金镜像，并已规划机器标识、SSH 主机密钥和实例数据重生流程时，才应单独评估此操作。

### 7.2 验证系统、磁盘、网络、时区与时间同步

**执行位置：Ubuntu 虚拟机（UTM 控制台，任意目录）**

```bash
printf '%s\n' 'Architecture:'
uname -m
dpkg --print-architecture

printf '\n%s\n' 'Operating system:'
cat /etc/os-release

printf '\n%s\n' 'Root filesystem and block devices:'
findmnt /
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
df -h /

printf '\n%s\n' 'Network and route:'
ip -brief address
ip route

printf '\n%s\n' 'DNS and time:'
getent hosts ubuntu.com
timedatectl

printf '\n%s\n' 'Failed systemd units:'
systemctl --failed --no-pager
```

预期结果：

- `uname -m` 与 `dpkg --print-architecture` 对应所选 ISO。
- `/etc/os-release` 显示计划安装的 Ubuntu 系列。
- 根文件系统挂载在虚拟磁盘上，容量符合规划。
- 网卡有地址，并存在默认路由。
- `getent hosts ubuntu.com` 能解析域名。
- `timedatectl` 能显示当前时区与时间同步状态。
- `systemctl --failed` 没有未解释的失败单元。

`timedatectl` 中的 `Time zone` 与 `System clock synchronized` 是两个不同维度。`Time zone: Etc/UTC` 只表示 `Local time` 当前按 UTC 显示，不代表系统时钟错误；`System clock synchronized: yes` 表示系统时钟已同步，`NTP service: active` 表示网络时间同步服务处于活动状态。`RTC in local TZ: no` 表示 RTC 保持 UTC，通常无需改为本地时间。

若 IP 地址或 DNS 异常，不要先假设“Ubuntu 安装坏了”。按 [[虚拟机网络模式与可达性]] 和 [[Linux 网络接口、IP 地址、路由与 DNS 基础]] 从 UTM 虚拟网卡、客户机接口、DHCP、默认路由和 DNS 逐层检查。如果 `Time zone` 与该主机的用途不符，继续按 [[Ubuntu Server 初始化与基础安全#5. 设置时区并验证时间同步]] 设置；如果 `System clock synchronized` 为 `no` 或 `NTP service` 未激活，再检查客户机实际使用的时间同步服务、网络与日志。

## 8. 常见故障与恢复

| 现象 | 优先检查 | 恢复方式 |
| --- | --- | --- |
| 启动后进入 UEFI Interactive Shell | ISO 是否挂载、是否为正确架构、是否存在可移动驱动器 | 重新挂载已校验 ISO；按 UTM 官方指南检查 EFI 启动项 |
| 提示 `Exec format error` 或无法启动内核 | 宿主机、ISO 和 VM 架构是否一致 | 改用匹配架构的 ISO；跨架构需求单独使用 Emulate |
| SHA-256 校验失败 | ISO 与 `SHA256SUMS` 是否来自同一目录 | 删除损坏下载并从官方来源重新获取 |
| 安装完成后又进入安装器 | ISO 未卸载或启动顺序错误 | 关机、卸载 ISO、让虚拟磁盘优先启动 |
| 安装完成后重启停在黑屏 | ISO、显示设备或启动状态 | 按 UTM Ubuntu 指南关闭 VM、卸载 ISO后重新启动 |
| 客户机没有地址 | UTM 网卡、网络模式、客户机接口 | `ip link`、`ip route`；确认 UTM 虚拟网卡启用 |
| 能访问 IP，不能解析域名 | DNS 或时间 | `resolvectl status`、`getent hosts`、`timedatectl` |
| 宿主机明显卡顿 | vCPU、内存或真实磁盘空间分配过激 | 完全关机后降低资源，检查宿主与客户机空间 |
| 虚拟磁盘容量不够 | 初始规划不足、缓存或镜像增长 | 先备份，再按 UTM 官方磁盘扩容流程扩大并扩展客户机文件系统 |

### 安装失败时如何重新开始

如果安装从未完成，且虚拟磁盘中没有需要保留的数据，可以：

1. 关闭虚拟机。
2. 修正 ISO、架构、磁盘或网络配置。
3. 重新挂载已校验 ISO。
4. 重新运行安装器。

> [!danger] 从 UTM 列表移除虚拟机可能同时删除其数据
> UTM 官方 Actions 文档说明：位于默认路径的 VM 被移除时，数据也可能被删除。只有确认虚拟磁盘中没有需要保留的系统、源码或数据，并且独立备份可用时，才能删除 VM。不要把“移出列表”和“仅取消显示”视为同一操作。

如果系统已投入使用，应先按 [[UTM 虚拟机快照、备份与恢复]] 创建独立副本，再决定修复、重装或回退。

## 9. 安装完成后的下一步

虚拟机能够启动只代表操作系统安装完成，还不是可用的开发环境。下一步应按顺序完成：

1. [[Ubuntu Server 初始化与基础安全]]
2. [[OpenSSH 连接、密钥与主机指纹]]
3. [[Linux 开发工作区与本地文件系统规划]]
4. 项目所需的 Git、语言工具链与容器工具
5. [[UTM 虚拟机快照、备份与恢复]]

## 官方资料

- [UTM Documentation：Ubuntu 指南](https://docs.getutm.app/guides/ubuntu/)
- [UTM Documentation：QEMU System 与架构](https://docs.getutm.app/settings-qemu/system/)
- [UTM Documentation：Actions](https://docs.getutm.app/basics/actions/)
- [UTM Documentation：Controls](https://docs.getutm.app/basics/controls/)
- [UTM Documentation：Resize and Compress](https://docs.getutm.app/settings-qemu/drive/resize-and-compress/)
- [Ubuntu 官方镜像完整性验证](https://documentation.ubuntu.com/security/software-integrity/image-verification/)
- [Ubuntu installation documentation：Screen-by-screen installer walk-through](https://canonical-subiquity.readthedocs-hosted.com/en/latest/tutorial/screen-by-screen.html)
- [Ubuntu installation documentation：Configuring storage](https://canonical-subiquity.readthedocs-hosted.com/en/latest/howto/configure-storage.html)
- [Ubuntu Server：About Logical Volume Management](https://ubuntu.com/server/docs/explanation/storage/about-lvm/)
- [Ubuntu installation documentation：LVM sizing policy](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html#sizing-policy)
- [Ubuntu installation documentation：Autoinstall source reference](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html#source)
- [Ubuntu installation documentation：Cloud-init and autoinstall interaction](https://canonical-subiquity.readthedocs-hosted.com/en/latest/explanation/cloudinit-autoinstall-interaction.html)
- [Cloud-init documentation：Query cloud-init status](https://docs.cloud-init.io/en/latest/howto/status.html)
- [Ubuntu Server 下载](https://ubuntu.com/download/server)

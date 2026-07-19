---
title: APT 软件包管理基础
aliases:
  - Ubuntu APT 软件包管理
  - apt apt-get dpkg 基础
  - Ubuntu 软件安装更新与卸载
tags:
  - Linux
  - Linux/系统管理
  - Ubuntu
  - APT
  - 软件包管理
created: 2026-07-19T23:47:26
updated: 2026-07-20T00:49:15
---

本文建立 Ubuntu Server 软件包管理的最小模型：先理解软件源、软件包索引和已安装状态的区别，再按“刷新索引 → 查看计划 → 执行变更 → 验证结果”的顺序操作。目标不是收集所有 APT 选项，而是能够安全地安装、更新和移除软件。

## 1. APT 管理的不是一个下载命令

Ubuntu 通常以 Debian 软件包（`.deb`）分发系统软件。软件包除了文件，还包含版本、依赖关系、维护脚本和配置文件等元数据。

APT 工作时涉及三个不同层次：

```text
软件源
  -> 本机的软件包索引
  -> 依赖解析与变更计划
  -> dpkg 管理的已安装状态和文件
```

- **软件源**：系统信任并配置的仓库位置。
- **软件包索引**：本机保存的“仓库当前提供什么版本”的元数据副本。
- **已安装状态**：本机实际安装了哪些包、版本和文件。

`apt update` 只刷新第二层，不会把所有已安装软件更新到新版本；`apt upgrade` 才依据当前索引计算并执行更新。两条命令名称相近，但职责完全不同。

本篇只讨论 Ubuntu/Debian 的 APT 路线。Snap、容器镜像、语言包管理器和手工解压的软件不自动纳入同一套升级与卸载规则。

## 2. 掌握目标

| 层级 | 应达到的程度 | 命令与能力 |
| --- | --- | --- |
| 必须熟练 | 能区分刷新索引与升级，审查摘要后安装明确软件包或执行常规升级 | `apt update`、`apt install`、`apt upgrade` 的最小骨架 |
| 理解会查 | 能搜索与查看候选包、列出升级项、选择移除方式并核对安装状态 | `apt search`、`apt show`、`apt list`、`apt remove`、`apt purge`、`apt autoremove`、`dpkg-query`，理解 `apt-get` 与 `dpkg` 边界 |
| 认识即可 | 知道风险，用到时按官方流程处理 | `full-upgrade`、hold/pinning、第三方软件源、离线包、损坏依赖修复、发行版升级 |

熟练的标志不是不加判断地运行 `sudo apt upgrade -y`，而是能读懂 APT 给出的“将安装、升级、移除、保留哪些包”的摘要，并在异常时停止。

## 3. 四个工具的职责边界

| 工具 | 主要角色 | 适合场景 | 不应误解为 |
| --- | --- | --- | --- |
| `apt` | 面向人的高级交互式界面 | 日常搜索、查看、安装、升级和移除 | 稳定的脚本输出接口 |
| `apt-get` | APT 的较低层命令行界面 | 自动化脚本或官方流程明确要求 | 另一套独立软件包数据库 |
| `dpkg` | 安装状态与本地 `.deb` 的底层管理 | 查询底层状态、处理已取得的包文件 | 自动从仓库解决全部依赖的工具 |
| `dpkg-query` | 只读查询 dpkg 数据库 | 精确列出已安装包、版本、状态和文件 | 搜索仓库中尚未安装的软件 |

交互操作优先使用 `apt`。脚本优先遵循 Ubuntu 官方建议使用 `apt-get`，并固定非交互策略、日志和失败处理；不要解析给人阅读的 `apt` 输出。

安装本地 `.deb` 时，`sudo apt install ./package.deb` 通常比直接 `sudo dpkg -i package.deb` 更方便，因为 APT 可以在已配置的软件源中解析依赖。任何第三方包仍然意味着信任其发布者和维护脚本。

## 4. 命令骨架

查询仓库或本机状态：

```text
apt search 搜索词
apt show 软件包名
apt list [--installed|--upgradable] [模式]
dpkg-query [选项] [软件包名或模式]
```

改变软件包状态：

```text
sudo apt update
sudo apt upgrade
sudo apt install 软件包名...
sudo apt remove 软件包名...
sudo apt purge 软件包名...
sudo apt autoremove
```

“软件包名”是说明性占位符，不能原样输入。变更命令需要管理员权限，并可能启动、停止或重启由软件包提供的服务。

## 5. 五种高频任务模式

### 5.1 搜索、查看并确认软件包

```bash
apt search 'openssh server'
apt show openssh-server
apt list --installed openssh-server
dpkg-query -W -f='package=${binary:Package}\nversion=${Version}\nstatus=${Status}\n' openssh-server
```

- `apt search` 在当前本地索引的名称和描述等信息中搜索，结果可能较多。
- `apt show` 查看候选版本、依赖、描述和下载大小等元数据。
- `apt list --installed PACKAGE` 只看匹配包的已安装列表。
- `dpkg-query -W` 直接查询本机 dpkg 数据库；自定义 `-f` 可以稳定选择需要的字段。

如果本机索引很久没有刷新，`apt search` 和 `apt show` 看到的仓库信息可能过时。先判断是否可以联网、软件源是否可信，以及当前是否处于允许维护的时间窗口。

### 5.2 刷新索引并查看可升级项

```bash
sudo apt update
apt list --upgradable
```

`apt update` 从已配置软件源获取新的索引，写入本机 APT 状态目录。它可能暴露软件源签名、网络、代理或发行版配置问题，但不会直接升级已安装软件。

`apt list --upgradable` 使用当前索引列出候选版本高于已安装版本的包。某些更新可能因分阶段发布、依赖关系或 hold 暂时不执行；不要为了消除“kept back”提示就绕过默认策略。

检查当前软件源配置时，应先只读查看系统真实文件。Ubuntu 24.04 LTS 及之后的默认配置通常位于 `/etc/apt/sources.list.d/ubuntu.sources`；旧版本常使用 `/etc/apt/sources.list`。不要未经核对就复制其他发行版或版本的软件源。

### 5.3 审查并执行安装或升级计划

安装明确需要的软件包：

```bash
sudo apt install PACKAGE_NAME
```

升级当前发行版中允许升级的软件包：

```bash
sudo apt update &&
  apt list --upgradable &&
  sudo apt upgrade
```

运行前后都要读 APT 摘要，重点关注：

- 将新安装哪些依赖。
- 将升级哪些关键组件。
- 是否计划移除软件包。
- 需要下载和新增占用多少空间。
- 是否有服务会重启或弹出配置选择。

只预览依赖变更时，可以使用模拟模式：

```bash
apt -s install PACKAGE_NAME
apt -s upgrade
```

模拟不执行变更，但它基于当前索引和配置，不能证明真实安装时网络、磁盘、维护脚本和并发状态一定成功。

`apt upgrade` 不等于把 Ubuntu 升级到下一个发行版。发行版升级由 `do-release-upgrade` 等专门流程负责，必须单独规划备份、兼容性和维护窗口。

### 5.4 在 remove、purge 与 autoremove 之间选择

```bash
sudo apt remove PACKAGE_NAME
sudo apt purge PACKAGE_NAME
sudo apt autoremove
```

- `remove` 移除软件包主体，通常保留被 dpkg 认定为配置文件的内容，方便以后重新安装。
- `purge` 连同 dpkg 管理的系统配置一起移除。
- `autoremove` 清理曾作为依赖自动安装、当前已不再需要的软件包。

`purge` 并不承诺删除应用创建的全部数据、数据库、日志或用户主目录配置。`autoremove` 也不是无条件安全的磁盘清理：执行前必须审查候选列表，确认没有仍由人工流程依赖的软件包。

如果只想了解计划，先模拟：

```bash
apt -s remove PACKAGE_NAME
apt -s purge PACKAGE_NAME
apt -s autoremove
```

生产服务或开发工具链的移除，还要检查 systemd unit、数据目录、备份和恢复方法。

### 5.5 处理锁冲突与重启提示

APT 和 dpkg 使用锁来避免两个包管理进程同时改写数据库。遇到“Could not get lock”时，先观察谁正在工作：

```bash
pgrep -a -f 'apt|dpkg|unattended-upgrade'
systemctl status apt-daily.service apt-daily-upgrade.service --no-pager
sudo journalctl -u apt-daily.service -u apt-daily-upgrade.service -n 80 --no-pager
```

Ubuntu Server 可能通过 `unattended-upgrades` 自动应用安全更新。正常做法是等待活动任务结束，或在明确的维护策略下处理它，而不是删除锁文件。

> [!warning] 不要直接删除 APT 或 dpkg 锁文件
> 锁是并发保护信号，不是导致问题的普通缓存文件。活动事务期间删除锁并启动第二个事务，可能破坏软件包数据库。只有确认没有包管理进程运行后，才根据实际错误使用 Ubuntu/Debian 官方恢复步骤，例如完成被中断的 `dpkg --configure -a`；不要把恢复命令当成日常固定步骤。

更新后检查系统是否留下常见的重启标记：

```bash
if [ -f /var/run/reboot-required ]; then
  cat /var/run/reboot-required
  if [ -r /var/run/reboot-required.pkgs ]; then
    cat /var/run/reboot-required.pkgs
  fi
else
  printf '%s\n' '当前未发现 /var/run/reboot-required 标记'
fi
```

该标记提示某些更新建议重启，不会替你选择安全的重启时机。标记不存在也不能证明所有长期运行进程都已经加载新库。还应结合服务重启提示、`needrestart` 输出（若系统安装并启用）和维护窗口判断。

远程主机重启前必须确认控制台或备用访问路径、业务影响和启动后验证步骤，见 [[Ubuntu Server 初始化与基础安全]]。

## 6. 只读、变更与风险边界

| 命令 | 性质 | 主要风险 |
| --- | --- | --- |
| `apt search`、`apt show`、`apt list`、`dpkg-query` | 只读查询 | 仓库结果受本地索引新旧影响；输出可能很长 |
| `apt -s ...` | 模拟 | 不修改包状态，但结果不能覆盖真实执行时的全部失败条件 |
| `apt update` | 更新本地索引 | 需要网络并写 APT 状态；会暴露源、签名或代理问题 |
| `apt upgrade`、`apt install` | 系统变更 | 改文件、运行维护脚本、占用磁盘，服务可能重启 |
| `apt remove`、`apt purge` | 系统变更 | 服务和能力被移除；`purge` 还删除受管配置 |
| `apt autoremove` | 系统变更 | 审查不足可能移除仍被人工依赖的软件包 |
| `dpkg -i`、`dpkg --configure -a` | 底层系统变更 | 可能留下未满足依赖或继续被中断的事务，只在明确场景使用 |

不要把 `-y` 当成通用便利选项。它会自动确认计划，却不能替你判断计划是否正确。学习和人工维护阶段先保留交互确认。

第三方软件源和 `.deb` 的维护脚本通常以管理员权限执行。只添加来源、密钥、发行版与支持周期都经过核对的软件源。

## 7. 安全练习

以下练习只查询当前索引和已安装数据库，不安装、升级或删除任何软件，也不需要 `sudo`：

```bash
apt search '^bash$'
apt show bash
apt list --installed bash
apt list --upgradable

dpkg-query -W -f='package=${binary:Package}\nversion=${Version}\nstatus=${Status}\n' bash
dpkg-query -L bash | head -n 20

test -f /var/run/reboot-required \
  && printf '%s\n' '发现重启标记' \
  || printf '%s\n' '当前未发现重启标记'
```

练习时回答：`apt show bash` 中的仓库候选信息和 `dpkg-query` 中的本机已安装信息分别来自哪一层？如果 `apt list --upgradable` 没有输出可升级包，能否证明软件源索引刚刚刷新过？

需要练习真实安装时，应在可恢复的测试虚拟机中选择一个明确的非关键软件包，先运行 `apt -s install PACKAGE_NAME`，审查计划后再执行，并记录验证和移除路径。

## 8. 不看笔记自测

1. 软件源、本地索引和已安装状态分别是什么？
2. 为什么 `apt update` 不会把已安装软件全部升级？
3. `apt search` 与 `dpkg-query -W` 查询的范围有什么不同？
4. 交互命令与自动化脚本为什么分别偏向 `apt` 和 `apt-get`？
5. `remove`、`purge`、`autoremove` 分别解决什么问题？
6. 为什么 `purge` 不能保证删除数据库和用户主目录中的全部数据？
7. 遇到 APT 锁冲突时，为什么不能直接删除锁文件？
8. `/var/run/reboot-required` 存在和不存在分别能说明什么、不能说明什么？
9. 为什么 `apt upgrade` 与发行版升级不是同一件事？

## 9. 常见误解

| 误解 | 正确认识 |
| --- | --- |
| `apt update` 会更新已安装的软件 | 它主要刷新本地软件包索引 |
| `apt upgrade` 会升级到下一版 Ubuntu | 它更新当前发行版的软件包，发行版升级是单独流程 |
| 搜索不到包就代表仓库没有 | 本地索引可能过期，软件源组件或架构也可能不同 |
| `apt` 与 `dpkg` 是两套互不相关的数据库 | APT 做高级解析，底层安装状态由 dpkg 管理 |
| 直接 `dpkg -i` 总比 APT 简单 | dpkg 不负责从仓库完整解决依赖，`apt install ./FILE.deb` 常更合适 |
| `remove` 会清掉全部配置和数据 | 它通常保留受管配置，更不会自动识别所有应用数据 |
| `purge` 等于彻底擦除软件所有痕迹 | 它只针对包管理系统掌握的范围，不等于数据清理保证 |
| `autoremove` 可以不看列表直接确认 | 自动依赖判断不理解你的人工使用约定，必须审查计划 |
| 锁文件妨碍安装，删掉即可 | 锁用于保护事务并发，删除可能损坏数据库 |
| 加 `-y` 只是少按一次回车 | 它跳过人工计划确认，会放大错误命令的影响 |
| 没有重启标记就无需任何服务重启 | 长期运行进程是否加载新版本还需结合具体更新判断 |

## 相关笔记

- [[Linux 命令行学习路线与命令地图]]
- [[Shell 命令结构、类型与帮助系统]]
- [[Shell 标准流、管道、重定向与退出状态]]
- [[Linux 进程与系统资源常用命令]]
- [[systemd 服务与日志基础]]
- [[Ubuntu Server 初始化与基础安全]]

## 官方参考资料

以下资料于 **2026-07-19** 核对：

- [Ubuntu Server：Install and manage packages](https://documentation.ubuntu.com/server/how-to/software/package-management/)
- [Ubuntu Server：Managing your software](https://documentation.ubuntu.com/server/tutorial/managing-software/)
- [Ubuntu Server：Automatic updates](https://documentation.ubuntu.com/server/how-to/software/automatic-updates/)
- [Ubuntu Server：About apt upgrade and phased updates](https://documentation.ubuntu.com/server/explanation/software/about-apt-upgrade-and-phased-updates/)
- [Debian：APT User's Guide](https://www.debian.org/doc/manuals/apt-guide/index.en.html)
- [Debian Manpages：dpkg(1)](https://manpages.debian.org/testing/dpkg/dpkg.1.en.html)
- [Debian Manpages：dpkg-query(1)](https://manpages.debian.org/testing/dpkg/dpkg-query.1.en.html)

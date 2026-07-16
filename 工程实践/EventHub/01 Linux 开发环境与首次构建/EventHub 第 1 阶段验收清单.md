---
title: EventHub 第 1 阶段验收清单
aliases:
  - EventHub Linux 开发环境完成标准
  - EventHub 第 1 阶段恢复基线
tags:
  - 工程实践
  - EventHub
  - Linux
  - 验收
  - 备份恢复
created: 2026-07-17T00:48:00
updated: 2026-07-17T01:12:07
---

本文定义 EventHub 第 1 阶段的退出标准。它只说明需要证明什么，不把尚未执行的步骤写成已经成功。实际版本、提交和命令结果写入新的执行记录。

## 验收原则

- 每个勾选项都应有命令输出、截图或恢复记录支撑。
- “预期结果”与“当前实际结果”分开记录。
- 总命令退出码为 0 不代表内部没有 `SKIP`。
- Compose 静态解析成功不代表服务已启动或健康。
- 虚拟机快照不等于源码、数据库和密钥备份。
- 失败项应记录原因和下一步，不通过删除证据来获得干净结果。

## 1. 虚拟机与系统

- [ ] Ubuntu Server 能从虚拟磁盘正常启动，安装 ISO 已卸载。
- [ ] `uname -m` 与 `dpkg --print-architecture` 显示预期 ARM64 架构。
- [ ] vCPU、内存和磁盘与本次批准基线一致，或偏差已经记录原因。
- [ ] 虚拟机能访问网络、解析 DNS，并确认系统时间同步。
- [ ] 当前用户是普通用户，`sudo` 权限经过实际验证。
- [ ] 主机名、时区、HOME、目录属主和权限符合预期。
- [ ] APT 软件包索引和更新状态已经审查。
- [ ] 能使用 `systemctl` 和 `journalctl` 检查服务与日志。
- [ ] UFW 启用时已经先放行 SSH，并通过新会话验证。

建议证据：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
uname -a
dpkg --print-architecture
nproc
free -h
df -hT
id
sudo -v
timedatectl status
ip -brief address
ip route
getent hosts archive.ubuntu.com
```

## 2. SSH 日常主路径

- [ ] Mac mini 能通过传统 SSH 密钥登录 Ubuntu 虚拟机。
- [ ] 首次连接前已经通过 UTM 控制台核对主机指纹。
- [ ] 新开 SSH 会话仍然成功，不依赖仍打开的旧会话。
- [ ] `~/.ssh/config` 使用可维护的主机别名，不把动态 IP 当作永久事实。
- [ ] 能解释客户端私钥、服务端 `authorized_keys` 和客户端 `known_hosts` 的区别。
- [ ] 知道如何通过 UTM 控制台恢复错误的 `sshd` 或 UFW 配置。
- [ ] MacBook Air/Tailscale 路径若未配置，已经明确记录为可选而非失败项。

## 3. Linux 本地工作区与工具链

- [ ] 两个仓库位于 `$HOME/src` 下的 Linux 本地文件系统。
- [ ] 仓库不长期运行在 UTM 共享挂载或 iCloud 路径。
- [ ] Git、Go、Java、Maven或 Wrapper、Docker、Compose 版本符合当前项目约束。
- [ ] Docker daemon 正常，当前用户能按批准方式访问。
- [ ] `docker run --rm hello-world` 成功。
- [ ] Node.js 只在复现 Go OpenAPI lint 门禁时安装，并符合当前 CI。
- [ ] 日常 Git、Go、Maven 和项目构建不依赖 `sudo`。

## 4. 仓库身份与迁移完整性

对两个仓库分别确认：

- [ ] 远程地址正确。
- [ ] 当前分支正确。
- [ ] 本地 HEAD 与预期源提交一致。
- [ ] 上游状态和未推送提交已经解释。
- [ ] 未提交、未跟踪和忽略文件的迁移范围已经审计。
- [ ] fresh clone 无法获得的本地工程文件没有被误认为远端基线。
- [ ] 使用 bundle、patch 或 `rsync` 时已经完成来源、目标和传输后校验。

**执行位置：Ubuntu 虚拟机（各项目根目录）**

```bash
git remote -v
git branch --show-current
git rev-parse HEAD
git status --short --branch
git diff --check
```

## 5. EventHub Go 质量门禁

- [ ] `go version` 满足当前 `go.mod`。
- [ ] `go mod download` 成功。
- [ ] `go build ./...` 成功。
- [ ] 在干净 checkout 上完成 `make quality-check`。
- [ ] `make openapi-lint` 成功。
- [ ] 在干净 checkout 上完成 `make generated-check`。
- [ ] PR 场景针对真实 base ref 完成 `make openapi-breaking-check`。
- [ ] MySQL repository 集成测试明确显示 `PASS`，没有因为 Docker provider 不健康而 `SKIP`。
- [ ] `docker compose config --quiet` 成功。
- [ ] `make docker-build` 成功并能检查本地镜像。
- [ ] 验证前后 Git 状态已经对照。

## 6. EventHub Java 质量门禁

- [ ] `java`、`javac` 和 Maven 实际运行在当前 POM 要求的 JDK 版本。
- [ ] 已证明当前 revision 是否包含已跟踪 Maven Wrapper。
- [ ] fresh clone 使用当前 revision 实际存在的 Maven入口完成测试。
- [ ] 测试输出为 `BUILD SUCCESS`。
- [ ] 生产 profile 打包成功并生成预期 Jar。
- [ ] `docker compose config --quiet` 成功。
- [ ] 若完整迁移了本地质量文件，`make ci` 或当前 CI 等价入口成功。
- [ ] 已明确 H2 + Flyway 测试不等于真实 MySQL 行为验证。
- [ ] 验证前后 Git 状态已经对照。

## 7. 备份与恢复

- [ ] 已确认当前 UTM backend 和版本实际支持的快照、Duplicate 或导出能力。
- [ ] 已在正常关机状态创建独立 VM 基线副本。
- [ ] 已记录副本位置、日期、UTM 版本、客户机版本和阶段提交。
- [ ] 已为已提交源码建立 Git 远端或经过验证的 bundle。
- [ ] 必须保留的未提交和未跟踪内容已有独立处理。
- [ ] 已明确未来数据库和 Docker 卷需要独立逻辑备份。
- [ ] 已在不覆盖原 VM 的前提下做恢复核对。
- [ ] 恢复副本的 SSH 和 Tailscale 身份不会与原机未经处理地同时上线。

详细原理见 [[UTM 虚拟机快照、备份与恢复]]。

## 8. 阶段边界

- [ ] 没有把“能构建”描述成“已经部署”。
- [ ] 没有在第 1 阶段扩张到云主机、域名、TLS 或正式 CI/CD。
- [ ] 没有提前引入 Kubernetes。
- [ ] 已知第 2 阶段将从云主机和 systemd Go 原生部署开始。

## 执行记录模板

每次验收新建一篇记录，至少包含：

1. 日期、执行人和目标环境。
2. 宿主机、客户机和工具版本。
3. 两仓库分支、完整 SHA 和工作区状态。
4. 实际执行命令、退出结果和 `SKIP` 情况。
5. 失败处理和残留风险。
6. 创建的快照、备份和恢复验证。
7. 最终结论：通过、部分通过或未通过。

准备阶段的只读事实见 [[2026-07-16 第 1 阶段准备检查记录]]。

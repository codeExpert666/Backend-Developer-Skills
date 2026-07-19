---
title: systemd 服务与日志基础
aliases:
  - systemctl 与 journalctl
  - Linux 服务和日志基础
tags:
  - Linux
  - Linux/系统管理
  - systemd
  - systemctl
  - journalctl
created: 2026-07-17T00:48:00
updated: 2026-07-20T00:49:15
---

本文介绍 systemd、unit、service、启动状态和 journal 日志的基础使用。目标是能回答“服务是否运行、为什么失败、是否会开机启动、配置从哪里加载”，而不是遇到问题就反复 `restart`。

> [!abstract] 本篇掌握目标
> - **必须熟练**：区分 active 与 enabled，使用 `systemctl status` 和 `journalctl -u` 读取状态与日志，先检查再变更。
> - **理解会查**：按需使用 start、stop、restart、reload、enable、disable 和 `daemon-reload`，找到 unit 主文件、drop-in 与实际 `ExecStart`，并保留远程恢复通道。
> - **认识即可**：其他 unit 类型、复杂依赖关系和更高级的 journal 筛选；遇到服务编排或排障时再深入。
>
> 通用命令结构与帮助检索见 [[Linux 命令行学习路线与命令地图]] 和 [[Shell 命令结构、类型与帮助系统]]；本文较长的变量、判断和循环代码块可配合 [[Shell 脚本阅读基础]] 阅读。程序、进程与服务的边界见 [[Linux 进程与系统资源常用命令]]。

## 1. systemd 与 unit

在常见 Ubuntu Server 安装中，systemd 是 PID 1，负责系统启动、服务生命周期和依赖关系。

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
ps -p 1 -o pid=,comm=,args=
systemctl --version
```

systemd 管理的不只有 service：

| unit 类型 | 典型用途 |
| --- | --- |
| `.service` | 后台服务 |
| `.socket` | socket 激活 |
| `.timer` | 定时任务 |
| `.mount` | 挂载点 |
| `.path` | 监视路径变化 |
| `.target` | 一组启动目标 |

## 2. active 与 enabled 是两个维度

- `active`：unit 当前被 systemd 认为处于 active 状态；`active (exited)` 等状态不保证存在常驻进程，还要结合 unit 类型、`MainPID` 和 `status` 判断。
- `enabled`：系统进入对应 target 时是否计划自动启动。

unit 可以 active 但 disabled，例如手工启动；也可以 enabled 但因为启动失败而不 active。

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
systemctl is-active ssh.socket
systemctl is-enabled ssh.socket
systemctl status ssh.socket ssh.service --no-pager
```

Ubuntu 的 OpenSSH 可能由 `ssh.socket` 监听并按需激活 `ssh.service`，因此 service 显示 inactive 或 disabled 时，SSH 入口仍可能正常；这里同时查看两个 unit，不能只凭 service 一项下结论。具体机制见 [[OpenSSH 连接、密钥与主机指纹#3. 在服务端准备 sshd]]。

列出本次系统中的失败 unit：

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
systemctl --failed --no-pager
```

`status` 只展示部分近期日志。深入排查还需要 `journalctl`。

## 3. 启动、停止、重启与 reload

| 操作 | 含义 | 风险 |
| --- | --- | --- |
| `start` | 启动未运行服务 | 可能占用端口或访问数据 |
| `stop` | 停止服务 | 会中断依赖该服务的请求 |
| `restart` | 停止后重新启动 | 可能中断现有连接 |
| `reload` | 让支持该能力的服务重读配置 | 并非所有服务都实现 |

> [!warning] 远程服务先保留恢复通道
> 操作 `ssh.service`、防火墙、网络或远程代理前，保留控制台或另一个已验证会话。不要在唯一 SSH 会话中直接停止服务。

下面一次只执行所选动作。

**执行位置：Ubuntu 主机（具有 sudo 权限的会话，任意目录）**

```bash
(
printf '请输入 service 基名，不含 .service：'
IFS= read -r SERVICE_NAME
printf '请输入动作 start、stop、restart 或 reload：'
IFS= read -r SERVICE_ACTION

case "$SERVICE_NAME" in
  ''|-*|*[!A-Za-z0-9@_.-]*)
    printf '%s\n' '停止：service 名称格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac

case "$SERVICE_ACTION" in
  start|stop|restart|reload)
    sudo systemctl "$SERVICE_ACTION" "$SERVICE_NAME.service"
    ;;
  *)
    printf '%s\n' '停止：不支持的动作。' >&2
    exit 1
    ;;
esac

systemctl status "$SERVICE_NAME.service" --no-pager
)
```

这些含输入校验的代码块显式运行在圆括号创建的子 Shell 中；校验失败时的 `exit` 只停止当前代码块，不会退出登录 Shell。后文同类代码块采用相同边界。

如果服务配置有自己的校验命令，应在 restart/reload 前运行。例如 OpenSSH 使用 `sshd -t`，Nginx 使用 `nginx -t`。systemd 的启动失败信息不能替代服务自身语法校验。

## 4. enable、disable 与开机启动

`enable` 主要创建开机启动关系，不一定立刻启动；`disable` 取消这些关系，通常也不会立刻停止当前服务。

**执行位置：Ubuntu 主机（具有 sudo 权限的会话，任意目录）**

```bash
(
printf '请输入 service 基名，不含 .service：'
IFS= read -r SERVICE_NAME
printf '请输入动作 enable 或 disable：'
IFS= read -r ENABLE_ACTION

case "$SERVICE_NAME" in
  ''|-*|*[!A-Za-z0-9@_.-]*)
    printf '%s\n' '停止：service 名称格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac

case "$ENABLE_ACTION" in
  enable|disable)
    sudo systemctl "$ENABLE_ACTION" "$SERVICE_NAME.service"
    ;;
  *)
    printf '%s\n' '停止：不支持的动作。' >&2
    exit 1
    ;;
esac

systemctl is-enabled "$SERVICE_NAME.service"
)
```

如果明确需要“设置开机启动并立即启动”，可在核对依赖、端口和数据目录后执行：

**执行位置：Ubuntu 主机（具有 sudo 权限的会话，任意目录）**

```bash
(
printf '请输入 service 基名，不含 .service：'
IFS= read -r SERVICE_NAME

case "$SERVICE_NAME" in
  ''|-*|*[!A-Za-z0-9@_.-]*)
    printf '%s\n' '停止：service 名称格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac

sudo systemctl enable --now "$SERVICE_NAME.service"
systemctl status "$SERVICE_NAME.service" --no-pager
)
```

不要批量 enable 不理解的服务。

## 5. 使用 journalctl 查看日志

### 当前启动以来的服务日志

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
(
printf '请输入 service 基名，不含 .service：'
IFS= read -r SERVICE_NAME

case "$SERVICE_NAME" in
  ''|-*|*[!A-Za-z0-9@_.-]*)
    printf '%s\n' '停止：service 名称格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac

sudo journalctl -u "$SERVICE_NAME.service" -b --no-pager
)
```

### 最近日志

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
(
printf '请输入 service 基名，不含 .service：'
IFS= read -r SERVICE_NAME

case "$SERVICE_NAME" in
  ''|-*|*[!A-Za-z0-9@_.-]*)
    printf '%s\n' '停止：service 名称格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac

sudo journalctl -u "$SERVICE_NAME.service" -n 100 --no-pager
)
```

### 持续跟踪

**执行位置：Ubuntu 主机（交互式终端，任意目录；按 `Ctrl-C` 退出）**

```bash
(
printf '请输入 service 基名，不含 .service：'
IFS= read -r SERVICE_NAME

case "$SERVICE_NAME" in
  ''|-*|*[!A-Za-z0-9@_.-]*)
    printf '%s\n' '停止：service 名称格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac

sudo journalctl -u "$SERVICE_NAME.service" -f
)
```

### 系统级警告

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
sudo journalctl -b -p warning --no-pager
```

日志可能包含用户名、主机名、路径、地址、请求参数或令牌片段。分享前必须脱敏。

## 6. 修改 unit 后的固定顺序

修改 unit 文件或 drop-in 后：

1. 保存原配置或记录恢复方式。
2. 检查新文件权限和内容。
3. 运行服务自身配置校验。
4. 执行 `daemon-reload`。
5. restart 或 reload 目标服务。
6. 检查状态和 journal。
7. 从新的客户端路径验证。

`daemon-reload` 只让 systemd 重新读取 unit 定义，不会自动重启服务。

**执行位置：Ubuntu 主机（具有 sudo 权限的会话，任意目录）**

```bash
(
printf '请输入 service 基名，不含 .service：'
IFS= read -r SERVICE_NAME

case "$SERVICE_NAME" in
  ''|-*|*[!A-Za-z0-9@_.-]*)
    printf '%s\n' '停止：service 名称格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac

if sudo systemctl daemon-reload &&
   sudo systemctl restart "$SERVICE_NAME.service"; then
  systemctl status "$SERVICE_NAME.service" --no-pager
  sudo journalctl -u "$SERVICE_NAME.service" -b -n 100 --no-pager
else
  printf '%s\n' 'daemon-reload 或 restart 失败；未继续假定新 unit 已生效。' >&2
  systemctl status "$SERVICE_NAME.service" --no-pager || true
  sudo journalctl -u "$SERVICE_NAME.service" -b -n 100 --no-pager || true
  exit 1
fi
)
```

若重启失败，先用 journal 和服务自身校验定位原因。需要恢复时还原之前保存的 unit/drop-in，再次 `daemon-reload` 和 restart。

## 7. 查找 unit 来源与最终配置

**执行位置：Ubuntu 主机（任意目录，只读）**

```bash
(
printf '请输入 service 基名，不含 .service：'
IFS= read -r SERVICE_NAME

case "$SERVICE_NAME" in
  ''|-*|*[!A-Za-z0-9@_.-]*)
    printf '%s\n' '停止：service 名称格式不符合本文保护规则。' >&2
    exit 1
    ;;
esac

systemctl cat "$SERVICE_NAME.service"
systemctl show "$SERVICE_NAME.service" \
  -p FragmentPath \
  -p DropInPaths \
  -p User \
  -p Group \
  -p ExecStart \
  -p Restart
)
```

不要直接修改 `/usr/lib/systemd/system` 或 `/lib/systemd/system` 中由软件包管理的文件。管理员覆盖通常放在：

```text
/etc/systemd/system/SERVICE_NAME.service.d/
```

这里的 `SERVICE_NAME` 是说明性路径片段，不是可直接执行的 Shell 占位符。具体服务应遵循其官方安装和部署说明。

## 8. 常见排查顺序

| 现象 | 优先检查 |
| --- | --- |
| `Unit ... not found` | 服务名、软件包是否安装、`systemctl list-unit-files` |
| 状态为 `failed` | `systemctl status` 与 `journalctl -u` |
| 配置改了没生效 | 服务自身校验、`daemon-reload`、reload/restart |
| 开机后未运行 | `is-enabled`、依赖、启动日志 |
| 服务运行但端口不通 | 监听地址、端口、路由、防火墙 |
| 立即反复重启 | `Restart=`、退出码、配置和依赖 |
| 用户服务找不到 | 是否应使用 `systemctl --user`、用户会话是否存在 |

## 完成标准

- [ ] 能区分 active 与 enabled。
- [ ] 能解释 start、stop、restart 和 reload。
- [ ] 能按服务和本次启动读取 journal。
- [ ] 知道 unit 改动后何时需要 `daemon-reload`。
- [ ] 能找到 unit 主文件、drop-in 和实际 `ExecStart`。
- [ ] 不通过无脑重启掩盖配置错误。

## 相关笔记

- [[Ubuntu Server 初始化与基础安全]]
- [[OpenSSH 连接、密钥与主机指纹]]
- [[EventHub 六阶段工程化实践路线图]]

## 官方参考资料

以下资料于 **2026-07-17** 核对：

- [systemd：systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [systemd：journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [systemd：systemd.unit](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html)

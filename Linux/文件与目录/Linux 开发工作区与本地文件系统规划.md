---
title: Linux 开发工作区与本地文件系统规划
aliases:
  - Linux 后端开发工作区规划
  - Linux 本地开发目录规划
  - Linux 开发目录与数据边界
tags:
  - Linux
  - Linux/文件与目录
  - Linux/开发环境
  - 后端开发
  - 文件系统
created: 2026-07-16T00:31:00
updated: 2026-08-18T23:22:57
---

本文解决三个问题：Linux 开发项目长期放在哪里、为什么不把宿主机或网络共享目录直接当作主工作区，以及怎样按生命周期分别管理源码、工具链、缓存、配置、服务数据和备份。

本篇只讨论目录选择、文件系统边界和数据管理原则。某个项目的目录名、精确工具版本、安装步骤、构建入口和质量门禁，应由对应的工具专题或项目实践笔记负责。

> [!abstract] 本篇掌握目标
> - **必须熟练**：使用 `$HOME` 和绝对路径定位用户级工作区，核对路径所在的挂载点，让当前用户在不使用 `sudo` 的情况下完成日常开发。
> - **理解会查**：解释 `findmnt` 的 `SOURCE`、`FSTYPE`、`OPTIONS` 和 `TARGET`，区分源码、生成物、工具链、缓存、配置与凭据、服务数据和备份。
> - **认识即可**：宿主机共享挂载、网络文件系统、Docker 守护进程（daemon）数据和不同语言缓存目录的实现差异；遇到具体环境时再按实际配置核对。
>
> 命令学习入口见 [[Linux 命令行学习路线与命令地图]]；路径、变量和引用见 [[Shell 路径、变量、引用与展开]]，日常文件操作见 [[Linux 文件与目录常用命令]]。

> [!info] 资料核对日期
> 本文涉及 Filesystem Hierarchy Standard（文件系统层次结构标准，FHS）、`findmnt`、GNU `install`、Go Modules（Go 模块机制）、Maven 本地仓库、Docker 存储，以及 `git clone` 能够迁移哪些内容的资料，均于 **2026-08-17** 根据官方文档核对。命令的实际输出仍应以目标 Linux 主机安装的版本为准。

## 1. 先按管理者和生命周期分类

规划工作区时，不要先画一棵看似整齐的目录树。先对每类内容回答四个问题：

1. **谁负责写入和变更**：当前用户、包管理器、构建工具，还是长期运行的服务。
2. **能否重新生成**：可以重新下载或构建，还是丢失后无法还原。
3. **依赖哪些文件系统语义**：是否需要稳定的权限位、锁、链接、Unix 域套接字（Unix socket）、文件监听和大量小文件性能。
4. **由什么机制保护**：Git、受控配置恢复、应用级备份，还是独立备份副本。

常见内容可以分成以下几类：

| 类型 | 常见位置或发现方式 | 主要管理者 | 丢失后的处理 |
| --- | --- | --- | --- |
| 项目源码与工作树 | `$HOME/src/$PROJECT_NAME` | 当前用户与 Git | 已提交并推送的内容可从远端恢复；本地修改需另行保护 |
| 构建产物与测试输出 | 项目约定的 `target/`、`build/`、`bin/` 等 | 构建工具 | 通常可重新生成，但应先确认是否混有日志或人工产物 |
| 语言工具链 | 包管理器、`/usr/local`、`/usr/lib/jvm` 或版本管理器决定的位置 | 系统管理员或版本管理器 | 按已记录版本和安装方法重建 |
| 依赖与构建缓存 | `go env`、Maven `settings.xml` 等工具配置显示的位置 | 对应构建工具 | 通常可重新下载，但会消耗网络和时间 |
| 用户配置与凭据 | `$HOME/.gitconfig`、`$HOME/.ssh`、`$HOME/.docker`、Maven `settings.xml` 等 | 当前用户与对应工具 | 需要受控恢复；不得混入公开源码或普通日志 |
| 服务数据 | Docker 卷（volume）、数据库、用户上传目录等项目明确的位置 | 守护进程、数据库或应用 | 通常不可仅靠重新构建恢复，需要应用级备份 |
| 备份与导出 | 独立磁盘、备份系统或其他故障域 | 备份流程 | 只有具备保留策略并完成恢复验证，才算有效保护 |

`$HOME/src` 是本篇建议主动建立的源码根目录。其他位置应通过工具、配置和服务实际状态发现，不要为了匹配示意结构而手工创建 `$HOME/go`、`$HOME/.m2`、`$HOME/.docker` 或 `$HOME/.ssh`。

## 2. 活跃仓库默认放在 Linux 本地文件系统

本篇所说的“Linux 本地文件系统”，是指由目标 Linux 主机的本地磁盘或虚拟磁盘直接承载并管理的文件系统。Linux 会将不同来源的文件系统通过挂载点统一接入同一棵目录树，因此，看到 `$HOME` 这样的 Linux 路径，只能确定它在目录树中的位置，不能确定其所在文件系统的来源。若 `$HOME` 实际落在网络文件系统或宿主机共享挂载上，承载该路径的文件系统就不属于本文所说的“Linux 本地文件系统”。

虚拟机能访问宿主机共享目录，不代表共享目录适合长期承载活跃仓库。宿主机共享挂载或网络文件系统需要跨越额外的协议和实现边界，可能在以下方面表现不同：

- 文件所有者、用户组和权限位。
- 大小写敏感性与文件名规则。
- 符号链接、硬链接和 Unix socket。
- Git 索引、锁文件和可执行位。
- Linux 文件事件通知机制 `inotify` 等文件监听能力。
- 大量小文件、依赖缓存和编译输出性能。
- 容器绑定挂载（bind mount）与数据库写入语义。

后端项目会频繁读写 `.git/objects`、依赖缓存、编译产物、测试报告、容器构建上下文和集成开发环境（Integrated Development Environment，IDE）索引。主工作区因此默认放在 Linux 本地文件系统：

```text
宿主机终端或 IDE
    -> SSH
    -> Linux 主机
         -> 本地文件系统中的 $HOME/src
```

两类位置的职责不同：

| 位置 | 适合承载 | 不应默认承载 |
| --- | --- | --- |
| Linux 本地文件系统 | 活跃 Git 工作树、构建输出、需要稳定监听的源码树 | 唯一备份副本 |
| 宿主机或网络共享目录 | 安装包、归档、补丁、报告和受控迁移介质 | 活跃工作树、数据库数据目录、Docker 卷的替代目录 |

共享目录不是绝对不能用于开发。如果团队有意使用经过验证的远程开发文件系统，应先验证权限、链接、锁、文件监听、大小写规则、性能和备份边界，再把它记录为该环境的明确例外，而不是根据目录名称直接假定可用。

## 3. 核对并解释路径所在的挂载点

先让 Linux 显示 `$HOME` 实际位于哪个挂载点。显式指定输出列，可以避免依赖 `findmnt` 可能变化的默认输出格式。

**执行位置：Linux 开发机（任意目录，只读）**

```bash
findmnt -o SOURCE,FSTYPE,OPTIONS,TARGET --target "$HOME"
df -hT "$HOME"
stat -f -c 'filesystem_type=%T path=%n' "$HOME"
```

准备好工作区后，再核对最终路径：

**执行位置：Linux 开发机（任意目录，只读）**

```bash
if [ -d "$HOME/src" ]; then
  findmnt -o SOURCE,FSTYPE,OPTIONS,TARGET --target "$HOME/src"
  df -hT "$HOME/src"
  namei -l "$HOME/src"
else
  printf '%s\n' 'SKIP: $HOME/src 尚未建立。'
fi
```

前面几条命令和 Shell 结构各自回答不同问题：

- `findmnt -o SOURCE,FSTYPE,OPTIONS,TARGET --target TARGET_PATH` 查找包含指定路径的挂载点，并只输出挂载来源、文件系统类型、挂载选项和挂载目标这四列。需要理解磁盘、文件系统与挂载点的上下游关系时，继续阅读 [[Linux 磁盘、分区、文件系统与 LVM 基础#4. 安装后的只读检查|存储布局的只读检查]]。
- `df -hT TARGET_PATH` 查看指定路径所在文件系统的类型、总容量、已用容量和剩余容量；`-h` 将容量换算为便于阅读的单位，`-T` 显示文件系统类型。它回答的是整个文件系统还有多少空间，不是某个目录占用了多少空间；进一步比较 `df` 与 `du` 的观察范围，见 [[Linux 文件与目录常用命令#4.4 检查目录占用与文件系统容量|du 与 df 的观察边界]]。
- `stat -f -c FORMAT TARGET_PATH` 查询路径所在文件系统的信息，并按 `FORMAT` 指定的格式输出；本节中的 `%T` 表示文件系统类型，`%n` 表示被查询的路径。普通模式下的 `stat` 用于查看文件对象的类型、大小、权限、所有者和时间戳，基础用法见 [[Linux 文件与目录常用命令#4.2 识别对象并解析路径|stat 基础]]。
- `namei -l TARGET_PATH` 把路径从根目录到最终对象逐级展开，显示每一级的对象类型、所有者、用户组和权限模式，适合发现某一级目录不可进入的问题。目录权限如何作用于整条路径，见 [[Linux 用户、用户组、sudo 与文件权限#5.3 目录的 x 是整条路径能否通过的门槛|逐级检查目录权限]]。
- `[ -d TARGET_PATH ]` 判断路径解析后是否指向目录，并用退出状态报告结果；`if` 根据这个状态选择执行 `then` 或 `else` 分支。需要继续理解 `test`、`[ ... ]` 和条件分支，见 [[Shell 脚本阅读基础#6. 使用 test 表达条件|test 条件判断]]。
- `printf '%s\n' TEXT` 按格式输出文本；这里的 `%s` 接收一段字符串，`\n` 在末尾换行，用来给出跳过检查的提示。若要理解正常输出、错误提示与重定向的关系，见 [[Shell 标准流、管道、重定向与退出状态#1. 一条命令同时具有数据通道和退出状态|标准输出与标准错误]]。

其中，`findmnt` 输出的四个字段需要这样理解：

| 字段 | 回答的问题 | 不能单独证明什么 |
| --- | --- | --- |
| `SOURCE` | 当前文件系统来自哪个设备、远端或虚拟来源 | 只看名称不能确定虚拟化平台中的全部实现细节 |
| `FSTYPE` | 内核把它识别成哪类文件系统 | 不能仅凭某个类型名称证明性能和语义完全符合项目要求 |
| `OPTIONS` | 当前挂载是否可写，以及是否存在只读标记 `ro`、禁止执行标记 `noexec` 等约束 | 不能代替用户权限、访问控制列表（ACL）和安全模块检查 |
| `TARGET` | 哪个挂载点覆盖了当前路径 | 目录名本身不代表挂载类型 |

本地 Ubuntu 常见 ext4，但不能把文件系统类型或设备名写成固定答案。若 `SOURCE`、`FSTYPE` 或虚拟机配置表明路径来自宿主机共享或网络挂载，应把它视为需要额外验证的边界；若 `OPTIONS` 中出现 `ro`，则当前挂载本身不可写。

## 4. 建立并验证用户级源码根目录

个人开发机可以把 `$HOME/src` 作为统一源码根目录：

- 当前用户天然拥有主目录。
- Git、构建和测试不需要 `sudo`。
- 路径与编程语言无关。
- 源码与系统软件、部署目录和服务数据保持边界。

这是一项开发约定，不是 Linux 强制标准。多人共享机器需要重新设计用户、组、目录写权限和继承规则，不能直接把个人方案扩展成共享方案。

先检查目标；若目录已经存在，只报告现状，不自动重设权限。下面的创建分支把个人工作区设为 `0700`，即只有当前用户可以读取、写入和进入。

**执行位置：Linux 开发机（任意目录；可能新建 `$HOME/src`）**

```bash
(
WORKSPACE_ROOT="$HOME/src"

if [ -L "$WORKSPACE_ROOT" ]; then
  printf '停止：工作区根目录是符号链接，请先核对其真实目标：%s\n' \
    "$WORKSPACE_ROOT" >&2
  exit 1
fi

if [ -e "$WORKSPACE_ROOT" ]; then
  if [ ! -d "$WORKSPACE_ROOT" ]; then
    printf '停止：目标已存在但不是目录：%s\n' "$WORKSPACE_ROOT" >&2
    exit 1
  fi
  printf 'INFO: 沿用已有目录，不修改其权限：%s\n' "$WORKSPACE_ROOT"
else
  if install -d -m 0700 -- "$WORKSPACE_ROOT"; then
    printf 'PASS: 已创建个人工作区：%s\n' "$WORKSPACE_ROOT"
  else
    printf 'FAIL: 无法创建个人工作区：%s\n' "$WORKSPACE_ROOT" >&2
    exit 1
  fi
fi

stat -c 'owner=%U group=%G mode=%a path=%n' "$WORKSPACE_ROOT"

if test -r "$WORKSPACE_ROOT" &&
   test -w "$WORKSPACE_ROOT" &&
   test -x "$WORKSPACE_ROOT"; then
  printf '%s\n' 'PASS: 当前用户可以读取、写入并进入工作区。'
else
  printf '%s\n' 'FAIL: 当前用户缺少工作区所需权限。' >&2
  exit 1
fi

findmnt -o SOURCE,FSTYPE,OPTIONS,TARGET --target "$WORKSPACE_ROOT"
)
```

`install -d -m 0700` 在创建目标目录的同时显式设置权限模式；前面的存在性检查确保它不会顺带改写已有目录的权限。`test -r`、`test -w` 和 `test -x` 从当前执行身份的视角检查可读、可写和可进入，详见 [[Shell 脚本阅读基础#6. 使用 test 表达条件|test 条件判断]]。

`0700` 是个人工作区的保守默认值；只有确实需要同组读取时才考虑 `0750`。需要多人共同写入时，还要设计组成员、组写权限、setgid（组继承）或 ACL，具体判断见 [[Linux 用户、用户组、sudo 与文件权限]]。

不要把个人活跃仓库默认放到：

| 路径 | 不适合作为个人工作区的原因 |
| --- | --- |
| `/root` | 只属于 root，日常开发会被迫提权 |
| `/usr/local/src` | 属于系统级本地层次，通常由管理员维护 |
| `/opt` | 常用于安装可选应用软件 |
| `/srv` | 常用于服务对外提供的数据 |
| `/tmp` | 生命周期和清理策略不适合作为长期源码 |

## 5. 把项目放入工作区

若准备执行 `git clone`，通常只建立 `$HOME/src`，让 Git 创建最终项目目录。若要创建非 Git 工作目录，可以输入一层项目名，再拒绝覆盖已有对象：

**执行位置：Linux 开发机（已确认 `$HOME/src` 可用；会创建输入的项目目录）**

```bash
(
WORKSPACE_ROOT="$HOME/src"
printf '请输入一层项目目录名：'
IFS= read -r PROJECT_NAME

case "$PROJECT_NAME" in
  ''|*/*|.|..)
    printf '%s\n' \
      '停止：目录名不能为空、不能包含斜杠，也不能是点目录。' >&2
    exit 1
    ;;
esac

PROJECT_DIR="$WORKSPACE_ROOT/$PROJECT_NAME"

if [ ! -d "$WORKSPACE_ROOT" ] || [ -L "$WORKSPACE_ROOT" ]; then
  printf '停止：工作区根目录不存在或是符号链接：%s\n' \
    "$WORKSPACE_ROOT" >&2
  exit 1
fi

if [ -e "$PROJECT_DIR" ] || [ -L "$PROJECT_DIR" ]; then
  printf '停止：项目目录已经存在：%s\n' "$PROJECT_DIR" >&2
  exit 1
fi

if ! install -d -m 0700 -- "$PROJECT_DIR"; then
  printf 'FAIL: 无法创建项目目录：%s\n' "$PROJECT_DIR" >&2
  exit 1
fi

stat -c 'owner=%U group=%G mode=%a path=%n' "$PROJECT_DIR"
findmnt -o SOURCE,FSTYPE,OPTIONS,TARGET --target "$PROJECT_DIR"
)
```

已有 Git 仓库应根据源仓库的远端、分支、提交和工作区状态，选择全新克隆（fresh clone）、`git bundle` 归档、补丁文件（patch）或受控复制，完整流程见 [[Git 仓库跨机器迁移与工作区保留]]。

无论采用哪种迁入方式，都要对最终项目目录重新运行 `findmnt --target`；上面的非 Git 目录示例已经在变量仍然有效时完成了这项检查。检查目标是确认项目目录没有通过符号链接或子挂载离开预期的本地文件系统。共享目录只承担交换作用，源副本应保留到目标工作区完成 Git 状态核对、项目验证和备份确认。

## 6. 发现工具链、缓存、配置和服务数据

工具链、缓存和服务数据的位置应以工具的实际配置为准，不根据示意目录猜测。下面用 `PASS` 表示成功读取，`SKIP` 表示当前环境不适用，`FAIL` 表示命令存在但检查失败；`PATH` 是 Shell 查找命令时使用的搜索路径。

**执行位置：Linux 开发机（任意目录；只读取工具状态）**

```bash
if command -v go >/dev/null 2>&1; then
  if go env GOPATH GOMODCACHE GOCACHE; then
    printf '%s\n' 'Go: PASS'
  else
    printf '%s\n' 'Go: FAIL，无法读取 Go 环境。' >&2
  fi
else
  printf '%s\n' 'Go: SKIP，当前 PATH 中没有 go。'
fi

if command -v mvn >/dev/null 2>&1; then
  if mvn -version; then
    if [ -f "$HOME/.m2/settings.xml" ]; then
      printf '%s\n' \
        'Maven: INFO，存在用户 settings.xml；本地仓库位置可能被覆盖。'
    else
      printf '%s\n' \
        'Maven: INFO，未发现用户 settings.xml；仍需检查全局 settings.xml 和项目启动参数。'
    fi
  else
    printf '%s\n' 'Maven: FAIL，命令存在但无法正常读取版本。' >&2
  fi
else
  printf '%s\n' 'Maven: SKIP，当前 PATH 中没有 mvn。'
fi

if command -v docker >/dev/null 2>&1; then
  if DOCKER_ROOT="$(docker info --format '{{.DockerRootDir}}')"; then
    printf 'Docker: PASS，daemon_data=%s\n' "$DOCKER_ROOT"
  else
    printf '%s\n' \
      'Docker: FAIL，CLI 存在，但无法读取 daemon 状态或当前身份无权访问。' >&2
  fi
else
  printf '%s\n' 'Docker: SKIP，当前 PATH 中没有 docker。'
fi
```

这些结果需要按边界解释：

- `go env` 输出的是当前 Go 环境实际采用的工作区与缓存位置；现代 Go Modules 项目不要求位于 `$HOME/go/src`。
- `mvn -version` 只确认 Maven、Java 和 Maven 安装目录（Maven home），不会显示本地仓库。Maven 默认使用 `$HOME/.m2/repository`，但用户或全局 `settings.xml`、`-Dmaven.repo.local` 和项目入口都可能覆盖它。
- `$HOME/.docker` 是 Docker 命令行客户端（Command-Line Interface，CLI）的用户配置位置，可能包含上下文（context）和认证信息；Docker 守护进程负责管理镜像、容器和卷数据。
- Docker 卷、绑定挂载和容器可写层不是同一种存储。不要把共享目录中的普通文件夹当成 Docker 管理的卷，也不要手工移动守护进程数据目录中的单个子目录。
- `$HOME/.ssh`、Maven settings.xml 和 Docker 用户配置可能包含敏感信息。检查时只确认路径、权限和所需字段，不把完整内容复制到日志或公开笔记。

## 7. 维护容量并建立保护边界

先识别空间增长发生在哪一层，再决定是否清理。

**执行位置：Linux 开发机（任意目录，只读）**

```bash
df -hT "$HOME"

for CHECK_PATH in "$HOME/src" "$HOME/.m2" "$HOME/go" "$HOME/.cache"; do
  if [ -e "$CHECK_PATH" ]; then
    if ! du -sh -- "$CHECK_PATH"; then
      printf 'FAIL: 无法统计目录：%s\n' "$CHECK_PATH" >&2
    fi
  else
    printf 'SKIP: 路径不存在：%s\n' "$CHECK_PATH"
  fi
done

if command -v docker >/dev/null 2>&1; then
  if ! docker system df; then
    printf '%s\n' 'FAIL: 无法读取 Docker 空间占用。' >&2
  fi
else
  printf '%s\n' 'Docker: SKIP，当前 PATH 中没有 docker。'
fi
```

根据增长来源选择处理方式：

- 源码目录异常大：检查构建产物、日志、数据库、用户上传文件和未忽略文件。
- Maven 或 Go 缓存较大：评估重新下载成本后，使用工具支持的清理方法。
- Docker 占用较大：先区分镜像、已停止容器、构建缓存、卷和绑定挂载。
- 虚拟机宿主磁盘紧张：同时检查客户机文件系统占用和虚拟磁盘的实际增长。

> [!warning] 不要为了腾空间盲目执行全局清理
> Docker 卷、数据库目录、被忽略文件和本地配置可能包含不可重建数据。清理前必须确认对象、影响范围、备份和恢复方法。

不同内容需要不同保护层：

| 内容 | 合适的保护方式 | 不能误认为 |
| --- | --- | --- |
| 已提交并推送的源码历史 | Git 远端、Git 镜像副本或其他 Git 级副本 | 已自动保护未提交、未跟踪和被忽略文件 |
| 本地工作现场与敏感配置 | 按敏感度建立受控副本或可重建清单 | 可以直接提交到公开仓库 |
| 依赖与构建缓存 | 通常记录来源和重建方法，不必常规备份 | 清理没有网络、时间和可用性成本 |
| 数据库、上传文件和卷 | 应用或数据库支持的备份，并验证恢复 | 复制源码或重建镜像即可恢复 |
| 虚拟机快照 | 短期回退和变更前保护 | 独立故障域中的长期备份 |
| 独立备份 | 保留策略、完整性检查和恢复演练 | 只要文件存在就一定可恢复 |

Git 仓库迁移与工作现场保护见 [[Git 仓库跨机器迁移与工作区保留]]；虚拟机快照和独立备份的区别见 [[UTM 虚拟机快照、备份与恢复]]。

## 8. 常见问题与恢复顺序

| 现象 | 优先检查 | 安全处理 |
| --- | --- | --- |
| 文件修改监听不稳定 | 项目是否位于宿主机或网络共享挂载 | 复制到新的 Linux 本地暂存目录，验证后再切换 |
| Git 报所有权或安全目录异常 | `stat`、`namei`、挂载来源和迁入方式 | 修复明确目标的所有者，不放宽全局权限 |
| 日常构建必须使用 `sudo` 才成功 | 工作区、缓存或构建产物是否由 root 创建 | 先停止导致所有者错误的操作，再做最小范围修复 |
| `findmnt` 显示只读或出现禁止执行标记 `noexec` | `OPTIONS` 和真正需要执行的路径 | 先确认挂载设计，不用 `chmod 777` 掩盖挂载问题 |
| 磁盘持续增长 | 源码、缓存、镜像、卷、数据库和日志 | 分层审计，不删除未知数据 |
| 想撤销目录规划 | 是否有未推送、未提交或未备份内容 | 先建立可恢复副本，再复制、验证和切换 |

权限问题按以下顺序收集证据：

**执行位置：Linux 开发机（工作区已存在；只读）**

```bash
namei -l "$HOME/src"
stat -c '%U:%G %a %n' "$HOME" "$HOME/src"
find "$HOME/src" -xdev -not -uid "$(id -u)" \
  -printf '%u:%g %m %p\n'
findmnt -o SOURCE,FSTYPE,OPTIONS,TARGET --target "$HOME/src"
```

先判断异常对象来自共享挂载、容器写入、迁移过程还是曾经误用 `sudo`，再按 [[Linux 用户、用户组、sudo 与文件权限]] 修复最小范围。移动重要工作区时采用“复制到新目录 → 核对 Git 与数据状态 → 运行项目验证 → 确认备份 → 切换”的顺序，不直接覆盖原目录，也不在首次复制时使用无边界删除选项。

## 完成标准

- [ ] 能按管理者、可重建性、文件系统语义和保护方式对开发数据分类。
- [ ] 活跃仓库位于 Linux 本地文件系统，或已经记录并验证共享文件系统例外。
- [ ] 能解释 `findmnt` 的 `SOURCE`、`FSTYPE`、`OPTIONS` 和 `TARGET`。
- [ ] 当前用户可以读取、写入并进入工作区，日常 Git 和构建不依赖 `sudo`。
- [ ] 知道 `$HOME/src` 是工作区约定，工具链、缓存和服务数据位置应从实际配置发现。
- [ ] 共享目录只作为受控交换区，不承担唯一副本或默认服务数据目录。
- [ ] 能区分 Git 远端、缓存重建、应用数据备份、虚拟机快照和独立备份。
- [ ] 清理或迁移前会先确认范围、不可重建内容和恢复方法。

## 相关笔记

- [[Linux 命令行学习路线与命令地图]]
- [[Linux 文件与目录常用命令]]
- [[Linux 用户、用户组、sudo 与文件权限]]
- [[Linux 磁盘、分区、文件系统与 LVM 基础]]
- [[Git 仓库跨机器迁移与工作区保留]]
- [[Git 安装与初始配置概览]]
- [[Ubuntu 安装 Go]]
- [[Java 与 Maven 环境搭建概览]]
- [[Docker 安装概览]]
- [[UTM 虚拟机快照、备份与恢复]]

## 官方参考资料

以下资料于 **2026-08-17** 核对：

- [Filesystem Hierarchy Standard 3.0](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)
- [util-linux：findmnt 手册源文件](https://github.com/util-linux/util-linux/blob/master/misc-utils/findmnt.8.adoc)
- [GNU Coreutils：install](https://www.gnu.org/software/coreutils/install)
- [Go：Managing module source](https://go.dev/doc/modules/managing-source)
- [Maven：Settings Reference](https://maven.apache.org/settings.html)
- [Maven：Local Repositories](https://maven.apache.org/repositories/local.html)
- [Docker：Storage](https://docs.docker.com/engine/storage/)
- [Git：git-clone](https://git-scm.com/docs/git-clone)

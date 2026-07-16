---
title: Ubuntu 安装 Go
aliases:
  - Ubuntu 安装 Go 工具链
  - Linux ARM64 安装 Go
  - Ubuntu Go 开发环境配置
tags:
  - Go
  - Go/安装
  - Go/Linux
  - Go/Ubuntu
  - Linux/开发环境
created: 2026-07-16T00:31:00
updated: 2026-07-16T01:17:59
---

本文用于在 Ubuntu 开发机上从零安装 Go 官方工具链，重点覆盖项目版本约束、CPU 架构、官方压缩包校验、安全替换、PATH、验证、升级与回滚。本文的实操案例是 Ubuntu Server ARM64；其他 Linux 架构必须重新选择对应安装包，不能照抄文件名。

如果正在搭建完整后端开发机，先阅读 [[Linux 后端开发目录与工具链规划]]。Git、JDK、Docker 与项目首次构建分别由对应专题负责；本文只解决“当前 Ubuntu 用户能可靠地调用项目要求的 Go 工具链”。

> [!important] 项目约束优先
> 不要先安装“当时最新的 Go”，再修改项目去迁就本机。应先读取项目的 `go.mod`、README、CI 和 Dockerfile。本次 EventHub Go 仓库在 2026-07-16 的只读核对结果是：`go.mod` 声明 `go 1.24.0`，README 要求 Go 1.24+，Docker 构建阶段使用 Go 1.24.0，CI 从 `go.mod` 读取版本。因此本文以 Go 1.24.0 为可复现实操版本；仓库声明变化后，应跟随项目重新核对。

## 目标与完成标准

完成后，应能区分“下载正确的二进制包”“Shell 能找到 `go`”“工具链版本满足项目”“项目可以编译”这四层验证。

| 检查项 | 预期结果 | 验证命令 |
| --- | --- | --- |
| 操作系统与架构 | Linux ARM64 虚拟机通常显示 `aarch64` / `arm64` | `uname -m`、`dpkg --print-architecture` |
| 安装来源 | 压缩包来自 `go.dev`，SHA-256 校验通过 | `sha256sum --check` |
| 命令路径 | 首选 `/usr/local/go/bin/go` | `type -a go`、`command -v go` |
| 工具链版本 | 本案例显示 `go1.24.0 linux/arm64` | `go version` |
| Go 环境 | `GOOS=linux`、`GOARCH=arm64`，路径属于当前用户或预期系统目录 | `go env` |
| 最小程序 | 能创建 module、运行和测试 | `go run .`、`go test ./...` |
| 项目约束 | `go.mod`、本机工具链、CI/Docker 基线已经核对 | `awk`、`go version` |

## 1. 先理解版本、系统与架构

Go 下载文件名同时编码版本、操作系统与 CPU 架构。例如：

```text
go1.24.0.linux-arm64.tar.gz
│        │     │
版本     系统  Go 架构名
```

Apple Silicon Mac 上通过 UTM **虚拟化**运行的 Ubuntu ARM64 客户机，通常应使用 Linux ARM64 包，而不是 macOS ARM64 包，也不是 Linux AMD64 包。宿主机是 macOS，不会改变客户机内工具链所面向的操作系统。

| Linux 命令常见输出 | Go 下载架构名 | 典型 CPU |
| --- | --- | --- |
| `uname -m` 为 `aarch64`，`dpkg` 为 `arm64` | `arm64` | Apple Silicon 虚拟机、ARM 服务器 |
| `uname -m` 为 `x86_64`，`dpkg` 为 `amd64` | `amd64` | Intel / AMD 64 位服务器或虚拟机 |

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
. /etc/os-release
printf 'system=%s\n' "$PRETTY_NAME"
printf 'kernel_arch=%s\n' "$(uname -m)"
printf 'debian_arch=%s\n' "$(dpkg --print-architecture)"
getconf LONG_BIT
```

本案例预期包含 `Ubuntu 24.04`、`aarch64`、`arm64` 和 `64`。如果输出是 `x86_64` / `amd64`，应改用 Linux AMD64 包；如果输出不是 Linux，不要继续执行本文的 `/usr/local` 安装步骤。

## 2. 从项目声明确定目标版本

不要只相信聊天记录或旧笔记。获得仓库后，直接读取可追踪的项目文件。

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
pwd
test -f go.mod
awk '$1 == "go" { print "go directive=" $2; exit }' go.mod
awk '$1 == "toolchain" { print "toolchain directive=" $2; exit }' go.mod
grep -nE 'Go [0-9]+|go-version-file|^FROM golang:' \
  README.md .github/workflows/*.yml Dockerfile 2>/dev/null || true
```

本次 EventHub Go 案例中，`go.mod` 输出 `1.24.0`，且没有额外的 `toolchain` 指令。以后如果这些文件发生变化，以新提交中的声明为准。不要为了消除本机报错而擅自编辑 `go.mod`；升级项目版本属于独立代码变更，需要 CI 和团队共同验证。

## 3. 检查已有 Go，避免覆盖来源不明的安装

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
command -v go || true
type -a go 2>/dev/null || true
go version 2>/dev/null || true
go env GOROOT GOPATH GOTOOLCHAIN 2>/dev/null || true
ls -ld /usr/local/go 2>/dev/null || true
```

如果已有 Go，记录命令路径、版本与 `GOROOT`。APT、Snap、手工压缩包、版本管理器可能同时提供多个 `go`；PATH 中靠前的命令才会实际执行。不要直接删除 `/usr/bin/go`、Snap 目录或未知版本管理器目录。

本文主线使用 Go 官方 tarball 安装到 `/usr/local/go`。它适合需要与项目精确对齐的个人开发虚拟机。若组织已统一使用 APT 仓库、镜像、asdf 或其他版本管理器，应遵守组织基线，不要混装。

## 4. 下载官方 ARM64 压缩包并验证 SHA-256

下列文件名与 SHA-256 已在 **2026-07-16** 通过 Go 官方下载 JSON 元数据核对：

- 文件：`go1.24.0.linux-arm64.tar.gz`
- SHA-256：`c3fa6d16ffa261091a5617145553c71d21435ce547e44cc6dfb7470865527cc7`

版本或架构变化后，这个校验值也必须变化。应从 [Go 官方下载页](https://go.dev/dl/) 或 [Go 官方完整下载元数据](https://go.dev/dl/?mode=json&include=all) 重新取得，不能沿用旧值，也不能从第三方教程复制。

### 4.1 安装下载与校验工具

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

`ca-certificates` 用于验证 HTTPS 服务器证书，`curl` 用于下载；`sha256sum` 通常由 Ubuntu 基础系统的 `coreutils` 提供。

### 4.2 下载并校验

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
GO_VERSION='1.24.0'
GO_OS='linux'
GO_ARCH='arm64'
GO_ARCHIVE="go${GO_VERSION}.${GO_OS}-${GO_ARCH}.tar.gz"
GO_SHA256='c3fa6d16ffa261091a5617145553c71d21435ce547e44cc6dfb7470865527cc7'
GO_DOWNLOAD_DIR="$HOME/Downloads/go"

mkdir -p "$GO_DOWNLOAD_DIR"
cd "$GO_DOWNLOAD_DIR"
curl --fail --location --remote-name "https://go.dev/dl/${GO_ARCHIVE}"
printf '%s  %s\n' "$GO_SHA256" "$GO_ARCHIVE" | sha256sum --check
```

预期看到：

```text
go1.24.0.linux-arm64.tar.gz: OK
```

> [!warning] 校验失败时立即停止
> `FAILED` 表示文件不完整、文件名与校验值不匹配，或下载内容不是预期归档。不要使用 `--ignore-missing`、不要跳过校验、不要继续 `sudo tar`。删除本次下载的单个归档，重新核对官方元数据后再下载：

**执行位置：Ubuntu 虚拟机（`$HOME/Downloads/go`）**

```bash
rm -- "$GO_ARCHIVE"
```

该删除命令只针对当前 Shell 中已明确设置的归档文件名，不要把它改成通配符。

可以在安装前只读查看归档前几项，确认顶层目录是 `go/`：

**执行位置：Ubuntu 虚拟机（`$HOME/Downloads/go`）**

```bash
tar -tzf "$GO_ARCHIVE" | sed -n '1,20p'
```

## 5. 使用“暂存验证—备份旧版—切换”安装

Go 官方明确警告：不要把新归档直接解压进已有 `/usr/local/go`，否则新旧文件混合会形成损坏安装。本文在官方原则上增加暂存与备份步骤：先解压到独立目录并运行其中的 `go`，成功后才切换。

> [!important] 保持当前终端不关闭
> 下列步骤把备份路径保存在 `GO_BACKUP` 变量中。完成安装、PATH 和最小程序验证前不要关闭当前终端，并记录命令打印的备份目录。

### 5.1 解压到独立暂存目录并验证

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
GO_VERSION='1.24.0'
GO_ARCHIVE="go${GO_VERSION}.linux-arm64.tar.gz"
GO_ARCHIVE_PATH="$HOME/Downloads/go/$GO_ARCHIVE"
GO_STAGE="/usr/local/go-stage-${GO_VERSION}-$(date +%Y%m%d%H%M%S)"

if ! test -f "$GO_ARCHIVE_PATH"; then
  printf '停止：归档不存在：%s\n' "$GO_ARCHIVE_PATH" >&2
elif sudo test -e "$GO_STAGE"; then
  printf '停止：暂存目录已经存在：%s\n' "$GO_STAGE" >&2
else
  sudo install -d -m 0755 "$GO_STAGE"
  if sudo tar --strip-components=1 -C "$GO_STAGE" -xzf "$GO_ARCHIVE_PATH" \
    && sudo "$GO_STAGE/bin/go" version; then
    printf '暂存工具链验证通过：%s\n' "$GO_STAGE"
  else
    printf '停止：解压或暂存工具链验证失败，请保留输出并移除本次暂存目录后重试。\n' >&2
  fi
fi
```

最后一条应输出 `go version go1.24.0 linux/arm64`。若不是，停止切换；检查变量、架构和归档来源。暂存目录的路径带时间戳，不会与正常 `/usr/local/go` 混合。

### 5.2 备份旧目录并切换

**执行位置：Ubuntu 虚拟机（与上一步相同的 Shell）**

```bash
GO_BACKUP=''

if ! sudo test -x "$GO_STAGE/bin/go"; then
  printf '停止：暂存工具链尚未验证：%s\n' "$GO_STAGE" >&2
else
  if sudo test -e /usr/local/go || sudo test -L /usr/local/go; then
    GO_BACKUP="/usr/local/go.backup.$(date +%Y%m%d%H%M%S)"
    sudo mv /usr/local/go "$GO_BACKUP"
  fi

  if ! sudo mv "$GO_STAGE" /usr/local/go; then
    if [ -n "$GO_BACKUP" ]; then
      if ! sudo mv "$GO_BACKUP" /usr/local/go; then
        printf '紧急：新旧目录切换均失败，请保持当前终端并检查 %s。\n' "$GO_BACKUP" >&2
      fi
    fi
    printf '%s\n' 'Go 切换失败，旧安装已尝试恢复。' >&2
  else
    /usr/local/go/bin/go version
    printf 'previous_go_backup=%s\n' "${GO_BACKUP:-none}"
  fi
fi
```

这里使用 `mv` 保留旧目录，而不是立即 `rm -rf /usr/local/go`。新版本完成实际项目验证前不要删除备份。首次安装没有旧目录时会打印 `none`。

## 6. 为当前用户配置 PATH

Go 官方安装目录的可执行文件位于 `/usr/local/go/bin`。`go install` 安装的个人工具默认进入 `$HOME/go/bin`。两者都加入当前用户的登录环境即可，不需要设置全局 `/etc/profile`，也不要手工设置 `GOROOT`。

Ubuntu Server 默认 Bash 登录环境可以使用 `$HOME/.profile`：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
profile_file="$HOME/.profile"
go_path_line='export PATH="/usr/local/go/bin:$HOME/go/bin:$PATH"'

touch "$profile_file"
grep -qxF "$go_path_line" "$profile_file" || printf '\n%s\n' "$go_path_line" >> "$profile_file"
. "$profile_file"
hash -r
```

新开一个 SSH 会话后再次验证，确认配置不是只在当前终端临时生效。

**执行位置：Ubuntu 虚拟机（新 SSH 会话，任意目录）**

```bash
command -v go
type -a go
go version
```

预期首选路径是 `/usr/local/go/bin/go`。如果同时看到 `/usr/bin/go`，不一定是故障；关键是 `/usr/local/go/bin` 在 PATH 中更靠前且版本正确。

> [!note] 为什么不设置 `GOROOT`
> 官方二进制能确定自身安装根目录。手工把 `GOROOT` 固定为旧版本路径，反而容易让升级、IDE 和工具调用错位。只配置 PATH，并用 `go env GOROOT` 查看实际结果。

## 7. 验证 Go 环境

### 7.1 检查版本、架构与目录

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
go version
go env GOVERSION GOOS GOARCH GOROOT GOPATH GOBIN GOCACHE GOMODCACHE GOTOOLCHAIN
go env GOENV
```

本案例的关键预期是：

- `GOVERSION` 为 `go1.24.0`。
- `GOOS` 为 `linux`。
- `GOARCH` 为 `arm64`。
- `GOROOT` 指向 `/usr/local/go`。
- `GOPATH` 通常为 `$HOME/go`。
- `GOBIN` 为空时，`go install` 默认把工具放入 `$GOPATH/bin`，通常即 `$HOME/go/bin`。

`$HOME/go` 主要承载 module 缓存和个人工具，不要求把现代 Go Modules 项目放入 `$HOME/go/src`。本阶段统一将源码放在 `$HOME/src`，详见 [[Linux 后端开发目录与工具链规划]]。

### 7.2 运行最小 module

该验证只在 `mktemp` 创建的临时目录中写文件，不污染项目仓库。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
work_dir="$(mktemp -d)"
cd "$work_dir"

go mod init example.com/go-env-check
printf '%s\n' \
  'package main' \
  '' \
  'import "fmt"' \
  '' \
  'func main() {' \
  '    fmt.Println("Go environment works")' \
  '}' \
  > main.go

go fmt ./...
go run .
go test ./...
```

预期看到 `Go environment works`，测试命令正常退出。完成后只删除刚创建的临时目录：

**执行位置：Ubuntu 虚拟机（与上一步相同的 Shell）**

```bash
cd "$HOME"
rm -rf -- "$work_dir"
```

### 7.3 回到真实项目核对

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
cd "$HOME/src/eventhub-go"
printf 'project_go=%s\n' "$(awk '$1 == "go" { print $2; exit }' go.mod)"
go version
go env GOOS GOARCH GOMOD
go mod download
go build ./...
git status --short
```

`go build ./...` 只验证所有 package 能编译，不会在项目根目录留下应用二进制；依赖和构建缓存会写入 Go 的用户缓存目录。完整测试、lint、生成物和 Docker 门禁见 [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]]。

## 8. 安全升级、降级与回滚

### 项目版本变化时怎么做

1. 先读取新提交中的 `go.mod`、`toolchain` 指令、README、CI 和 Dockerfile。
2. 到 Go 官方下载页选择与 Linux/CPU 架构匹配的归档。
3. 重新取得该文件的官方 SHA-256，不复用本文的 1.24.0 校验值。
4. 重复“暂存验证—备份旧版—切换”，不要覆盖现有目录。
5. 新开 SSH 会话，执行版本、最小 module 和项目质量门禁。
6. 只有新版本稳定后，才按明确路径删除不再需要的备份。

Go 1.21 及以后支持工具链选择。`go env GOTOOLCHAIN` 可查看当前策略；但自动下载工具链不是忽略团队基线的理由。开发机、CI 和 Docker 构建尽量使用仓库声明的同一版本，结果更容易复现。完整语义见 [Go 官方工具链文档](https://go.dev/doc/toolchain)。

### 新版本失败时恢复最近备份

下面操作会把当前 `/usr/local/go` 移走，再恢复最近的备份；执行前先用 `printf` 确认两个路径。它不会删除失败版本，仍可用于复盘。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
backup_dir="$(find /usr/local -maxdepth 1 -mindepth 1 -type d -name 'go.backup.*' -print | sort | tail -n 1)"

if [ -z "$backup_dir" ]; then
  printf '%s\n' '没有找到 /usr/local/go.backup.*，停止回滚。' >&2
elif ! sudo test -e /usr/local/go; then
  printf '%s\n' '当前 /usr/local/go 不存在；停止自动回滚，请先检查目录状态。' >&2
else
  failed_dir="/usr/local/go.failed.$(date +%Y%m%d%H%M%S)"
  printf 'current_to=%s\nbackup_from=%s\n' "$failed_dir" "$backup_dir"

  if ! sudo mv /usr/local/go "$failed_dir"; then
    printf '%s\n' '停止：无法移动当前 Go 目录，尚未修改备份。' >&2
  elif sudo mv "$backup_dir" /usr/local/go; then
    hash -r
    /usr/local/go/bin/go version
    go version
  else
    printf '%s\n' '恢复备份失败，正在把刚移走的当前版本放回。' >&2
    if ! sudo mv "$failed_dir" /usr/local/go; then
      printf '紧急：自动放回也失败，请保持终端并检查：%s\n' "$failed_dir" >&2
    fi
  fi
fi
```

如果旧版是一个符号链接，`find -type d` 可能不会选中它；此时不要猜测，先运行 `sudo ls -la /usr/local`，依据安装时记录的 `previous_go_backup` 手工确认路径。

## 9. 权限、安全与卸载边界

- `/usr/local/go` 由 root 管理；普通用户只需读取和执行。
- `$HOME/go`、Go 缓存、项目源码应属于当前用户。
- 不要用 `sudo go build`、`sudo go test` 或 `sudo go install`，否则会产生 root 所有的缓存和项目文件。
- 不要执行来源不明的 `curl ... | sh` 安装脚本。
- 不要为了依赖下载成功而关闭 TLS 校验、设置全局 `GOINSECURE=*`，或写入未经组织批准的代理、模块镜像。

检查用户目录所有权：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
for path_name in "$HOME/go" "$HOME/.cache/go-build" "$HOME/src"; do
  if [ -e "$path_name" ]; then
    stat -c '%U:%G %a %n' "$path_name"
  fi
done
```

若确认某个目录完全属于当前个人开发账号，却因误用 `sudo go ...` 变成 root 所有，可只修复明确的用户缓存目录；不要对 `/usr/local` 或多人共享目录递归 `chown`。

卸载官方 tarball 时，Go 官方步骤是移除 `/usr/local/go` 并删除 PATH 条目。为了可恢复，个人开发机可先改名停用，而不是立即删除：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
disabled_dir="/usr/local/go.disabled.$(date +%Y%m%d%H%M%S)"
sudo mv /usr/local/go "$disabled_dir"
printf 'disabled_go=%s\n' "$disabled_dir"
```

然后从 `$HOME/.profile` 删除本文添加的精确 PATH 行，新开 SSH 会话并确认 `command -v go`。确认不再需要后，再单独评估是否删除停用目录、`go env GOCACHE`、`go env GOMODCACHE` 与 `$HOME/go/bin`；这些目录用途不同，不应一次性清空。

## 10. 常见问题排查

### `go: command not found`

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
test -x /usr/local/go/bin/go
grep -nF '/usr/local/go/bin' "$HOME/.profile" || true
printf '%s\n' "$PATH" | tr ':' '\n'
. "$HOME/.profile"
hash -r
command -v go
```

如果当前会话修好但新 SSH 会话仍失败，检查登录 Shell 实际读取哪个启动文件；不要把同一 PATH 行反复追加到 `.profile`、`.bashrc`、`.zshrc` 和系统文件。

### 安装后仍显示旧版本

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
type -a go
command -v go
readlink -f "$(command -v go)"
go env GOROOT
```

常见原因是 APT、Snap 或版本管理器路径排在前面，或 Shell 缓存了旧命令位置。先确认路径顺序再调整，不要直接删除系统包文件。

### 出现 `Exec format error`

这通常表示下载了错误架构的二进制。重新比较：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
uname -m
dpkg --print-architecture
file /usr/local/go/bin/go
```

ARM64 Ubuntu 应使用 `linux-arm64`，AMD64 Ubuntu 应使用 `linux-amd64`。

### module 下载失败

先确认基础网络、DNS、系统时间和 Go 配置：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
getent hosts proxy.golang.org
date --iso-8601=seconds
go env GOPROXY GOSUMDB GOPRIVATE GONOSUMDB GOINSECURE
curl -I https://proxy.golang.org/
```

中国大陆或企业网络下，应使用组织批准的代理、私有模块仓库和证书方案。不要把来源不明、可能失效的第三方地址写进全局环境。私有模块的 `GOPRIVATE` 与 Git 凭据问题，应结合 [[Git 凭据、SSH 与常见问题排查]] 排查。

## 相关笔记

- [[Linux 后端开发虚拟机搭建概览]]
- [[Linux 后端开发目录与工具链规划]]
- [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]]
- [[Ubuntu 从零安装 Git]]
- [[Ubuntu 安装 Docker]]

## 官方参考资料

以下资料于 **2026-07-16** 核对；版本与页面内容会变化，执行新版本安装前应重新访问。

- [Go：Download and install](https://go.dev/doc/install)
- [Go：官方下载页与 SHA-256](https://go.dev/dl/)
- [Go：完整下载 JSON 元数据](https://go.dev/dl/?mode=json&include=all)
- [Go：Managing Go installations](https://go.dev/doc/manage-install)
- [Go：Toolchains](https://go.dev/doc/toolchain)
- [Go：Command documentation and environment variables](https://go.dev/cmd/go/)
- [Go：Release History](https://go.dev/doc/devel/release)

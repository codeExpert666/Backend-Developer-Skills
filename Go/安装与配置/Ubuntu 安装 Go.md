---
title: Ubuntu 安装 Go
aliases:
  - Ubuntu 安装 Go 工具链
  - Linux ARM64 安装 Go
  - Ubuntu Go 开发环境配置
tags:
  - Go
  - Go/安装
  - Go/工具链
  - Go/Linux
  - Go/Ubuntu
created: 2026-07-16T00:31:00
updated: 2026-07-17T01:12:07
---

本文用于在 Ubuntu 上安装 Go 官方工具链，覆盖项目版本约束、CPU 架构、官方下载与 SHA-256 校验、安全切换、PATH、验证、升级、回滚和排障。

主线使用 Go 官方 tarball 安装到 `/usr/local`。它适合需要和项目精确版本对齐的开发机；如果组织已经统一使用受管理的软件仓库或版本管理器，应遵守组织基线，不要混装多个来源。

> [!important] 项目约束优先
> 先读取项目的 `go.mod`、可选 `toolchain` 指令、README、CI 和 Dockerfile，再选择版本。不要先安装“当时最新版本”，也不要为了迁就本机而擅自降低项目声明。

EventHub 第 1 阶段的当前版本选择属于工程基线，见 [[EventHub 第 1 阶段环境与版本基线]]；本文只讲可迁移到其他项目和架构的安装方法。

## 1. 完成标准

| 检查项 | 应达到的状态 |
| --- | --- |
| 系统与架构 | 确认当前是 Linux，并映射到 Go 的 `arm64` 或 `amd64` 名称 |
| 版本来源 | 从项目声明得出目标版本 |
| 下载来源 | 归档和 SHA-256 来自 Go 官方 |
| 安装切换 | 新版本先在暂存目录验证，旧版本保留可恢复备份 |
| PATH | 新登录会话能够找到 `/usr/local/go/bin/go` |
| 工具链验证 | `go version` 与 `go env` 符合选择 |
| 最小程序 | 能创建 module、运行和测试 |
| 项目验证 | 项目能下载依赖、构建，并继续执行自身质量门禁 |

## 2. 确认操作系统和 CPU 架构

Go 下载文件名同时编码版本、操作系统和架构：

```text
go1.24.0.linux-arm64.tar.gz
│        │     │
版本     系统  Go 架构名
```

示例文件名只用于解释结构，不代表所有项目都应安装该版本。

| Linux 常见输出 | Go 下载架构名 | 典型平台 |
| --- | --- | --- |
| `uname -m` 为 `aarch64`，`dpkg` 为 `arm64` | `arm64` | ARM 服务器、Apple Silicon ARM64 虚拟机 |
| `uname -m` 为 `x86_64`，`dpkg` 为 `amd64` | `amd64` | Intel / AMD 64 位主机 |

**执行位置：Ubuntu 主机（任意目录）**

```bash
. /etc/os-release
printf 'system=%s\n' "$PRETTY_NAME"
printf 'kernel_arch=%s\n' "$(uname -m)"
printf 'debian_arch=%s\n' "$(dpkg --print-architecture)"
getconf LONG_BIT
```

根据 Ubuntu 架构映射 Go 架构名：

**执行位置：Ubuntu 主机（任意目录）**

```bash
case "$(dpkg --print-architecture)" in
  arm64)
    GO_ARCH='arm64'
    ;;
  amd64)
    GO_ARCH='amd64'
    ;;
  *)
    printf '停止：本文主线尚未覆盖该架构：%s\n' \
      "$(dpkg --print-architecture)" >&2
    exit 1
    ;;
esac

printf 'go_arch=%s\n' "$GO_ARCH"
```

如果出现 `Exec format error`，通常是下载架构与客户机架构不匹配，而不是 Go 源码错误。

## 3. 从项目声明确定目标版本

**执行位置：Ubuntu 主机（Go 项目根目录）**

```bash
pwd
test -f go.mod
awk '$1 == "go" { print "go directive=" $2; exit }' go.mod
awk '$1 == "toolchain" { print "toolchain directive=" $2; exit }' go.mod

grep -nE 'go-version-file|go-version|^FROM golang:' \
  README.md .github/workflows/*.yml Dockerfile 2>/dev/null || true
```

判断顺序：

1. `go.mod` 的 `go` 与 `toolchain` 指令。
2. CI 是否从 `go.mod` 读取版本，或另有固定版本。
3. Dockerfile 的构建镜像。
4. README 的开发前置条件。

如果多处声明不一致，应先把它作为项目问题记录并澄清，而不是随意选择最容易安装的版本。

## 4. 检查现有 Go 与安装来源

**执行位置：Ubuntu 主机（任意目录）**

```bash
command -v go || true
type -a go 2>/dev/null || true
go version 2>/dev/null || true
go env GOROOT GOPATH GOTOOLCHAIN 2>/dev/null || true
ls -ld /usr/local/go 2>/dev/null || true
```

APT、Snap、官方 tarball 和版本管理器可能同时提供 `go`。PATH 中最靠前的命令才会执行。记录现有路径和版本，不直接删除 `/usr/bin/go`、Snap 目录或未知版本管理器文件。

## 5. 从官方获取归档和 SHA-256

**官方资料核对日期：2026-07-17。** Go 版本、文件和校验值会变化，执行安装时重新访问：

- [Go 官方下载页](https://go.dev/dl/)
- [Go 官方完整下载 JSON](https://go.dev/dl/?mode=json&include=all)

不要从普通博客、聊天记录或第三方镜像复制 SHA-256。版本或架构改变时，校验值也必须重新取得。

### 5.1 安装下载工具

**执行位置：Ubuntu 主机（任意目录）**

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

### 5.2 设置并校验变量

从项目声明和官方下载页获得版本与 SHA-256 后输入。

**执行位置：Ubuntu 主机（任意目录）**

```bash
printf '请输入项目要求的 Go 版本号：'
IFS= read -r GO_VERSION

case "$(dpkg --print-architecture)" in
  arm64)
    GO_ARCH='arm64'
    ;;
  amd64)
    GO_ARCH='amd64'
    ;;
  *)
    printf '%s\n' '停止：无法自动映射 Go 架构。' >&2
    exit 1
    ;;
esac

printf '请输入官方页面中对应归档的 SHA-256：'
IFS= read -r GO_SHA256

case "$GO_VERSION" in
  ''|*[!0-9.]*)
    printf '%s\n' '停止：版本号格式不符合本文主线。' >&2
    exit 1
    ;;
esac

if [ "${#GO_SHA256}" -ne 64 ]; then
  printf '%s\n' '停止：SHA-256 应为 64 个十六进制字符。' >&2
  exit 1
fi

case "$GO_SHA256" in
  *[!0-9a-fA-F]*)
    printf '%s\n' '停止：SHA-256 含非十六进制字符。' >&2
    exit 1
    ;;
esac

GO_OS='linux'
GO_ARCHIVE="go${GO_VERSION}.${GO_OS}-${GO_ARCH}.tar.gz"
printf 'archive=%s\nsha256=%s\n' "$GO_ARCHIVE" "$GO_SHA256"
```

`GO_VERSION` 不带 `go` 前缀，例如输入 `1.24.0`。真正选择应以项目当前约束为准。

### 5.3 下载并校验

**执行位置：Ubuntu 主机（与上一代码块相同的 Shell）**

```bash
GO_DOWNLOAD_DIR="$HOME/Downloads/go"
mkdir -p "$GO_DOWNLOAD_DIR"
cd "$GO_DOWNLOAD_DIR"

curl --fail --location --remote-name \
  "https://go.dev/dl/${GO_ARCHIVE}"

if ! printf '%s  %s\n' "$GO_SHA256" "$GO_ARCHIVE" \
  | sha256sum --check; then
  printf '%s\n' '停止：SHA-256 校验失败，删除本次下载的单个归档。' >&2
  rm -- "$GO_DOWNLOAD_DIR/$GO_ARCHIVE"
  exit 1
fi
```

预期输出以归档文件名开头，并以 `OK` 结束。

> [!warning] 校验失败立即停止
> 不使用 `--ignore-missing`，不跳过校验，也不继续 `sudo tar`。上面的失败分支只删除当前变量指向的单个归档，不使用通配符；重新核对版本、架构、文件名和官方 SHA-256 后再下载。

下载成功后可只读查看归档结构：

**执行位置：Ubuntu 主机（`$HOME/Downloads/go`）**

```bash
tar -tzf "$GO_ARCHIVE" | sed -n '1,20p'
```

顶层应为 `go/`。

## 6. 暂存验证后再切换

Go 官方明确警告不要把新归档直接解压进已有 `/usr/local/go`，否则新旧文件可能混合。本文采用：

1. 解压到独立暂存目录。
2. 直接运行暂存工具链。
3. 移动为带版本的安装目录。
4. 备份当前 `/usr/local/go`。
5. 创建新的 `/usr/local/go` 符号链接。

> [!important] 保持当前终端
> 以下变量需要在同一 Shell 中使用。切换、PATH 和最小程序验证完成前，不要关闭终端；记录命令打印的备份路径。

### 6.1 解压到暂存目录

**执行位置：Ubuntu 主机（与下载步骤相同的 Shell）**

```bash
GO_ARCHIVE_PATH="$GO_DOWNLOAD_DIR/$GO_ARCHIVE"
GO_STAGE="/usr/local/go-stage-${GO_VERSION}-${GO_ARCH}-$(date +%Y%m%d%H%M%S)"
GO_INSTALL_DIR="/usr/local/go-${GO_VERSION}-${GO_ARCH}"

if [ ! -f "$GO_ARCHIVE_PATH" ]; then
  printf '停止：归档不存在：%s\n' "$GO_ARCHIVE_PATH" >&2
  exit 1
fi

if sudo test -e "$GO_STAGE" || sudo test -e "$GO_INSTALL_DIR"; then
  printf '%s\n' '停止：暂存目录或目标版本目录已经存在。' >&2
  exit 1
fi

sudo install -d -m 0755 "$GO_STAGE"
sudo tar --strip-components=1 \
  -C "$GO_STAGE" \
  -xzf "$GO_ARCHIVE_PATH"

"$GO_STAGE/bin/go" version
"$GO_STAGE/bin/go" env GOOS GOARCH GOROOT
```

输出版本和架构必须与刚才选择一致。若验证失败，不切换 `/usr/local/go`，先保留现场检查。

只有在暂存验证失败、并且确认不再需要保留现场时，才使用下面的带路径保护清理。验证成功准备继续切换时，**跳过这段命令**。

**执行位置：Ubuntu 主机（与上一代码块相同的 Shell）**

```bash
case "${GO_STAGE:-}" in
  /usr/local/go-stage-*)
    sudo rm -rf -- "$GO_STAGE"
    ;;
  *)
    printf '%s\n' '停止：暂存路径不符合保护规则。' >&2
    exit 1
    ;;
esac
```

这是递归删除命令，只能用于已经确认失败且不再需要的本次暂存目录。

### 6.2 安装版本目录并切换链接

**执行位置：Ubuntu 主机（与上一代码块相同的 Shell）**

```bash
GO_BACKUP=''

if ! sudo test -x "$GO_STAGE/bin/go"; then
  printf '%s\n' '停止：暂存工具链尚未验证。' >&2
  exit 1
fi

if ! sudo mv "$GO_STAGE" "$GO_INSTALL_DIR"; then
  printf '%s\n' '停止：无法把暂存目录移动为版本目录。' >&2
  exit 1
fi

if sudo test -e /usr/local/go || sudo test -L /usr/local/go; then
  GO_BACKUP="/usr/local/go.backup.$(date +%Y%m%d%H%M%S)"
  sudo mv /usr/local/go "$GO_BACKUP"
fi

if sudo ln -s "$GO_INSTALL_DIR" /usr/local/go; then
  /usr/local/go/bin/go version
  printf 'installed=%s\nbackup=%s\n' \
    "$GO_INSTALL_DIR" "${GO_BACKUP:-none}"
else
  printf '%s\n' '创建 /usr/local/go 链接失败，开始恢复。' >&2

  if [ -n "$GO_BACKUP" ]; then
    if ! sudo mv "$GO_BACKUP" /usr/local/go; then
      printf '紧急：旧版本也未能恢复，请保持终端并检查：%s\n' \
        "$GO_BACKUP" >&2
    fi
  fi

  exit 1
fi
```

不要删除 `GO_BACKUP`，直到 PATH、新会话、最小程序和真实项目都验证通过。

## 7. 为当前用户配置 PATH

Go 工具链位于 `/usr/local/go/bin`，`go install` 的个人工具通常位于 `$HOME/go/bin`。不需要手工设置 `GOROOT`。

Ubuntu Server 默认 Bash 登录环境可使用 `$HOME/.profile`。

**执行位置：Ubuntu 主机（任意目录）**

```bash
PROFILE_FILE="$HOME/.profile"
GO_PATH_LINE='export PATH="/usr/local/go/bin:$HOME/go/bin:$PATH"'

touch "$PROFILE_FILE"
grep -qxF "$GO_PATH_LINE" "$PROFILE_FILE" \
  || printf '\n%s\n' "$GO_PATH_LINE" >>"$PROFILE_FILE"

. "$PROFILE_FILE"
hash -r
command -v go
go version
```

新开 SSH 会话后再次验证：

**执行位置：Ubuntu 主机（新 SSH 会话，任意目录）**

```bash
command -v go
type -a go
go version
go env GOROOT GOPATH
```

若同时存在 `/usr/bin/go`，关键是 PATH 首选路径与版本正确，而不是强行删除其他来源。

## 8. 验证工具链

### 8.1 版本、架构和目录

**执行位置：Ubuntu 主机（任意目录）**

```bash
go version
go env GOVERSION GOOS GOARCH GOROOT GOPATH GOBIN \
  GOCACHE GOMODCACHE GOTOOLCHAIN GOENV
```

应重点确认：

- `GOOS=linux`。
- `GOARCH` 与客户机架构一致。
- `GOROOT` 指向当前安装。
- `GOPATH`、缓存和个人工具目录属于当前用户。

源码目录不需要放回 `$HOME/go/src`，可按 [[Linux 开发工作区与本地文件系统规划]] 使用 `$HOME/src`。

### 8.2 最小 module

以下代码只写入带固定前缀的 `/tmp` 临时目录，并通过 trap 清理。

**执行位置：Ubuntu 主机（任意目录）**

```bash
WORK_DIR="$(mktemp -d /tmp/go-env-check.XXXXXX)"

cleanup_go_check() {
  case "$WORK_DIR" in
    /tmp/go-env-check.*)
      rm -rf -- "$WORK_DIR"
      ;;
    *)
      printf '跳过清理：路径不符合保护规则：%s\n' "$WORK_DIR" >&2
      ;;
  esac
}

trap cleanup_go_check EXIT
cd "$WORK_DIR"

go mod init example.com/go-env-check
printf '%s\n' \
  'package main' \
  '' \
  'import "fmt"' \
  '' \
  'func main() {' \
  '    fmt.Println("Go environment works")' \
  '}' \
  >main.go

go fmt ./...
go run .
go test ./...
```

预期看到 `Go environment works`，测试正常退出。Shell 结束时只删除本次临时目录。

### 8.3 真实项目

**执行位置：Ubuntu 主机（任意目录）**

```bash
printf '请输入 Go 项目绝对路径：'
IFS= read -r PROJECT_DIR

if [ ! -f "$PROJECT_DIR/go.mod" ]; then
  printf '停止：目标目录没有 go.mod：%s\n' "$PROJECT_DIR" >&2
  exit 1
fi

cd "$PROJECT_DIR"
printf 'project_go=%s\n' \
  "$(awk '$1 == "go" { print $2; exit }' go.mod)"

go version
go env GOOS GOARCH GOMOD
git status --short --branch 2>/dev/null || true
go mod download
go build ./...
git status --short --branch 2>/dev/null || true
```

`go build ./...` 只证明包能够编译，不等于测试、lint、生成物和容器门禁都通过。EventHub 的完整入口见 [[EventHub 仓库迁移与首次质量门禁]]。

## 9. 升级、降级与回滚

项目版本变化时：

1. 重新读取 `go.mod`、`toolchain`、README、CI 和 Dockerfile。
2. 重新选择匹配架构的官方归档。
3. 重新取得对应官方 SHA-256。
4. 重复暂存验证与备份切换。
5. 新开会话运行最小 module 和项目质量门禁。
6. 只有新版本稳定后，才评估清理旧安装。

Go 1.21 及以后支持工具链选择，具体语义见 [Go Toolchains](https://go.dev/doc/toolchain)。自动下载工具链不是忽略团队基线的理由；开发机、CI 和 Docker 构建应尽量对齐。

### 回滚到已记录的备份

先从安装记录中找到准确的备份路径，不自动猜测“最新目录”。

**执行位置：Ubuntu 主机（任意目录）**

```bash
printf '请输入已核对的 Go 备份绝对路径：'
IFS= read -r GO_BACKUP

case "$GO_BACKUP" in
  /usr/local/go.backup.*)
    ;;
  *)
    printf '%s\n' '停止：备份路径不符合保护规则。' >&2
    exit 1
    ;;
esac

if ! sudo test -e "$GO_BACKUP" && ! sudo test -L "$GO_BACKUP"; then
  printf '停止：备份不存在：%s\n' "$GO_BACKUP" >&2
  exit 1
fi

if ! sudo test -e /usr/local/go && ! sudo test -L /usr/local/go; then
  printf '%s\n' '停止：当前 /usr/local/go 不存在，请先人工检查。' >&2
  exit 1
fi

FAILED_DIR="/usr/local/go.failed.$(date +%Y%m%d%H%M%S)"
sudo mv /usr/local/go "$FAILED_DIR"

if sudo mv "$GO_BACKUP" /usr/local/go; then
  hash -r
  /usr/local/go/bin/go version
  printf 'failed_version_saved=%s\n' "$FAILED_DIR"
else
  printf '%s\n' '恢复失败，正在放回刚移走的当前版本。' >&2
  sudo mv "$FAILED_DIR" /usr/local/go
  exit 1
fi
```

该流程保留失败版本用于复盘。恢复后仍需新开会话并运行最小程序与项目门禁。

## 10. 权限与卸载边界

- `/usr/local/go*` 由 root 管理，普通用户只读执行。
- `$HOME/go`、Go 缓存和源码属于当前开发用户。
- 不运行 `sudo go build`、`sudo go test` 或 `sudo go install`。
- 不执行来源不明的 `curl ... | sh`。
- 不关闭 TLS 校验，不全局设置 `GOINSECURE=*`。
- 私有 module 应使用组织批准的 `GOPRIVATE`、Git 凭据和证书方案。

**执行位置：Ubuntu 主机（任意目录）**

```bash
for PATH_NAME in "$HOME/go" "$HOME/.cache/go-build" "$HOME/src"; do
  if [ -e "$PATH_NAME" ]; then
    stat -c '%U:%G %a %n' "$PATH_NAME"
  fi
done
```

若误用 `sudo go ...` 造成用户缓存归 root，先确认真实目录和单用户边界，再按 [[Linux 用户、用户组、sudo 与文件权限]] 最小修复。

停用当前官方安装时，先改名而不是立即删除：

**执行位置：Ubuntu 主机（任意目录）**

```bash
if sudo test -e /usr/local/go || sudo test -L /usr/local/go; then
  DISABLED_DIR="/usr/local/go.disabled.$(date +%Y%m%d%H%M%S)"
  sudo mv /usr/local/go "$DISABLED_DIR"
  printf 'disabled_go=%s\n' "$DISABLED_DIR"
else
  printf '%s\n' '/usr/local/go 不存在，没有执行停用。'
fi
```

随后从 `$HOME/.profile` 删除本文添加的精确 PATH 行，并在新会话确认。缓存、个人工具和版本目录用途不同，不一次性清空。

## 11. 常见问题

### `go: command not found`

**执行位置：Ubuntu 主机（任意目录）**

```bash
test -x /usr/local/go/bin/go
grep -nF '/usr/local/go/bin' "$HOME/.profile" || true
printf '%s\n' "$PATH" | tr ':' '\n'
. "$HOME/.profile"
hash -r
command -v go
```

若当前会话恢复但新 SSH 会话仍失败，检查登录 Shell 实际读取的启动文件，不要把相同行反复追加到多个配置文件。

### 安装后仍显示旧版本

**执行位置：Ubuntu 主机（任意目录）**

```bash
type -a go
command -v go
readlink -f "$(command -v go)"
go env GOROOT
```

先判断 PATH 顺序和 Shell 命令缓存，不直接删除其他安装来源。

### `Exec format error`

**执行位置：Ubuntu 主机（任意目录）**

```bash
uname -m
dpkg --print-architecture
file /usr/local/go/bin/go
```

重新选择与当前 Linux 架构匹配的官方归档。

### module 下载失败

**执行位置：Ubuntu 主机（任意目录）**

```bash
getent hosts proxy.golang.org
date --iso-8601=seconds
go env GOPROXY GOSUMDB GOPRIVATE GONOSUMDB GOINSECURE
curl -I https://proxy.golang.org/
```

在中国大陆或企业网络中，按组织批准的代理、私有 module 仓库和证书方案处理。不要把来源不明、可能失效的第三方镜像地址写入全局配置。

## 相关笔记

- [[Linux 开发工作区与本地文件系统规划]]
- [[Linux 用户、用户组、sudo 与文件权限]]
- [[Git 凭据、SSH 与常见问题排查]]
- [[EventHub 第 1 阶段环境与版本基线]]
- [[EventHub 仓库迁移与首次质量门禁]]

## 官方参考资料

- [Go：Download and install](https://go.dev/doc/install)
- [Go：官方下载页与 SHA-256](https://go.dev/dl/)
- [Go：完整下载 JSON 元数据](https://go.dev/dl/?mode=json&include=all)
- [Go：Managing Go installations](https://go.dev/doc/manage-install)
- [Go：Toolchains](https://go.dev/doc/toolchain)
- [Go：Command documentation](https://go.dev/cmd/go/)
- [Go：Release History](https://go.dev/doc/devel/release)

---
title: Linux 用户、用户组、sudo 与文件权限
aliases:
  - Linux 用户与权限基础
  - Ubuntu sudo 与 umask
tags:
  - Linux
  - Linux/用户
  - Linux/权限
  - sudo
  - umask
created: 2026-07-17T00:48:00
updated: 2026-07-20T00:49:15
---

本文解释 Linux 用户、用户组、UID/GID、文件权限、`sudo` 和 `umask` 的边界，并提供适合个人开发机的验证与恢复方法。

> [!abstract] 本篇掌握目标
> - **必须熟练**：确认当前用户与用户组，读懂基本 `rwx` 权限，安全使用 `sudo`，完成常见 `chmod` 操作。
> - **理解会查**：解释 UID/GID、主组与补充组，根据任务核对 `chown`、目录执行权限、权限链和 `umask`，修改前先限定目标范围。
> - **认识即可**：ACL、特殊权限位和更复杂的共享权限设计；遇到对应场景时再深入。
>
> 命令行如何拆解及怎样查参数见 [[Linux 命令行学习路线与命令地图]] 与 [[Shell 命令结构、类型与帮助系统]]；文件和目录操作的通用命令见 [[Linux 文件与目录常用命令]]。

## 用户与用户组

Linux 权限判断主要依赖数字 UID/GID，而不仅是显示名称。

**执行位置：Linux 主机（任意目录，只读）**

```bash
id
whoami
groups
getent passwd "$USER"
getent group "$(id -gn)"
```

输出中的：

- `uid` 标识用户。
- `gid` 标识主组。
- `groups` 列出补充组。
- 同名用户在不同机器上可能拥有不同 UID。

## root 与 sudo

root 拥有系统级权限，不应作为日常开发账号。`sudo` 让被授权用户按需以更高权限执行单条管理命令。

**执行位置：Linux 主机（普通用户会话）**

```bash
sudo -v
sudo -l
```

`sudo -v` 验证当前凭据，`sudo -l` 显示被允许的命令。不要用 `sudo` 运行普通 Git、Go、Maven 或项目构建，否则容易在 HOME 中留下 root 所有文件。

## 创建额外管理用户

仅在确有需要时创建，不为了教程完整强制增加账号。

**执行位置：Ubuntu 控制台（已有 sudo 用户）**

```bash
(
printf '新用户名称: '
IFS= read -r NEW_USER

case "$NEW_USER" in
  ''|-*|*[!a-z0-9_-]*)
    printf '停止：用户名为空或包含不支持的字符。\n' >&2
    exit 1
    ;;
esac

if id "$NEW_USER" >/dev/null 2>&1; then
  printf '停止：用户已经存在。\n' >&2
else
  sudo adduser "$NEW_USER" &&
    sudo usermod -aG sudo "$NEW_USER" &&
    id "$NEW_USER"
fi
)
```

必须从新的控制台或 SSH 会话实际登录并运行 `sudo -v`，不能只看 `usermod` 退出码。

## 文件权限

基础权限分为 owner、group、others 三组，每组包含：

- `r`：读取。
- `w`：写入。
- `x`：文件可执行或目录可进入。

**执行位置：Linux 主机（任意目录，只读）**

```bash
ls -ld "$HOME" "$HOME/src" 2>/dev/null || true
stat -c 'owner=%U group=%G uid=%u gid=%g mode=%a path=%n' "$HOME"
```

目录的 `x` 表示能否穿过和访问目录内项目；只有 `r` 而没有 `x` 并不能正常进入目录。

## chmod 与 chown

`chmod` 改权限位，`chown` 改所有者和组。两者解决的问题不同。

`chmod` 常见两种写法：符号模式直接表达“给谁增加或移除什么权限”，数字模式一次声明 owner、group、others 三组权限。以下是结构示例，不要对未知真实文件直接照抄：

```text
chmod u+x script.sh
chmod g-w shared.txt
chmod 640 config.ini
```

- `u+x` 表示给 owner 增加执行权限，`g-w` 表示移除 group 的写权限。
- 数字模式中 `r=4`、`w=2`、`x=1`；`640` 表示 owner 为 `rw-`、group 为 `r--`、others 无权限。
- 目录的 `x` 表示能否穿过目录，不能机械套用普通文件的期望。
- `chmod` 不改变所有者；递归 `chmod -R` 会扩大范围，初学阶段不要把它当作权限修复捷径。

个人工作区通常应由当前用户拥有：

**执行位置：Linux 主机（任意目录）**

```bash
install -d -m 0750 "$HOME/src"
stat -c 'owner=%U group=%G mode=%a path=%n' "$HOME/src"
```

不要习惯性执行递归 `chown`。需要修复单个路径时，先规范化并限制范围：

**执行位置：Linux 主机（任意目录）**

```bash
(
SOURCE_ROOT=$(realpath -e -- "$HOME/src") || exit 1
OWNER_UID=$(id -u) || exit 1
OWNER_GID=$(id -g) || exit 1
printf '要修复的单个绝对路径: '
IFS= read -r AFFECTED_PATH

if [ -L "$AFFECTED_PATH" ]; then
  printf '停止：目标本身是符号链接。\n' >&2
elif ! CANONICAL_PATH=$(realpath -e -- "$AFFECTED_PATH"); then
  printf '停止：目标不存在或无法解析。\n' >&2
else
  case "$CANONICAL_PATH" in
    "$SOURCE_ROOT"/*)
      if ! before_record="$(stat -c 'before owner=%U:%G uid=%u gid=%g mode=%a path=%n' "$CANONICAL_PATH")"; then
        printf '%s\n' '停止：无法记录修改前的属主与权限。' >&2
        exit 1
      fi
      printf '%s\n' "$before_record"
      if ! sudo chown -- "$OWNER_UID:$OWNER_GID" "$CANONICAL_PATH"; then
        printf '%s\n' '停止：chown 失败。' >&2
        exit 1
      fi
      if ! stat -c 'after  owner=%U:%G uid=%u gid=%g mode=%a path=%n' "$CANONICAL_PATH"; then
        printf '%s\n' '警告：chown 已执行，但修改后验证失败，请立即人工核对。' >&2
        exit 1
      fi
      ;;
    *)
      printf '停止：规范化路径不在 %s/ 内。\n' "$SOURCE_ROOT" >&2
      ;;
  esac
fi
)
```

恢复时使用修改前记录的数字 UID/GID，而不是猜测用户名。

## umask

`umask` 是新文件和目录权限的屏蔽规则，不会追溯修改现有文件。

**执行位置：Linux 主机（任意目录）**

```bash
umask
umask -S
```

常见 `0022` 通常产生：

- 新普通文件约为 `0644`。
- 新目录约为 `0755`。

较严格的 `0027` 或 `0077` 是否合适取决于共享需求。不要只看到数字就全局修改 Shell 配置；先在临时目录验证：

**执行位置：Linux 主机（任意目录）**

```bash
(
if ! test_dir="$(mktemp -d /tmp/permission-check.XXXXXX)"; then
  printf '%s\n' '停止：无法创建权限练习目录。' >&2
  exit 1
fi
readonly test_dir

cleanup_permission_check() {
  case "$test_dir" in
    /tmp/permission-check.*)
      rm -rf -- "$test_dir"
      ;;
    *)
      printf '跳过清理：路径不符合保护规则：%s\n' "$test_dir" >&2
      ;;
  esac
}

trap cleanup_permission_check EXIT

(
  umask 0027
  touch "$test_dir/sample-file"
  mkdir "$test_dir/sample-dir"
)

stat -c 'mode=%a path=%n' "$test_dir/sample-file" "$test_dir/sample-dir"
chmod u+x "$test_dir/sample-file"
stat -c 'after u+x mode=%a path=%n' "$test_dir/sample-file"
chmod 640 "$test_dir/sample-file"
stat -c 'after 640 mode=%a path=%n' "$test_dir/sample-file"
)
```

这些带输入校验或 `trap` 的示例显式运行在圆括号创建的子 Shell 中；其中的 `exit` 和退出清理只结束该代码块，不会退出当前登录 Shell 或把 `trap` 留到会话结束。

## 常见问题

### 构建生成 root 文件

先找出范围：

**执行位置：Linux 主机（任意目录，只读）**

```bash
find "$HOME/src" -xdev -not -user "$USER" \
  -printf '%u:%g %m %p\n' | sed -n '1,80p'
```

修复根因通常是停止使用 `sudo go`、`sudo mvn`、`sudo npm` 或以 root 身份运行 IDE。不要先递归改属主再继续产生同类文件。

### 新组权限没有生效

补充组在登录会话创建时加载。加入组后最可靠的方式是退出并新建会话，再运行：

**执行位置：Linux 主机（新登录会话，任意目录，只读）**

```bash
id
groups
```

### 目录能列出但不能进入

检查父目录的 `x` 权限、ACL 和实际路径：

**执行位置：Linux 主机（任意目录，只读）**

```bash
namei -l "$HOME/src"
getfacl "$HOME/src" 2>/dev/null || true
```

## 完成标准

- [ ] 能解释 UID/GID、主组和补充组。
- [ ] 日常开发不依赖 root 登录。
- [ ] 能区分 `chmod`、`chown` 和 `umask`。
- [ ] 能安全验证 sudo 权限。
- [ ] 不使用无限制递归 `chown` 修复工作区。

## 官方参考资料

以下资料于 **2026-07-16** 核对：

- [Ubuntu Server：用户管理](https://documentation.ubuntu.com/server/how-to/security/user-management/)
- [GNU Bash：umask](https://www.gnu.org/software/bash/manual/bash.html#index-umask)

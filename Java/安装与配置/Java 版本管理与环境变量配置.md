---
title: Java 版本管理与环境变量配置
aliases:
  - Java 多版本管理
  - JAVA_HOME 与 PATH 配置
tags:
  - Java
  - Java/配置
  - Java/版本管理
  - Java/环境变量
  - Maven/Toolchains
created: 2026-07-14T23:30:24
updated: 2026-07-14T23:42:04
---

一台开发机可以同时安装多个 JDK，但在某个具体进程中，最终只能选择一个 Java 运行时。版本管理的核心不是“安装了多少个 JDK”，而是明确终端、Maven、IDE、CI 和应用服务各自选择了哪一个 JDK，以及项目如何把这一选择记录下来。

首次安装请先完成 [[Ubuntu 安装 Java 与 Maven]] 或 [[macOS 安装 Java 与 Maven]]。本文重点讲解 `JAVA_HOME`、`PATH`、Ubuntu alternatives、macOS `java_home`、SDKMAN、Maven Toolchains 与项目级版本约束。

## 1. Java 版本是如何被选中的

当 Shell 执行 `java` 时，会按照 `PATH` 从左到右查找可执行文件。`JAVA_HOME` 是另一个独立变量，供 Maven、Gradle、IDE、Tomcat 和脚本寻找 JDK 根目录。

| 选择入口 | 影响范围 | 常见误区 |
| --- | --- | --- |
| `PATH` | 当前进程及其子进程执行哪个 `java`、`javac` | 只改 `JAVA_HOME`，却没有调整 `PATH` |
| `JAVA_HOME` | Maven、构建脚本和部分工具选择哪个 JDK | 把它设成 `$JAVA_HOME/bin` 或 `bin/java` |
| Ubuntu alternatives | `/usr/bin/java`、`/usr/bin/javac` 等系统命令链接 | 只切换 `java`，没有检查 `javac` |
| macOS `java_home` | 从已注册 JDK 中按版本、架构选择目录 | JDK 没有注册到 JavaVirtualMachines |
| SDKMAN | 当前用户和当前 Shell 的候选 SDK | 与系统包管理器交叉覆盖后不检查 `PATH` |
| Maven Toolchains | Maven 构建插件使用的 JDK | 误以为它也改变了运行 Maven 自身的 JDK |
| IDE 设置 | IDE 编译、导入、运行和测试使用的 JDK | 以为 IDE 一定读取终端 `JAVA_HOME` |

## 2. 建立诊断基线

每次调整版本前后，都运行同一组命令：

```bash
printf 'configured shell=%s\n' "$SHELL"
printf 'current process=' && ps -p $$ -o comm=
printf 'JAVA_HOME=%s\n' "${JAVA_HOME:-<未设置>}"

type -a java 2>/dev/null || true
type -a javac 2>/dev/null || true
type -a mvn 2>/dev/null || true

java -version 2>&1 || true
javac -version 2>&1 || true
mvn -version 2>&1 || true
```

再让 JVM 报告自己的真实安装目录：

```bash
java -XshowSettings:properties -version 2>&1 \
  | grep -E 'java.home|java.version|os.arch'
```

`command -v java` 只告诉你 Shell 找到的入口，它可能是符号链接或包装器；`java.home` 才是 JVM 进程认定的运行时目录。

## 3. 正确理解 `JAVA_HOME`

正确的 `JAVA_HOME` 指向 JDK 根目录：

```text
<JAVA_HOME>/bin/java
<JAVA_HOME>/bin/javac
<JAVA_HOME>/bin/jar
```

可以用通用检查确认：

```bash
test -n "${JAVA_HOME:-}"
test -x "$JAVA_HOME/bin/java"
test -x "$JAVA_HOME/bin/javac"
"$JAVA_HOME/bin/java" -version
"$JAVA_HOME/bin/javac" -version
```

以下都是错误或脆弱的配置：

```bash
export JAVA_HOME=/path/to/jdk/bin
export JAVA_HOME=/path/to/jdk/bin/java
export PATH="$PATH:$JAVA_HOME/bin"
```

前两条层级错误；第三条把目标 JDK 放在 `PATH` 末尾，可能仍被前面的旧 Java 覆盖。通常应写成：

```bash
export JAVA_HOME=/path/to/jdk
export PATH="$JAVA_HOME/bin:$PATH"
```

## 4. Ubuntu：使用 alternatives 管理系统默认 JDK

查看系统候选项：

```bash
update-alternatives --list java
update-alternatives --list javac
```

交互式选择：

```bash
sudo update-alternatives --config java
sudo update-alternatives --config javac
```

切换后验证二者属于同一 JDK 主版本：

```bash
readlink -f "$(command -v java)"
readlink -f "$(command -v javac)"
java -version
javac -version
```

根据 `javac` 计算 `JAVA_HOME`：

```bash
export JAVA_HOME="$(dirname "$(dirname "$(readlink -f "$(command -v javac)")")")"
export PATH="$JAVA_HOME/bin:$PATH"
```

Ubuntu 还可能提供 `update-java-alternatives`，用于一次切换同一 JDK 注册的一组工具：

```bash
update-java-alternatives --list
```

具体名称和可用性取决于安装包。执行系统级切换前，先查看候选列表并确认服务影响；共享服务器上不要为了个人项目改变所有用户的系统默认值。

## 5. macOS：使用 `/usr/libexec/java_home`

列出 macOS 已注册 JDK：

```bash
/usr/libexec/java_home -V
```

按主版本查询路径：

```bash
/usr/libexec/java_home -v 25
/usr/libexec/java_home -v 21
/usr/libexec/java_home -v 17
```

只在当前终端切换到 JDK 21：

```bash
export JAVA_HOME="$(/usr/libexec/java_home -v 21)"
export PATH="$JAVA_HOME/bin:$PATH"
java -version
javac -version
mvn -version
```

永久默认可以写进 `~/.zshrc`，但只适合“全局默认版本”这一层。多个项目需要不同版本时，频繁手改 `~/.zshrc` 容易残留重复配置，优先使用 SDKMAN、项目脚本、IDE 配置或 Maven Toolchains。

## 6. 使用 SDKMAN 管理多个 JDK 和 Maven

SDKMAN 支持 Linux、macOS 和 WSL，并能管理 Java、Maven、Gradle 等 JVM 工具。它安装在当前用户目录中，适合个人开发机，不应在没有评估的生产服务器上替代系统包管理。

### 安装 SDKMAN

官方提供远程安装脚本。为了先检查内容，可以下载后再执行：

```bash
installer="$HOME/sdkman-install.sh"
curl -fsSL https://get.sdkman.io -o "$installer"
less "$installer"
bash "$installer"
rm -f "$installer"
```

新开终端，或者在当前 Shell 加载：

```bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk version
```

SDKMAN 安装器会向 Bash 或 Zsh 启动文件追加初始化片段。不要再手工复制多份相同片段。

### 查看并安装精确 JDK

先刷新候选列表，并从当前列表选择完整标识符：

```bash
sdk update
sdk list java
```

安装时使用 `sdk list java` 实际显示的标识符，不要长期照抄笔记里的旧补丁版本。下面以 2026-07-14 可用的 Temurin 25.0.3 为例，执行时应替换变量值：

```bash
JAVA_IDENTIFIER='25.0.3-tem'
sdk install java "$JAVA_IDENTIFIER"
```

查看当前版本：

```bash
sdk current java
java -version
javac -version
```

### 临时切换与默认版本

只切换当前 Shell：

```bash
sdk use java "$JAVA_IDENTIFIER"
```

设置以后新 Shell 的默认版本：

```bash
sdk default java "$JAVA_IDENTIFIER"
```

安装和切换 Maven 的方式相同：

```bash
sdk list maven
MAVEN_VERSION='3.9.16'
sdk install maven "$MAVEN_VERSION"
sdk use maven "$MAVEN_VERSION"
mvn -version
```

### 使用 `.sdkmanrc` 固定项目环境

进入项目根目录，在已经选择好 Java 和 Maven 版本的情况下生成配置：

```bash
sdk env init
```

检查生成的 `.sdkmanrc`：

```bash
sed -n '1,120p' .sdkmanrc
```

进入项目后按配置切换：

```bash
sdk env
```

缺少声明的 SDK 时：

```bash
sdk env install
```

离开项目并恢复默认：

```bash
sdk env clear
```

是否提交 `.sdkmanrc` 由团队决定。提交前应确保其中只有公开的版本标识符，不含个人路径和敏感配置。自动切换需要在 `~/.sdkman/etc/config` 中显式启用 `sdkman_auto_env=true`；首次接触的项目建议先查看 `.sdkmanrc`，再手动执行 `sdk env`。

## 7. Maven 运行 JDK 与编译 JDK 并不一定相同

`mvn -version` 显示的是“运行 Maven 自身”的 JDK。Maven Toolchains 可以让编译器、测试、Javadoc 等支持 Toolchains 的插件使用另一个 JDK。

例如，Maven 可以由 JDK 25 启动，但项目明确要求用 JDK 21 编译。此时用户级 `~/.m2/toolchains.xml` 可以声明本机 JDK 路径：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<toolchains>
  <toolchain>
    <type>jdk</type>
    <provides>
      <version>21</version>
      <vendor>openjdk</vendor>
    </provides>
    <configuration>
      <jdkHome>/absolute/path/to/jdk-21</jdkHome>
    </configuration>
  </toolchain>
</toolchains>
```

macOS 的 `jdkHome` 可以通过下面命令获得：

```bash
/usr/libexec/java_home -v 21
```

Ubuntu 可以从 `javac` 路径计算，或者查看具体安装目录：

```bash
find /usr/lib/jvm -maxdepth 2 -type f -name javac -print
```

项目还需要在 `pom.xml` 中配置 Maven Toolchains Plugin 和所需版本。用户级文件只描述“这台机器有哪些 JDK”，项目 POM 才描述“这个构建需要哪种 Toolchain”。不要把个人绝对路径写进项目 POM。

完整 Maven 配置参见 [[Maven 常用配置与仓库管理]]，官方 Toolchains 说明见文末链接。

## 8. 项目级版本约束的推荐顺序

| 层次 | 解决的问题 | 建议是否提交到 Git |
| --- | --- | --- |
| `README` 和开发文档 | 告诉开发者需要的 JDK、Maven 与安装入口 | 是 |
| Maven Wrapper | 固定 Maven 版本 | 是 |
| `pom.xml` 编译器配置 | 固定语言级别和字节码目标 | 是 |
| Maven Enforcer | 在环境不符合要求时尽早失败 | 是 |
| Maven Toolchains Plugin | 让构建插件选择指定 JDK | 是 |
| `.sdkmanrc` | 为使用 SDKMAN 的开发者切换本机工具 | 团队决定 |
| `~/.m2/toolchains.xml` | 描述本机 JDK 绝对路径 | 否 |
| IDE SDK 设置 | 控制本机 IDE 导入、编译与运行 | 通常不提交个人配置 |
| CI `setup-java` 或构建镜像 | 固定自动化环境 | 是 |

> [!important] `source` 和 `target` 不能替代真实 JDK
> Maven Compiler Plugin 的 `release`、`source`、`target` 控制语言和字节码兼容性，但它们不会凭空提供另一个 JDK 的标准库和工具。需要严格选择构建 JDK 时，应使用对应 JDK、Toolchains 和 CI 配置。

## 9. IDE、终端和服务进程的边界

同一台机器上可能同时存在：

- 外部终端启动的 Maven。
- IDE 内置终端启动的 Maven。
- IDE Maven Importer 和 Maven Runner。
- IDE 项目 SDK 与模块 SDK。
- systemd、launchd 或容器中的 Java 服务。

它们不一定读取同一份 Shell 配置。排查时分别记录每个入口的 `java -version`、`mvn -version` 和 JDK 路径，不要用外部终端的结果代替 IDE 或服务进程的真实配置。

## 10. 常见错误与检查方法

### `JAVA_HOME` 与 `java` 指向不同版本

```bash
printf 'JAVA_HOME=%s\n' "${JAVA_HOME:-<未设置>}"
"$JAVA_HOME/bin/java" -version
java -version
type -a java
```

搜索重复配置：

```bash
grep -nE 'JAVA_HOME|sdkman|java_home|openjdk' \
  "$HOME/.profile" "$HOME/.bashrc" "$HOME/.zprofile" "$HOME/.zshrc" \
  2>/dev/null || true
```

### 切换后 Shell 仍执行旧路径

Bash：

```bash
hash -r
```

Zsh：

```zsh
rehash
```

然后新开终端并重新运行完整基线检查。

### SDKMAN 与系统 JDK 互相覆盖

SDKMAN 初始化通常会调整 `PATH`。加载后检查：

```bash
sdk current java
type -a java
printf '%s\n' "$PATH" | tr ':' '\n'
```

如果决定使用 SDKMAN 管理当前用户的 JDK，就让它成为明确的用户级入口；系统 JDK仍可保留给系统服务，但不要在同一个 Shell 启动文件中反复覆盖顺序。

## 官方参考资料

- [SDKMAN：安装](https://sdkman.io/install/)
- [SDKMAN：使用](https://sdkman.io/usage/)
- [Apache Maven：使用 Toolchains](https://maven.apache.org/guides/mini/guide-using-toolchains.html)
- [Apache Maven Toolchains Plugin](https://maven.apache.org/plugins/maven-toolchains-plugin/)
- [Apache Maven Wrapper](https://maven.apache.org/tools/wrapper/)
- [Eclipse Adoptium：安装](https://adoptium.net/installation/)
- [Ubuntu：配置 Java 开发环境](https://ubuntu.com/developers/docs/howto/java-setup/)

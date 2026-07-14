---
title: Maven 常用配置与仓库管理
aliases:
  - Maven settings.xml 配置
  - Maven 本地仓库与镜像配置
tags:
  - Java
  - Maven
  - Maven/配置
  - Maven/仓库
  - Maven/Wrapper
created: 2026-07-14T23:30:24
updated: 2026-07-14T23:41:06
---

Apache Maven 安装完成后，即使没有创建 `settings.xml` 也能使用默认配置访问 Maven Central。只有在需要企业仓库、代理、认证、自定义本地仓库、Profile、Toolchains 或项目级 JVM 参数时，才应增加对应配置。

首次安装请先完成 [[Ubuntu 安装 Java 与 Maven]] 或 [[macOS 安装 Java 与 Maven]]。JDK 多版本选择和 `toolchains.xml` 的原理见 [[Java 版本管理与环境变量配置]]；依赖下载或认证异常见 [[Java 与 Maven 环境排障与维护]]。

## 1. Maven 配置分布在哪里

| 位置 | 作用范围 | 是否适合提交到项目 Git |
| --- | --- | --- |
| `${maven.home}/conf/settings.xml` | 当前 Maven 安装的全局配置 | 否；升级或重装可能被替换 |
| `~/.m2/settings.xml` | 当前用户的仓库、代理、认证和 Profile | 否；通常包含机器或账号信息 |
| `~/.m2/toolchains.xml` | 当前用户机器上可用的 JDK Toolchain 路径 | 否；包含本机绝对路径 |
| `~/.m2/repository/` | 默认本地依赖和插件缓存 | 否 |
| 项目 `pom.xml` | 项目依赖、插件、构建生命周期和共享约束 | 是 |
| 项目 `.mvn/maven.config` | 每次构建都应使用的 Maven CLI 参数 | 是，确认不含个人和敏感参数后 |
| 项目 `.mvn/jvm.config` | 启动 Maven JVM 的项目级 JVM 参数 | 是，确认适合所有构建环境后 |
| 项目 `.mvn/wrapper/`、`mvnw`、`mvnw.cmd` | 固定 Maven Wrapper 版本和启动脚本 | 是 |

Maven 会合并全局和用户 `settings.xml`，用户配置优先。不要直接修改 Homebrew、apt 或 SDKMAN 安装目录中的全局配置；开发者个人配置应放在 `~/.m2/settings.xml`。

## 2. 创建最小用户配置

先创建目录并限制文件权限：

```bash
mkdir -p "$HOME/.m2"
touch "$HOME/.m2/settings.xml"
chmod 600 "$HOME/.m2/settings.xml"
```

一个没有自定义项的最小结构如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
</settings>
```

修改后检查 Maven 计算出的最终配置：

```bash
mvn help:effective-settings
```

该目标默认隐藏密码。不要使用 `-DshowPasswords=true` 后把输出复制到工单、聊天或构建日志。

## 3. 本地仓库

默认本地仓库是：

```text
~/.m2/repository
```

通常不需要修改。需要把依赖缓存放到另一个磁盘时，可以在 `settings.xml` 顶层设置：

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository>/absolute/path/to/maven-repository</localRepository>
</settings>
```

路径应满足：

- 当前用户可读写。
- 位于可靠的本地文件系统。
- 不提交到项目 Git。
- 不与多个并发用户共享同一个没有隔离和权限设计的目录。
- 不放在容易被同步软件部分占位或自动清理的位置。

临时为一次构建指定本地仓库：

```bash
mvn -Dmaven.repo.local="$HOME/.cache/maven-repository" test
```

这适合诊断“默认本地仓库是否损坏”，不适合在每个项目中随意创建多套长期缓存。

## 4. 远程仓库、镜像与仓库管理器

Maven Central 的仓库 ID 是 `central`，默认地址是 `https://repo.maven.apache.org/maven2/`。普通开源项目无需为 Central 重复声明 `<repository>`。

企业通常通过 Nexus、Artifactory 或其他仓库管理器代理 Central 和内部制品。此时管理员会提供准确地址、证书、认证方式和 `mirrorOf` 规则。下面只使用占位域名，不能直接运行：

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <mirrors>
    <mirror>
      <id>company-maven</id>
      <name>Company Maven Repository Manager</name>
      <url>https://repo.example.com/repository/maven-public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>
</settings>
```

`mirrorOf` 常见语义：

| 值 | 含义 | 使用提醒 |
| --- | --- | --- |
| `central` | 只替代 Maven Central | 最容易控制影响范围 |
| `*` | 替代所有仓库 | 企业仓库必须能代理构建需要的全部仓库 |
| `external:*` | 替代所有非本机外部仓库 | 适合集中治理，但仍应按组织方案验证 |
| `*,!inhouse` | 替代除 `inhouse` 外的所有仓库 | ID 必须与项目声明精确匹配 |

Maven 不会把一个制品请求同时分发到多个同 ID 镜像；配置多个 `mirrorOf=central` 并不形成自动容灾或负载均衡。若组织需要聚合和缓存，应由仓库管理器完成。

> [!warning] 不要复制来源不明的公共镜像
> 镜像控制 Maven 获取依赖和插件的来源。错误或被篡改的镜像可能返回过期、不完整甚至恶意制品。个人网络慢时，优先使用 Maven Central、组织仓库或当时仍由可信组织维护的服务，并记录来源与生效范围。

## 5. 配置 HTTP 代理

Maven 的 JVM 进程不一定自动继承浏览器、Docker Desktop 或系统 GUI 的代理配置。组织提供明确的代理参数后，在 `settings.xml` 中配置：

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">
  <proxies>
    <proxy>
      <id>company-proxy</id>
      <active>true</active>
      <protocol>http</protocol>
      <host>proxy.example.com</host>
      <port>3128</port>
      <nonProxyHosts>localhost|127.0.0.1|*.example.internal</nonProxyHosts>
    </proxy>
  </proxies>
</settings>
```

代理需要认证时可增加 `<username>` 和 `<password>`，但不要把真实凭据放进公开示例、项目 POM 或 Git。多个代理配置中同时只应有一个 `<active>true</active>`。

验证 Maven 实际看到的配置：

```bash
mvn help:effective-settings
mvn -U -X help:effective-settings
```

`-X` 会输出大量调试信息，可能包含内部主机名、仓库地址和环境细节。分享日志前必须脱敏。

## 6. 私有仓库认证

项目 POM 或 `distributionManagement` 使用一个仓库 ID，`settings.xml` 的 `<server><id>` 必须与它完全相同：

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">
  <servers>
    <server>
      <id>company-releases</id>
      <username>${env.MAVEN_REPO_USERNAME}</username>
      <password>${env.MAVEN_REPO_PASSWORD}</password>
    </server>
  </servers>
</settings>
```

然后在当前受控会话或 CI Secret 中提供环境变量。不要把真实值写进 `~/.zshrc`、`~/.bashrc`、`.env` 后又意外提交。

Maven 3 也支持使用 `settings-security.xml` 加密服务器密码。生成时让 Maven交互提示，不把密码放进命令历史：

```bash
mvn --encrypt-master-password
mvn --encrypt-password
```

第一条输出放入 `~/.m2/settings-security.xml` 的 `<master>`，第二条输出放入 `settings.xml` 对应 `<password>`。官方文档明确说明 Maven 3 的主密码使用硬编码密钥保护，应把它视为仍需严密保护的本地秘密，而不是等同于操作系统密钥链或专业 Secrets Manager。

优先级建议：

1. CI 使用平台 Secret 与短期令牌。
2. 开发机使用组织批准的凭据注入或密钥管理方案。
3. 必须使用 Maven 3 内置加密时，限制 `settings.xml` 和 `settings-security.xml` 权限，并保护备份。
4. 永远不要提交这两个用户级文件。

## 7. Profile 应放在哪里

`pom.xml` Profile 适合描述项目共享的构建差异；`settings.xml` Profile 适合描述当前用户或环境的仓库和属性。不要用用户 Profile 隐藏项目构建必需的依赖或插件，否则 CI 和其他开发者无法复现。

用户 Profile 示例：

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">
  <profiles>
    <profile>
      <id>company-network</id>
      <properties>
        <company.environment>development</company.environment>
      </properties>
    </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>company-network</activeProfile>
  </activeProfiles>
</settings>
```

查看可用和已激活 Profile：

```bash
mvn help:all-profiles
mvn help:active-profiles
```

临时激活项目 Profile：

```bash
mvn -Pdevelopment test
```

## 8. Maven Wrapper

Maven Wrapper 让项目固定 Maven 版本。已经包含 Wrapper 的项目应使用：

```bash
./mvnw -version
./mvnw test
```

而不是直接执行：

```bash
mvn test
```

为已有 Maven 项目添加或更新 Wrapper：

```bash
mvn wrapper:wrapper
```

生成后通常包含：

```text
.mvn/wrapper/maven-wrapper.properties
mvnw
mvnw.cmd
```

检查并提交这些文件，使 Linux、macOS 和 Windows 开发者使用同一 Maven 版本。Unix 脚本应有执行权限：

```bash
chmod +x mvnw
git add --chmod=+x mvnw
```

Wrapper 首次运行会下载 Maven 发行版，默认缓存到 `~/.m2/wrapper/dists`。安全要求较高的项目可以在 `maven-wrapper.properties` 中配置官方文档支持的 `distributionSha256Sum`，校验下载的 Maven 分发包。

> [!important] Wrapper 固定 Maven，不自动安装 JDK
> 运行 `mvnw` 前仍需要可用 JDK。Wrapper 选择的 Maven 与 Maven Toolchains 选择的编译 JDK 是两个不同层次。

## 9. Maven Toolchains

先让 Maven Toolchains Plugin 显示它发现的 JDK：

```bash
mvn org.apache.maven.plugins:maven-toolchains-plugin:3.2.0:display-discovered-jdk-toolchains
```

也可以生成候选 `toolchains.xml` 内容：

```bash
mvn org.apache.maven.plugins:maven-toolchains-plugin:3.2.0:generate-jdk-toolchains-xml
```

生成内容需要人工检查，再决定是否写入 `~/.m2/toolchains.xml`。JDK 路径和项目 POM 的分工见 [[Java 版本管理与环境变量配置]]。

## 10. Maven 与项目级 JVM 参数

`MAVEN_OPTS` 传给“运行 Maven 的 JVM”，例如内存或系统属性：

```bash
export MAVEN_OPTS="-Xms256m -Xmx2g -Dfile.encoding=UTF-8"
```

不应把大内存配置无条件写入所有开发者的全局 Shell 文件。项目确实需要统一参数时，可以放在 `.mvn/jvm.config`：

```text
-Xms256m
-Xmx2g
-Dfile.encoding=UTF-8
```

Maven 3.9.0 起支持 `MAVEN_ARGS`，用于在命令行参数前增加 Maven 参数；项目级共享 CLI 参数则可放在 `.mvn/maven.config`。两者都不要用于隐藏开发者不知道的发布、删除或凭据参数。

## 11. 常用生命周期与诊断命令

| 命令 | 作用 | 使用提醒 |
| --- | --- | --- |
| `mvn validate` | 检查项目模型和早期生命周期 | 不代表已经编译或测试 |
| `mvn compile` | 编译主代码 | 输出通常在 `target/classes` |
| `mvn test` | 编译并运行单元测试 | 日常最小验证入口 |
| `mvn verify` | 完成测试并执行集成验证阶段 | CI 常用，但具体行为取决于插件配置 |
| `mvn package` | 生成 JAR、WAR 等制品 | 不安装到本地仓库 |
| `mvn install` | 把当前项目制品安装到本地仓库 | 不等于安装 Maven 软件 |
| `mvn deploy` | 发布到远程仓库 | 会产生外部写入，执行前确认目标和凭据 |
| `mvn clean` | 删除项目 `target/` | 不清理 `~/.m2/repository` |
| `mvn dependency:tree` | 查看依赖树 | 排查版本冲突常用 |
| `mvn help:effective-pom` | 查看继承和 Profile 合并后的 POM | 输出可能很长 |
| `mvn help:effective-settings` | 查看合并后的 Maven Settings | 默认隐藏密码，日志仍可能暴露内部地址 |
| `mvn -o test` | 离线构建 | 本地缺少依赖时会失败 |
| `mvn -U test` | 强制检查 Snapshot 和缺失更新 | 不应作为每次构建的默认参数 |

## 12. 本地仓库清理原则

不要遇到任何下载错误就删除整个 `~/.m2/repository`。先定位具体坐标，例如：

```text
groupId: com.example
artifactId: example-library
version: 1.2.3
```

它对应的默认路径大致是：

```text
~/.m2/repository/com/example/example-library/1.2.3/
```

先停止并发 Maven 构建，备份或删除这个明确的版本目录，再执行：

```bash
mvn -U test
```

也可以评估 Maven Dependency Plugin 的 `dependency:purge-local-repository`，但要先阅读当前插件文档和参数，避免清除超出预期的依赖。

## 中国大陆网络环境提示

Maven 下载慢或超时不等于必须配置公共镜像。按顺序检查：

1. `curl -I https://repo.maven.apache.org/maven2/` 是否能建立 TLS 连接。
2. `mvn -version` 使用的 JDK 是否信任当前网络证书。
3. Maven 是否需要组织代理。
4. `settings.xml` 是否存在失效镜像或仓库地址。
5. 企业仓库是否可用、认证是否有效。
6. CI 与开发机是否使用同一网络和 Settings。

如果组织提供 Nexus 或 Artifactory，应优先使用组织发布的地址和证书策略。不要把个人代理端口、账号密码或临时镜像提交到共享项目。

## 官方参考资料

- [Apache Maven：配置](https://maven.apache.org/configure.html)
- [Apache Maven Settings Reference](https://maven.apache.org/settings.html)
- [Apache Maven：Mirror 配置](https://maven.apache.org/guides/mini/guide-mirror-settings.html)
- [Apache Maven：Proxy 配置](https://maven.apache.org/guides/mini/guide-proxies.html)
- [Apache Maven 3：密码加密](https://maven.apache.org/guides/mini/guide-encryption.html)
- [Apache Maven Wrapper](https://maven.apache.org/tools/wrapper/)
- [Apache Maven Toolchains Plugin](https://maven.apache.org/plugins/maven-toolchains-plugin/)
- [Apache Maven Help Plugin：effective-settings](https://maven.apache.org/plugins/maven-help-plugin/effective-settings-mojo.html)

# Spring Boot 多环境 Logback 日志配置

## 目标

本文配置适用于 Spring Boot 2.2+、3.x 和 4.x，约定：

- `local` 是开发环境：输出 `DEBUG` 及以上日志，同时写入控制台和文件。
- 非 `local` 是生产环境：输出 `INFO` 及以上日志，只写入文件。
- 普通日志文件保留全部达到阈值的日志，包括 `ERROR`。
- `ERROR` 日志额外复制到独立的 error 文件。
- 普通日志和 error 日志均按日期及文件大小轮转。
- 生产、开发环境通过 Spring 配置文件使用不同的日志目录。

Spring Boot 官方支持在 `logback-spring.xml` 中使用 `<springProfile>` 区分环境，使用 `<springProperty>` 从 Spring `Environment` 读取配置。不要用普通的 `<property>` 直接读取 `logging.file.path`。

参考文档：

- [Spring Boot Logging](https://docs.spring.io/spring-boot/reference/features/logging.html)
- [Logback RollingFileAppender](https://logback.qos.ch/manual/appenders.html#SizeAndTimeBasedRollingPolicy)

## application.yml

基础配置作为生产默认配置：

```yaml
spring:
  application:
    name: demo-service

logging:
  level:
    root: INFO

  file:
    # 生产环境可通过 APP_LOG_PATH 环境变量覆盖
    path: ${APP_LOG_PATH:/var/log/demo-service}
```

Spring Boot 配置文件中的默认值语法是 `${属性:默认值}`，不是 Logback 的 `:-` 语法。

## application-local.yml

```yaml
logging:
  level:
    root: DEBUG

  file:
    # 开发环境可通过 APP_LOG_PATH 环境变量覆盖
    path: ${APP_LOG_PATH:./logs}
```

激活 `local` 后，Spring Boot 会使用这里的日志级别和目录覆盖 `application.yml` 中的生产默认值。

## logback-spring.xml

文件位置：`src/main/resources/logback-spring.xml`。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 读取合并 application.yml 和 profile 配置后的 Spring Environment -->
    <springProperty
            scope="context"
            name="APP_NAME"
            source="spring.application.name"
            defaultValue="application"/>

    <springProperty
            scope="context"
            name="LOG_DIR"
            source="logging.file.path"
            defaultValue="./logs"/>

    <springProperty
            scope="context"
            name="ROOT_LEVEL"
            source="logging.level.root"
            defaultValue="INFO"/>

    <property
            name="LOG_PATTERN"
            value="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{50} - %msg%n"/>

    <!-- 开发环境控制台输出 -->
    <appender name="CONSOLE"
              class="ch.qos.logback.core.ConsoleAppender">

        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>DEBUG</level>
        </filter>

        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <charset>UTF-8</charset>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!--
        普通日志文件：
        local 环境写入 DEBUG、INFO、WARN、ERROR；
        生产环境写入 INFO、WARN、ERROR。

        这里没有排除 ERROR，因此 ERROR 会同时出现在普通日志和独立 error 日志中。
    -->
    <appender name="APPLICATION_FILE"
              class="ch.qos.logback.core.rolling.RollingFileAppender">

        <file>${LOG_DIR}/${APP_NAME}.log</file>
        <append>true</append>

        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>${ROOT_LEVEL}</level>
        </filter>

        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- 每天轮转；同一天超过 100MB 时使用递增序号 -->
            <fileNamePattern>
                ${LOG_DIR}/archive/${APP_NAME}.%d{yyyy-MM-dd}.%i.log.gz
            </fileNamePattern>

            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
            <cleanHistoryOnStart>true</cleanHistoryOnStart>
        </rollingPolicy>

        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <charset>UTF-8</charset>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- ERROR 日志额外复制到独立文件 -->
    <appender name="ERROR_FILE"
              class="ch.qos.logback.core.rolling.RollingFileAppender">

        <file>${LOG_DIR}/${APP_NAME}-error.log</file>
        <append>true</append>

        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>

        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>
                ${LOG_DIR}/archive/${APP_NAME}-error.%d{yyyy-MM-dd}.%i.log.gz
            </fileNamePattern>

            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>5GB</totalSizeCap>
            <cleanHistoryOnStart>true</cleanHistoryOnStart>
        </rollingPolicy>

        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <charset>UTF-8</charset>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- 开发环境：DEBUG+，控制台和文件同时输出 -->
    <springProfile name="local">
        <root level="${ROOT_LEVEL}">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="APPLICATION_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
        </root>
    </springProfile>

    <!-- 生产环境：INFO+，只写文件 -->
    <springProfile name="!local">
        <root level="${ROOT_LEVEL}">
            <appender-ref ref="APPLICATION_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
        </root>
    </springProfile>

</configuration>
```

## 输出结果

| 环境 | 控制台 | 普通日志文件 | ERROR 文件 |
| --- | --- | --- | --- |
| `local` | DEBUG、INFO、WARN、ERROR | DEBUG、INFO、WARN、ERROR | ERROR |
| 非 `local` | 不输出 | INFO、WARN、ERROR | ERROR |

归档文件示例：

```text
logs/
├── demo-service.log
├── demo-service-error.log
└── archive/
    ├── demo-service.2026-07-22.0.log.gz
    ├── demo-service.2026-07-22.1.log.gz
    └── demo-service-error.2026-07-22.0.log.gz
```

`SizeAndTimeBasedRollingPolicy` 同时使用日期和大小轮转，因此 `fileNamePattern` 中的 `%d` 与 `%i` 都不能省略。

## 启动方式

开发环境：

```bash
java -jar app.jar --spring.profiles.active=local
```

生产环境：

```bash
java -jar app.jar --spring.profiles.active=prod
```

这里约定所有非 `local` 环境都采用生产日志策略，因此未指定 profile 时也只写文件。

## 注意事项

1. 文件名必须是 `logback-spring.xml`，不能改成普通的 `logback.xml`，否则 Spring Boot 的 `<springProfile>` 和 `<springProperty>` 扩展无法正常使用。
2. 不要在 `<configuration>` 上配置 `scan="true"`，Spring Boot 的 Logback 扩展与 Logback 配置扫描不兼容。
3. `logging.file.path` 表示日志目录；Spring Boot 将其映射为 `LOG_PATH`。`LOG_FILE` 对应的是 `logging.file.name`，不要混用。
4. 生产服务器必须提前创建日志目录，并确保运行应用的系统用户有写权限，例如：

   ```bash
   sudo install -d -o appuser -g appuser /var/log/demo-service
   ```

5. 如果使用 Spring Boot 2.1 或更早版本，应使用旧属性 `logging.path`，同时将 `<springProperty>` 的 `source` 改为 `logging.path`。

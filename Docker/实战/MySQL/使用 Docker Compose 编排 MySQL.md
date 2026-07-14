---
title: 使用 Docker Compose 编排 MySQL
aliases:
  - Docker Compose 运行 MySQL
  - MySQL Compose 配置
tags:
  - Docker
  - Docker/实战
  - Docker/MySQL
  - Docker/Compose
  - MySQL
created: 2026-07-15T00:15:50
updated: 2026-07-15T00:29:12
---

Docker Compose 将镜像、端口、环境变量、Secret、健康检查和数据卷集中声明在 `compose.yaml` 中。它特别适合项目本地开发：团队成员不必复制一长串 `docker run` 参数，启动和销毁环境的行为也更容易审查。

如果尚不理解单个参数的作用，先完成 [[使用 docker run 启动 MySQL]]。本文继续使用 `mysql:8.4.10`，并将密码从 Compose 文件中分离出来。

## 示例目录

```text
mysql-compose-demo/
├── compose.yaml
├── .gitignore
├── initdb/
│   └── 001-schema.sql
└── secrets/
    ├── mysql_app_password.txt
    └── mysql_root_password.txt
```

创建目录并进入：

```bash
mkdir -p mysql-compose-demo/initdb mysql-compose-demo/secrets
cd mysql-compose-demo
```

密码文件不得提交到 Git：

```gitignore
secrets/*.txt
!secrets/.gitkeep
.env
```

> [!warning] Secret 文件仍需保护
> 本地 Compose 的文件型 Secret 可以避免把密码直接写进 `compose.yaml`，但文件内容仍存在宿主机磁盘上。应限制文件权限，并由团队的安全配置或密码管理流程分发；不要把示例密码当作真实密码。

## 准备密码文件

下面命令会交互读取密码，不会把真实值直接写在命令参数中：

```bash
read -r -s MYSQL_ROOT_SECRET
printf '\n'
printf '%s' "$MYSQL_ROOT_SECRET" > secrets/mysql_root_password.txt
unset MYSQL_ROOT_SECRET

read -r -s MYSQL_APP_SECRET
printf '\n'
printf '%s' "$MYSQL_APP_SECRET" > secrets/mysql_app_password.txt
unset MYSQL_APP_SECRET

chmod 600 secrets/mysql_root_password.txt secrets/mysql_app_password.txt
```

在 Bash 中，`read -r -s` 会等待输入。部分 Zsh 环境的提示方式不同，但同样可以输入后回车。密码文件末尾不强制需要换行。

## 编写 `compose.yaml`

```yaml
services:
  db:
    image: mysql:8.4.10
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
      MYSQL_DATABASE: app_db
      MYSQL_USER: app_user
      MYSQL_PASSWORD_FILE: /run/secrets/mysql_app_password
    ports:
      - "127.0.0.1:3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./initdb:/docker-entrypoint-initdb.d:ro
    secrets:
      - mysql_root_password
      - mysql_app_password
    healthcheck:
      test:
        - CMD-SHELL
        - mysqladmin ping -h localhost -u root -p"$$(cat /run/secrets/mysql_root_password)"
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    stop_grace_period: 1m

volumes:
  mysql_data:

secrets:
  mysql_root_password:
    file: ./secrets/mysql_root_password.txt
  mysql_app_password:
    file: ./secrets/mysql_app_password.txt
```

### 为什么使用 `_FILE`

MySQL Official Image 支持将 `MYSQL_ROOT_PASSWORD`、`MYSQL_USER`、`MYSQL_PASSWORD` 等初始化变量改写为对应的 `_FILE` 形式。容器入口脚本会从 `/run/secrets/...` 文件读取值，因此 Compose 文件不必直接包含密码。

### 为什么健康检查中使用 `$$`

Compose 会处理 `$` 插值。`$$` 表示把一个字面量 `$` 传入容器，最终由容器内的 Shell 执行 `$(cat ...)`。如果误写成单个 `$`，Compose 可能在宿主机解析变量，产生空值或错误配置。

### 为什么端口绑定到 `127.0.0.1`

```yaml
ports:
  - "127.0.0.1:3306:3306"
```

这让 MySQL 只通过宿主机回环地址访问，适合本地开发。写成 `3306:3306` 通常会监听宿主机所有可用接口，可能让同一网络中的其他设备访问该端口。

如果只有同一 Compose 项目中的应用需要访问数据库，可以完全删除 `ports`；容器仍可通过 Compose 网络和服务名 `db` 连接 MySQL。

## 添加初始化脚本

在 `initdb/001-schema.sql` 中写入：

```sql
CREATE TABLE IF NOT EXISTS app_db.docker_compose_check (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    message VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO app_db.docker_compose_check (message)
VALUES ('initialized by docker-entrypoint-initdb.d');
```

脚本只会在数据目录为空的首次初始化阶段执行。更完整的文件类型、执行顺序和重新初始化规则见 [[MySQL 容器配置与初始化]]。

## 启动前验证配置

```bash
docker compose config --quiet
docker compose config --images
```

`docker compose config --quiet` 检查 Compose 模型是否有效。它不能验证 SQL 语法，也不能证明密码文件内容正确。

确认解析后的配置没有意外发布到所有接口：

```bash
docker compose config
```

不要把包含敏感插值的完整输出复制到工单或公开日志中。

## 启动并观察初始化

```bash
docker compose pull
docker compose up -d
docker compose ps
docker compose logs -f db
```

初始化完成后按 `Ctrl+C` 退出日志跟随，再检查健康状态：

```bash
docker compose ps
docker inspect \
  --format '{{json .State.Health}}' \
  "$(docker compose ps -q db)"
```

## 连接和验证

使用应用用户进入数据库：

```bash
docker compose exec db mysql -u app_user -p app_db
```

按提示输入应用用户密码，然后执行：

```sql
SELECT VERSION();
SELECT CURRENT_USER();
SELECT * FROM docker_compose_check;
```

## 常用 Compose 命令

| 命令 | 作用 | 是否影响数据卷 |
| --- | --- | --- |
| `docker compose up -d` | 创建或更新并后台启动服务 | 复用现有命名卷 |
| `docker compose ps` | 查看当前项目服务状态 | 否 |
| `docker compose logs -f db` | 跟随 MySQL 日志 | 否 |
| `docker compose exec db ...` | 在运行中的 MySQL 容器执行命令 | 取决于命令内容 |
| `docker compose stop` | 停止服务但保留容器 | 否 |
| `docker compose start` | 启动已停止服务 | 否 |
| `docker compose restart db` | 重启 MySQL 服务 | 否 |
| `docker compose down` | 删除容器和项目网络 | 默认保留命名卷 |
| `docker compose down -v` | 删除容器、网络和项目命名卷 | 是，具有破坏性 |

> [!danger] 谨慎执行 `down -v`
> `-v` 会删除 Compose 管理的命名卷，MySQL 数据也会随之删除。先按 [[MySQL 容器备份恢复与版本升级]] 完成备份，并在新数据库中实际验证恢复。

## 与应用服务一起编排

应用容器连接 MySQL 时，主机名是 Compose 服务名 `db`，端口是容器端口 `3306`：

```yaml
services:
  app:
    image: example/app:1.0.0
    environment:
      DB_HOST: db
      DB_PORT: "3306"
      DB_NAME: app_db
      DB_USER: app_user
      DB_PASSWORD_FILE: /run/secrets/mysql_app_password
    secrets:
      - mysql_app_password
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mysql:8.4.10
    # 其余配置与前文相同
```

短语法 `depends_on: [db]` 只保证创建和启动顺序，不等待数据库健康。长语法的 `condition: service_healthy` 会等待 MySQL 健康检查通过，但应用自身仍应支持连接重试：数据库可能在应用运行期间重启或短暂不可用。

更完整的地址、端口和连接串说明见 [[MySQL 容器网络与应用连接]]。

## 更新配置时发生什么

修改 `compose.yaml` 后再次执行：

```bash
docker compose config --quiet
docker compose up -d
```

Compose 会比较声明并按需重建容器。重建容器不会自动删除命名卷，因此 MySQL 通常继续使用原有数据目录。

以下修改需要分别判断：

- 修改端口、重启策略、挂载或健康检查：通常重建容器即可。
- 修改 `MYSQL_DATABASE`、`MYSQL_USER` 或密码文件：对已有数据卷不会重新初始化。
- 修改 MySQL 配置项：先确认新配置与现有数据、当前版本兼容。
- 修改镜像版本：属于数据库升级，应先备份和检查升级路径。

## 可选的开发环境简化方案

如果只是个人短期实验，可以使用 `.env` 插值减少文件数量，但不要提交真实值：

```yaml
services:
  db:
    image: mysql:8.4.10
    environment:
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD:?set MYSQL_ROOT_PASSWORD}"
      MYSQL_DATABASE: "${MYSQL_DATABASE:-app_db}"
      MYSQL_USER: "${MYSQL_USER:-app_user}"
      MYSQL_PASSWORD: "${MYSQL_PASSWORD:?set MYSQL_PASSWORD}"
```

`.env` 便于开发，但环境变量会出现在容器配置中。需要更清晰的敏感信息边界时使用前面的 Compose Secret 方案。

## 常见问题

### `docker compose up` 后应用仍连接失败

检查 MySQL 健康状态和日志，而不是只检查容器是否显示 `Up`：

```bash
docker compose ps
docker compose logs --tail 200 db
```

确认应用使用 `db:3306`，并配置连接重试。

### 修改初始化 SQL 后没有执行

初始化目录只在空数据目录首次启动时处理。开发环境确实允许丢弃全部数据时，才可以备份后执行 `docker compose down -v` 再重新启动；保留数据时应使用 Flyway、Liquibase、Goose 等数据库迁移机制。

### Compose 项目名变化后出现新卷

Compose 默认会将项目名加入卷的实际名称。更改目录名或 `--project-name` 可能创建另一组资源。先运行：

```bash
docker compose ls
docker volume ls
```

不要在没确认旧卷内容前删除它。

## 相关笔记

- [[使用 Docker 运行 MySQL 概览]]
- [[MySQL 容器配置与初始化]]
- [[MySQL 容器数据持久化]]
- [[MySQL 容器网络与应用连接]]
- [[MySQL 容器日常维护与故障排查]]

## 官方参考资料

- [Docker：Use containerized databases](https://docs.docker.com/guides/databases/)
- [Docker：Compose file services reference](https://docs.docker.com/reference/compose-file/services/)
- [Docker：Compose Secrets](https://docs.docker.com/reference/compose-file/secrets/)
- [Docker Hub：MySQL Official Image](https://hub.docker.com/_/mysql)

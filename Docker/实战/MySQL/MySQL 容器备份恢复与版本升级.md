---
title: MySQL 容器备份恢复与版本升级
aliases:
  - Docker MySQL 备份恢复
  - MySQL 容器版本升级
tags:
  - Docker
  - Docker/实战
  - Docker/MySQL
  - MySQL/备份恢复
  - MySQL/升级
created: 2026-07-15T00:15:50
updated: 2026-07-15T00:29:12
---

可靠备份的标准不是“生成了一个文件”，而是能够在独立数据库中恢复，并通过结构、行数和关键业务查询验证结果。MySQL 镜像升级也不是简单修改 `image:` 标签；必须先确认受支持的升级路径、数据兼容性和回滚条件。

本文以单机 Docker Compose 和逻辑备份为主。生产环境的大数据量、严格恢复时间目标、时间点恢复、复制拓扑和高可用场景，需要专门的备份产品与数据库运维方案。

## 备份前先明确目标

| 指标 | 要回答的问题 |
| --- | --- |
| RPO | 最多能够接受丢失多长时间的数据 |
| RTO | 故障后允许多久恢复服务 |
| 备份范围 | 单库、全部业务库、账户、存储过程、事件还是完整实例 |
| 保存位置 | 是否独立于 Docker 宿主机和原数据卷 |
| 加密与权限 | 谁可以读取备份，其中是否含个人或敏感数据 |
| 恢复验证 | 在哪里、多久一次、用哪些查询确认恢复成功 |

如果备份和原卷位于同一块磁盘，同一次磁盘故障可能同时摧毁两者。重要数据至少还应复制到独立、受控的存储位置。

## 创建逻辑备份

以下示例假设 [[使用 Docker Compose 编排 MySQL]] 中的 `db` 服务使用文件型 root Secret。

### 创建备份目录

```bash
mkdir -p backups
chmod 700 backups
```

### 备份单个业务数据库

```bash
backup_file="backups/app_db-$(date '+%Y%m%d-%H%M%S').sql"

docker compose exec -T db sh -c '
  export MYSQL_PWD="$(cat /run/secrets/mysql_root_password)"
  exec mysqldump \
    --user=root \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --default-character-set=utf8mb4 \
    app_db
' > "$backup_file"

test -s "$backup_file"
ls -lh "$backup_file"
```

参数含义：

| 参数 | 作用 |
| --- | --- |
| `-T` | 关闭 Compose 的伪终端分配，避免污染重定向输出 |
| `--single-transaction` | 对支持事务的表获取一致性视图，减少锁定 |
| `--routines` | 包含存储过程和函数 |
| `--triggers` | 包含触发器 |
| `--events` | 包含 Event Scheduler 事件 |
| `--default-character-set=utf8mb4` | 明确客户端导出字符集 |

`--single-transaction` 主要保证 InnoDB 等事务表的一致性；非事务表、导出期间的 DDL 变更以及超大数据集需要额外设计。生产备份应使用专用备份账户和受控凭据，不要长期依赖 root。

### 压缩备份

```bash
gzip "$backup_file"
ls -lh "${backup_file}.gz"
```

压缩成功后原 `.sql` 会被替换为 `.sql.gz`。保存前可以生成校验值：

```bash
shasum -a 256 "${backup_file}.gz" > "${backup_file}.gz.sha256"
```

校验值只能检测文件是否变化，不能证明 SQL 能够成功恢复。

## 恢复到独立数据库并验证

不要第一次恢复就覆盖原数据库。先创建独立恢复库：

```bash
docker compose exec db mysql -u root -p \
  -e 'CREATE DATABASE app_db_restore CHARACTER SET utf8mb4;'
```

该命令会交互提示 root 密码。如果使用 Secret 且不便交互，可以在受控环境中使用容器内 Secret：

```bash
docker compose exec -T db sh -c '
  export MYSQL_PWD="$(cat /run/secrets/mysql_root_password)"
  exec mysql --user=root \
    --execute="CREATE DATABASE IF NOT EXISTS app_db_restore CHARACTER SET utf8mb4"
'
```

恢复未压缩备份：

```bash
docker compose exec -T db sh -c '
  export MYSQL_PWD="$(cat /run/secrets/mysql_root_password)"
  exec mysql --user=root app_db_restore
' < backups/app_db-YYYYMMDD-HHMMSS.sql
```

恢复压缩备份：

```bash
gzip -dc backups/app_db-YYYYMMDD-HHMMSS.sql.gz | \
  docker compose exec -T db sh -c '
    export MYSQL_PWD="$(cat /run/secrets/mysql_root_password)"
    exec mysql --user=root app_db_restore
  '
```

### 验证恢复结果

```bash
docker compose exec db mysql -u root -p app_db_restore
```

```sql
SHOW TABLES;
SELECT COUNT(*) FROM <关键表>;
CHECK TABLE <关键表>;
```

还应比较：

- 关键表行数和业务汇总值。
- 存储过程、函数、触发器和事件是否存在。
- 字符集、排序规则和时区相关数据是否正确。
- 应用能否在只读或隔离环境连接恢复库并完成关键查询。

验证完成后再决定是否删除恢复库。

## 备份整个实例时的注意事项

`mysqldump --all-databases` 可以导出多个数据库，但系统账户、授权、系统表和不同版本之间的兼容性比单业务库更复杂。不要把“全实例 SQL 文件”当作可在任意 MySQL 版本直接恢复的通用镜像。

应记录：

- 导出时的完整 MySQL 版本。
- 镜像名称与摘要。
- 使用的参数和备份工具版本。
- 数据库字符集与排序规则。
- GTID、二进制日志和复制相关状态。
- 目标恢复版本与验证结论。

## 为什么不能只备份 Docker Volume

Docker 可以归档一个卷，但存储层复制不了解 MySQL 正在执行的事务。运行中直接打包 `/var/lib/mysql` 可能得到不一致的数据文件。

卷级备份只有在以下条件明确时才可作为方案的一部分：

- MySQL 已经完整停止并确认干净关闭；或
- 使用了支持 MySQL 一致性的快照与物理备份流程；
- 恢复环境使用兼容的文件系统、MySQL 版本和配置；
- 已实际完成恢复演练。

日常本地环境优先从逻辑备份开始。有关卷生命周期见 [[MySQL 容器数据持久化]]。

## 升级前检查

1. 记录当前版本和镜像。
2. 阅读目标版本的 Release Notes 和升级章节。
3. 确认官方支持从当前版本到目标版本的升级路径。
4. 使用 MySQL Shell Upgrade Checker 等官方工具检查兼容性。
5. 完成逻辑备份，并验证恢复。
6. 在独立环境复制真实数据规模和应用流量进行测试。
7. 检查字符集、排序规则、认证插件、SQL Mode、保留字和驱动兼容性。
8. 设计停机窗口、监控、验收查询和回滚条件。

查看当前版本：

```bash
docker compose exec db mysql -u app_user -p \
  -e 'SELECT VERSION(), @@version_comment;'

docker compose images
docker image inspect mysql:8.4.10 \
  --format '{{index .RepoDigests 0}}'
```

## 理解升级路径

MySQL 的 LTS、Bugfix 和 Innovation 系列具有明确升级路径。不能因为两个镜像都能挂载 `/var/lib/mysql`，就认为可以直接跨版本启动。

例如，MySQL 官方 8.4 文档明确说明：从 MySQL 5.7 到 8.4 不能跳过 8.0，应先升级到 8.0，再升级到 8.4。实际执行时必须以目标版本当前官方文档为准。

> [!danger] 不要用旧镜像直接“回滚”已升级的数据目录
> 新版本启动后可能升级数据字典或系统表。把同一个卷重新挂给旧版本通常不构成安全回滚。可靠回滚依赖升级前保留的独立备份、卷快照或经过验证的旧环境副本。

## 推荐的升级演练方式

对于学习和中小型本地环境，优先采用“新卷 + 逻辑恢复”演练：

1. 从旧实例生成并验证逻辑备份。
2. 使用目标镜像和一个全新命名卷启动独立 MySQL。
3. 将备份恢复到新实例。
4. 执行应用迁移、集成测试和关键查询。
5. 对比性能、字符集、权限和执行计划。
6. 只有验证完成后，才安排正式切换。

这种方式不会让目标版本直接改写唯一的数据卷，回退也更清晰。数据量很大或 RTO 很短时，应采用官方支持的物理备份、复制或克隆方案，而不是无验证地复用旧卷。

## Compose 中更新镜像

完成前述验证后，将精确标签更新为已验证目标版本：

```yaml
services:
  db:
    image: mysql:<已验证的目标版本>
```

执行前再次检查：

```bash
docker compose config --quiet
docker compose pull db
docker compose images
```

正式升级必须使用已批准的运行手册。不要在唯一实例上直接执行 `docker compose pull && docker compose up -d`，然后才开始阅读升级日志。

## 备份与恢复检查清单

- [ ] 备份文件不在原数据卷的唯一故障域中。
- [ ] 备份记录了来源 MySQL 版本和镜像。
- [ ] 备份包含所需的表、触发器、过程、函数和事件。
- [ ] 文件权限和加密符合数据敏感级别。
- [ ] 已恢复到独立数据库或独立实例。
- [ ] 已运行结构、行数和关键业务查询。
- [ ] 已记录恢复耗时，能够评估 RTO。
- [ ] 已验证应用驱动和迁移工具可以连接目标版本。
- [ ] 升级和回滚使用不同、可识别的数据副本。

## 常见问题

### 备份文件大小为 0

检查 `docker compose exec` 是否使用 `-T`、数据库名是否正确、命令是否在错误发生时仍被重定向，以及备份目录是否可写。查看退出码，不要只看文件是否存在。

### SQL 可以导入，但应用无法使用

检查用户授权、迁移版本、字符集、排序规则、时区、驱动兼容性和初始化数据。数据表存在不等于完整恢复成功。

### 修改标签后容器反复重启

停止继续重试，保留日志和原数据副本，检查升级路径、配置项和数据字典错误。不要用多个不同版本反复挂载唯一卷尝试“碰运气”。

## 相关笔记

- [[MySQL 容器数据持久化]]
- [[MySQL 容器配置与初始化]]
- [[使用 Docker Compose 编排 MySQL]]
- [[MySQL 容器日常维护与故障排查]]

## 官方参考资料

- [Docker Hub：MySQL Official Image 的备份与恢复示例](https://hub.docker.com/_/mysql)
- [MySQL 8.4：Backup and Recovery](https://dev.mysql.com/doc/refman/8.4/en/backup-and-recovery.html)
- [MySQL 8.4：Upgrade Paths](https://dev.mysql.com/doc/refman/8.4/en/upgrade-paths.html)
- [MySQL 8.4：Upgrade Best Practices](https://dev.mysql.com/doc/refman/8.4/en/upgrade-best-practices.html)
- [MySQL 8.4：Upgrading a Docker Installation](https://dev.mysql.com/doc/refman/8.4/en/docker-mysql-getting-started.html#docker-mysql-upgrading)

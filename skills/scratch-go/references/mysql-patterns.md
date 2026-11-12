# MySQL 模式

根据用户提供的 `mysql-patterns` 文档整理，原文来源标记为 `ECC`。用于 MySQL/MariaDB 表结构、索引、迁移、慢查询、锁等待、队列事务、连接池、复制及生产配置的开发与审查。

跨数据库规则以 [数据库规范](database-spec.md)为准，包括禁止数据库外键、由应用维护引用完整性及避免 N+1 查询。配置、依赖构造与资源生命周期遵守 [Go 工程模式](go-patterns.md)，错误转换遵守 [错误处理](error.md)。本文件的 SQL、连接池参数和服务器配置均为示例，不自动覆盖项目约定。

## 1. 引擎与版本

先确认引擎和精确版本，再应用特定语法或配置；MySQL 与 MariaDB 已存在多处差异。

```sql
SELECT VERSION();
SHOW VARIABLES LIKE 'version_comment';
```

- MySQL 8.0.19 起支持 upsert 行别名；8.0.20 起弃用 `ON DUPLICATE KEY UPDATE` 中的 `VALUES(col)`，优先使用行别名。
- MariaDB 使用 `VALUES(col)` 引用待插入值。混合部署仅在所有目标版本都支持时采用该写法，并记录 MySQL 的弃用风险。
- `SKIP LOCKED` 仅用于允许跳过已锁行的队列式工作，可能返回不一致视图，不用于账务或其他完整性敏感读取；先确认目标版本支持。
- JSON、生成列、约束、执行计划分析命令、复制命令及 binlog 参数也按目标引擎和版本确认，不套用另一引擎的手册。

## 2. 表结构与类型

```sql
CREATE TABLE orders (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    account_id BIGINT UNSIGNED NOT NULL,
    status VARCHAR(32) NOT NULL,
    total DECIMAL(15, 2) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME NULL,
    PRIMARY KEY (id),
    KEY idx_orders_account_status_created (account_id, status, created_at),
    KEY idx_orders_active (account_id, deleted_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

| 用途 | 优先选择 | 避免 |
| --- | --- | --- |
| 代理主键 | `BIGINT UNSIGNED AUTO_INCREMENT` | 未评估增长上限就采用较小整数类型 |
| UUID 查询键 | `BINARY(16)`，应用明确编码与转换规则 | 热表使用 `VARCHAR(36)` 主键 |
| 金额与精确数量 | 按范围和精度选择 `DECIMAL(p, s)` | `FLOAT`、`DOUBLE` |
| 用户文本 | `utf8mb4`，显式选择排序规则 | 依赖 `utf8` / `utf8mb3` 默认值 |
| 应用时间戳 | `DATETIME`，应用和数据库会话统一使用 UTC | 假设 `DATETIME` 保存时区元数据 |
| 软删除 | `deleted_at DATETIME NULL`，按查询设计索引 | 过滤已删除行却没有适配索引 |
| 经常扩展的状态 | 查询表或经目标版本确认有效的受约束 `VARCHAR` | 频繁修改 `ENUM` |

使用 `CURRENT_TIMESTAMP` 默认值时也要明确数据库会话时区。表间关联采用普通字段及必要索引，不添加数据库外键约束。

## 3. 索引与迁移

联合索引通常按等值条件在前、范围或排序字段在后设计，并通过实际执行计划确认：

```sql
SELECT id, total
FROM orders
WHERE account_id = ?
  AND status = 'pending'
  AND created_at >= ?
ORDER BY created_at DESC
LIMIT 50;

EXPLAIN
SELECT id, total
FROM orders
WHERE account_id = 123 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 50;
```

上述查询可评估使用 `(account_id, status, created_at)` 索引；建表已有该索引时不重复创建。

| 字段 | 需调查的信号 |
| --- | --- |
| `type` | 大表上的 `ALL` |
| `key` | 存在高选择性条件却为 `NULL` |
| `rows` | 交互请求的估算扫描行数过高 |
| `Extra` | `Using temporary`、`Using filesort` 或大范围 `Using where` |

这些信号用于定位问题，不单凭一个字段认定查询错误。索引会增加写入、迁移、备份和 buffer pool 成本，不盲目添加。

大表迁移前确认表大小、数据分布、DDL 算法、元数据锁、额外磁盘需求和复制延迟；在接近生产数据规模的环境演练。按引擎与版本确认在线 DDL 能力，明确中止阈值、应用兼容窗口及回滚方案，不假定增加索引或修改列不会阻塞。

## 4. 查询模式

### 4.1 Upsert

先确认冲突由业务需要的主键或唯一索引触发，例如 `user_settings` 的 `(user_id, setting_key)` 唯一索引。MariaDB 及确认兼容的混合部署可使用：

```sql
INSERT INTO user_settings (user_id, setting_key, setting_value)
VALUES (?, ?, ?)
ON DUPLICATE KEY UPDATE
    setting_value = VALUES(setting_value),
    updated_at = CURRENT_TIMESTAMP;
```

MySQL 支持行别名的版本使用：

```sql
INSERT INTO user_settings (user_id, setting_key, setting_value)
VALUES (?, ?, ?) AS new
ON DUPLICATE KEY UPDATE
    setting_value = new.setting_value,
    updated_at = CURRENT_TIMESTAMP;
```

### 4.2 游标分页

```sql
SELECT id, name, created_at
FROM products
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 50;

CREATE INDEX idx_products_created_id ON products (created_at, id);
```

排序和游标包含唯一的决胜字段，明确空值处理；通过 `EXPLAIN` 确认目标版本能有效利用该索引。大表避免深度 `OFFSET`，否则服务器需扫描并丢弃前面的行。

### 4.3 JSON 扩展数据

JSON 用于扩展数据；需要频繁关联、过滤、约束的归属、租户和生命周期字段保持关系型列。

```sql
CREATE TABLE events (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    payload JSON NOT NULL,
    event_type VARCHAR(64)
        GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(payload, '$.type'))) STORED,
    KEY idx_events_type (event_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

对高频查询路径评估生成列及其索引。确认目标引擎的 JSON 存储、校验和生成列表达式行为，不假设 MySQL 与 MariaDB 完全相同。

### 4.4 全文搜索

```sql
ALTER TABLE articles ADD FULLTEXT KEY ft_articles_title_body (title, body);

SELECT id, title, MATCH(title, body) AGAINST (? IN NATURAL LANGUAGE MODE) AS score
FROM articles
WHERE MATCH(title, body) AGAINST (? IN NATURAL LANGUAGE MODE)
ORDER BY score DESC
LIMIT 20;
```

需要拼写容错、复杂排名、跨表分面或超出内置能力的语言分析时，评估外部搜索服务。

## 5. 事务与锁

保持事务短小，各代码路径按一致顺序锁定行：

```sql
START TRANSACTION;

SELECT id, balance
FROM accounts
WHERE id IN (?, ?)
ORDER BY id
FOR UPDATE;

UPDATE accounts SET balance = balance - ? WHERE id = ?;
UPDATE accounts SET balance = balance + ? WHERE id = ?;

COMMIT;
```

这是锁顺序示例；实际转账还需验证账户存在、余额、金额及幂等约束。检查实际执行计划与锁行为，不能仅靠书写顺序推断锁定顺序。

- 外部 API 调用放在事务外，避免持锁等待网络。
- 为 `UPDATE`、`DELETE` 和锁定读的条件设计索引，避免扫描扩大锁范围。
- Go 使用 `BeginTx` 创建事务，事务内所有读写使用同一个 `*sql.Tx`；处理失败时回滚，并检查 `Commit` 的返回错误。
- 死锁后回滚整笔事务。确需重试时，按 Go 工程模式明确错误范围、幂等性、次数与退避，受上下文约束，不无条件自动重试。
- 提交结果未知时先确认幂等键或业务结果，不直接重放写入。
- 死锁发生后尽快采集 `SHOW ENGINE INNODB STATUS\G`，后续事件可能覆盖诊断。

队列领取示例：

```sql
START TRANSACTION;

SELECT id
FROM jobs
WHERE status = 'pending'
ORDER BY created_at, id
LIMIT 1
FOR UPDATE SKIP LOCKED;

UPDATE jobs
SET status = 'processing', started_at = CURRENT_TIMESTAMP
WHERE id = ?;

COMMIT;
```

无结果时结束事务，不执行更新；更新使用本事务选出的 ID。按领取条件评估 `(status, created_at, id)` 索引，实际工作在提交后执行，并根据业务定义失败恢复和重复执行规则。`SKIP LOCKED` 只适用于允许跳过锁定行的任务，不替代正常事务一致性。

## 6. Go 连接池

复用入口构造并注入的 `*sql.DB`，它本身就是并发安全的连接池，不按请求创建。参数从统一配置读取；以下片段假设 `db` 和已校验的连接池配置存在：

```go
db.SetMaxOpenConns(poolConfig.MaxOpenConns)
db.SetMaxIdleConns(poolConfig.MaxIdleConns)
db.SetConnMaxLifetime(poolConfig.ConnMaxLifetime)
db.SetConnMaxIdleTime(poolConfig.ConnMaxIdleTime)
```

- 根据全部应用实例、后台任务及管理连接核算服务器连接预算；例如单实例最大 10 个连接只是容量评估起点。
- `MaxIdleConns` 不超过 `MaxOpenConns`。关注 `DB.Stats()` 的使用中、空闲连接、等待次数和等待时长，避免因池过小或事务内再取连接而耗尽。
- 将连接生命周期及最大空闲时间配置在服务器或代理的断开阈值以内；例如 `wait_timeout = 300` 秒时，可评估 240 秒的回收设置，不将其视为所有部署的固定值。
- 驱动连接超时及读写超时按实际驱动能力配置；查询、执行、事务和健康检查使用带期限的上下文，通过 `QueryContext`、`ExecContext`、`BeginTx`、`PingContext` 传递。
- `sql.Open` 不代表连接已建立；是否以 `PingContext` 阻止启动遵守项目的依赖策略。驱动的失效连接检测能力需单独确认，不将 Python 的 `pool_pre_ping` 当作 Go 配置项。
- 查询结束及时关闭 `Rows` 并检查 `Rows.Err()`；服务排空后由依赖所有者关闭连接池。

## 7. 诊断

```sql
SHOW FULL PROCESSLIST;
SHOW ENGINE INNODB STATUS\G;
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';
```

在受控环境启用慢日志的示例：

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

先确认权限、日志输出位置、敏感数据及容量影响，记录原值和恢复方式；全局变量对现有连接的生效方式按目标版本确认。无索引查询日志可能产生大量记录，不直接作为所有生产环境的固定设置。

仅在允许实际执行查询时使用执行分析工具。MySQL 的 `EXPLAIN ANALYZE` 会执行查询，在生产规模数据上可能昂贵；MariaDB 的分析命令按对应版本确认，不直接照搬语法。

## 8. 复制与一致性

副本可能延迟。写后立即读取、结算、权限校验及幂等键检查等要求一致性的路径走主库，或采用经验证能满足一致性要求的读取策略。

```sql
-- Legacy terminology, common in existing fleets.
SHOW SLAVE STATUS\G;

-- Newer terminology where supported.
SHOW REPLICA STATUS\G;
```

按引擎与版本选择命令，同时检查 SQL 线程、IO 线程、复制错误及延迟；TCP 可连接不代表复制健康。

## 9. 安全

以下为授权语法示意，账户来源地址需按实际网络限制，密码来自平台 secret manager，不提交真实值：

```sql
CREATE USER 'app'@'%' IDENTIFIED BY 'use-a-secret-manager';
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'%';
ALTER USER 'app'@'%' REQUIRE SSL;

SELECT user, host
FROM mysql.user
WHERE user = '';

DROP USER IF EXISTS ''@'localhost';
DROP USER IF EXISTS ''@'%';
```

- 运行账户不授予 `ALL PRIVILEGES` 或全局 `*.*` 权限，与迁移和管理账户分离。
- 跨主机或网络连接要求 TLS，客户端配置证书及服务器身份校验。
- 使用账户管理语句，不直接修改 `mysql.user`；清理匿名账户前确认实际账户与影响。
- 性能调优前检查公网暴露、绑定地址和访问来源。

## 10. 服务器配置

专用数据库主机的评估示例；参数名及支持情况先核对目标引擎与版本：

```ini
[mysqld]
innodb_buffer_pool_size = 4G
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1

max_connections = 300
thread_cache_size = 50

wait_timeout = 300
interactive_timeout = 300
innodb_lock_wait_timeout = 10

slow_query_log = ON
long_query_time = 1
log_queries_not_using_indexes = ON

log_bin = mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800
```

这些值用于评审，不是通用预设。内存、连接数、日志保留及持久性参数根据负载、硬件、备份和恢复目标确定；MariaDB 的 binlog 保留配置及各版本弃用参数另行确认。

## 11. 反模式

| 反模式 | 风险 | 替代方式 |
| --- | --- | --- |
| 热路径 `SELECT *` | 过量传输、客户端依赖列结构 | 显式选择列 |
| 深度 `OFFSET` | 大量扫描和丢弃 | 游标分页 |
| 关联字段缺少索引 | 慢 JOIN、更新或删除锁范围扩大 | 按查询设计关联字段索引，不创建外键约束 |
| 长事务 | 锁等待、undo 历史膨胀 | 缩小事务单元 |
| 直接修改 `mysql.user` | 绕过账户管理机制 | 使用账户管理语句 |
| 应用账户拥有管理权限 | 故障影响面扩大 | 最小权限运行账户 |
| 连接池回收晚于服务器超时 | 失效连接 | 提前回收，确认驱动失效检测 |
| 写后立即读副本 | 读取过期状态 | 一致性敏感路径走主库 |

## 12. 审查与交付

数据库审查输出包括：

1. 引擎、精确版本及尚未验证的假设。
2. 优先级最高的正确性、锁、安全和迁移风险。
3. 可执行的 SQL 或 Go 修改方案，以及受影响的索引、事务与配置。
4. 验证计划与实际结果：`EXPLAIN`、迁移演练、锁与死锁检查、中止和回滚条件；不适用项说明原因。
5. 影响建议的 MySQL/MariaDB 语法差异。

工程命令与公共交付要求见 [SKILL.md](../SKILL.md)，不因文档中提供了管理 SQL 就自动执行生产变更。

## 13. 官方参考

- [MySQL INSERT 与行别名](https://dev.mysql.com/doc/refman/8.0/en/insert.html)：确认行别名支持的起始版本。
- [MySQL 8.0.20 变更](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-20.html)：确认 upsert 中 `VALUES(col)` 的弃用。
- [MariaDB Upsert](https://mariadb.com/docs/server/reference/sql-statements/data-manipulation/inserting-loading-data/insert-on-duplicate-key-update)：确认 MariaDB 的 `VALUES(col)` 用法。
- [Go 连接管理](https://go.dev/doc/database/manage-connections)：确认 `sql.DB` 连接池及生命周期参数的语义。

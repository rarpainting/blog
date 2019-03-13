# MariaDB

## 数据库语言

### DDL(Data Definition Language) -- 数据定义语言

操作对象和对象的属性(主键/索引等), 会导致锁表的 DDL Lock

- Create
- Drop
- Alter

### DML(Data Manipulation Language) -- 数据操作语言

Insert/Delete/Update

### DCL(Data Control Language) -- 数据控制语句

数据库对象的权限, 与 数据安全 相关

- Grant
- Revoke

### TCL(Transaction Control Language) 事务控制语言

- Savepoint 设置保存点
- Rollback
- set Transaction

## MariaDB SSL 开启

```sql
GRANT ALL PRIVILEGES ON *.* TO 'user'@'host' IDENTIFIED BY 'password' REQUIRE SSL;
```

在通过 SSL 认证下, 赋予某用户(user:password@host)所有权限(*.*)

## 配置文件

### General Log

```conf
general_log             = 1 # 最大文件数?
```

### Slow Query Log

```conf
slow_query_log # 开启慢查询日志
long_query_time     = 2 # 限定查询时间 2s , 界限这么宽 ?
log-queries-not-using-indexes
```

### Binlog

```conf
sync_binlog = 0 # 事务提交后, 仅把 binlog_cache 中的数据写入 binlog 文件中, 但不执行 fsync 同步磁盘
```

## SQL 语句

### SQL 语句分析

```sql
EXPLAIN SELECT * FROM table;
```

获得该此 sql 语句的执行信息:
- `id`: (该连接的?) SQL 执行顺序
- `select_type`: 子查询中每个 select 子句的类型
  - `SIMPLE`: (简单 SELECT, 不使用 UNION 或子查询等)
  - `PRIMARY`: (查询中若包含任何复杂的子部分, 最外层的 select 被标记为 PRIMARY)
  - `UNION`: (UNION 中的第二个或后面的 SELECT 语句)
  - `DEPENDENT UNION`: (UNION 中的第二个或后面的 SELECT 语句, 取决于外面的查询)
  - `UNION RESULT`: (UNION 的结果)
  - `SUBQUERY`: (子查询中的第一个 SELECT)
  - `DEPENDENT SUBQUERY`: (子查询中的第一个 SELECT, 取决于外面的查询)
  - `DERIVED`: (派生表的 SELECT, FROM 子句的子查询)
  - `UNCACHEABLE SUBQUERY`: (一个子查询的结果不能被缓存, 必须重新评估外链接的第一行)
- `table`: 这一行的数据是属于那一个 table 的, 如果是临时表也会有临时表的表名称
- `type`: mysql 在表中找到所需行的方式, 又称 **访问类型**
  - `ALL`: Full Table Scan, MySQL 将遍历全表以找到匹配的行
  - `index`: Full Index Scan，index 与 ALL 区别为 index 类型只遍历索引树
  - `range`: 只检索给定范围的行, 使用一个索引来选择行
  - `ref`: 表示上述表的连接匹配条件, 即哪些列或常量被用于查找索引列上的值
  - `eq_ref`: 类似 ref, 区别就在使用的索引是唯一索引, 对于每个索引键值, 表中只有一条记录匹配, 简单来说, 就是多表连接中使用 primary key 或者 unique key 作为关联条件
  - `const`、`system`: 当 MySQL 对查询某部分进行优化, 并转换为一个常量时, 使用这些类型访问. 如将主键置于 where 列表中, MySQL 就能将该查询转换为一个常量; `system` 是 `const` 类型的特例, 当查询的表只有一行的情况下, 使用 `system`
  - `NULL`: MySQL 在优化过程中分解语句, 执行时甚至不用访问表或索引, 例如从一个索引列里选取最小值可以通过单独索引查找完成
- `possible_keys`: mysql 能使用哪个索引在表中找到记录, 查询涉及到的字段上若存在索引, 则该索引将被列出, 但 不一定 被当前查询使用
- `key`: mysql 在当前查询中实际使用的索引
- `key_len`: **索引** 中使用的字节数, 可通过该列计算查询中使用的索引的长度
  - 显示的值为索引字段的 **最大可能长度** , 并非实际使用长度, 即 `key_len` 是根据表定义计算而得, 不是通过表内检索出的
- `ref`: 表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值
- `rows`: 表的行数 -- 根据表统计信息及索引选用情况, 估算(?)的找到所需的记录所需要读取的行数
- `extra`: 用于表示 mysql 解决查询的详细信息
  - `using where`: 列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的, 这发生在对表的全部的请求列都是同一个索引的部分的时候, 表示 mysql 服务器将在存储引擎检索行后再进行过滤
  - `using temporary`: 表示 MySQL 需要使用临时表来存储结果集, 常见于排序和分组查询
  - `using filesort`: MySQL 中无法利用索引完成的排序操作称为 **文件排序**
  - `using join buffer`: 改值强调了在获取连接条件时没有使用索引, 并且需要连接缓冲区来存储中间结果. 如果出现了这个值, 那应该注意, 根据查询的具体情况可能需要添加索引来改进能
  - `impossible where`: 这个值强调了 where 语句会导致没有符合条件的行
  - `select tables optimized away`: 这个值意味着仅通过使用索引, 优化器可能仅从聚合函数结果中返回一行
  
总结;
- `EXPLAIN` 不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况
- `EXPLAIN` 不考虑各种 Cache
- `EXPLAIN` 不能显示 MySQL 在执行查询时所作的优化工作
- 部分统计信息是估算的, 并非精确值
- `EXPALIN` 只能解释 SELECT 操作，其他操作要重写为 SELECT 后查看执行计划

### 获取有关数据库和表的信息

- `SELECT DATABASE();` -- 当前数据库
- `SHOW DATABASES;` -- 所有数据
- `SHOW TABLES;` -- 当前数据库的表
- `DESCRIBE table;` -- 获得一个表列信息 === `SHOW COLUMNS FROM table;`

- `SELECT column(s) FROM table ORDER BY RULE DESC;` -- *倒序* 排序所获得的行
- `SELECT column(s) FROM table1 {LEFT|RIGHT|INNER} JOIN table2 ON table1.columns { < | > | = | ... } table2.columns;` -- 返回{包括左表的所有记录和右表符合联结关系的字段 | 包括右表的所有记录和左表符合联结关系的字段 | 两个表中符合联结关系的字段}
- `SELECT column1(s) FROM table WHERE column2(s) IS NULL;` -- 从 table 中找出能同时满足 column2(s) 是 NULL 字段的 column1(s) 行

- `SQL SECURITY { DEFINER | INVOKER }`: 指明谁有权限执行添加该信息的(view/trigger/function/procedure/event); DEFINER 表示按定义者拥有的权限执行, INVOKER 表示按调用者的权限来执行, 默认下 系统指定为 **DEFINER**

- `PURGE {BINARY|MASTER} LOGS {TO 'log_name' | BEFORE datatime_expr}`
  - 删除指定日志文件名或者日期之前的 **二进制** 文件
  - **注意**: 如果从站处于活跃状态且 **未从** 将删除的文件中读取内容, 则文件删除后, 从站将无法继续复制
  - `RESET MASTER` -- 删除所有日志文件; `FLUSH LOGS` -- 写入记录到磁盘
  - 所记录的二进制文件将在 **expire_logs_days/binlog_expire_logs_seconds** 后自动删除; 0 -- 关闭自动删除

- `RESET reset_option[, reset_option]` -- 用于清除各种服务器操作的状态, **重置** 服务器状态到初始状态

- `SHOW ENGINE innodb status;` -- 最新一次记录的死锁日志(记录的是 **等待锁的 sql 语句** 记录, 而不是 完整事务 的 sql 记录)
  - 在死锁检查里面:
    - lock_mode X waiting -- next-key lock
    - lock_mode X locks rec but not gap -- 行锁
    - locks gap before rec -- 间隙锁

- `SHOW BINARY LOGS;` -- 当前 Master 未过期的 binlog

- `SHOW MASTER STATUS;` -- MySQL Mater 的 binlog 状态(最新位置-position 等)

- `SHOW BINLOG EVENTS;` -- MySQL Master 的 binlog 简单记录

generated column(>=5.7):
- 实现列数据的关联更新
- `ALTER TABLE table ADD COLUMN column TYPE GENERATED ALWAYS AS (such as [id%100])`

## MYSQL 数据目录

- performace_schema: 提供运行时检查服务器内部执行的信息

- sys: sys 模式, 提供了用于解释性能模式信息的一组对象

- mysql: 系统数据库
  - `columns_stats`: 引擎无关的表统计信息, 基于直方图的统计功能
  - `columns_priv`: **列** 级权限相关的信息 -- <a href="#permission">权限</a>
  - `db`: 与 **数据库** 权限相关的信息
  - `event`
  - `func`
  - `general_log` -- 常规日志(general log)内容
  - `gitd_slave_pos` -- 用于从属服务器的复制
  - `help_category` -- HELP 命令使用的表, 用于地理信息相关, 其他的 help 信息相关的还有 help_keyword, help_relation, help_topic
  - `host` -- 主机及其权限的表, 将在 MariaDB_10.4 版本去除
  - `index_stats` -- 存储与引擎无关的表统计信息, 索引相关(?)
  - `innodb_index_stats` -- 与 InnoDB 持久统计相关的信息, 包含每个索引的多个行
  - `innodb_table_stats` -- 与 InnoDB 持久统计相关的信息, 每个表一行
  - `ndb_binlog_index` -- 兼容 MySQL Cluster , 将在 MariaDB_10.4 版本去除
  - `plugin` -- 使用 INSTALL_SONAME, INSTALL_PLUGIN 或者 mysql_plugin 等插件列表; 不包含有关内置插件或使用 --plugin-load 选项加载的插件的信息
  - `proc` -- 包含存储过程和存储函数的信息, 该信息与 INFORMATION SCHEMA.ROUTINES 表中存储的信息相似
  - `procs_priv` -- 包含存储过程和存储函数 **权限** 的信息, 同时 INFORMATION SCHEMA.ROUTINES 派生自 mysql.procs_priv
  - `roles_mapping` -- 包含与角色(roles)相关的信息
  - `servers -- 包含有关 Spinder/FEDERATED/FederatedX 存储引擎(Storage Engine)使用的服务器信息
  - `slow_log` -- 慢日志(slow log)内容
  - `tables_priv` -- 表级权限, 派生表 -- INFORMATION_SCHEMA.TABLE_PRIVILEGES 
  - `table_stats` -- 表统计信息相关
  - `time_zone` -- 与时区相关的信息
  - `time_zone_leap_second` -- 与时区相关的信息
  - `time_zone_name` -- 与时区相关的信息
  - `time_zone_transition` -- 与时区相关的信息
  - `time_zone_transition_type` -- 与时区相关的信息
  - `transaction_registry` -- **事务**精确版本控制
  - `user` -- 访问服务器上的用户的权限, 及其全局权限的信息

<h2 id="permission">MYSQL/MARIADB 权限</h2>

- Global Privileges
  - CREATE USER
  - FILE
  - GRANT OPTION -- 授予(当前用户拥有的)全局权限
  - PROCESS -- 运行 SHOW PROCESSLIST 或者 显示与活动进程相关的信息
  - RELOAD -- 运行 FLUSH , 相当于 mysqladmin 命令
  - REPLICATION CLIENT
  - REPLICATION SLAVE -- 从服务器使用的账号登录在主服务器上时需要, 用于在主服务器上进行更新
  - SHOW DATABASES
  - SHUTDOWN
  - SUPER -- super-user
    - 语句: CREATE MASTER TO, KILL, PURGE LOGS, SET global system variables, mysqladmin debug 命令
    - 同时该权限允许用户: 写入数据即使设置了 **read_only**, 启用/禁用日志记录, 启用/禁用从站上的复制, 为支持该子句的语句指定 DEFINER , 到达 MAX_CONNECTIONS 后启动一次连接
    - 如果服务器已经为 初始连接 指定了一个 *mysqld option*, 那么该命令(?)将不会在 SUPER 连接时执行

- Database Privileges
  - CREATE
  - CREATE ROUTINE -- 通过 CREATE PROCEDURE/CREATE FUNCTION 创建存储过程
  - CREATE TEMPORARY TABLES -- 通过 CREATE TEMPORARY TABLE 创建临时表
  - DROP
  - EVENT
  - GRANT OPTION -- 授予(当前用户拥有的)数据库权限
  - LOCK TABLES -- 使用 LOCK TABLES 语句获得显式锁; 需要同时拥有 SELECT 目标表的权限, 用于锁定该表
  
- Table Privileges
  - ALTER
  - CREATE
  - CREATE VIEW
  - DELETE
  - DELETE HISTORY
  - DROP
  - GRANT OPTION
  - INDEX
  - INSERT
  - REFERENCES
  - SELECT
  - SHOW VIEW
  - TRIGGER
  - UPDATE -- 更新表中的现有行, 需要同时具备 SELECT 该表的权限 或者 适用于 WHERE 子句的列
  
- Columns Privileges
  - INSERT
  - REFERENCES
  - SELECT
  - UPDATE
  
- Function Privileges
  - ALTER ROUTINE -- 通过 ALTER FUNCTION 更改存储函数的特征
  - EXECUTE -- 执行存储函数
  - GRANT OPTION
  
- Procedure Privileges
  - ALTER ROUTINE
  - EXECUTE
  - GRANT OPTION

- Proxy Privileges
  - PROXY -- 允许一个用户成为另一个用户的代理
  
- 账户的资源限制
  - `MAX_QUERIES_PER_HOUR`
  - `MAX_UPDATE_PER_HOUR`
  - `MAX_CONNECTIONS_PER_HOUR`
  - `MAX_USER_CONNECTIONS`
  - `MAX_STATEMENT_TIME`
  
- 账户的 SSL/TLS 选项
  - REQUIRE 只使用一次, 通过 AND 分隔各个选项
  - REQUIRE NONE -- 不需要 TLS
  - REQUIRE SSL
  - REQUIRE X509 -- 启用 TLS, 有效的 X509 证书
  - REQUIRE ISSUER 'issuer' -- TLS, 有效的 X509 证书; 同时必须通过证书颁发机构 'issuer'
  - REQUIRE SUBJECT 'subject' -- TLS, 有效的 X509 证书; 同时证书的主题必须是 'subject'
  - REQUIRE CIPHER 'cipher' -- -- TLS, 有效的 X509 证书; 同时用于连接的加密必须使用字符串指定的 'cipher'
  

## MariaDB 日志

### Error Log

错误日志

### General Log

mysql 接收到的每一个命令, 无论成功与否都记录下来
需要相当高的系统开销, 一般用于 **调试** 阶段

### Slow Log

长时间的 **SQL 查询** 记录

- `slow_query_log = 1`
- 慢查询设置(`log_query_time`)时间级别是 微秒级
- 写入文件到 `slow_query_log_file`, 写入表到 mysql.slow_log
- `log_queries_not_using_indexes` -- 执行 不使用索引或者不限制行数 的查询都会被记录
- `log_slow_admin_statements` -- 设置存储缓慢的管理查询, 包括 ALTER TABLE / ANALYZE TABLE / CHECK TABLE / CREATE INDEX / DROP INDEX / OPTIMIZE TABLE / REPAIR TABLE
- `log_slow_disabled_statements` -- 禁止某些类型语句的记录(admin / call / slave / sp(存储过程))
- `min_examined_row_limit` -- 如果查询结果的行数超过该值, 则将该查询记录到 `slow_query_log`

#### 统计扩展(Extended Statistics)

- `log_slow_rate_limit` -- 限制实际写入慢查询的 行数/比例(?) ...
- `log_slow_verbosity` -- { (Empty) | query_plan | innodb | explain }
- `log_slow_filter` -- 通过 ',' 分割; 过滤器: 如果需要记录的查询同时 **匹配过滤器中的某类型** , 则将其记录到慢日志中:
  - `admin` / `filesort` / `filesort_on_disk` / `filesort_priority_queue`(>= MariaDB 10.3.1) / `full_join` / `full_scan` / `query_cache` / `query_cache_miss` / `tmp_table` / `tmp_table_on_disk`

### Bin Log

包含数据库 **所有** 更改(CREATE ALTER INSERT UPDATE DELETE)的记录(即使该记录对数据不影响)
是 **复制** 所必备的, 可用于备份后还原数据

- `sql_log_bin` = 1 -- 开启 二进制日志
- `max_binlog_size` -- 单日志文件限制尺寸
- 写入文件到 `log_bin_basename`, 通过 SHOW BINARY LOGS; 查询二进制日志文件列表

完整 Binlog 的标志:
- statement 格式: 以 COMMIT 结尾
- row 格式: 以 XID event 结尾(redo log 也有相应的 XID event)

#### 复制时安全清除二进制文件

- 通过 `SHOW BINARY LOGS` 获取主服务器上的二进制日志文件列表
- 到从服务器上通过 `SHOW SLAVE STATUS` 检查每个从属服务器正在读取的二进制日志文件
- **找到从站正在读取的 最早 的日志文件, 删除在这之前的所有日志文件**
- 如果需要的话, 在删除日志文件之前, 备份日志文件

#### 选择性的记录

`binlog_do_db`/`binlog_ignore_db` -- 特意记录的数据库和需要忽略的数据库

- 不接受 ',' 分割的数据库, 多个设置则需要多次使用

#### 二进制日志格式

`binlog_format` = { STATEMENT | ROW | MIXED } -- 二进制日志格式

##### STATEMENT

记录所有对表或结构进行更改的 SQL 语句

##### ROW

记录影响各个表行的事件

`binlog_row_image`: { MINIMAL | FULL }:
- 默认为 FULL, 记录每一行的变更
- MINIMAL : 只记录影响后的行 (注: 腾讯云)


##### MIXED

默认基于 **语句** 复制, 当确定语句对于 基于语句的复制可能不安全时, 将启用基于 **行** 的格式:

- INSERT-DELEYED
- 更新具有 **ATUO_INCREMENT 列** 的表并启用 **触发器(trigger)** 或者 **存储函数(stored function)**
- LOAD_FILE()
- `ROW_COUNT()` / `FOUND_ROWS()`
- USER() / CURRENT_USER()
- UUID()
- 其中有某些表是 mysql database 的 log table
- 语句使用了 **系统变量**; 如果与 会话范围(session scope) 一起用, 则可能会有例外
- 调用自定义的函数
- [MIXED](https://mariadb.com/kb/en/library/binary-log-formats/)

**注意**:

- 直接编辑 mysql 数据库, 则根据 binlog_format 执行日志记录
- 间接编辑 mysql 数据库, 则统一通过 **语句日志** 记录

### Redo Log: 重做日志

脏页记录, 确保事务的持久性

### Undo Log: 回滚日志

保存事务前的 mysql-MVCC 版本(视图)

### 附议

#### 崩溃恢复

- 如果 redo log 里面的事务时完整的, 即已经有了 commit 标识, 则直接提交
- 如果 redo log 里面的事务只有完整的 prepare 标识, 则判断对应的事务 binlog 是否存在并完整
  - 事务 binlog 完整, 则提交事务
  - 否则 回滚事务

#### redo log 与 binlog 关联

依靠 XID 关联, 崩溃恢复时, 按顺序扫描 redo log 并恢复

## 复制

复制的目地
- 可扩展性
- 备份帮助
- 分发数据

### 设置复制

```yaml
[master]
log-bin
server-id = 1 # 无论 主服务器还是从服务器 都需要设置 唯一 的 server-id
log-basename = master1 # mariadb 独有
```

- skip-networking = 1 -- 仅允许于本地通信; 禁止远程连接
- bind-address = x.x.x.x

#### 锁定 Master 表

- `FLUSH TABLES WITH READ LOCK` -- 并且获得读锁
- `SHOW MASTER STATUS` -- 获得 binlog 文件名
- 获得 文件(File)和位置(Position)
- 开始复制
- `UNLOCK TABLES`

#### 开启 Slave 服务器

### Relay Log

- 文件格式与 BinLog 相同, 因此能用 mysqlbinlog* 导出

## 线程池

- thread_handling = pool-of-threads # 开启线程池
- thread_handling = one-thread-per-connection # 一连接一线程

### 线程池功能


在 MariaDB 5.5 后, 线程池目标:

- 使线程池动态增长和缩小
- 最大限度的减少维护线程池本身的开销
- 充分利用底层操作系统功能
- 限制线程池使用的资源

### 底层实现及差别

- Windows: 使用 Windows 本地线程池
  - `thread_pool_min_threads`
- 类 Unix: 通用实现
  - `thread_pool_size`
  
## MySQL 同步策略

###  async

### semi-sync: 只要有一个 slave 响应成功, master 就 commit

配置

在主库安装 semisync_master 插件：

```shell
mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so'; //linux
mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.dll'; //windows
```

在备库安装 semisync_slave 插件:

```shell
mysql> INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so'; //linux
mysql> INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';//windows
```

```conf
rpl_semi_sync_master_enabled=1
rpl_semi_sync_master_timeout=1000
```

### sync

## <<极客时间: MySQL 实战>>

![MySQL 的逻辑架构图](0d2070e8f84c4801adbfa03bda1f98d9.png)

### 一次查询运行流程

#### 连接器

- Mysql 在执行过程中临时使用的内存是管理在 **连接对象** 里面的, 该资源只有在断开才释放; 此时建议在执行 较大的操作/长时间的操作 后执行 **mysql_reset_connection** 重建连接, 该过程不执行重连和权限验证, 仅将 [连接恢复](https://mariadb.com/kb/en/library/mysql_reset_connection/)

#### 查询缓存
  
- 查询缓存会因为一次 插入/更新/删除 操作而失效, 因此一般建议关闭 **查询缓存(query_cache_type=DEMAND)**
- 一般建议在需要使用查询缓存的地方设置 **SQL_CACHE**, 例如: 
  - `SELECT SQL_CACHE * FROM T WHERE ID=10;`
- **注**: MySQL 8.0 将删除查询缓存功能; Mariadb 呢?

#### 分析器

- 词法分析
- 语法分析
  - 其中包括对字段是否正确的判断 -- 依据: 字段不同于表数据, 结构事先已经缓存在 server 中
- 语义解析

#### 优化器

MySQL 对 ast 执行树的优化

`optimizer_switch`: 优化器策略
// TODO:
- index_merge=on
- index_merge_union=on
- index_merge_sort_union=on
- index_merge_intersection=on
- index_merge_sort_intersection=off
- engine_condition_pushdown=off
- index_condition_pushdown=on
- derived_merge=on
- derived_with_keys=on
- firstmatch=on
- loosescan=on
- materialization=on
- in_to_exists=on
- semijoin=on
- partial_match_rowid_merge=on
- partial_match_table_scan=on
- subquery_cache=on
- mrr=off
- mrr_cost_based=off
- mrr_sort_keys=off
- outer_join_with_cache=on
- semijoin_with_cache=on
- join_cache_incremental=on
- join_cache_hashed=on
- join_cache_bka=on
- optimize_join_buffer_size=off
- table_elimination=on
- extended_keys=on
- exists_to_in=on
- orderby_uses_equalities=on
- condition_pushdown_for_derived=on
- split_materialized=on

#### 执行器

以 `select * from where ID=10` 为例

1. 权限验证
  - 命中查询缓存: 在查询缓存 **返回结果** 时, 做权限验证
  - 查询: 在优化器之前调用 `precheck` 验证权限
  - **注**: 由于 MySQL 的验证存在 存储过程 等需要在运行中才能确定的权限申请, 因此只能在执行其中
2. 查询
  - 调用 InnoDB 引擎接口取第一行, 判断 `ID` 是否符合, 是则保存在结果集中, 否则跳过该条

慢查询日志 -- `rows_examined`
- 引擎扫描行数和 `rows_examined` 并不完全相同 ??

### 一次更新运行流程(WAL: Write-Ahead Logging)

![redo log](b075250cad8d9f6c791a52b6a600f69c.jpg)

- `checkpoint` 是擦除的位置
- `write pos` 是当前记录的位置

如果 `checkpoint` - `write pos` 过小, 则表示 redo log 将满, 此时 server 会选择 取消 wirte(?)

redo log 与 binlog 差异:
- redo log 是 InnoDB 特有的; binlog 是 MySQL 的 Server 层实现的, 对所有引擎开放
- redo log 是物理日志; binlog 是逻辑日志
  - redo log 记录了 这个页 "做了什么改动"
  - binlog 有两种模式: `statement` 记录 `sql` 语句; `row` 格式记录 **更新前 和 更新后** 两条记录
- `redo log` 是循环写的, 空间固定可能用完; `binlog` 可以追加, 即 写到在一定大小后, 会切换到下一份文件, 而不是覆盖以前的日志 -- 因此: `binlog` 的 **归档** 功能确定了目前它还是无法被 `redo log` 替代

同时 redo log 和 binlog 通过 "事务 ID" 对应

![update 语句执行流程](2e5bff4910ec189fe1ee6e2ecc7b4bbe.png)

WAL 得益于两方面:
- redo log 和 binlog 都是顺序写, 磁盘的顺序写比随机写速度快
- 组提交机制, 可以大幅度降低磁盘的 IOPS 消耗

**WAL 机制只保证写完了 redo log 和 内存, 不保证写数据到磁盘**

#### update 的实际执行流程:

- 先获取 `ID=2` 这一行
- 执行器拿到引擎的行数据, 在值之上 +1, 再通过引擎接口写入新数据
- 引擎将该数据更新到 内存 中, 同时记录到 `redo log` 中, 此时 `redo log` 处于 `prepare` 状态; 状态设置后告知 执行器 可以随时提交事务
- 执行器生成这个操作的 `binlog`, 并把 `binlog` 写入到 磁盘
- 执行器 调用 引擎 的提交事务接口, 引擎把 `redo log` 改成 `提交(commit)` 状态, 更新完成

#### 两阶段提交与事务

用于保证提交都是成功的

`redo log`:
- `innodb_flush_log_at_trx_commit`: 每次事务的 `redo log` 都直接持久化到磁盘
- `sync_binlog`: 每次事务的 `binlog` 都直接持久化到磁盘

避免长事务对业务的影响

从应用端看:
- 使用 `set autocommit=1` 确保自动提交事务
- 确认是否有不必要的只读事务
- 通过 `SET MAX_EXECUTION_TIME` 控制每个语句执行的最长时间
从数据库端看:
- 监控 `information_schema.Innodb_trx` 表, 设置长事务阈值
- Percona 的 `pt-kill` 工具
- 功能测试阶段开启 general_log, 分析日志行为
- 在 **初始化 DB** 时设置 `innodb_undo_tablespaces`(undo log 表个数) 为 2

### 索引

- 主键
  - 主键索引的叶子节点存的是 **整行数据**
  - 非主键索引的叶子节点存的是 **主键的值**, 即查询非主键索引比查询主键索引多了一次查询 B 树的次数(回表)
- 尽量自增主键
  - B 树为了维护索引 **有序性**, 在插入新值的时候需要做必要的维护(页分裂/页合并), 影响性能以及空间利用率
  - 自增主键长度小, 普通索引的叶子节点就越小, 普通索引占用的空间也就越小
- 使用 **业务字段(如身份证)** 作为主键的场景
  - 只有一个索引
  - 该索引必须是唯一索引
  
回到主键索引树搜索的过程，称为回表
 
#### 索引优化

1. 覆盖索引
  - 如果查询条件使用的是普通索引(或是联合索引的最左原则字段), 查询结果是 联合索引的字段 或是 主键, **不用回表操作**, 直接返回结果, 减少 IO 磁盘读写读取正行数据
2. 最左前缀
  - 联合索引的最左 N 个字段, 也可以是字符串索引的最左 M 个字符
3. 联合索引
  - 根据创建联合索引的顺序, 以最左原则进行 where 检索, 比如 (age, name) 以 `age=1` 或 `age=1 and name='张三'` (同时 `name='张三' and age=1` 由于 server 会优化顺序, 与前者结果相同) 可以使用索引; 单以 name=‘张三’ 不会使用索引, 考虑到存储空间的问题, 还请根据业务需求, 将查找频繁的数据进行 **靠左** 创建索引
4. 索引下推
  - like 'hello%’and age > 10 检索, MySQL 5.6 版本之前, 会对匹配的数据进行回表查询; 5.6 版本后, 会先过滤掉 age<10 的数据, 再进行回表查询, 减少回表率, 提升检索速度

#### 字符串索引

- 直接建立完整索引, 占空间
- 建立 前缀索引 重点关注 **区分度**, 要求在满足区分度的同时利用上 前缀索引 带来的优化
- 前缀索引 不能同时利用上 覆盖索引 的优化(因为前缀索引必须回表, 以确定查询是否正确)
- 倒序存储, 再建立前缀索引, 以绕过字符串本身前缀的区分度不够的问题
- 创建 hash 字段索引, 查询性能稳定, 有额外的存储和计算消耗

特殊方案:
- (前缀)区分度差的数据:
  - 倒序存储: 存储时数据时是倒序, 查询时通过 `reverse` 摆正或者应用程序处理
  - Hash 字段: 注意 散列冲突
- 两者的相同点:
  - 都不支持范围查询
- 两者的差异:
  - 倒序依然是同样的索引, 但是 Hash 需要添加一个比对用的字段
  - 从 CPU 计算复杂度: 一个是 `reverse`, 一个是 `hashEncoding`
  - 从查询效率看: 比起使用的依然是 前缀索引 方式的倒序索引, Hash 一般更为稳定

### 优化器

- 优化器只有在大量(万以上)查询中, 才会游离在使用索引与否中(?)
- 优化器会同时 估计 使用普通索引后 **回表** 的次数

#### MySQL 优化指令

```sql
ANALYZE TABLE table;
```
> 优化索引的散列程度(cardinality)

```sql
CHECKSUM TABLE table;
```
> 错误校验

```sql
OPTIMIZE TABLE table;
```
> 碎片整理, 等价于 `recreate` + `analyze`

用于碎片整理的还有:

```sql
ALTER TABLE table ENGINE=innodb,ALGORITHM=inplace; # 默认行为, 避免了原表的复制, 仅是对表重组
ALTER TABLE table ENGINE=innodb,ALGORITHM=copy; # 不仅复制了原表, 还进行了表的重组, 一般用于修改主键(primary key)时使用
```
> 以上两者执行中都需要创建 tmp_table

```sql
CHECK TABLE table;
```
> 数据库错误检查

部分可选项:
- `UPGRADE`: 测试更早版本的表是否与当前版本兼容
- `QUICK`: 速度最快的选项, 检查时, 不会检查 链接(`link`) 的正确与否
- `FAST`: 只检查表时候正确关闭
- `CHANGED`: 只检查上次检查时间之后更新的数据
- `MEDIUM`: 默认选项, 会检查 索引文件 和 数据文件 之间的链接正确性
- `EXTENDED`: 最慢的选项, 会执行全面的检查

```sql
RAPAIR TABLE table;
```
> 修复表(只对 MyISAM 和 ARCHIVE 有效), 及其索引

部分可选项:
- `QUICK`: 最快的选项, 只修复索引树
- `EXTENDED`: 最慢的选项, 需要逐行重建索引
- `USE_FRM`: 只有当 MYI 文件丢失时才使用这个选项, 全面重建整个索引

> 注意:
> 在重建表的过程中, 页会按照 90% 的比例重新整理页数据(10% 留给 UPDATE 使用)
> 因此如果该表在重建前利用率已经超过 90% , 重建后反而会导致文件更大

### 脏页及其控制处理

#### `innodb_io_capacity`: innodb 一次刷新到磁盘的脏页数

`innodb_io_capacity` 与 磁盘配置 的参考对应

| `Innodb_io_capacity` | 磁盘配置       |
|                   -: | :-             |
|                  200 | 单盘 SAS/SATA  |
|                 2000 | SAS*12 RAID 10 |
|                 5000 | SSD            |
|                50000 | FUSION-IO      |


#### `innodb_flush_neighbors`

该参数在
- mysql<8.0 / mariadb<10.3.9/?? 时默认为 1, 在 flush 脏页时会把相邻的脏页同时 flush , 可能会出现连锁反应, 适用于 SATA 等 IOPS 较低的设备
- mysql>=8.0 / mariadb>=10.3.9/?? 时默认为 0, 在 flush 脏页时只 flush 当前脏页然后返回, 适用于 SSD 等 IOPS 较高的设备, 减少 SQL 语句响应时间

### Order by 工作原理

mysql 的 `order by` 排序分为两种: 全字段排序 和 rowid 排序

以以下内容作为分析基准:

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;
```

```sql
select city,name,age from t where city='杭州' order by name limit 1000;
```

![city 字段的索引示意图](5334cca9118be14bde95ec94b02f0a3e.png)

#### 全字段排序

- 初始化 sort_buffer, 确定放入 name, city, age 三个字段
- 从索引 city 找到第一个满足 `city='杭州'` 条件的主键 id, 也就是图中的 ID_X
- 从主键 id 索引取出整行, 取 name, city, age 三个字段的值, 存入 sort_buffer 中
- 重复步骤 3/4 直到 city 的值不满足查询条件为止, 对应的主键 id 也就是图中 ID_Y
- 对 sort_buffer 中的数据按照字段 name 做快速排序
- 按照排序结果取前 1000 行返回给客户端

![全字段排序](6c821828cddf46670f9d56e126e3e772.jpg)

#### rowid 排序

- 初始化 sort_buffer, 确定放入两个字段 -- 用于索引的 name 和 id
- 从索引 city 找到第一个满足 `city='杭州'` 条件的主键 id, 即图中的 ID_X
- 到主键 id 索引取出整行, 取 name, id 字段, 存入 sort_buffer
- 重复步骤 3/4 直到不满足 `city='杭州'` 条件为止, 即图中的 ID_Y
- 对 sort_buffer 中的数据按照字段 name 进行排序
- 遍历排序结果, 取前 1000 行, 并按照 id 的值回到元表中取出 city, name 和 age 三个字段并返回到客户端

![rowid 排序](dc92b67721171206a302eb679c83e86d.jpg)

#### 相关全局变量

##### `sort_buffer_size`

mysql 为排序设置的 sort_buffer 大小

如果需要排序的数据下雨 `sort_buffer_size`, 排序会在内存中完成

如果排序数据量太大, 内存不够用, 就不得不利用磁盘临时文件辅助排序

可通过以下方法确定排序语句是否使用了临时文件(MariaDB 无效)

```sql
/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a 保存 Innodb_rows_read 的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b 保存 Innodb_rows_read 的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算 Innodb_rows_read 差值 */
select @b-@a;
```

**注意: 如果需要用到临时文件, 则会使用外部排序(仅是临时文件是归并, 还是 sort_buffer 的算法也替换到归并?), 外部排序一般使用归并排序**

##### max_length_for_sort_data

控制用于排序的行数据长度

- 小于 `max_length_for_sort_data`: 全字段排序
- 大于 `max_length_for_sort_data`: `rowid` 排序

#### 通过索引优化

##### 为 (city, name) 建立联合索引

```sql
alter table t add index city_user(city, name);
```

步骤:
- 从索引 (city, name) 找到第一个满足 `city='杭州'` 条件的主键 id
- 到主键 id 索引取出整行, 取 (name, city, age) 三个字段的值, 作为结果集的一部分直接返回
- 从索引 (city, name) 取下一个记录主键 id
- 重复步骤 2/3 , 直到查到第 1000 条记录, 或者是不满足 `city='杭州'` 条件时循环结束

![city 和 name 的联合索引示意图](f980201372b676893647fb17fac4e2bf.png)

![(city, name) 联合索引后, 查询执行计划](3f590c3a14f9236f2d8e1e2cb9686692.jpg)

##### 为 (city, name, age) 建立联合索引, 以使用 覆盖索引

```sql
alter table t add index city_user(city, name, age);
```

步骤:
- 从索引 (city, name, age) 找到第一个满足 `city='杭州'` 条件的记录, 取出其中 city, name 和 age 这三个字段的值, 作为结果集的一部分直接返回
- 从索引 (city, name, age) 取出下一个记录, 如果满足条件, 则同样取出三个字段的值, 直接返回到结果集
- 重复步骤 2 , 直到满足 `limit` 或者 不满足 `city='杭州'` 条件时循环结束

![(city, name, age) 联合索引后, 查询执行计划](df4b8e445a59c53df1f2e0f115f02cd6.jpg)

#### 结论

- MySQL 在认为内存足够的情况下, 会多利用内存, 尽量减少磁盘访问
- MySQL 只在原数据是无序的时候才需要生成临时文件, 并且在临时文件中排序
- 对于 **InnoDB 表** 来说, 执行 **全字段排序** 会减少磁盘访问, 因此会被优先选择
- 对于 **内存表** 来说, 回表过程只是简单的根据数据行的位置, **直接访问内存** 得到数据, 根本不会导致多访问磁盘, 此时 **rowid** 会被优先考虑

### 随机获取行

```sql
CREATE TABLE `words` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `word` varchar(64) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=0;
  while i<10000 do
    insert into words(word) values(concat(char(97+(i div 1000)), char(97+(i % 1000 div 100)), char(97+(i % 100 div 10)), char(97+(i % 10))));
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```

```sql
select word from words order by rand() limit 3;
```

内部流程:
- 创建一个临时表; 这个临时表使用的是 memory 引擎, 表里有两个字段, 第一个字段是 *double* 类型(R), 第二个字段是 *varchar(64)* 类型(W); 且该表没有索引
- 从 `words` 表中, 按主键顺序取出所有的 word 值, 对于每一个 word 值, 调用 rand() 生成一个大于 0 小于 1 的随机小数, 并把这个 随机小数 和 word 分别存入临时表的 R 和 W 字段中, 到此, 扫描行数是 10000
- 目前临时表中有 10000 行数据, 接下来需要在没有索引的内存临时表中, 按照字段 R 排序
- 初始化 sort_buffer, sort_buffer 中有两个字段, double 和 整数
- 从 内存临时表 中一行一行的取出 R 和位置信息, 分别存入 sort_buffer 中的两个字段中; 这个过程要对内存临时表作全表扫描, 此时扫描行数增加 10000, 成了 20000
- 在 sort_buffer 中根据 R 的值进行排序, 注意: 这个过程没有涉及到表操作, 所以不会增加扫描行数
- 扫描完成后, 取出前三个结果的位置信息, 依次到内存临时表中取出 word 值, 返回给客户端; 该过程访问了表的三行数据, 扫描总行数变成 20003

![随机排序完整流程图](2abe849faa7dcad0189b61238b849ffc.png)

#### 内存临时表

`rowid`:
- 引擎用于唯一定位一行数据的信息
- 对于有主键的 InnoDB 表来说, 这个 rowid 就是主键 ID
- 对于没有主键的 InnoDB 表来说, 这个 rowid 就是由 **InnoDB** 生成的长度为 6 字节的 rowid
- memory 引擎不是索引组织表, 在这个例子里面, 你可以认为它就是一个数组, 这个 rowid 其实就是数组的下标
- rowid 是对 **InnoDB** 生效, 对 MySQL 透明的, 所以 **优化器** 不能利用 rowid 作为主键优化

即 **order by rand() 使用了内存临时表, 内存临时表排序的时候使用了 rowid 排序方法**

#### 磁盘临时表

- `tmp_table_size` 这个配置限制了内存临时表的大小, 默认值是 16M
- 如果临时表大小超过 `tmp_table_size`, 那么内存临时表就会转成磁盘临时表
- 磁盘临时表使用的默认引擎由 `internal_tmp_disk_storage_engine` 设置(默认是 InnoDB), MariaDB 则是 `default_tmp_storage_engine` ?

#### 优先队列(最大/最小堆)算法

- 如果需要存储的临时表内存 size 小于 `innodb_sort_buffer_size`, 则启用 优先队列算法
- 如果启用了 `OPTIMIZER_TRACE`, `information_schema`.`OPTIMIZER_TRACE`.`filesort_priority_queue_optimiaztion`.`chosen` 为 `true`, 则确定启动排序的 优先队列算法
- 如果需要存储的临时表内存 size 大于 `innodb_sort_buffer_size`, 仍然会启用 归并排序算法

相对而言, 严格随机的方法

```sql
select count(*) into @C from t;
set @Y = floor(@C * rand());
set @sql = concat("select * from t limit ", @Y, ",1");
prepare stmt from @sql;
execute stmt;
DEALLOCATE prepare stmt;
```

### 函数与索引

- sql 语句的索引一般观察的是 **被驱动表** 的索引(select l from table)

#### 条件字段的函数操作

- (由于 B+ 树的有序性是按深度递增, )对索引字段作函数操作, 可能会破坏索引值的 **有序性** , 因而导致优化器决定放弃树搜索功能
- 即使运算不会破坏有序性, MySQL 也有可能不会考虑索引

#### 隐式类型转换

- 字符串和数字作比较, 是将 字符串 转换到 **数字**(对字符串做数字转换方法), 再做比较

#### 隐式字符编码转换

- 字符串默认以"按数据长度增加的方向"进行转换, 而转换的过程动用函数

### 事务与查询与锁

- 查询(select) 的时候使用 `lock in share mode` 或者 `for update` 会导致该查询放弃该事务的视图, 而转用最新的视图, 如果使用了行锁, 则分别是添加 S 锁 或者 X 锁; 破坏了读可重复性, 但是由于不需要回滚 MVCC 版本, 查询效率更高(尤其是该查询的目标表已经进行了大量的更改)
- 间隙锁, 锁的是查询到数据的间隙; 间隙锁之间不冲突; 间隙锁是 前开后开 区间

### MySQL 加锁规则

- 原则:
  1. 加锁的基本单位是 next-key lock(前开后闭区间)
  2. 查找过程中 访问到的对象(索引, 行等) 才加锁
- 优化:
  1. 索引上的 **等值查询** , 给 唯一索引 加锁的时候, next-key lock 退化为 **行锁**
  2. 索引上的 **等值查询**, **向右遍历** 时且 (在一个 next-key lock 区间)最后一个值 **不等于** 等值条件 的时候, next-key lock 退化为 **间隙锁**
- BUG:
  1. (唯一?)索引上的范围查询会访问到不满足条件的第一个值为止
  
- 锁是加载索引上的
- 如果目的是通过 `lock in share mode` 加 S 锁并且要求防止数据更新, 则需要同时考虑避开 覆盖索引 的优化(可能直接 `for update` 加 X 锁更好 ?)
- 由于索引都需要查询到不满足的条件, 所以建议尽量使用 **limit** , 访问安全且减小了加锁的范围
- **读提交(Read-Commit) 没有间隙锁(或者说间隙锁更小), 因此 debug 分析时逻辑更为清晰**
- `desc` 语句导致优化器倒序搜索满足条件的值, 那么上面 **优化 2** 的 **向右遍历** , 就变成 **向左遍历**
- **间隙**, 是由 **这个间隙右边的那个记录** 定义的

### binlog 写入规则

![binlog 写盘状态](9ed86644d5f39efb0efec595abb92e3e.png)

- **write**: 把日志写入到文件系统的 page cache, 没有持久化数据到磁盘, 速度较块
- **fsync**: 数据持久化操作; 主要占用 IOPS 的操作

`sync_binlog` 变量:
- `sync_binlog=0`: 每次提交事务都只 write, 不 fsync
- `sync_binlog=1`: 每次提交事务都执行 fsync
- `sync_binlog=N(N>1)`: 每次提交事务都 write, 在积累 N 个事务后 sync, 推荐 N 在 (100, 1000) 这个范围, 但是故障重启后会丢失最近 N 个事务的 binlog

![MySQL redo log 存储状态](9d057f61d3962407f413deebc80526d4.png)

redo log 的三种状态:
- 存在 redo log buffer 种, 物理上是在 MySQL 进程内存中, 即图中的红色部分
- 写到磁盘(write), 但是没有持久化(fsync), 物理上是在文件系统的 page cache 里, 即黄色部分
- 持久化到磁盘, 对应 hard disk, 即绿色部分

已经提交的事务, `innodb_flush_log_trx_commit` 会控制 redo log 的写入策略:
- 0: 表示每次事务提交时, 都只是把 redo log 留在 redo log buffer
- 1: 每次事务提交后, 都将 redo log 持久化到 磁盘
- 2: 每次事务提交后, 都只是把 redo log 写到 page cache

**没有提交的事务 redo log** 在以下情况写入到磁盘:
- InnoDB 每秒 1 次的写磁盘操作
- redo log buffer 占用的空间即将达到 **innodb_buffer_size** 一半的时候, 后台进程会主动写盘
- 并行的事务提交的时候, 顺带将这个事务的 redo log buffer 持久化到磁盘

> 双 1 配置:
> `sync_binlog` 和 `innodb_flush_log_at_trx_commit` 都为 1:
> 一个事务完整提交的同时需要 redo log (prepare 阶段) 和 binlog 两次写盘

组提交机制:
- LSN(日志逻辑序列号): 单调递增, 对应 redo log 的 **写入点**; 每次写入 length 的 redo log, LSN 的值就加上 LSN
- 并发事务在 perpare 阶段, 写完 redo log buffer , 准备持久化到磁盘; 该组的 leader (一般是第一个写盘的事务)以该组的 LSN 总和的方式写盘

![redo log 组提交](933fdc052c6339de2aa3bf3f65b188cc.png)

![两阶段提交, 组提交细节](5ae7d074c34bc5bd55c82781de670c28.png)

binlog 的 fsync 变量(mysql/mariadb):
- `binlog_group_commit_sync_delay` / `binlog_commit_wait_usec`: 等待时间
- `binlog_group_commit_sync_no_delay_count` / `binlog_commit_wait_count`: 等待累计数

### MySQL 设计问答

#### 为什么 binlog cache 是每个线程独立维护, 而 redo log buffer 是全局共用

- binlog 不能被打断: 一个事务的 binlog 必须连续写, 因此要整个事务完成后, 再一起写到文件
- redo log 为了减少写盘次数, 会携带并发时的其他事务一起写盘

#### binlog 已写盘 , redo log 也 commit 了, 但是因为网络原因, 客户端接受不到响应, 此时仍然是正常情况

数据库的 "crash-safe" 保证以下内容:
- 如果客户端收到事务成功的消息, 事务就一定持久化了
- 如果客户端收到事务失败(主键冲突, 回滚等)的消息, 事务就一定失败了
- 如果客户端收到 "执行异常" 的消息, 应用需要重连后通过查询当前状态来继续后续的逻辑; 此时数据库只保证内部(数据和日志/主库和备库之间)一致

### 主备一致

预备: 备库节点设置成 readonly
- 有时候一些运营类的查询语句会被放到备库查询, 设置为只读更安全
- 放置 切换时 出现双写, 造成主备不一致
- 可以用 readonly 状态, 来判断节点的角色
- readonly 设置对 超级(super)权限 用户无效

![主备流程图](a66c154c1bc51e071dd2cc8c1d6ca6a3.png)

一个事务日志同步的完整流程:
- 在备库 B 上通过 `change master` 命令, 设置主库 A 的 IP, 端口, 用户名, 密码, 以及要从呢个位置开始请求 binlog, 这个位置包含 文件名和日志偏移量
- 在备库 B 上执行 `start slave` 命令, 这时候备库会启动两个线程 -- `io_thread`/`sql_thread`; 其中 io_thread 负责与主库建立连接
- 主库 A 验证完(用户名, 密码)后, 开始按照备库 B 传来的位置, 从本地读取 binlog , 发给 B
- 备库 B 拿到 binlog 后, 写到本地文件, 即 中转日志(relay log)
- sql_thread 读取中转日志, 解析出日志里的命令, 并执行

`binlog_format`:
- ROW:
  - FULL: 记录改动的 行 的 所有字段的值; 同时记录所有的 前镜像(表中修改前的内容) 和 后镜像(表中修改后的内容)
  - MINIMAL: 只记录必要的信息(主键, 修改列等); 前镜像只记录唯一识别列(唯一索引列, 主键列), 后镜像只记录 修改列
- STATEMENT:
  - 只记录 sql 语句
  - 通过添加 `SET TIMESTAMP=XXX` 确保主备数据一致性
- MIXED:
  - 在可能主备数据不一致的地方使用 ROW 格式, 否则使用 STATEMENT
  
#### 复制循环问题

双 M 结构(互为主备)

![MySQL 主备切换流程 - 双 M 结构](20ad4e163115198dc6cf372d5116c956.png)

双 M 结构互为主备关系, 在切换时不需要修改主备关系

`log_slave_update`: 备库执行 relay log 后生成 binlog

循环复制问题: 节点 A 更新了一条语句, 生成的 binlog 发给节点 B, 节点 B 执行完后也生成 binlog; 此时如果节点 A 也是节点 B 的备库(双 M 结构), 相当于又把节点 B 新生成的 binlog 拿过来

MySQL 是在 binlog 中记录该命令的 server id, 以解决复制循环的问题的

双 M 结构规定:
- 两个库的 server id 必须不同, 如果相同, 则它们之间不能设定为主备关系
- 一个备库接收到 binlog 并在重放的过程中, 生成于原 binlog 的 server id 相同的 binlog
- 每个库在收到从字节的主库发过来的日志后, 先判断 server id , 如果与自己相同, 表示这个日志是自己生成的, 就直接丢弃这个日志

按照上面逻辑, 如果我们设置了双 M 结构, 日志的执行流程应该是:
- 从节点 A 更新的事务, binlog 里面记的都是 A 的 server id
- 传到节点 B 执行一次以后, 节点 B 生成的 binlog 的 server id 也是 A 的 server id
- 再传回给节点 A , A 判断到这个 server id 与自己的相同, 就不会再处理这个日志, 所以, 死循环再这里断掉

##### 数据库迁移导致的循环复制

数据库迁移途中如果由双 M 结构, 可能会导致 循环复制, 建议做法

```sql
stop slave;
CHANGE MASTER TO IGNORE_SERVER_IDS=(server_id_of_B);
start slave;
```

停止非主备双方的复制; 一段时间后, 把 ignore 改回来

```sql
stop slave;
CHANGE MASTER TO IGNORE_SERVER_IDS=();
start slave;
```


### 高可用

#### 主备延迟

与数据同步有关的时间点主要包括以下三点:
1. 主库 A 执行完成一个事务, 写入 binlog, 我们把这个时刻记为 T1
2. 之后传给备库 B , 我们把备库 B 接收这个 binlog 的时刻记为 T2
3. 备库 B 执行完成这个事务, 我们把这个时刻记为 T3

主备延迟(`seconds_behind_master`): 同一个事务, 在备库执行完成的时间 和 主库执行完成的时间差 , 即 T3 - T1, mysql 的计算方式:
- 每个事务的 binlog 里面有一个时间字段, 用于记录主库上写入的时间
- 备库取出当前正在执行的事务的时间字段的值, 计算它与当前系统时间的差值, 即 `seconds_behind_master`

#### 主备切换策略

结论: 在满足可靠性的前提下, 可用性依赖于 `seconds_behind_master`(SBM); SBM 越小, 可用性越高

##### 可靠性优先策略

在 双 M 结构 下, 从状态 1 到状态 2 切换:
- 判断备库 B 现在的 `seconds_behind_master`(SBM), 如果小于某个值, 继续下一步, 否则持续重试这一步
- 把主库 A 改成 只读 状态, 即把 readonly 设置为 true
- 判断备库 B 的 `seconds_behind_master` 的值, 直到这个值变成 0 位置
- 把备库改成 可读写 状态, 即把 readonly 设置为 false
- 把业务请求切换到备库 B

![MySQL 可靠性优先主备切换流程](54f4c7c31e6f0f807c2ab77f78c8844a.png)

##### 可用性优先策略

把 可靠性优先 的后两步提前执行, 来略过中间的服务不可用(主备 readonly)的环节

可用性可能导致数据不一致的情况, 因此更建议 `binlog_format` 设置为 `ROW` 格式

### 备库并行复制

`slave_parallel_workers`: 用于控制 work 线程数量, 一般取 8-16(32 核物理机)

![多线程模型](bcf75aa3b0f496699fd7885426bc6245.png)

- coordinator: 原先的 sql_thread , 不直接更新数据, 而只负责中转日志和分发事务
- worker: 真正更新日志的线程, 个数受 `slave_parallel_workers` 决定

#### MySQL 5.5 版本的并行复制策略 -- 丁奇

##### 按表分发策略

如果两个事务更新不同的表, 则它们可以并行

用于命中 worker 的 hash 为 hash.tableX

每个事务在分发的时候, 更所有 worker 的冲突关系包括以下三种情况:
- 如果跟所有 worker 都不冲突, coordinator 线程就会把这个事务分配给最空闲的 worker
- 如果跟多于一个 worker 冲突, coordinator 线程就进入等待状态, 直到和这个事务存在冲突关系的 worker 只剩下一个
- 如果只跟一个 worker 冲突, coordinator 线程就会把这个事务分配给这个存在冲突关系的 worker

##### 按行分发策略

**必须为 row 格式**

按行分发 还需要考虑 唯一键 的 顺序问题, 即 hash 应为 "库名+表名+索引 a 的名字 + a 的值"

相比于按表并行分发策略, 按行并行策略在决定线程分发的时候, 需要消耗更多的计算资源

#### MySQL 5.6 版本的并行复制策略

##### 按库分发策略

可以以各种格式存储

颗粒过大

#### MariaDB 的并行复制策略

利用 redo log 的 组提交(group commit) 特性:
- 能够在同一个组里提交的事务, 一定不会修改同一行
- 主库上可以并行执行的事务, 备库上一定是可以并行执行的

于是, 实现上:
- 在一组里面一起提交的事务, 有一个相同的 commit_id, 下一个组就是 commit_id+1
- commit_id 直接写在 binlog 里面
- 传到备库应用的时候, 相同 commit_id 的事务分发到多个 worker 执行
- 这一组全部执行完成后, coordinator 再去取下一批

#### MySQL 5.7 的并行复制策略

`slave-parallel-type`: 控制并行复制策略:
- `DATABASE`: 使用 5.6 的按库并行策略
- `LOGICAL_CLOCK`: 再两阶段提交中, 只要能达到 redo log perpare 阶段, 就表示事务已经通过锁冲突的检验
  - 同时处于 prepare 状态的事务, 再备库执行时时可以并行的
  - 处于 prepare 状态的事务, 与处于 commit 状态的事务之间, 在备库执行时也是可以并行的
  - 那么通过组提交的策略, 通过控制 `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no_delay_count`, 提高备库复制的并行度

#### MySQL 5.7.22 的并行复制策略

**注意: 该策略导致该版本的 binlog 不向下兼容**

`binlog_transation_dependency_tracking`: 控制是否开启新策略
- `COMMIT_ORDER`: 根据 redo log 的规则, 是否同时进入 `prepare` 和 `commit` 来判断是否可以并行的策略
- `WRITESET`: 对于事务涉及更新的每一行, 计算出这一行的 hash 值, 组成集合 `writeset`; 如果两个事务的 `writeset` 没有交集, 就认为没有操作相同的行, 可以并行
- `WRITESET_SESSION`: 在 `WRITESET` 之上, 在主库上 同一个线程(**线程约束**) 先后执行的两个事务, 在备库执行的时候, 要保证相同的先后顺序

相比 丁奇版 的 优势:
- **writeset** 是在主库生成后直接写入到 binlog 里; 那么在备库执行时, 不需要解析 binlog 内容(event 里的数据行) , 节省解析时的计算量
- 不需要把整个事务的 binlog 都扫一遍, 才能决定分发到哪个 worker
- 由于备库分发策略不依赖 binlog 内容, 即使 binlog 格式是 statement 也没问题, 通用性更强

### 主备故障排除

当在主库 A 发生故障停机, 主备切换时, 可能在备库 A' 和 从库 B 之间发生不一致的情况; 在无官方方案时, 可选用:
在主备切换且发生数据冲突时, 临时主动跳过错误, 直到错误停止:
- `sql_slave_skip_counter`: 跳过指定数量的事务
- `sql_skip_errors`: 跳过指定错误

**注: 该方案要求错误停止后关闭以上连个参数, 以免掩盖后续的错误, 导致数据不一致**

#### GTID(Global Transaction Identifier)

全局事务 ID , 是一个事务在提交时生成的, 是这个事务的唯一标识; 格式是: GTID=`server_uuid:gno`/`source_id:transaction_id`
- `server_id`/`source_id`: 一个实例第一次启动自动生成, 是一个全局唯一的值
- `gno`/`transaction_id`: 整数, 初始值为 1 , 每次 **提交事务(commit)** 即 +1
- 在 binlog 文件头部的 `Previous_gtids` 有该文件的 GTID 范围信息, mysql 就是靠这个粗定位 GTID 位置的

使能 GTID:
- `gtid_mode`
- `enforce_gtid_consistency`

GTID 生成方式:
- `gtid_next`: session 变量/当前会话变量
  - `automatic`: 默认值
    -  记录 binlog 的时候, 先记录一行: `set @@SESSION.GTID_NEXT='server_uuid:gno'`
    - 把这个 GTID 假如本实例的 GTID 集合
  - `current_gtid`: 私自指定的一个 GTID 值
    - 如果 `current_gtid` 存在, 则该事务会被系统 **直接忽略**
    - 如果 `current_gtid` 不存在, 则将该 GTID 分配到接下来要执行的事务; 此时系统不生成新的 GTID, gno 值也不会 +1

使用 GTID 后原主库的从库设置为新主库的从库语句:
```sql
CHANGE MASTER TO
MASTER_HOST=$host_name
MASTER_PORT=$port
MASTER_USER=$user_name
MASTER_PASSWORD=$password
master_auto_position=1
```

`master_auto_position`: 主备关系启用 GTID 协议

#### GTID 与在线 DDL

#### 课后问题

在需要创建主从关系时由于 binlog 不充足, 导致主从关系建立失败, 可从以下方面考虑:
1. 如果业务允许主从不一致的情况那么可以在主上先 show global variables like 'gtid_purged', 得到 `gtid_purged1`; 然后在从上执行 set global gtid_purged ='gtid_purged1' . 指定从库从哪个 gtid 开始同步; 此时 binlog 缺失那一部分数据在从库上会丢失, 会造成主从不一致
2. 需要主从数据一致的话, 最好还是通过重新搭建从库来做
3. 如果有其它的从库保留有全量的 binlog 的话，可以把从库指定为保留了全量 binlog 的从库为主库(级联复制)
4. 如果 binlog 有备份的情况,可以先在从库上应用缺失的 binlog,然后在 start slave

### 读写分离

#### 客户端直连

- 不需要 proxy 转发请求, 性能更好, 架构简单
- 直接连接, 需要客户端处理对不同服务的连接, 耦合高(可以用 zookeeper 之类的负责后端管理)

#### proxy

- 服务的两端耦合低
- proxy 本身对可用性的需求在系统中是最高的, 对团队要求高

#### 过期读

这种 "在从库上会读到系统的一个过期状态" 的现象, 暂且称之为 "过期读"

##### 强制走主库

对实时性要求高的查询, 直接查询主库

##### Sleep

在更新完主库后, 等待一定时间再查询备库(1s?)

##### 判读主备无延迟

![show slave status](00110923007513e865d7f43a124887c1.png)

- 每次读从库先读取 `seconds_behind_master` , 直到该值为 0 再读取
- 对比位点确保主备无延迟
  - `Master_Log_File` 和 `Read_Master_Log_Pos` 表示的是读到的主库的最新位置
  - `Relay_Master_Log_File` 和 `Exec_Master_Log_Pos` 表示的是备库执行到的最新位置
  - 如果 `Master_Log_File` 和 `Relay_Master_Log_File`, `Read_Master_Log_Pos` 和 `Exec_Master_Log_Pos` 两两相等, 则认为接收到的日志已经同步完
- 对比 GTID 集合确保主备无延迟
  - `Auto_Position=1` -- 主备关系启用 GTID 协议
  - `Retrieved_Gtid_Set` -- 备库收到的所有日志的 GTID 集合
  - `Executed_Gtid_Set` -- 备库所有已经执行的 GTID 集合
  - 如果两个集合相同, 即备库接收到日志都已经同步完成

##### 配合 semi-sync

semi-sync 的设计:
- 事务提交的时候, 主库把 binlog 发给从库
- 从库收到 binlog 后, 发回给从库一个 ack , 表示收到了
- 主库收到这个 (多个从库中随意一个) ask , 才给客户端返回 "事务完成" 的确认

但是 semi-sync 仍然会有以下问题:
- 一主多从时, 从库执行查询请求可能仍然有过期读
- 在事务繁忙时, 可能由于持续延迟, 而导致客户端过度等待

##### 等主库位点

如果在 从库 执行以下命令:

```sql
select master_pos_wait(file, pos[, timeout]);
```

return:
- NULL: 执行期间, 备库同步线程发生 **异常**
- -1: 等待时间超过 timeout
- 0: 刚开始执行, 就发现已经执行过这个位置
- >0: 从命令开始执行, 到从库执行完 file 和 pos 表示的 binlog 位置, 执行了多少事务

从以上信息, 如果只要求获得查询语句要求的最新数据:
- 目标事务执行完, 马上执行 `show master status` 得到当前主库执行到的 file 和 pos
- 选定一个从库准备执行查询语句
- 先在从库执行 `select master_pos_wait(File, Position, 1)`
- 返回 >=0 , 则在该从库执行查询语句
- 否则, 在主库执行查询语句(退化机制)


##### 等 GTID

```sql
select wait_for_executed_gtid_set(gtid_set, 1);
```

return:
- 等待, 直到该(从)库执行的事务中包含传入的 gtid_set , 返回 0
-  超时返回 1

`session_track_gtids`=`OWN_GTID`: 开启事务执行后返回 GTID , 在 C 中需要调用 `mysql_session_track_get_first` , GO 有这东西吗 ?

#### 课后问题

假设一个一主多从且主备方案是 GTID , 现在需要在一张大表上做 DDL , 可能会出现什么情况

大表做 DDL 的时候可能导致 **主从延迟** , 导致 等 GTID 的方案可能导致这个时间段的流量全部流到主库, 或者全部超时

如果流量太大, 则需要避免该请况, 可以考虑先在从库做 DDL , 主备切换后, 主库再做 DDL
- 在各个从库先 `SET sql_log_bin=OFF` , 做 DDL , 所有从库及备库做完后, 做主从换, 最后在原来的主库用同样的方式做 DDL
- 在从库执行 DDL; 将从库上执行 DDL 产生的 GTID 在主库上利用生成一个空事务 GTID 的方式将这个 GTID 在主库上生成出来

但是以上方案, 在 DDL 期间主库完成的事务需要同步到 DDL 过的从库, 如果 DDL 内容是 **删除列** 的操作, 会导致同步出错

因此, 该方案下, 支持的 DDL 只有:
- 创建/删除索引
- 新增最后一列, 删除最后一列

### 如何判断数据库是否出问题

#### select 1

`innodb_thread_concurrency`: InnoDB  **并发查询** 的并发线程上限

不足: 如果 并发线程数 已经到上限了, 那么 `select 1` 能够返回成功, 但是 SQL 语句依然会被阻塞

**在线程进入锁等待之后, 并发线程的计数会减一**, 即等 行锁/间隙锁 的线程不算在 `innodb_thread_concurrency` 之内

#### 查表判断

在 mysql 库创建一个表, 里面只放一行数据, 定期执行

```sql
select * from mysql.health_check;
```

不足: 这种情况可以判断查询, 不能判断事务更新

#### 更新判断

```sql
UPDATE mysql.health_check SET t_modified=now();
```

放一个 timestamp 字段, 更新最后一次执行检测的时间

但是如果是 双 M 结构, 则需要考虑可能出现 行冲突 , 导致同步停止的情况; 此时可以在该更新行上添加上 `server_id` .

```sql
CREATE TABLE `health_check` (
  `id` int(11) NOT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

/* 检测命令 */
insert into mysql.health_check(id, t_modified) values
  (@@server_id, now()) on duplicate key update t_modified=now();
```

不足: 不能充分反映 IO 资源的短缺

#### 内部统计 -- `performance_schema`

- `performance`.`file_summary_by_event_name`: 统计各个类型, 每次 IO 请求的各种时间结果(平均, 最大最小)
- `performance`.`setup_instruments`: 各个统计数据类型的开关

其中:
- `wait/io/file/innodb/innodb_log_file` -- 关于 redo log 的统计
- `wait/io/file/sql/binlog` -- 关于 binlog 的统计

使能监控:

```sql
update setup_instruments set `ENABLED`='YES', `TIMED`='YES' where name like '?';
```

例如:

```sql
select `EVENT_NAME`, `MAX_TIMER_WAIT` FROM `performance_schema`.`file_summary_by_event_name` where `EVENT_NAME` in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and `MAX_TIMER_WAIT`>200*1000000000;
```

检测 binlog 和 redo log 的单次 IO 请求时间是否超过 200 ms

### 动态观点看死锁

1. 锁是一个一个加的, 对同一组资源的访问, 要按照尽量相同的顺序访问
2. 在发生死锁时, `for update` 这条语句占用的资源更多, 回滚成本大, 即如果有的话, InnoDB 会选择了回滚成本更小的 `lock in share mode`

```sql
truncate table `performance_schema`.`file_summary_by_event_name`;
```

清空统计值, 以免统计值影响下次判断

#### 课后问题: 如果对空表加锁, 锁的是什么?

对空表加锁, 锁的范围是 next-key lock (-∞, supermum]

### 误删数据

- `sql_safe_updates`: `delete` 或者 `update` 没有 `where` 条件或者 `where` 条件里面没有包含索引字段, 那么就会报错
- 保证 `binlog_format=ROW` 和 `binlog_row_image=FULL`

#### 误删库/表 -- 常规操作

如果误删库: 要求有全量备份, 加增量日志(binlog)

1. 取最近一次全量备份
2. 通过备份恢复出一个临时库
3. 从日志备份里面, 取出备份后的日志(通过 start-position/stop-position / GTID 定位)
4. 除了 误操作的语句外 , 应用到临时库

![误删库时, 数据恢复流程](2fafd0b75286e0163f432f85428ff8db.png)

如果误删表:
1. 上述 2 之后, 将临时实例设置为线上的备库的从库
2. 在 `start slave` 之前, 先通过执行 `change replication filter replicate_do_table=(tbl_name)`, 就可以让临时库只同步误操作的表(???)
- 也可以用上并行复制, 加速数据恢复过程

![误删表时, 数据恢复流程](65bb04929b8235fb677c7a78b5bd67f1.png)

优点:
- 稳定, 没有业务限制

缺点:
- 可恢复的期限与 `expire_logs_days`(binlog 保留时间)相关
- 恢复时间长

#### 延迟复制备库

通过 `CHANGE MASTER TO MASTER_DELAY=N` 设置备库与主库的延迟, 保持延迟在 **N 秒** 内

优点:
- 定位错误越早, 恢复效率越快

缺点:
- 这种情况会导致主库压力大(实时数据都从主库操作查询), 规模越大, 可适用的业务越严格

#### 预防删除库/表

预防措施:
1. 账号分离, 具体业务 只有 DML 权限, DDL 要求通过开发管理系统获得支持
2. 要删除数据表前, 先对目标表做改名操作(固定前后缀 , 如 `_to_be_deleted`); 观察一段时间, 确保对业务无影响后再删除表; 改表名的操作要通过管理系统执行, 并且要对表名有检查(如 必须是前面的 固定前后缀)

#### rm 删除数据

- 集群 方案解决
- 跨机房/跨城市 备份
  - 如果数据量大, 用物理备份 [xtrabackup](https://github.com/percona/percona-xtrabackup)
  - 如果数据量小, 用 mysqldump / mysqldumper

#### 生产规范

**重要!重要!** 修改生产的数据, 或者添加索引优化, 都要先写好四个脚本:
- 备份脚本: 是对需要变更的数据备份到一张表中, 固定需要操作的数据行, 以便误操作或业务要求进行回滚
- 执行脚本: 对数据变更的脚本, 为防 Update 错数据, 一般连备份表进行 Update 操作
- 验证脚本: 验证数据变更或影响行数是否达到预期要求效果
- 回滚脚本: 将数据回滚到修改前的状态。

小 tip:
**chattr +i**: (无论什么权限)不能 添加/修改/删除/重命名 文件

### kill

当收到 `kill query thread_id` 时, mysql 处理 kill 命令的线程做了两件事
- 把 session 的运行状态(变量 killed)改成 `THD::KILL_QUERY`
- 对 session 的执行线程发一个信号, 让该 session 退出等待

当收到 `kill thread_id` 时, mysql 处理 kill 命令的线程做了两件事
- 把 session 的运行状态(变量 killed)改成 `THD::KILL_CONNECTION`
- 关掉 session 的网络连接, 此时客户端收到 断开连接 的提示

`show processlist` 部分逻辑:
- 如果一个线程的状态是 `KILL_CONNECTION` , 就把 Command 列 显示成 Killed

#### kill 无效的原因

##### 线程没有执行到判断线程状态的逻辑

```sql
select sleep(100) from t;
```

`sleep` 的等待逻辑不同于 等待行锁(`pthread_cond_timewait`), 该逻辑没有唤醒点

##### 终止逻辑耗时较长

- 超大事务执行期间被 kill; 此时回滚事务, 耗时长
- 大查询回滚; 查询期间生成较大的临时文件, 或者此时文件系统压力大, 在等待 IO 资源
- DDL 命令执行到最后阶段, 被 kill, 需要删除临时文件, 也会受文件系统压力影响

#### mysql 客户端资源请求

- 本地缓存(默认 -- `mysql_store_result`), 向服务器请求结果, 并在客户端本地缓存结果
- 不存储(`-quick` -- `mysql_use_result`), 读一行处理一行

#### 课后问题: 长时间回滚大事务, 应该是重启还是等待完成

- 重启其实还是会继续回滚, 因为 redo log 此时还未提交(Commit); 如果系统资源充足, 更建议继续回滚
- 如果系统资源紧缺(尤其是 IO) , 那么关闭 -> 切换到备库执行回滚 -> 主备切换 -> 同步 , 是个不错的选择

### 全表扫描

`net_buffer_length`: 默认 16 k, 网络发送包的一次发送的最大长度

#### 全表扫描时, server 层的影响:

- 获取一行, 写到 net_buffer; 重复获取行, 直到 net_buffer 写满, 调用网络接口发送
- 发送成功, 就清空 net_buffer , 取下一行, 继续写入
- 如果 发送函数 返回 `EAGAIN` 或 `WSAEWOULDBLOCK`, 就表示本地网络栈(socket send buffer) 写满了, 进入等待; 直到网络栈重新可写

查询语句的状态变化(`show processlist`.`state`):
- MySQL 查询语句进入执行阶段, 首先把状态设置为 `Sending data`
- 发送执行结果的列相关的信息(meta data)给客户端
- 继续执行语句的流程(还有哪些流程? 写 binlog?redo log commit?)
- 执行完成后, 把状态(state)设置为空字符串

#### 全表扫描时, InnoDB 层的影响

1. WAL 机制缓存的数据存储在 Buffer Pool(BP), 该机制加速了更新和加速查询的效率

**内存命中率**: WAL 机制加速效果的重要指标

`innodb_buffer_pool_instances`: Buffer Pool 数量

2. InnoDB 通过 最少使用(LRU)算法, 淘汰最久未使用的数据

![InnoDB 的 LRU 算法](25e18920dd204cf99eec2d62755fe99e.png)

InnoDB 按照 5:3 比例把整个 LRU 链表分成了 young 和 old 区域, 通过 LRU_old 指向 old 的首位

InnoDB 的 LRU 完整流程:
1. 状态 1 , 访问数据页 P3 , 由于 P3 在 young 区域, 因此和优化前的 LRU 算法一样, 将其移到链表头部, 即状态 2
2. 访问新的不存在于 LRU 链表的数据页, 这时候依然是淘汰数据页 Pm, 但是新插入的数据页 Px, 是在 LRU_old
3. 处于 old 区域的数据页, 每次被访问都需要基于 `innodb_old_blocks_time`(LRU 链表的 old 区域中判断的时间阈值 -- 默认 1000ms) 做以下判断
  - 如果这个数据页在 LRU 链表中存在的时间 **超过 innodb_old_blocks_time**, 就把它移动到链表头部
  - 如果这个数据页在 LRU 链表中存在的时间 **短于 innodb_old_blocks_time**, 位置保持不变

## 课后问题

如果客户端压力过大, **长时间不接收数据**, 会对服务端产生什么影响?

- 会造成 长事务
- 如果该事务有更新, 会锁住目标的行锁/间隙锁, 会导致其他语句的更新被锁住
- 导致 undo log 不能被回收, 导致回滚段空间膨胀
  
### JOIN

`STRAIGHT_JOIN`: 固定的, 以左边的表作为驱动表, 驱动右表; 强制 优化器 对于 联表查询 的执行顺序

#### NLJ(Index Nested-Loop Join)

使用上 被驱动表 的索引

#### Simple Nested-Loop Join

无索引无优化

#### BNL(Block Nested-Loop Join)

MySQL 无索引优化

![Block Nested-Loop Join](15ae4f17c46bf71e8349a8f2ef70d573.jpg)

`join_buffer_size`: 设置 join_buffer 的大小

- 如果放不下 驱动表 需要的数据, 那么 驱动表 会被分段放置
- 把驱动表上的数据需要读到的全部读入到 join_buffer 中
- 扫描 被驱动表, 把 被驱动表 中的数据和 join_buffer 中的数据对比, 满足 join 条件的, 作为结果集的一部分返回

如果使用 join , 应该选择大表做驱动表还是小表做驱动表
- 如果是 `Index Nested-Loop Join` , 应该选择小表做驱动表
- 如果是 `Block Nested-Loop Join`:
  - 在 `join_buffer_size` 足够大的时候, 是一样的
  - 在 `join_buffer_size` 不够大的时候, 应该选择小表做驱动表(避免多次表分段)
- 小表的判断, 需要考虑到 表行数 和 表列数

#### 课后问题 (BNL 的性能问题)

如果被驱动表是一张 **大表** , 且是一个 **冷数据库**, 查询中会对 MySQL 服务器产生什么影响

如果 驱动表 太大导致分段, 那么 被驱动表 就会被多次读取:
- 如果大表的大小 M 页比 old 区域 N 小(M<N), 循环读取间隔很可能超过 `innodb_old_blocks_time` 的时间, 那么该数据会被迁移到 **young** 区域, 把部分 **热点数据被淘汰**, 导致 "Buffer pool hit rate" 命中率极低; 热点数据的清求需要读磁盘, 因此响应慢, 请求被阻塞等
- 如果大表的大小 M 页比 old 区域 N 页大(M>N), 则使得读取一次(M 页)大表后再读, 都需要从头开始读取大表(从第 1 页开始), 影响 缓存(buffer pool) 区域的作用

#### MRR(Multi-Range Read) 优化

目的是尽量顺序读盘

前提: **如果数据是按照主键递增顺序插入得到的, 那么可以认为: 如果按照主键的递增顺序查询的话, 对磁盘的读比较接近顺序读, 能够提升读性能**

`read_rnd_buffer_size`: 控制 `read_rnd_buffer_size` 大小

现在版本的优化器会倾向于不适用 MRR, 开启 MRR 优化(开启 MRR, 并且在使用前不计算消耗):
```sql
set optimizer_switch="mrr=on,mrr_cost_based=off";
```

开启 MRR 后, 逻辑:
- 根据索引, 定位到满足条件的记录, 将 id 值放入到 read_rnd_buffer
- 将 read_rnd_buffer 中的 id 递增排序
- 以排序后的 id 序列, 依次到主键 id 索引中查记录, 并作为结果返回

#### BKA(Batched Key Access)

开启 BKA -- NLJ + MRR:
```sql
-- mysql
set optimizer_switch="mrr=on,mrr_cost_based=off,batched_key_access=on";

-- mariadb
set optimizer_switch="";
```

#### 课后问题

```sql
CREATE TABLE `t1` (
 `id` int(11) NOT NULL,
 `a` int(11) DEFAULT NULL,
 `b` int(11) DEFAULT NULL,
 `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

create table t2 like t1;
create table t3 like t2;
-- 初始化三张表的数据
insert into ... 
```

```sql
select * from t1 join t2 on(t1.a=t2.a) join t3 on (t2.b=t3.b) where t1.c>=X and t2.c>=Y and t3.c>=Z;
```

分析:
- @Mr.Strive.Z.H.L, 使用 `join`:
  - (调用 innodb 接口,)从 t1 取一行数据, 返回到 server
  - 从 t2 取满足条件的数据
  - 从 t3 取满足条件的数据
- 如果采用 BKA 进行优化(只有 BKA 可以?), 是直接的嵌套查询; 且每多一个 `join` 部分，就多一个 join_buffer
- 如果没有 `straight_join` , 那么第一个驱动表 MySQL 会在经过 `where t1.c>=X and t2.c>=Y and t3.c>=Z` 过滤后, 以数据最少的表(在 t1/t2/t3 中选), 作为第一个驱动表, 此时可能出现以下情况:
  - 如果驱动表是 t1, 则连接顺序是 t1->t2->t3, 则需要在(被驱动表) t2.a 和 t3.b 上创建索引
  - 如果驱动表是 t3, 连接顺序: t3->t2->t1, 则需要在 t2.b 和 t1.a 上创建索引
  - 如果驱动表是 t2, 则考虑另外拎个条件的过滤效果
  - 同时需要在 第一个驱动表 的 c 上创建索引

### 临时表

#### 特性

- 建表语句: `create temporary table`
- 一个临时表只能被创建它的 **session** 访问, 对其他线程不可见, 且在 session 结束时销毁
- 临时表可以与普通表同名
- session 中如果有同名的临时表和普通表, (`show create`/增删查改)语句优先访问 **临时表**
- `show tables` 显示普通表, `select @@tmpdir` 显示实例的临时文件目录
- 关于 `InnoDB` 引擎的临时表:
  - 表结构: 会生成一个放置在 临时文件夹 , ".frm" 后缀, "#sql{进程 id}_{线程 id}_序列号" 的文件
  - 表中数据的 存放方式 和版本相关:
    - <=5.6: 在临时文件目录下创建一个相同前缀, 以 ".idb" 为后缀的文件
    - >=5.7: MySQL 引入了 临时文件表空间 , 专门用于存放临时文件的数据
- 关于数据表在 内存 中的区分命名 -- `table_def_key`
  - 普通表: `table_def_key` 的值是由 "库名+表名"
  - 临时表: `table_def_key` 的值是由 "库名+表名", 又加上 "{`server_id`}+{`thread_id`}"
  
#### 注:

如果设置了 `binlog_format={ statement | mixed }`

1. MySQL 在记录 binlog 的 drop table 行为时, 会以标准格式:

```sql
DROP TABLE `t_normal` /* generated by server */
```

以标注该命令被服务器端修改过

2. MySQL 在记录 binlog 时, 会把主库执行这个语句的 `thread_id` 写到 binlog 中; 当需要 **主备复制** 临时表的时候, 备库通过与该语句关联的 `thread_id` 生成临时表

#### 课后思考

```sql
create temporary table temp_t(id int primary key)engine=innodb;

-- 通过
alter table temp_t rename to temp_t2; ------ [1]

-- error, Table 'test.temp_t2' doesn't exist
-- 注意: mariadb 中没有报错 !!
rename table temp_t2 to temp_t3; -------- [2]
```

[1]: `alter table` 操作的是 `table_def_key`
[2]: `rename table` 是通过到 MySQL 的 database 目录下修改该数据库的表("库名/表名.frm"); 但是临时表是放置在 tmpdir 目录下, 而且文件命名规则是 "#sql{process_id}_{thread_id}_no.frm"

### 内部临时表

#### UNION

`UNION`:
- 如果只有一列数据, 那么该列作为 主键(PRIMARY) , 享受唯一性约束
- 如果有两列或以上数据, 那么不存在 主键 , 无论实际开始 union 的数据列是否有主键
- 在表上使用上了临时表, 而且该表依然满足各种约束规则

`UNION ALL`:
- 对 联合 的语句依次执行子查询, 得到的结果直接作为结果集的一部分, 发给客户端; 即不需要 临时表

#### GROUP BY

## 附: 杂记

### inplace 与 online 的关系

- DDL 过程如果是 online , 就一定是 inplace ()
- 如果时 inplace 有可能不是 online(全文索引--FULLTEXT index 空间索引--SPATIAL index)

### Update

如果 `update` 查询后要更改的值和目标值一样, InnoDB 依然会认真的执行该语句(加锁, 数据更新)

因为 MySQL 的隔离规则(可重复读 等), 在一个事务(08 | 事务到底是隔离的还是不隔离的?--图 5)中, 如果该语句(Update)不记录, 不执行
会直接影响事务后续的语句

### MySQL 配置

mysql >= 5.7 后, 默认开启 ssl, 即使用 unixsock 连接, 也会读取 /etc/mysql/conf.d 里的 ssl 配置先行验证

但是 mycli 用不了是 ??

### go-sql-driver

```shell
  - fieldTypeDate, fieldTypeNewDate --> Date YYYY-MM-DD
  - fieldTypeTime --> Time [-][H]HH:MM:SS[.fractal]
  - fieldTypeTimestamp, fieldTypeDateTime --> Timestamp YYYY-MM-DD HH:MM:SS[.fractal]
```

> connect 时, 加 `?parseTime=true` , MySQL-Server 读时间时会返回时间的 二进制 ?
> 好像不是, 好像只是, go-sql-driver 在 client 对时间的 二进制 内容转换为 time.Time, 没有加 `parseTime=true` 的时候只是返回字符串

从上面的类型看:
- `binaryRows`.`readRow`: 新版 stmt 协议 "prepared statement protocol", >= mysql/4.1
  - fieldTypeTime: -- "-838:59:59" 到 "838:59:59" 3 字节
    - formatBinaryTime -- 二进制 编码到 time 字符串
  - !fieldTypeTime && hasParams(conn, "parseTime=true"):
    - parseBinaryDateTime -- 二进制 直接编码到 time.Time
  - 其余:
    - formatBinaryDateTime -- 二进制 编码到 time 字符串
- `textRows`.`readRow`: 旧版协议, >= mysql/3.20
  - hasPrarms(conn, "parseTime=true") && fieldTypeTimestamp, fieldTypeDateTime, fieldTypeDate, fieldTypeNewDate:
    - parseDateTime -- "YYYY-MM-DD HH:MM:SS.MMMMMM" -> time.Parse(sql.timeFormat) -> time.Time

由于 go 里面不能直接赋值到 `time`.`Time` , 所以如果没有添加 "parseTime=true" , 那么在结构体里面不能使用 `time`.`Time` , 而是用 `database/sql`.`NullString` , 再自行转换到 `time`.`Time`

### `show processlist`.`state`
- `Checking table`: 正在检查数据表(这是自动的)
- `Closing tables`: 正在将表中修改的数据刷新到磁盘中, 同时正在关闭已经用完的表. 这是一个很快的操作, 如果不是这样的话, 就应该确认磁盘空间是否已经满了或者磁盘是否正处于重负中
- `Connect Out`: 复制从服务器正在连接主服务器。
- `Copying to tmp table on disk`: 由于临时结果集大于 tmp_table_size , 正在将临时表从内存存储转为磁盘存储以此节省内存
- `Creating tmp table`: 正在创建临时表以存放部分查询结果
- `deleting from main table`: 服务器正在执行多表删除中的第一部分, 刚删除第一个表
- `deleting from reference tables`: 服务器正在执行多表删除中的第二部分, 正在删除其他表的记录
- `Flushing tables`: 正在执行 `FLUSH TABLES`, 等待其他线程关闭数据表
- `Killed`: 发送了一个 `kill` 请求给某线程, 那么这个线程将会检查 `kill` 标志位, 同时会放弃下一个 `kill` 请求. MySQL 会在每次的主循环中检查 kill 标志位, 不过有些情况下该线程可能会过一小段才能死掉. 如果该线程程被其他线程锁住了, 那么 kill 请求会在锁释放时马上生效
- `Locked`: 被其他查询锁住了
- `Sending data`: 正在处理 `SELECT` 查询的记录, 同时正在把结果发送给客户端
- `Sorting for group`: 正在为 `GROUP BY` 做排序
- `Sorting for order`: 正在为 `ORDER BY` 做排序
- `Opening tables`: 正尝试打开一个表. 这个过程应该会很快, 除非受到其他因素的干扰 -- 例如, 在执 `ALTER TABLE` 或 `LOCK TABLE` 语句行完以前, 数据表无法被其他线程打开
- `Removing duplicates`: 正在执行一个 `SELECT DISTINCT` 方式的查询, 但是 MySQL 无法在前一个阶段优化掉那些重复的记录. 因此, MySQL 需要再次去掉重复的记录, 然后再把结果发送给客户端
- `Reopen table`: 获得了对一个表的锁, 但是必须在表结构修改之后才能获得这个锁; 已经释放锁, 关闭数据表, 正尝试重新打开数据表
- `Repair by sorting`: 修复指令正在排序以创建索引
- `Repair with keycache`: 修复指令正在利用索引缓存一个一个地创建新索引. 它会比 `Repair by sorting` 慢些
- `Searching rows for update`: 正在讲符合条件的记录找出来以备更新. 它必须在 UPDATE 要修改相关的记录之前就完成了
- `Sleeping`: 正在等待客户端发送新请求
- `System lock`: 正在等待取得一个外部的系统锁; 如果当前没有运行多个 mysqld 服务器同时请求同一个表, 那么可以通过增加 `--skip-external-locking` 参数来禁止外部系统锁
- `Upgrading lock`: INSERT DELAYED 正在尝试取得一个锁表以插入新记录
- `Updating`: 正在搜索匹配的记录, 并且修改它们
- `User Lock`: 正在等待 `GET_LOCK()`
- `Waiting for tables`: 该线程得到通知, 数据表结构已经被修改了, 需要重新打开数据表以取得新的结构; 然后, 为了能的重新打开数据表, 必须等到所有其他线程关闭这个表. 以下几种情况下会产生这个通知: `FLUSH TABLES tbl_name`, `ALTER TABLE`, `RENAME TABLE`, `REPAIR TABLE`, `ANALYZE TABLE`, 或 `OPTIMIZE TABLE`
- `Send to client`: 客户端接收数据慢, 或者服务端的网络栈写满
- `Waiting for handler insert`: `INSERT DELAYED` 已经处理完了所有待处理的插入操作, 正在等待新的请求

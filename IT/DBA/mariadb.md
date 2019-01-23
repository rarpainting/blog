# MariaDB

## MariaDB SSL 开启

GRANT ALL PRIVILEGES ON *.* TO 'user'@'host' IDENTIFIED BY 'password' REQUIRE SSL;
在通过 SSL 认证下, 赋予某用户(user:password@host)所有权限(*.*)

## 配置文件

### General Log

```conf
general_log             = 1 # 最大文件数?
```

### Slow Log

```conf
slow_query_log # 开启慢查询日志
long_query_time     = 2 # 限定查询时间 2s , 界限这么宽 ?
log-queries-not-using-indexes
```

### Bin Log

```conf
sync_binlog = 0 # 事务提交后, 仅把 binlog_cache 中的数据写入 binlog 文件中, 但不执行 fsync 同步磁盘
```

## SQL 语句

### 获取有关数据库和表的信息

- SELECT DATABASE(); -- 当前数据库
- SHOW DATABASES; -- 所有数据
- SHOW TABLES; -- 当前数据库的表
- DESCRIBE table; -- 获得一个表列信息 === SHOW COLUMNS FROM table;

- SELECT column(s) FROM table ORDER BY RULE DESC; -- *倒序* 排序所获得的行
- SELECT column(s) FROM table1 {LEFT|RIGHT|INNER} JOIN table2 ON table1.columns { < | > | = | ... } table2.columns; -- 返回{包括左表的所有记录和右表符合联结关系的字段 | 包括右表的所有记录和左表符合联结关系的字段 | 两个表中符合联结关系的字段}
- SELECT column1(s) FROM table WHERE column2(s) IS NULL; -- 从 table 中找出能同时满足 column2(s) 是 NULL 字段的 column1(s) 行

- SQL SECURITY { DEFINER | INVOKER }: 指明谁有权限执行添加该信息的(view/trigger/function/procedure/event); DEFINER 表示按定义者拥有的权限执行, INVOKER 表示按调用者的权限来执行, 默认下 系统指定为 **DEFINER**

- PURGE {BINARY|MASTER} LOGS {TO 'log_name' | BEFORE datatime_expr}
  - 删除指定日志文件名或者日期之前的 **二进制** 文件
  - **注意**: 如果从站处于活跃状态且 **未从** 将删除的文件中读取内容, 则文件删除后, 从站将无法继续复制
  - RESET MASTER -- 删除所有日志文件; FLUSH LOGS -- 写入记录到磁盘
  - 所记录的二进制文件将在 **expire_logs_days** 后自动删除; 0 -- 关闭自动删除

- RESET reset_option[, reset_option] -- 用于清除各种服务器操作的状态, **重置** 服务器状态到初始状态

## MYSQL 数据目录

- performace_schema: 提供运行时检查服务器内部执行的信息

- sys: sys 模式, 提供了用于解释性能模式信息的一组对象

- mysql: 系统数据库
  - columns_stats: 引擎无关的表统计信息, 基于直方图的统计功能
  - columns_priv: **列** 级权限相关的信息 -- <a href="#permission">权限</a>
  - db: 与 **数据库** 权限相关的信息
  - event
  - func
  - general_log -- 常规日志(general log)内容
  - gitd_slave_pos -- 用于从属服务器的复制
  - help_category -- HELP 命令使用的表, 用于地理信息相关, 其他的 help 信息相关的还有 help_keyword, help_relation, help_topic
  - host -- 主机及其权限的表, 将在 MariaDB_10.4 版本去除
  - index_stats -- 存储与引擎无关的表统计信息, 索引相关(?)
  - innodb_index_stats -- 与 InnoDB 持久统计相关的信息, 包含每个索引的多个行
  - innodb_table_stats -- 与 InnoDB 持久统计相关的信息, 每个表一行
  - ndb_binlog_index -- 兼容 MySQL Cluster , 将在 MariaDB_10.4 版本去除
  - plugin -- 使用 INSTALL_SONAME, INSTALL_PLUGIN 或者 mysql_plugin 等插件列表; 不包含有关内置插件或使用 --plugin-load 选项加载的插件的信息
  - proc -- 包含存储过程和存储函数的信息, 该信息与 INFORMATION SCHEMA.ROUTINES 表中存储的信息相似
  - procs_priv -- 包含存储过程和存储函数 **权限** 的信息, 同时 INFORMATION SCHEMA.ROUTINES 派生自 mysql.procs_priv
  - roles_mapping -- 包含与角色(roles)相关的信息
  - servers -- 包含有关 Spinder/FEDERATED/FederatedX 存储引擎(Storage Engine)使用的服务器信息
  - slow_log -- 慢日志(slow log)内容
  - tables_priv -- 表级权限, 派生表 -- INFORMATION_SCHEMA.TABLE_PRIVILEGES 
  - table_stats -- 表统计信息相关
  - time_zone -- 与时区相关的信息
  - time_zone_leap_second -- 与时区相关的信息
  - time_zone_name -- 与时区相关的信息
  - time_zone_transition -- 与时区相关的信息
  - time_zone_transition_type -- 与时区相关的信息
  - transaction_registry -- **事务**精确版本控制
  - user -- 访问服务器上的用户的权限, 及其全局权限的信息

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
  - MAX_QUERIES_PER_HOUR
  - MAX_UPDATE_PER_HOUR
  - MAX_CONNECTIONS_PER_HOUR
  - MAX_USER_CONNECTIONS
  - MAX_STATEMENT_TIME
  
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

### General Log

### Slow Log

长时间的 **SQL 查询** 记录

- slow_query_log = 1
- 慢查询设置(log_query_time)时间级别是 微秒级
- 写入文件到 slow_query_log_file, 写入表到 mysql.slow_log
- log_queries_not_using_indexes -- 执行 不使用索引或者不限制行数 的查询都会被记录
- log_slow_admin_statements -- 设置存储缓慢的管理查询, 包括 ALTER TABLE / ANALYZE TABLE / CHECK TABLE / CREATE INDEX / DROP INDEX / OPTIMIZE TABLE / REPAIR TABLE
- log_slow_disabled_statements -- 禁止某些类型语句的记录(admin / call / slave / sp(存储过程))
- min_examined_row_limit -- 如果查询结果的行数超过该值, 则将该查询记录到 slow_query_log

#### 统计扩展(Extended Statistics)

- log_slow_rate_limit -- 限制实际写入慢查询的 行数/比例(?) ...
- log_slow_verbosity -- { (Empty) | query_plan | innodb | explain }
- log_slow_filter -- 通过 ',' 分割; 过滤器: 如果需要记录的查询同时 **匹配过滤器中的某类型** , 则将其记录到慢日志中:
  - admin / filesort / filesort_on_disk / filesort_priority_queue(>= MariaDB 10.3.1) / full_join / full_scan / query_cache / query_cache_miss / tmp_table / tmp_table_on_disk

### Bin Log

包含数据库 **所有** 更改(CREATE ALTER INSERT UPDATE DELETE)的记录(即使该记录对数据不影响)
是 **复制** 所必备的, 可用于备份后还原数据

- sql_log_bin = 1 -- 开启 二进制日志
- max_binlog_size -- 单日志文件限制尺寸
- 写入文件到 log_bin_basename, 通过 SHOW BINARY LOGS; 查询二进制日志文件列表

#### 复制时安全清除二进制文件

- 通过 SHOW BINARY LOGS 获取主服务器上的二进制日志文件列表
- 到从服务器上通过 SHOW SLAVE STATUS 检查每个从属服务器正在读取的二进制日志文件
- **找到从站正在读取的 最早 的日志文件, 删除在这之前的所有日志文件**
- 如果需要的话, 在删除日志文件之前, 备份日志文件

#### 选择性的记录

binlog_do_db/binlog_ignore_db -- 特意记录的数据库和需要忽略的数据库

- 不接受 ',' 分割的数据库, 多个设置则需要多次使用

#### 二进制日志格式

binlog_format = { STATEMENT | ROW | MIXED } -- 二进制日志格式

##### STATEMENT

记录所有对表或结构进行更改的 SQL 语句

##### ROW

记录影响各个表行的事件

##### MIXED

默认基于 **语句** 复制, 当确定语句对于 基于语句的复制可能不安全时, 将启用基于 **行** 的格式:

- INSERT-DELEYED
- 更新具有 **ATUO_INCREMENT 列** 的表并启用 **触发器(trigger)** 或者 **存储函数(stored function)**
- LOAD_FILE()
- ROW_COUNT() / FOUND_ROWS()
- USER() / CURRENT_USER()
- UUID()
- 其中有某些表是 mysql database 的 log table
- 语句使用了 **系统变量**; 如果与 会话范围(session scope) 一起用, 则可能会有例外
- 调用自定义的函数
- [MIXED](https://mariadb.com/kb/en/library/binary-log-formats/)

**注意**:

- 直接编辑 mysql 数据库, 则根据 binlog_format 执行日志记录
- 间接编辑 mysql 数据库, 则统一通过 **语句日志** 记录

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

- FLUSH TABLES WITH READ LOCK -- 并且获得读锁
- SHOW MASTER STATUS -- 获得 binlog 文件名
- 获得 文件(File)和位置(Position)
- 开始复制
- UNLOCK TABLES

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

### 低级实现及差别

- Windows: 使用 Windows 本地线程池
  - thread_pool_min_threads
- 类 Unix: 通用实现
  - thread_pool_size
  
## <<极客时间: MySQL 实战>>

[MySQL 的逻辑架构图](0d2070e8f84c4801adbfa03bda1f98d9.png)

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

优化 ast 执行树

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

[redo log](b075250cad8d9f6c791a52b6a600f69c.jpg)

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

[update 语句执行流程](2e5bff4910ec189fe1ee6e2ecc7b4bbe.png)

#### update 的实际执行流程:

- 先获取 `ID=2` 这一行
- 执行器拿到引擎的行数据, 在值之上 +1, 再通过引擎接口写入新数据
- 引擎将该数据更新到 内存 中, 同时记录到 `redo log` 中, 此时 `redo log` 处于 `prepare` 状态; 状态设置后告知 执行器 可以随时提交事务
- 执行器生成这个操作的 `binlog`, 并把 `binlog` 写入到 磁盘
- 执行器 调用 引擎 的提交事务接口, 引擎把 `redo log` 改成 `提交(commit)` 状态, 更新完成

#### 两阶段提交

用于保证提交都是成功的

`redo log`:
- `innodb_flush_log_at_trx_commit`: 每次事务的 `redo log` 都直接持久化到磁盘
- `sync_binlog`: 每次事务的 `binlog` 都直接持久化到磁盘

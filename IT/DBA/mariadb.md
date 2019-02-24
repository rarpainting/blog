# MariaDB

## MariaDB SSL 开启

GRANT ALL PRIVILEGES ON *.* TO 'user'@'host' IDENTIFIED BY 'password' REQUIRE SSL;
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
- `id`: (该连接?)的 SQL 执行顺序
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

- SHOW ENGINE innodb status; -- 最新一次记录的死锁日志(记录的是 **等待锁的 sql 语句** 记录, 而不是 完整事务 的 sql 记录)

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

错误日志

### General Log

mysql 接收到的每一个命令, 无论成功与否都记录下来
需要相当高的系统开销, 一般用于 **调试** 阶段

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

完整 Binlog 的标志:
- statement 格式: 以 COMMIT 结尾
- row 格式: 以 XID event 结尾(redo log 也有相应的 XID event)

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

`binlog_row_image`: { MINIMAL | FULL }:
- 默认为 FULL, 记录每一行的变更
- MINIMAL : 只记录影响后的行 (注: 腾讯云)


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
- mysql<8.0 / mariadb<?? 时默认为 1, 在 flush 脏页时会把相邻的脏页同时 flush , 可能会出现连锁反应, 适用于 SATA 等 IOPS 较低的设备
- mysql>=8.0 / mariadb>=?? 时默认为 0, 在 flush 脏页时只 flush 当前脏页然后返回, 适用于 SSD 等 IOPS 较高的设备, 减少 SQL 语句响应时间

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
select city,name,age from t where city='杭州' order by name limit 1000  ;
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

##### sort_buffer_size

mysql 为排序设置的 sort_buffer 大小

如果需要排序的数据下雨 sort_buffer_size, 排序会在内存中完成

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
- 对于没有主键的 InnoDB 表来说, 这个 rowid 就是由系统生成的长度为 6 字节的 rowid
- memory 引擎不是索引组织表, 在这个例子里面, 你可以认为它就是一个数组, 这个 rowid 其实就是数组的下标

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
  - 加锁的基本单位是 next-key lock(前开后闭区间)
  - 查找过程中 访问到的对象(索引, 行等) 才加锁
- 优化:
  - 索引上的 **等值查询** , 给 唯一索引 加锁的时候, next-key lock 退化为 **行锁**
  - 索引上的 **等值查询**, 向右遍历时且 (在一个 next-key lock 区间)最后一个值不等于等值条件 的时候, next-key lock 退化为 **间隙锁**
- BUG:
  - 唯一索引上的范围查询会访问到不满足条件的第一个值为止
  
- 锁是加载索引上的
- 如果目的是通过 `lock in share mode` 加 S 锁并且要求防止数据更新, 则需要同时考虑避开 覆盖索引 的优化(可能直接 `for update` 加 X 锁更好 ?)
- 由于索引都需要查询到不满足的条件, 所以建议尽量使用 **limit** , 访问安全且减小了加锁的范围
- *读提交(Read-Commit) 没有间隙锁(或者说间隙锁更小), 因此 debug 分析时逻辑更为清晰*

### 附: 杂记

#### inplace 与 online 的关系

- DDL 过程如果时 online , 就一定是 inplace ()
- 如果时 inplace 有可能不是 online(全文索引--FULLTEXT index 空间索引--SPATIAL index)

#### Update

如果 `update` 查询后要更改的值和目标值一样, InnoDB 依然会认真的执行该语句(加锁, 数据更新)

因为 MySQL 的隔离规则(可重复读 等), 在一个事务(08 | 事务到底是隔离的还是不隔离的?--图 5)中, 如果该语句(Update)不记录, 不执行
会直接影响事务后续的语句

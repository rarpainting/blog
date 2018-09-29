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



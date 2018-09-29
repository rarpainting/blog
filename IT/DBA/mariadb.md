# MariaDB

## MariaDB SSL 开启

GRANT ALL PRIVILEGES ON *.* TO 'user'@'host' IDENTIFIED BY 'password' REQUIRE SSL;
在通过 SSL 认证下, 赋予某用户(user:password@host)所有权限(*.*)

## 性能分析日志

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

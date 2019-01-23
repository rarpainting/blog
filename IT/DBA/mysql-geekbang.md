# MySQL 流程

1. 首先客户端通过 tcp/ip 发送一条 sql 语句到 server 层的 SQL interface
2. SQL interface 接到该请求后, 先对该条语句进行解析, 验证权限是否匹配
3. 验证通过以后, 分析器会对该语句分析, 是否语法有错误等
4. 接下来是优化器器生成相应的执行计划, 选择最优的执行计划
5. 之后会是执行器根据执行计划执行这条语句; 在这一步会去尝试 open table, 如果该 table 上有 MDL, 则等待; 如果没有, 则加在该表上加短暂的 MDL(S) (如果 `open_tables` 太大,表明 `table_open_cache` 太小; 需要不停的去打开 frm 文件)
6. 进入到引擎层，首先会去 `innodb_buffer_pool` 里的 data dictionary (元数据信息)得到表信息
7. 通过元数据信息,去 `lock info` 里查出是否会有相关的锁信息，并把这条 `update` 语句需要的锁信息写入到 lock info 里(锁这里还有待补充)
8. 然后涉及到的老数据通过快照的方式存储到 `innodb_buffer_pool` 里的 `undo page` 里, 并且记录 `undo log` 修改的 redo ; (如果 data page 里有就直接载入到 undo page 里, 如果没有, 则需要去磁盘里取出相应 page 的数据, 载入到 undo page 里)
9. 在 `innodb_buffer_pool` 的 `data page` 做 update 操作; 并把操作的物理数据页修改记录到 `redo log buffer` 里; 由于 `update` 这个事务会涉及到多个页面的修改, 所以 `redo log buffer` 里会记录多条页面的修改信息
因为 `group commit` 的原因, 这次事务所产生的 `redo log buffer` 可能会跟随其它事务一同 flush 并且 sync 到磁盘上
10. 同时修改的信息, 会按照 `event` 的格式, 记录到 `binlog_cache` 中; (这里注意 `binlog_cache_size` 是 `transaction` 级别的, 不是 `session` 级别的参数, 一旦 `commit` 之后, dump 线程会从 `binlog_cache` 里把 event 主动发送给 slave 的 I/O 线程)
11. 之后把这条 sql , 需要在二级索引上做的修改，写入到 `change buffer page`，等到下次有其他 sql 需要读取该二级索引时，再去与二级索引做 merge (随机 I/O 变为顺序 I/O, 但是由于现在的磁盘都是 SSD, 所以对于寻址来说, 随机 I/O 和顺序 I/O 差距不大)
12. 此时 update 语句已经完成, 需要 commit 或者 rollback; 这里讨论 commit 的情况，并且双 1
13. commit 操作，由于存储引擎层与 server 层之间采用的是内部 XA (保证两个事务的一致性,这里主要保证 `redo log` 和 `binlog` 的原子性), 所以提交分为 `prepare` 阶段与 `commit` 阶段
14. `prepare` 阶段,将事务的 xid 写入, 将 `binlog_cache` 里的进行 flush 以及 sync 操作(大事务的话这步非常耗时)
15. commit 阶段, 由于之前该事务产生的 `redo log` 已经 sync 到磁盘了; 所以这步只是在 `redo log` 里标记 `commit`
16. 当 binlog 和 redo log 都已经落盘以后, 如果触发了刷新脏页的操作, 先把该脏页复制到 `doublewrite buffer` 里, 把 `doublewrite buffer` 里的刷新到共享表空间, 然后才是通过 `page cleaner` 线程把脏页写入到磁盘中

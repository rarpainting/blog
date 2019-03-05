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

## 19 -- @某/人

版本5.7.13
rc模式下:
session 1:
begin;
select * from t where c=5 for update; 
session 2:
delete from t where c=10; --等待
session 3:
insert into t values(100001,8); --成功
session 4:
update t set c=100 where id=10; --成功
session 1:
commit
session 2:事务执行成功

rr模式下:
begin;
select * from t where c=5 for update; 
session 2:
delete from t where c=10 --等待
session 3:
insert into t values(100001,8) --等待
session 4:
update t set c=100 where id=10; --成功
session 1:
commit
session 2:事务执行成功
session 3：事务执行成功

从上面这两个简单的例子,可以大概看出上锁的流程
不管是 rr 模式还是 rc 模式,这条语句都会先在 server 层对表加上 MDL S 锁,然后进入到引擎层

rc模式下, 由于数据量不大只有10W. 通过实验可以证明 session 1 上来就把该表的所有行都锁住了
导致其他事务要对该表的所有现有记录做更新, 是阻塞状态. 为什么 insert 又能成功 ?
说明 rc 模式下 for update 语句没有上 gap 锁, 所以不阻塞 insert 对范围加插入意向锁, 所以更新成功
session 1 commit 后, session 2 执行成功. 表明所有行的 X 锁是在事务提交完成以后才释放

rr模式下, session 1 和 session 2 与 rc 模式下都一样, 说明 rr 模式下也对所有行上了 X 锁
唯一的区别是 insert 也等待了, 是因为 rr 模式下对没有索引的更新, 聚簇索引上的所有记录, 都被加上了 X 锁. 其次, 聚簇索引每条记录间的间隙(GAP), 也同时被加上了 GAP 锁. 由于 gap 锁阻塞了 insert 要加的插入意向锁, 导致 insert 也处于等待状态. 只有当 session 1 commit 完成以后. session 1 上的所有锁才会释放,S2,S3 执行成功

由于例子中的数据量还比较小, 如果数据量达到千万级别, 就比较直观的能看出, 上锁是逐行上锁的一个过程. 扫描一条上一条, 直到所有行扫描完, rc 模式下对所有行上 X 锁. rr 模式下不仅对所有行上 X 锁, 还对所有区间上 gap 锁. 直到事务提交或者回滚完成后, 上的锁才会被释放

## 30 -- IceGeek17

问题一：
对于文中的第一个例子（不等号条件里的等值查询），当试图去找 “第一个id<12的值"的时候，用的还是从左往右的遍历（因为用到了优化2），也就是说，当去找第一个等值的时候（通过树搜索去定位记录的时候），即使order by desc，但用的还是向右遍历，当找到了第一个等值的时候（例子中的id=15），然后根据order by desc，再向左遍历。
是不是这么理解？

问题二：
对于第21讲的思考题， select * from t where c>=15 and c<=20 order by c desc lock in share mode， 老师已经给出了答案，我这里再详细理解一下：
先定位索引c上最右边c=20的行，所以第一个等值查询会扫描到c=25，然后通过优化2，next-key lock退化为间隙锁，则会加上间隙锁（20，25），紧接着再向左遍历，会加 next-key lock (15, 20], (10, 15], 因为要扫描到c=10才停下来，所以也会加next-key lock (5,10]
理解的是否正确？

问题三：
对于上面的问题二的sql，在索引c上，把（10，25）这段区间锁上就应该是完备的了，理论上（5，10]这段区间是否锁上对结果应该没有影响呀。
是不是说MySQL就是这么实现的，next-key lock前开后闭，因为扫到了c=10，所以会加next-key lock (5,10]，这里MySQL的实现扩大了锁的区间范围，其实没有这个必要？

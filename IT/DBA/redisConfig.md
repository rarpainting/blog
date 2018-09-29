# redis 配置 -- 2.4.5
## 常用配置
- daemonize  -- 设置为守护进程

- pidfile -- 在守护进程下， redis会将 PID 写入 pidfile 中， 可配置写入特定文件

- unixsocket(perm) -- 服务器监听某特定端口， 默认不开启

- loglevel -- 记录文档级别(debug, verbose, notice, warning)， 默认 verbose 级别

- logfile -- 记录输出， 默认 stdout
- syslog-enabled -- 输出记录到系统记录器
- syslog-ident -- 系统记录器中的身份
- syslog-facility -- 系统记录器中的级别， 默认 local0

## 快照配置 -- RDB模式
- rdbcompression -- redis 保存数据到磁盘时默认进行 LZF 压缩， 通过该设置取消
- dbfilename/dir -- redis 保存文件名/路径

## (主从)复制配置
- slaveof [masterip] [masterport] -- 设置 redis 为slav服务， 以及 redis 的同步对象数据库
- masterauth [master-password]

- slave-serve-stale-data -- 当 redis 与主数据库的连接断开， 设置是否正常响应客户端的请求：(1)yes-任然响应请求， 而且(*possibly with out of data data, or data set may just be empty if this is the first synchronization*.);(2)no-除了 INFO 和 SLAVEOF 请求，任何请求返回一个异常
- repl-ping-slave-period -- slav 向 server 发送 PING 的周期
- repl-timeout -- 超时时间， 需要比 repl-ping-slave-period 更大(?)

## 安全配置
- rename-command -- 重命名命令

## 内存策略
- maxmemroy-policy -- 内存策略（内存不足时，任何策略都会返回错误）
	- volatile-lru(default) -- 通过 LRU 算法删除过期集中的某个键
	- allkeys-lru -- 通过 LRU 删除任意一个键
	- volatile-random -- 随机删除过期集中的某个键
	- allkeys-random -- 随机删除任意一个键
	- volatile-ttl -- 通过(minor-ttl)删除最近过期的键
	- noeviction -- 不删除键，仅返回错误

不同策略的经验规则 [一般情况下，allkeys-lru 是建议的选项]
> allkeys-lru：请求符合幂律分布(部分子集的访问比其他元素的多)
> allkeys-random：经常性的循环访问，或希望请求分布平稳
> volatile-ttl：通过对缓存对象设置 TTL ， 决定哪些对象应该被过期
> allkeys-lru + volatile-ttl：**缓存**部分实例的同时，**持久化**部分键

- maxmemory-samples -- 由于LRU 算法和(minimal TTL)都是近似算法， 通过设定样本数(sample size)进行检查配置

## 唯一追加模式 -- AOF模式
	前言:由于 Redis 默认采用异步的方式将数据存放到磁盘上，在断电时会丢失不同程度的数据，AOF模式使用 fsync策略 能够保证在掉电后仅丢失 1s 的数据；默认下 AOF 是关闭的，开启后 redis 会自动加载 AOF

- appendfsync -- AOF 策略
	- always -- 每次数据都写到磁盘
	- no -- 不同步，由系统决定何时写入到磁盘
	- everysec -- 每秒同步一次
- no-appendfsync-on-rewrite -- always / everysec 可能导致大量的磁盘读写(fsync)， 设置 yes 在 BGSAVE 或 BGREWRITEAOF 时，阻止主进程调用 fsync 
- auto-aof-rewrite-percentage(min-size) -- 以**最近一次**被重写的日志结果为基础，当AOF日志以(指定百分比)增长时，redis 触发 BGREWRITEAOF 以一个(指定最小尺寸)重写日志文件， （百分比为零默认禁用自动重写）
## SLOWLOG -- 记录查询执行时间的日志系统
	slow log 运行在内存中

- slowlog-log-slower-than (ms) -- 记录**执行时间**大于该设置的事务
- slowlog-max-len -- slowlog -- 最多保存的日志数
## 虚拟内存配置
	...
## ADVANCED CONFIG
Hash 储存格式
>成员较少 -- 采用类似一维数组的方式紧凑存储， value 的 redisObject 的 encoding 为 zipmap 
>超过某界限 -- 以真正的 HashMap 记录， encoding 为 HT

list 存储格式
>去指针的紧凑存储格式

set 存储格式
>去指针的紧凑存储格式

zset 存储格式
>紧凑存储格式

# Redis
## 1.事务处理
###Watch -- 乐观锁(CAS)

**{ watch key1 [key2 [key3...]] }**
被监视 watch 的锁在 exec 之前被修改 -- 该事务被取消, exec 返回 nil-reply
## 2.大量数据插入 -- ≥2.6

```
非 pipe mode
cat text | redis-cli
作为事务处理

pipe mode
cat text | redis-cli --pipe
返回All data transferred. Waiting for the last reply...
	Last reply received from server.
	errors: 0, replies: 1000000
```

## 3.分区
### 1)分区理论 -- 分区规则所在
#### 客户端分区
#### 代理分区
#### 查询路由

客户端--> redis 实例 --> redis 节点
#### 重要结论如下:

- 如果Redis被当做缓存使用, 使用一致性哈希实现动态扩容缩容;
- 如果Redis被当做一个 持久化存储 使用, 必须使用 固定的keys-to-nodes 映射关系, **节点的数量一旦确定不能变化**; 否则的话(即 Redis节点 需要动态变化的情况), 必须使用可以在**运行时进行数据再平衡**的一套系统, 而当前只有 **Redis集群** 可以做到这样

### 2)redis 分区实现

#### redis 集群

#### Twemproxy

#### 支持一致性哈希的客户端
- golang -- [**Radix**] [**Redigo**] [GO-Redis/redis]
- python -- [*aioredis*] [*asyncio_redis*] [**redis-py**]
- nodeJS -- [**ioredis**] [**node_redis**] [*thunk_redis*]
- C -- [**hiredis**]
- C++ -- [*async-redis*] [C+redis+client(base on hiredis)]

### 3)理论补充 ------ 一致性哈希(consistent hashing)

#### 在动态变化的Cache环境中,  哈希算法的优劣判断

- 平衡性
- 单调性
- 分散性
- 负载性能

## 4.redis 分布式锁 --- redlock

	目标的分布式锁应该有三种特性:
	1.安全属性 -- 互相排斥
	2.活性A:无死锁
	3.活性B:容错

### 1) 单 redis 实例实现分布式锁

```redis
SET resourceName myRandomValue PX (ms) NX
```

### 2) lua 脚本

复杂操作的原子性(事务)


### 3) redlock 算法

条件: 每个 master 节点都独立, 并且不存在主从复制

步骤:
1. 获取当前 unix 时间
2. 依次尝试从 N 个实例,  使用**相同的 key 和 全局唯一值**,  获取锁
3. 当且仅当从 **多数** 的 redis 中取得锁,  取锁时间 **小于** 锁失效时间,  锁才算获取成功
4. key 真正有效时间 = 锁失效事件 - 取锁时间
5. 当判断取锁失败(不能拿到 N/2+1 个 Reids 实例的锁 or 取锁时间超过了有效时间) ==> 客户端解除 **所有(无论是否加锁成功)** Redis 实例上占用的锁

- 缺点: 本身以 timestamp 作为 value , 对分布式环境的时间过于严格
- 解决: **value=unique_client_id+thread_id+timestamp**

#### 细节讨论

1. 活性安全特性
- server: key 失效导致的锁自动释放
- client: 客户端自动释放锁
- client: 客户端要求再次获取锁, 需要等待时间(该时间必须 **大于成功获取锁所需的时间** )
2. 性能,  崩溃恢复和 Redis
3. 锁的扩展

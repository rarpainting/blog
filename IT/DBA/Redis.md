# Redis

## 内存

### 内存使用统计

| 属性名                    | 属性说明                                                      |
| :-                        | :-                                                           |
| `used_memory`             | Redis 分配器分配的内存总量, 即 内部存储的所有数据的内存占用量 |
| `used_memory_human`       | 以可读的格式返回 `used_memory`                                |
| `used_memory_rss`         | 从 **操作系统** 的角度显示 Redis 进程占用的物理内存总量       |
| `used_memory_peak`        | 内存使用的最大值, 表示 `used_memory` 的峰值                   |
| `used_memory_peak_human`  | 同 `used_memory_human`                                        |
| `used_memory_lua`         | Lua 引擎所消耗的内存大小                                      |
| `mem_fragmentation_ratio` | `used_memory_rss`/`used_memory` 的比值, **内存碎片率**        |
| `mem_allocator`           | Redis 使用的内存分配器, 默认为 `jemalloc`                     |

> 注: 1) 当 mem_fragmentation_radio > 1 时, 说明 used_memory_rss - used_memory 多出的部分内存并没有用于数据存储, 而是被内存碎片所消耗, 如果两者相差很大, 说明碎片率严重
> 2) 当 mem_fragmentation_ratio < 1 时, 折衷情况一般出现在操作系统把 Redis 的内存交换 (Swap) 到硬盘导致, 由于硬盘速度远远慢于内存, Redis 性能会变得很差, 甚至僵死

### 内存消耗划分

只要包括: 自身内存 + 对象内存 + 缓冲内存 + 内存碎片

注意:
- 数据对齐
- 安全重启 -- 重启节点以便重新整理碎片

#### 自身内存

TODO

#### 对象内存

对象内存时 Redis 内存占用最大的一块

#### 缓冲内存

#### 内存碎片

### 子进程内存消耗

- 关闭 linux Transparent Huge Pages(THP) 机制, 防止 copy-on-write 期间内存过都消耗
- 预留部分内存以供 Redis 的子进程产生
- 设置 sysctl `vm.overcommit_memory = 1` 允许内核可以分配所有的物理内存, 防止 Redis 在 fork 时因系统剩余内存不足而失败

### 动态调整内存上限

- Redis 默认无限使用服务器内存, 为防止极端情况下导致的 系统内存耗尽, 建议为所有的 Redis 进程都要配置 `maxmemory`

### 内存回收策略

Redis 内存回收体现在:
- 删除到达过期时间的键对象
  - 惰性删除 -- 当客户端读取带有超时属性的键时, 如果已经超过键设置的过期时间, 则执行删除操作并返回空
  - 定时任务删除 -- 带 快/慢 速率的自适应算法
- 内存使用达到 `maxmemory` 上限时触发 内存溢出控制策略(maxmemory-policy)
  - neoviction
  - volatile-lru
  - allkeys-lru
  - allkeys-random
  - volatile-random
  - volatile-ttl

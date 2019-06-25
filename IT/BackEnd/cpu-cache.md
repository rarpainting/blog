# CPU 缓存和伪共享

CPU 的缓存是按缓存行组成的, 典型的是 64 字节

## CPU 缓存共享

1. 请求核直接访问目标核, 但是每次的跨核访问都会经过有限带宽的 **Memory Controller**
2. 缓存行的传输, 需要能获取目标行是否被修改

## MESI 协议

- M(修改, Modified): 本地处理器已经修改缓存行, 即行已脏; 此 cache 内容已经与内存中的不一样, 并且此 cache, 只有本地一个拷贝(专有)
- E(专有, Exclusive): 缓存行内容和内存中的一样, 并且其他处理器**都没有**这行数据
- S(共享, Shared): 缓存行内容和内存中的一样, 有可能其他处理器**也存在**此缓存的拷贝
- I(无效, Invalid): 缓存行失效, 不能使用

缓存行的四种状态转换规则:
- 初始: 缓存行没有加载任何数据, 即 I(无效)
- 本地读写
  - 本地写(Local Write): 如果本地处理器写数据至处于 I(无效) 状态的缓存行, 则缓存行的状态变成 M(修改)
  - 本地读(Local Read):
    - 其他处理器的缓存中*也没有*这行数据, 则从内存(内存可能再向磁盘请求)中加载该行数据, 并设为 E(专有)
    - 其他处理器中有此缓存行的状态, 则读进本地缓存, (无论之前缓存行状态是 S 或者 E)并设置该行数据为 S(共享)
- 远程读写
  - 远程读(Remote Read): 目标核通过**内存控制器(Memory Controller)**将请求核的目标数据发送过去, 将该数据设为 S(共享), 同时内存也缓存了该行数据
  - 远程写(Remote Write): 假设请求核知道(?)其他核也有目标数据的备份, 为了修改该数据, 请求核发出一个 **RFO(Request For Owner)请求**, (以需要拥有这行数据的权限), 其他核的相应缓存将设为 I(无效) -- 保障了数据安全, 同时处理 PFO 请求/设置 I(无效)带来了巨大的性能消耗

## 各级 IO 延迟数字

来源: [CPU性能和CACHE](https://plantegg.github.io/2021/07/19/CPU%E6%80%A7%E8%83%BD%E5%92%8CCACHE/)

### Cache、内存、磁盘、网络的延迟比较

假设主频2.6G的CPU，每个指令只需要 0.38ns

- 每次内存寻址需要 100ns
- 一次 CPU 上下文切换（系统调用）需要大约 1500ns, 也就是 1.5us（这个数字参考了这篇文章, 采用的是单核 CPU 线程平均时间）
- SSD 随机读取耗时为 150us
- 从内存中读取 1MB 的连续数据, 耗时大约为 250us
- 同一个数据中心网络上跑一个来回需要 0.5ms
- 从 SSD 读取 1MB 的顺序数据, 大约需要 1ms （是内存速度的四分之一）
- 磁盘寻址时间为 10ms
- 从磁盘读取 1MB 连续数据需要 20ms
- 如果 CPU 访问 L1 缓存需要 1 秒，那么访问主存需要 3 分钟、从 SSD 中随机读取数据需要 3.4 天、磁盘寻道需要 2 个月，网络传输可能需要 1 年多的时间

**延迟数字对比表:**

| Work | Latency |
| :--- | :--- |
| L1 cache reference | 0.5 ns |
| Branch mispredict	| 5 ns |
| L2 cache reference | 7 ns |
| Mutex lock/unlock | 25 ns |
| Main memory reference | 100 ns |
| Compress 1K bytes with Zippy | 3,000 ns |
| Send 1K bytes over 1 Gbps network | 10,000 ns |
| Read 4K randomly from SSD* | 150,000 ns |
| Read 1 MB sequentially from memory | 250,000 ns |
| Round trip within same datacenter | 500,000 ns |
| Read 1 MB sequentially from SSD* | 1,000,000 ns |
| Disk seek | 10,000,000 ns |
| Read 1 MB sequentially from disk | 20,000,000 ns |
| Send packet CA->Netherlands->CA | 150,000,000 ns |

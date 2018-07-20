# Lockfree programming

## 非阻塞同步(Non-blocking Synchronization)

- Wait-free

- Lock-free

- Obstruction-free

## 加锁的层级

复杂程度:

Lock-based < Lockless-based < Atomic < Obstruction-free < Lock-free < Wait-free

加锁粒度:

Wait-free < Lock-free < Obstruction-free < Atomic < Lockless-based < Lock-based


## Lock-free

### Spin Lock

Lock 操作被阻塞时, 不是进入等待队列, 而是死循环 CPU 空转等待其他线程释放锁

### Seqlock(CAS) -- 写者优先级更高

写操作会获得一把锁, 同时锁的序列值 +1 , 当读者读取数据之前和之后都读取该序列值, 如果读取前后的值相同, 则代表写没有发生, 否则放弃当前读操作, 进入下一次循环

注意: seqlock不阻止 Read 操作, 要小心注意写者修改数据结构时, 读者在访问一个无效的指针

### 环形缓冲区

# Nginx

Nginx 定义了一种基础类型的模块: 核心模块, 模块类型-NGX_CORE_MODULE, 目地是使非模块框架值关注于如何于 6 个核心模块通信

事件模块/HTTP 模块/mail 模块共性: 各有一个模块作为代言人, 并在同类模块中由 1 个作为核心业务于管理功能的模块

事件模块使 HTTP 模块和 mail 模块的基础

## Nginx 核心进程模型

### Master 进程

- 监控进程 - 进程组与用户的交互接口, 同时对进程进行监控
- 不处理网络事件, 不负责业务的执行
- 管理 worker 进程来实现 **重启服务/平滑升级/更换日志文件/配置文件实时生效**
- master 的 `for(;;)` 中的 `sigsuspend()` 负责挂起进程, 直到进程接收到信号

### Worker 进程

- 处理基本的网络事件
- Worker 之间的进程是对等的, 只可能在相同的 Worker 中处理
- 更多的 Worker 数, 只会导致进程相互竞争 CPU 资源, 从而带来不必要的上下文切换, 因而 Worker 数一般与 CPU 数对等

### 处理过程

- 略

#### 惊群现象

当一个连接进来后, 所有的 `accept` 在这个 socket 上面的进程, *都会* 收到通知, 然而只有一个进程能处理(`accept`)该连接, 其他的则 `accept` 失败

Nginx 对惊群现象的处理:

一个 worker 进程在 accept 这个连接之后, 所有的 连接/解析/处理/数据产生/数据返回/断开连接 都在一个连接中解决

### Nginx 事件处理机制

- 异步非阻塞的事件处理机制

### Nginx 与 Apache

- Apache 每个请求都独占一个线程, 抗压不行, 在 PHP 处理慢或者前端压力大的情况下, 会出现 Apache 进程数飙升而导致拒绝服务
- Nginx 一个进程只有一个主线程, 通过异步非阻塞的事件处理机制, 实现循环处理
- Nginx 网络模式是事件驱动(select/poll/epoll)
- Nginx 做前端负责进行抗并发/反向代理、负载均衡、静态文件缓存, 后端采用 Apache 处理动态请求

### 事件驱动

#### kernel 处理事件

- 数据量少: 每个数据包产生一次中断, kernel 响应中断并放进对应协议幀
- 数据量大: 启动 **interrupt coalescing** 机制, 让网卡做中断合并, *足够多的数据包 或者 等待一个 timeout* 才会产生一个中断, 随后 kernel 一次性处理
- 庞大的数据量: 轮询处理模式

#### select

`poll` 的实现与 `select` , 主要的不同是 `poll` 免去了对文件描述符的限制

#### epoll

工作模式:
- LT(level-triggered): 默认, 支持 block 和 non-block select
  - 内核提示 fd 就绪, 如果本次读取后缓冲区仍然未空, 则该 fd 继续为 就绪态
  - 内核的提示信息会一直发送, 直至缓冲区为空, fd 转为 未就绪态
- ET(edge-triggered): 仅支持 non-blocking socket
  - 内核提示 fd 就绪, 如果本次读取后无论缓冲区是否为空, 缓冲区都会被清空, 同时该 fd 转为 未就绪态
  - 内核对 fd 就绪消息只有一次(only once)

```c
// 创建最大支持 size 的 epoll fd
int epoll_create(int size);

// epfd:
// op: 对 fd 的操作
//  EPOLL_CTL_ADD
//  EPOLL_CTL_MOD: 修改已注册的 fd 的监听事件
//  EPOLL_CTL_DEL
// fd: 需要操作的 描述符
// event:
//  EPOLLIN: 表示对应的文件描述符可以读(包括对端 SOCKET 正常关闭)
//  EPOLLOUT: 表示对应的文件描述符可以写
//  EPOLLPRI: 表示对应的文件描述符有紧急的数据可读(这里应该表示有带外数据到来)
//  EPOLLERR: 表示对应的文件描述符发生错误
//  EPOLLHUP: 表示对应的文件描述符被挂断
//  EPOLLET: 将 EPOLL 设为边缘触发(Edge Triggered)模式, 这是相对于水平触发(Level Triggered)来说的
//  EPOLLONESHOT(2.6.2): 只监听一次事件, 当监听完这次事件之后, 如果还需要继续监听这个 socket 的话, 需要再次把这个 socket 加入到 EPOLL 队列里
//  EPOLLWAKEUP(3.5): 系统在 自动休眠 下, 当唤醒设备的事件发生时, 设备驱动会保持唤醒状态, 直到事件进入排队状态
//       此时如果需要 **保持设备唤醒状态** , 则设置 EPOLLWAKEUP , 系统将保持设备持续唤醒, *从 epoll_wait 调用开始, 直到下一次 epoll_wait 调用*
//  EPOLLEXCLUSIVE(4.5): 用于避免 惊群效应
//       如果 多个具有 EPOLLEXCLUSIVE 标签和多个没有 EPOLLEXCLUSIVE 的 epoll 实例同时监听同一事件, 则会向未指定 EPOLLEXCLUSIVE 的所有 epoll 实例以及至少一个指定 EPOLLEXCLUSIVE 的 epoll 实例提供事件
// return : ok - 0/error -- -1
//       EPOLLEXCLUSIVE 一般与这些标志一起使用: EPOLLIN, EPOLLOUT, EPOLLWAKEUP, and EPOLLET.
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

// epfd:
// events: 用于存放已就绪的事件列表
// maxevents: events 的可容纳大小, 即一次处理的事件数
// return : 已就绪的事件数
int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);
```

#### kqueue

### libuv

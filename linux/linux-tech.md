# Linux

## fork / vfork / clone

三者底层调用的都是 `do_fork`

| 系统调用 | 描述                                                                                                                        |
| :-       | :-                                                                                                                          |
| fork     | 父进程的完整副本, 复制了父亲进程的所有资源(已打开的描述符, namespace, 地址空间/内存数据), 包括 `task_struct` 等进程管理结构 |
| vfork    | 复制 `task_struct` 等进程管理结构, 与父进程共享数据段(data segment) , 且 vfork 出来的子进程会先于父进程运行                 |
| clone    | 真正用于创建 进程/线程(轻量级进程) 的系统调用(syscall)                                                                                          |

### copy-on-write/写时复制

完整的 fork 行为需要:
- 为子进程的页表分配页帧
- 为子进程的页分配页帧
- 初始化子进程的页表
- 把父进程的页复制到子进程相应的页中

为了规避这些耗时的操作, linux 采用 copy-on-write 的策略来进行 fork 行为:
- fork 后, 子进程仅复制了父进程的 `task_struct` 等必要的数据
- 一开始, 数据未被修改, 则父进程与子进程共享同一片页帧; 此时页帧被共享, 不能被修改, 即页帧被保护
- 子进程/父进程 尝试写入页帧内容, 则产生一个异常(page fault int14) , 此时内核将该页复制到一个新的页帧中并标记为 可写 , **原来的页帧依然被标注为写保护**; 如果后续内核捕捉异常, 检查页帧由当前 唯一属主 修改, 则将该页标记为 可写
- 异常处理 `do_wp_page()`: 对触发异常的页帧进行取消共享, 为写操作复制一份新的物理页面; 异常处理结束后, 重新执行写操作并继续运行程序

### vfork

- vfork 在 `exec` 之前只复制 `task_struct` 等必要数据, 彻底共享父进程数据; `exec` 后, 通过前一个 `task_struct` 创建新的进程上下文, 堆栈
- 由 vfork 创造出来的子进程还会导致 **父进程挂起** , 除非子进程 `exit` 或者 `execve` 才会唤起父进程
- 由 vfork 创建出来的子进程共享了父进程的所有内存, 包括栈地址, 直至子进程使用 `execve` 启动新的应用程序为止
- 由 vfork 创建出来得子进程应该使用 `_exit()` 函数来退出, 而不应该使用 `return` 或者 `exit()` 退出, 否则会导致 父进程 继续创建 子进程

### clone

| 标志                      | 含义                                                                                 |
| `CLONE_PARENT`(2.3.12)    | 创建的子进程的父进程是调用者的父进程, 新进程与创建它的进程成了 **兄弟** 而不是"父子" |
| `CLONE_FS`(2.0)           | 子进程与父进程共享相同的文件系统, 包括 root, 当前目录, umask                         |
| `CLONE_FILES`(2.0)        | 子进程与父进程共享相同的文件描述符(file descriptor)表                                |
| `CLONE_NEWNS`(2.4.19)	    | 在新的 namespace 启动子进程, namespace 描述了进程的文件 hierarchy                    |
| `CLONE_SIGHAND`(2.0)      | 子进程与父进程共享相同的信号处理(signal handler)表                                   |
| `CLONE_PTRACE`(2.2)	    | 若父进程被 trace, 子进程也被 trace                                                   |
| `CLONE_VFORK`(2.2)        | 父进程被挂起, 直至子进程释放虚拟内存资源                                             |
| `CLONE_VM`(2.0)           | 子进程与父进程运行于相同的内存空间                                                   |
| `CLONE_PID`(!obsolete!)   | 子进程在创建时 PID 与父进程一致                                                      |
| `CLONE_THREAD`(2.4.0)	    | 支持 POSIX 线程标准, 子进程与父进程共享相同的线程群                                  |

```c
int clone(int (fn)(void ), void *child_stack, int flags, void *arg);
```

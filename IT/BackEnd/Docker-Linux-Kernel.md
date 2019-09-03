# Docker & Linux Kernel

## Linux

### clone

```C
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

- child_func: 传入子进程运行的程序主函数
- child_stack: 传入子进程使用的栈空间
- flags: 表示使用哪些 `CLONE_*` 标志位
- args: 则可用于传入用户参数

### setns

```C
fd = open(argv[1], O_RDONLY);   /* 获取namespace文件描述符 */
setns(fd, 0);                   /* 加入新的namespace */
execvp(argv[2], &argv[2]);      /* 执行程序 */
```

### unshare -- 运行在原进程上

```C
int unshare(int flags);
```

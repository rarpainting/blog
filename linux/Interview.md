# Interview

## 僵尸进程

- 子进程在死亡后(PCB 等资源未被释放)需要被父进程 wait 捕获他们的状态
- 如果父进程没有 wait 而子进程结束, 会导致子进程的资源一直不释放(僵尸态)
- 如果父进程没有 wait 而结束进程, 子进程会被托管到 pid 为 1 的进程(`/sbin/init` OR `/lib/systemd/systemd`)

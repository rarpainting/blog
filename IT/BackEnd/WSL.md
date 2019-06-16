# WSL

## 特点

- 一套独立于 win32 子系统 / posix 子系统 / os/2 子系统的 linux 子系统
- 介于 user-syscall 和 windows kernel 之间
- 可直接运行原生 linux 程序
- linux 子系统逻辑上处于 picoprocess , 与外界(win32 子系统等)隔离, 不能直接调度 win32 的 **驱动** , 因此 **不能** 与 win32 文件系统直接交互, **不能** 调度 win32 的 GPU 驱动
- 但是 UoW 默认是把 win32 的文件系统挂载在 UoW 的 /mnt 上, 以提供两者的文件系统交互

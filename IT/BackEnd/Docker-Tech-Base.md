# Docker 技术基础

## Namespace

命名空间可用于分离进程树, 网络接口, 挂载点以及进程间通讯等资源

Linux 提供了以下 7 种不同的命名空间

- CLONE_NEWCGROUP
- CLONE_NEWIPC
- CLONE_NEWNET
- CLONE_NEWNS
- CLONE_NEWPID
- CLONE_NEWUSER
- CLONE_NEWUTS

在 Linux 系统中有两个特殊进程

- /sbin/init -- 负责执行内核部分初始化工作和系统配置
- [kthreadd] -- 内核进程, 负责管理和调度其他进程



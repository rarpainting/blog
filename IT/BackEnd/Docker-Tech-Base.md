# Docker 技术基础

## Namespace

命名空间可用于分离进程树, 网络接口, 挂载点以及进程间通讯等资源

Linux 提供了以下 7 种不同的命名空间

- `CLONE_NEWCGROUP`: CGroup root directory(CGroup 根目录)
- `CLONE_NEWIPC`: System V IPC (信号量, 消息队列, 共享内存) 和 POSIX message queues
- `CLONE_NEWNET`: network devices, stacks, port etc (网络设备/虚拟网络, 网络栈, 端口)
- `CLONE_NEWNS`: Mount points(文件系统挂载点)
- `CLONE_NEWPID`: Process IDs(进程编号)
- `CLONE_NEWUSER`: User & group IDs(用户和用户组)
- `CLONE_NEWUTS`: Hostname & NIS domain name(主机名与 NIS 域名)

在 Linux 系统中有两个特殊进程

- /sbin/init -- 负责执行内核部分初始化工作和系统配置
- [kthreadd] -- 内核进程, 负责管理和调度其他进程


## 网络

Docker 的网络模式
- Host
- Container
- None
- Bridge

## CGoup(Control Group)

CGroup 为一组进程分配(CPU 内存 网络等宽等)资源

功能:
- Resource Limiting
- Prioritization
- Accounting
- Control

### CGroup 支持的文件类型

|文件名|R/W|用途|
|:-|:-:|:-|
|Release_agent       |RW|删除分组时执行, 该文件只存在于 根分组|
|Notify_on_release   |RW|设置是否执行 release_agent|
|Tasks               |RW|属于分组的线程 TID 列表|
|CGroup.procs        |R |属于该分组的进程 PID . 仅包括多线程进程的线程 leader 的 TID (这点与 tasks 不同)|
|CGroup.event_control|RW|监视状态变化和分组删除事件的配置文件|

![cgroup 层级](cgroup-img001.png)

### CGroup 子系统

- blkio -- 这个子系统为块设备设定输入/输出限制, 比如物理设备(磁盘, 固态硬盘, USB 等等)
- cpu -- 这个子系统使用调度程序提供对 CPU 的 cgroup 任务访问
- cpuacct -- 这个子系统自动生成 cgroup 中任务所使用的 CPU 报告
- cpuset -- 这个子系统为 cgroup 中的任务分配独立 CPU（在多核系统）和内存节点
- devices -- 这个子系统可允许或者拒绝 cgroup 中的任务访问设备
- freezer -- 这个子系统挂起或者恢复 cgroup 中的任务
- memory -- 这个子系统设定 cgroup 中任务使用的内存限制, 并自动生成由那些任务使用的内存资源报告
- net_cls -- 这个子系统使用等级识别符(classid)标记网络数据包, 可允许 Linux 流量控制程序(tc)识别从具体 cgroup 中生成的数据包
- ns -- 名称空间子系统

### CGroup 设计解析

task_struct: 管理进程的数据结构

下面是与 CGroup 相关的部分

```C
#ifdef CONFIG_CGROUPS
struct css_set *cgroups;
struct list_head cg_list;
#endif
```

`cgroups`: 与进程相关的 CGroup 信息
`cg_list`: 归属于同一个 `css_set` 的进程链表

css_set:

```C
struct css_set {
	// 该 css_set 的引用数
	atomic_t refcount;
	struct hlist_node hlist;
	struct list_head tasks;
	struct list_head cg_links;
	// 指向 cgroup_subsys_state 的指针组
	// cgroup_subsys_state: 进程与特定 cgroup 子系统相关的信息
	struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
	struct rcu_head rcu_head;
};
```

```C
struct cgroup_subsys_state {
	// 该进程归属的 cgroup
	// 通过将进程归属到特定的子系统, 以控制进程
	struct cgroup *cgroup;
	atomic_t refcnt;
	unsigned long flags;
	struct css_id *id;
};
```

因此进程和 CGroup 的关系:

task_struct->css_set->cgroup_subsys_state->cgroup

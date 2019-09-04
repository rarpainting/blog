# Docker 技术基础

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

---

## 网络

Docker 的网络模式
- Host
- Container
- None
- Bridge

---

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
- cpuset -- 这个子系统为 cgroup 中的任务分配独立 CPU(在多核系统)和内存节点
- devices -- 这个子系统可允许或者拒绝 cgroup 中的任务访问设备
- freezer -- 这个子系统挂起或者恢复 cgroup 中的任务
- memory -- 这个子系统设定 cgroup 中任务使用的内存限制, 并自动生成由那些任务使用的内存资源报告
- net_cls -- 这个子系统使用等级识别符(classid)标记网络数据包, 可允许 Linux 流量控制程序(tc)识别从具体 cgroup 中生成的数据包
- ns -- 名称空间子系统

### CGroup 设计解析

#### 博客

[IBM 文章](https://www.ibm.com/developerworks/cn/linux/1506_cgroup/index.html)

task_struct: 管理进程的数据结构

`/usr/include/linux/sched.h`

下面是与 CGroup 相关的部分

```C
#ifdef CONFIG_CGROUPS
struct css_set *cgroups;
struct list_head cg_list;
#endif
```

- `cgroups`: 与进程相关的 CGroup 信息
- `cg_list`: 归属于同一个 `css_set` 的进程链表

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

#### linux-4.9

`include/linux/sched.h`

```C
struct task_struct {
...

#ifdef CONFIG_CGROUP_SCHED
	struct task_group *sched_task_group;
#endif

...
}
```

`kernel/sched/sched.h`

```C
/* task group related information */
struct task_group {
	struct cgroup_subsys_state css;

#ifdef CONFIG_FAIR_GROUP_SCHED
	/* schedulable entities of this group on each cpu */
	struct sched_entity **se;
	/* runqueue "owned" by this group on each cpu */
	struct cfs_rq **cfs_rq;
	unsigned long shares;

#ifdef	CONFIG_SMP
	/*
	 * load_avg can be heavily contended at clock tick time, so put
	 * it in its own cacheline separated from the fields above which
	 * will also be accessed at each tick.
	 */
	atomic_long_t load_avg ____cacheline_aligned;
#endif
#endif

#ifdef CONFIG_RT_GROUP_SCHED
	struct sched_rt_entity **rt_se;
	struct rt_rq **rt_rq;

	struct rt_bandwidth rt_bandwidth;
#endif

	/* 直接读-拷贝写 锁
	读多写少 */
	struct rcu_head rcu;
	struct list_head list;

	struct task_group *parent;
	struct list_head siblings;
	struct list_head children;

#ifdef CONFIG_SCHED_AUTOGROUP
	struct autogroup *autogroup;
#endif

	struct cfs_bandwidth cfs_bandwidth;
};
```

`include/linux/cgroup-defs.h`

```C
/*
 * Per-subsystem/per-cgroup state maintained by the system.  This is the
 * fundamental structural building block that controllers deal with.
 *
 * Fields marked with "PI:" are public and immutable and may be accessed
 * directly without synchronization.
 */
struct cgroup_subsys_state {
	/* PI: the cgroup that this css is attached to */
	/* 该进程归属的 cgroup */
	struct cgroup *cgroup;

	/* PI: the cgroup subsystem that this css is attached to */
	/* 该进程归属的 cgroup_subsys */
	struct cgroup_subsys *ss;

	/* reference count - access via css_[try]get() and css_put() */
	struct percpu_ref refcnt;

	/* PI: the parent css */
	struct cgroup_subsys_state *parent;

	/* siblings list anchored at the parent's ->children */
	struct list_head sibling;
	struct list_head children;

	/*
	 * PI: Subsys-unique ID.  0 is unused and root is always 1.  The
	 * matching css can be looked up using css_from_id().
	 */
	int id;

	unsigned int flags;

	/*
	 * Monotonically increasing unique serial number which defines a
	 * uniform order among all csses.  It's guaranteed that all
	 * ->children lists are in the ascending order of ->serial_nr and
	 * used to allow interrupting and resuming iterations.
	 */
	u64 serial_nr;

	/*
	 * Incremented by online self and children.  Used to guarantee that
	 * parents are not offlined before their children.
	 */
	atomic_t online_cnt;

	/* percpu_ref killing and RCU release */
	struct rcu_head rcu_head;
	struct work_struct destroy_work;
};
```

```C
struct cgroup {
	/* self css with NULL ->ss, points back to this cgroup */
	/* 指向自身的 css */
	struct cgroup_subsys_state self;

	unsigned long flags;		/* "unsigned long" so bitops work */

	/*
	 * idr allocated in-hierarchy ID.
	 *
	 * ID 0 is not used, the ID of the root cgroup is always 1, and a
	 * new cgroup will be assigned with a smallest available ID.
	 *
	 * Allocating/Removing ID must be protected by cgroup_mutex.
	 */
	int id;

	/*
	 * The depth this cgroup is at.  The root is at depth zero and each
	 * step down the hierarchy increments the level.  This along with
	 * ancestor_ids[] can determine whether a given cgroup is a
	 * descendant of another without traversing the hierarchy.
	 */
	int level;

	/*
	 * Each non-empty css_set associated with this cgroup contributes
	 * one to populated_cnt.  All children with non-zero popuplated_cnt
	 * of their own contribute one.  The count is zero iff there's no
	 * task in this cgroup or its subtree.
	 */
	int populated_cnt;

	struct kernfs_node *kn;		/* cgroup kernfs entry */
	struct cgroup_file procs_file;	/* handle for "cgroup.procs" */
	struct cgroup_file events_file;	/* handle for "cgroup.events" */

	/*
	 * The bitmask of subsystems enabled on the child cgroups.
	 * ->subtree_control is the one configured through
	 * "cgroup.subtree_control" while ->child_ss_mask is the effective
	 * one which may have more subsystems enabled.  Controller knobs
	 * are made available iff it's enabled in ->subtree_control.
	 */
	u16 subtree_control;
	u16 subtree_ss_mask;
	u16 old_subtree_control;
	u16 old_subtree_ss_mask;

	/* Private pointers for each registered subsystem */
	struct cgroup_subsys_state __rcu *subsys[CGROUP_SUBSYS_COUNT];

	struct cgroup_root *root;

	/*
	 * List of cgrp_cset_links pointing at css_sets with tasks in this
	 * cgroup.  Protected by css_set_lock.
	 */
	struct list_head cset_links;

	/*
	 * On the default hierarchy, a css_set for a cgroup with some
	 * susbsys disabled will point to css's which are associated with
	 * the closest ancestor which has the subsys enabled.  The
	 * following lists all css_sets which point to this cgroup's css
	 * for the given subsystem.
	 */
	struct list_head e_csets[CGROUP_SUBSYS_COUNT];

	/*
	 * list of pidlists, up to two for each namespace (one for procs, one
	 * for tasks); created on demand.
	 */
	struct list_head pidlists;
	struct mutex pidlist_mutex;

	/* used to wait for offlining of csses */
	wait_queue_head_t offline_waitq;

	/* used to schedule release agent */
	struct work_struct release_agent_work;

	/* ids of the ancestors at each level including self */
	int ancestor_ids[];
};
```

新的进程<->cgroup 关系:
`task_struct` -> `task_group` -> `cgroup_subsys_state` -> `cgroup`

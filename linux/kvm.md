# KVM

<!-- TOC -->

- [KVM](#kvm)
	- [KVM 介绍](#kvm-介绍)
		- [简单介绍](#简单介绍)
		- [KVM 功能列表](#kvm-功能列表)
		- [KVM 工具](#kvm-工具)
	- [CPU 和内存虚拟化](#cpu-和内存虚拟化)
		- [为什么需要 CPU 虚拟化](#为什么需要-cpu-虚拟化)
		- [基于二进制翻译的全虚拟化](#基于二进制翻译的全虚拟化)
		- [超虚拟化(半虚拟化/操作系统辅助虚拟化)](#超虚拟化半虚拟化操作系统辅助虚拟化)
		- [硬件辅助的全虚拟化](#硬件辅助的全虚拟化)
			- [Intel-VT(Virtualization Technology)](#intel-vtvirtualization-technology)
		- [SMP MPP NUMA](#smp-mpp-numa)
			- [SMP](#smp)
			- [MPP](#mpp)
			- [NUMA](#numa)
		- [KVM 的 CPU 虚拟化](#kvm-的-cpu-虚拟化)
		- [CPU 虚拟化](#cpu-虚拟化)
		- [执行客户机系统代码](#执行客户机系统代码)
		- [客户机线程到物理 CPU](#客户机线程到物理-cpu)
		- [客户端 vCPU 分配规则](#客户端-vcpu-分配规则)
		- [KVM 内存虚拟化](#kvm-内存虚拟化)
		- [KSM(Kernel SamePage Merging OR Kernel Shared Memory)](#ksmkernel-samepage-merging-or-kernel-shared-memory)
			- [原理](#原理)
			- [实现过程 -- 合并过程](#实现过程----合并过程)
		- [KVM Huge Page Backed Memory(巨页内存技术)](#kvm-huge-page-backed-memory巨页内存技术)
	- [I/O 全虚拟化和准虚拟化](#io-全虚拟化和准虚拟化)
		- [全虚拟化 I/O 设备](#全虚拟化-io-设备)
			- [QEMU 模拟网卡的实现](#qemu-模拟网卡的实现)
			- [qemu-kvm 关于磁盘设备和网络设备的主要选项](#qemu-kvm-关于磁盘设备和网络设备的主要选项)
			- [准虚拟化(Para-virtualization) I/O 驱动 virtio](#准虚拟化para-virtualization-io-驱动-virtio)
				- [virtio 框架](#virtio-框架)
				- [Virtio 的 Linux 下实现](#virtio-的-linux-下实现)
				- [Vhost-net(kernel-level virtio server)](#vhost-netkernel-level-virtio-server)
				- [virtio-balloon(Linux-2.6.27 qemu-kvm-0.13)](#virtio-balloonlinux-2627-qemu-kvm-013)
				- [RadHat 的多队列 Virtio](#radhat-的多队列-virtio)
		- [Tun/Tap](#tuntap)
			- [驱动程序](#驱动程序)
			- [发送过程:](#发送过程)
			- [接受过程:](#接受过程)
	- [KVM I/O 设备直接分配和 SR-IOV](#kvm-io-设备直接分配和-sr-iov)
		- [PCI/PCI-E 设备直接分配给虚机 (PCI Pass-through)](#pcipci-e-设备直接分配给虚机-pci-pass-through)
			- [PCI/PCIe Pass-through 原理](#pcipcie-pass-through-原理)
		- [SR-IOV 设备分配](#sr-iov-设备分配)
			- [SR-IOV 条件](#sr-iov-条件)
			- [优势和不足](#优势和不足)
			- [综合结论](#综合结论)
	- [Libvirt](#libvirt)
		- [Libvirt 提供了什么](#libvirt-提供了什么)
	- [Nova 通过 libvirt 管理 QEMU/KVM 虚机](#nova-通过-libvirt-管理-qemukvm-虚机)
		- [Libvirt 在 OpenStack 架构中的位置](#libvirt-在-openstack-架构中的位置)
		- [Nova 中的 libvirt 使用](#nova-中的-libvirt-使用)
	- [通过 libvirt 作 QEMU/KVM 快照和 Nova 实例的快照](#通过-libvirt-作-qemukvm-快照和-nova-实例的快照)
		- [QEMU/KVM 快照](#qemukvm-快照)
			- [概念](#概念)
			- [制造 snapshot 的各个 API](#制造-snapshot-的各个-api)
		- [OpenStack 的快照](#openstack-的快照)
			- [对 Nova Instance 进行快照](#对-nova-instance-进行快照)
			- [对卷做快照](#对卷做快照)
		- [从镜像文件启动的 Nova 虚机做快照](#从镜像文件启动的-nova-虚机做快照)
			- [Nova Live Snapshot -- 不停机快照](#nova-live-snapshot----不停机快照)
			- [Nova Cold Snapshot -- 停机快照](#nova-cold-snapshot----停机快照)
		- [当前 Nova Snapshot 的局限](#当前-nova-snapshot-的局限)
	- [使用 libvirt 迁移 QEMU/KVM 虚机和 Nova 虚机](#使用-libvirt-迁移-qemukvm-虚机和-nova-虚机)
		- [QEMU/KVM 迁移的概念](#qemukvm-迁移的概念)
		- [迁移效率的衡量](#迁移效率的衡量)
		- [KVM 迁移操作](#kvm-迁移操作)
			- [静态迁移](#静态迁移)
			- [动态迁移](#动态迁移)

<!-- /TOC -->

## KVM 介绍

### 简单介绍

- 基于虚拟化扩展(*Intel VT* OR *AMD-V*)的 X86 硬件的 Linux 原生的全虚拟化解决方案
- 通过 /dev/kvm 接口设置一个客户端虚拟服务器的地址空间 并提供模拟 I/O

![用户空间 内核空间与虚拟机](./311544349234041.jpg)

- Guset: 客户端系统, 提供 CPU(VCPU), 内存, 驱动; 被 KVM 置于一种受限制的 CPU 模式下运行
- KVM: 内核驱动, 提供 CPU 和 内存 的虚机化, 客户端的 I/O 拦截
- QEMU: 用户进程, 专供 KVM 虚机使用的 QEMU, Guest 的 I/O 被 KVM 拦截后, 将会最终由 QEMU 处理; 提供给 Guset 虚拟的硬件 I/O, 通过 IOCTL /dev/kvm 设备与 KVM 交互

### KVM 功能列表

- 支持 CPU 和 menory 超分 (Overcommit)
- 支持半虚拟化 I/O
- 支持热插拔(CPU, 块设备, 网络设备)
- 支持对称多处理器(SMP 架构)
- 支持实时迁移(Live Migration)
- 支持 PCI 设备直接分配和单根 I/O 虚拟化(SR-IOV)
- 支持内核同页合并(KSM)
- 支持 NUMA (非一致存储访问结构)

### KVM 工具

- libvirt: 操作/管理 KVM 虚机的虚拟化 API, 可操作 KVM vmware XEN Hyper-V LXC 等多重 Hypervisor
- Virsh: 基于 libvirt 的 CLI
- Virt-Manager: 基于 libvirt 的 GUI
- Virt-V2V: 虚机格式迁移工具
- Virt-install ...
- sVirt: 安全工具

## CPU 和内存虚拟化

### 为什么需要 CPU 虚拟化

X86 操作系统默认运行在 Ring0 的 CPU 级别上, 但是虚拟操作系统不知道, 此时, 需要 VMM(虚拟机管理程序) 来避免这种情况 -- 虚机通过 VMM 的以下方法访问硬件:

- 半虚拟化
- 全虚拟化
- 硬件辅助的虚拟化


### 基于二进制翻译的全虚拟化

![全虚拟化](011406053161018.jpg)

客户端操作系统(Guest OS) 运行在 Ring 1, 当它在执行特权指令(Ring 0)时, 触发异常, VMM 捕获该异常, 在异常里翻译, 模拟, 最终返回给 Guest OS

该流程性能损耗大

![异常捕获, 翻译, 模拟](011407133943983.jpg)

### 超虚拟化(半虚拟化/操作系统辅助虚拟化)

![超虚拟化](011408208321118.jpg)

通过修改操作系统内核, 替换不能虚拟化的指令, 通过**超级调用(hypercall)** 直接和底层的 **虚拟化层(hypervisor)** 通讯
hypervisor 同时也提供 **超级调用接口** 满足其他关键内核操作, 如*内存管理*, *中断*, *时间保持*

该流程不需要捕获和模拟, 性能损耗低

该流程只适用于 linux , 不适用于 Windows


|                           | 利用二进制翻译的全虚拟化               | 硬件辅助虚拟化                                                                       | 操作系统协助/半虚拟化                                                                                           |
| --                        | --                                     | --                                                                                   | --                                                                                                              |
| 实现技术                  | BT 和直接执行                          | 遇到特权指令转到 root 模式执行                                                       |                                                                                                                 |
| 客户端操作系统修改/兼容性 | 无需修改客户操作系统, 最佳兼容性       | 无需修改客户操作系统, 最佳兼容性                                                     | 客户操作系统需要修改来支持 hypercall , 因此不能运行在物理硬件本身和其他 Hypervisor 上, 兼容性差, 不支持 Windows |
| 性能                      | 差                                     | 全虚拟化下, CPU 需要在两种模式之间切换, 带来性能开销; 但是, 其性能在逐渐毕竟半虚拟化 | 好 半虚拟化下 CPU 性能开销几乎为 0, 虚机性能接近于物理机                                                        |
| 应用厂商                  | VMWare Workstation / QEMU / Virtual PC | VMware ESXi / Microsoft Hyper-V / Xen 3.0 / KVM                                      | Xen                                                                                                             |



### 硬件辅助的全虚拟化

Intel 和 AMD 的虚拟化技术

#### Intel-VT(Virtualization Technology)

该技术下 VMM 运行在 VMX root operation , 客户端 OS 运行在 VMX non-root operation

VM-entry: VMX root operation 通过调用 `VMLAUNCH/VMRESUME` 指令切换到 VMX non-root operation
VM-exit: 调用 `VMCALL` 指令调用 VMM 服务, 硬件挂起 Guest OS, 切换到 VMX root operation

### SMP MPP NUMA

#### SMP

多处理器架构

所有 CPU 共享全部资源(总线, 内存, IO 系统等) 操作系统或数据库只有一个副本

多个 CPU 之间没有区别, 平等的访问内存 外设 唯一一个操作系统

扩展能力有限, SMP 服务器 CPU 利用率最好在 2 到 4 个 CPU

#### MPP

海量并行处理结构: 分布式存储器模式

一个分布式存储器模式, 具有多个节点, 每个节点都有自己的(不共享的)存储器

能配置为 SMP 结构, 也能配置为非 SMP 结构

MPP 能理解为单一 SMP 的横向扩展集群, 需要软件实现

#### NUMA

非一致存储访问结构

由多个 SMP 服务器通过一定的节点以**互联网**的形式连接, 协同工作, 从用户的角度是一个完整的服务器系统

多个 SMP 系统通过节点互联网连接而成, 每个节点都只访问自己的本地资源(内存, 存储等), 是一种完全 **无共享(Share Nothing)** 结构

### KVM 的 CPU 虚拟化

1. qemu-kvm 通过对 /dev/kvm 的一系列 IOCTL 命令操作虚机

2. 一个 KVM 虚机即一个 Linux qemu-kvm 进程, 与其他 Linux 进程一样被 Linux 进程调度器调度

3. KVM 户级系统的内存是 qemu-kvm 进程的地址空间的一部分

4. KVM 虚机的 vCPU 作为进程运行在 qemu-kvm 进程的上下文中

![vCPU QEMU 进程 Linux 进程调度 物理 CPU 之间的逻辑关系](011509285517859.jpg)

### CPU 虚拟化

- 对于客户机, OS 运行在 ring 0 中, 应用运行在 ring 3 中
- 对于 KVM, QEMU 运行在 ring 3 中, KVM 运行在 ring 0 中

- QEMU 仅通过 KVM 控制虚机的代码被 CPU 执行, 本身不执行客户机代码
- 即, CPU 没有被虚级化成虚拟的 CPU 供客户机

![vSphere 中 CPU 虚拟化](697113-20150915144822773-366209755.jpg)

上图部分概念:
- socket - 颗 - CPU 的物理单位
- core - 核 - 每个 CPU 的物理内核
- thread - 超线程 - CPU 核的虚拟化, 导致生成多个逻辑 CPU, 同时运行(有缺陷的)多线程

在 KVM 等虚拟化平台中, 会在 VM 层和物理层之间加入 VMKernel 层, 从而允许所有的 VM 共享物理层资源

- VM 层
- 物理层
- VMKernel 层

### 执行客户机系统代码

Linux 内核的执行模式
- 用户模式 - User -- 代表客户机系统执行 I/O 操作
- 内核模式 - Kernel -- 负责将 CPU 切换到 Guest Mode 执行 Guest OS 代码, 并在 CPU 退出时回到 Kernel 模式
- 客户机模式 - Guest - KVM 驱动为 **支持虚拟化的 CPU** 而添加 -- 执行客户机系统非 I/O 代码, 同时在需要的时间驱动 CPU 退出该模式

KVM 内核驱动作为 User mode 和 Guest mode 之间的桥梁
- **User Mode** 中的 Qemu-KVM 通过 IOCTL 运行虚拟机
- KVM 内核模块收到请求后, 将 VCPU 上下文加载到 **VMCS(virtual machine control structure)** ,驱动 CPU 进入 VMX non-root 模式, 执行 客户机代码

![三种执行模块的分工](kvm_inst_heavy_bwutq8.png)

QEMU-KVM 相比原生 QEMU 的改动
- 原生 QEMU 通过 **指令翻译** 实现 CPU 虚拟化; 修改后的 QEMU-KVM 则通过 IOCTL 调用 KVM 模块
- 原生 QEMU 时单线程, QEMU-KVM 时多线程

一个 QEMU 进程的线程任务
- I/O 线程 -- 管理模拟设备
- vCPU 线程 -- 运行 Guest 代码
- 其他如 Event Loop, Offloaded Tasks

![kvm-qemu 线程](kvm_archi_cpu_gpzcpy.png)

### 客户机线程到物理 CPU

- 客户机线程调度到客户机物理 CPU(KVM vCPU)
- vCPU 线程调度到主机物理 CPU, 由 Hypervisor(Linux) 负责

Linux 进程调度

- Linux 进程将一个 可运行(runnable)进程(依靠 CPU 亲和性规则)调度
- 处理器亲和性: 可设置 vCPU 在指定的物理 CPU 上运行
- vCPU 数目不超过物理 CPU 数, 否则, 会出现线程间 CPU 内核资源竞争

### 客户端 vCPU 分配规则

1. 根据负载需要分配最少 vCPU
2. 客户机 vCPU 总数不应超过物理 CPU 内核总数, 否则, 存在 CPU 竞争, CPU 核内线程切换, 导致 overhead
3. 负载分为 计算负载和 I/O 负载; 对于计算负载, 需要分配较多 vCPU, 甚至需要考虑 CPU 亲和性

### KVM 内存虚拟化

利用 mmap 系统调用, 在 QEMU 主线程的虚拟地址空间中模拟 MMU

![两个到多个虚拟机](011413028483143.jpg)

- VA->PA -- *客户机虚拟地址* 到 *客户机内存物理地址* 的映射
- PA->MA -- VMM 负责的 *客户物理内存* 到 *实际机器内存* 的映射

![内存虚拟化](011413028483143.jpg)

VMM 内存虚拟化:
- 软件方式 -- 内存地址翻译, Shadow page table
- 硬件实现 -- 基于 CPU 的辅助虚拟化, AMD-NPT(Nested Page Tables) 和 Intel-EPT(Extended Page Tables)

![Intel-EPT](011445076768587.jpg)

EPT 硬件辅助虚拟化 -- **两阶段记忆体转换**:
- Guest Physical Address -> System Physical Address, VMM 不需要保留 SPT(Shadow Page Table), 不经过 SPT 转换过程
- 能耗低, 硬件指令集更可靠和稳定

### KSM(Kernel SamePage Merging OR Kernel Shared Memory)

#### 原理

作为 内核守护进程的 **KSMD** , 定期执行页面扫描, 识别副本页面并合并副本(借助 Linux 将内核相似的内存页合并成一个内存页)

增加了内核开销
提高了内存效率

能实现更多的内存超分, 运行更多的虚机

#### 实现过程 -- 合并过程

**TODO**
![合并](011433302547754.jpg)

![Guest1 写内存后](011434087852627.jpg)

### KVM Huge Page Backed Memory(巨页内存技术)

- 技术链接 -- MongoDB

## I/O 全虚拟化和准虚拟化

在 QEMU/KVM 中 客户机可以使用的设备大致分为三类:

- 模拟设备: 完全由 QEMU 纯软件模拟的设备
- Virtio: 实现 VIRTIO API 的半虚拟化设备
- PCI 设备直接分配

### 全虚拟化 I/O 设备

软件模拟设备

过程:
- 客户机的设备驱动程序发起 I/O 请求操作请求
- KVM 模块中的 I/O 模块捕获代码拦截该 I/O 请求
- 经过处理后将本次 I/O 请求放到 **I/O 共享页(sharing page)** , 并通知用户空间的 QEMU 程序
- QEMU 获得 I/O 操作的具体信息后, 交由 **硬件模拟代码** 来模拟出本次 I/O 操作
- 完成后, QEMU 将结果放回 I/O 共享页, 并通知 KVM 模块中的 I/O 操作捕获代码
- KVM 模块的捕获代码读取 I/O 共享页中的操作结果, 返回给客户机

- 能模拟出各种硬件设备
- 上下文切换多, 数据复制多, 性能差

**注: 当客户机通过 DMA(Direct Memory Access) 访问大块 I/O 时, QEMU 通过内存映射(Memory Map)方式将结果直接写在客户机的内存中, 然后通知 KVM 模块告诉客户机 DMA 操作已经完成**

#### QEMU 模拟网卡的实现

全虚拟化下, KVM 虚机可以选择的**网络**模式包括:

1. 默认用户模式(User) -- [-net user[,vlan=n]]: 不需要管理员权限来运行, 没有指定 -net 选项下, 默认 [-net tap[,vlan=n][,fd=h]]
2. 基于网桥模式(Bridge)的模式 -- [-net nic[,vlan=n][,macaddr=adr]]: 创建一个新的网卡, 与 VLAN n 连接, 如果没有指定 -net 选项, 则创建一个单一的 NIC
3. 基于 NAT(Network Address Teanslation)的模式 -- [-net tap[,vlan=n][,fd=h][,iframe=name][,script=file]]: 将 TAP 网络接口 name 与 VLAN n 进行连接, 并用 网络配置脚本文件(Default Is /etc/qemu-ifup) 配置, 如果没有指定 name , OS 自动分配; fd=h 可以用来指定一个已打开的 TAP 主机接口的句柄

#### qemu-kvm 关于磁盘设备和网络设备的主要选项

| 类型                      | 选项                                                                                                                                                                                                                                                                                                                     |
| --                        | --                                                                                                                                                                                                                                                                                                                       |
| 磁盘设备(软盘 磁盘 CDROM) | -drive option[,option[,option[,...]]]: 定义一个硬盘设备; 可用子选项有很多                                                                                                                                                                                                                                                |
|                           | file=/path/to/somefile: 硬件映像文件路径                                                                                                                                                                                                                                                                                 |
|                           | if=interface: 指定硬盘设备所连接的接口类型, 即控制器类型, 如 ide、scsi、sd、mtd、floppy、pflash 及 virtio 等                                                                                                                                                                                                             |
|                           | index=index: 设定同一种控制器类型中不同设备的索引号, 即标识号                                                                                                                                                                                                                                                            |
|                           | media=media: 定义介质类型为硬盘(disk)还是光盘(cdrom)                                                                                                                                                                                                                                                                     |
|                           | format=format: 指定映像文件的格式, 具体格式可参见 qemu-img 命令                                                                                                                                                                                                                                                          |
|                           | -boot [order=drives][,once=drives][,menu=on OR off]: 定义启动设备的引导次序, 每种设备使用一个字符表示; 不同的架构所支持的设备及其表示字符不尽相同, 在 x86 PC 架构上, a、b 表示软驱、c 表示第一块硬盘, d 表示第一个光驱设备, n-p 表示网络适配器; 默认为硬盘设备(-boot order=dc,once=d)                                    |
| 网络                      | -net nic[,vlan=n][,macaddr=mac][,model=type][,name=name][,addr=addr][,vectors=v]: 创建一个新的网卡设备并连接至vlan n中；PC架构上默认的NIC为e1000, macaddr用于为其指定MAC地址, name用于指定一个在监控时显示的网上设备名称；emu可以模拟多个类型的网卡设备；可以使用“qemu-kvm -net nic,model=?”来获取当前平台支持的类型   |
|                           | -net tap[,vlan=n][,name=name][,fd=h][,ifname=name][,script=file][,downscript=dfile]: 通过物理机的TAP网络接口连接至vlan n中, 使用script=file指定的脚本(默认为/etc/qemu-ifup)来配置当前网络接口, 并使用downscript=file指定的脚本(默认为/etc/qemu-ifdown)来撤消接口配置；使用script=no和downscript=no可分别用来禁止执行脚本 |
|                           | -net user[,option][,option][,...]: 在用户模式配置网络栈, 其不依赖于管理权限；有效选项有:                                                                                                                                                                                                                                 |
|                           | vlan=n: 连接至 vlan n, 默认 n=0                                                                                                                                                                                                                                                                                          |
|                           | name=name: 指定接口的显示名称, 常用于监控模式中                                                                                                                                                                                                                                                                          |
|                           | net=addr[/mask]: 设定 GuestOS 可见的 IP 网络, 掩码可选, 默认为 10.0.2.0/8                                                                                                                                                                                                                                                |
|                           | host=addr: 指定 GuestOS 中看到的物理机的 IP 地址, 默认为指定网络中的第二个, 即 x.x.x.2                                                                                                                                                                                                                                   |
|                           | dhcpstart=addr: 指定 DHCP 服务地址池中 16 个地址的起始 IP, 默认为第 16 个至第 31 个, 即 x.x.x.16-x.x.x.31                                                                                                                                                                                                                |
|                           | dns=addr: 指定 GuestOS 可见的 dns 服务器地址; 默认为 GuestOS 网络中的第三个地址, 即 x.x.x.3                                                                                                                                                                                                                              |
|                           | tftp=dir: 激活内置的 tftp 服务器, 并使用指定的 dir 作为 tftp 服务器的默认根目录                                                                                                                                                                                                                                         |
|                           | bootfile=file:BOOTP 文件名称, 用于实现网络引导 GuestOS; 如: qemu -hda linux.img -boot n -net user,tftp=/tftpserver/pub,bootfile=/pxelinux.0                                                                                                                                                                             |

#### 准虚拟化(Para-virtualization) I/O 驱动 virtio

Linux 的 设备驱动标准框架

##### virtio 框架

在 Guest OS 内核中安装前端驱动(Front-end driver) 和在 QEMU 中实现后端驱动(Back-end)的方式
前后端采用 **Vring** 直接通信, 从而绕过 KVM 模块

![Virtio Transport](011805499882916.jpg)

纯软件模拟和 Virtio 的区别:
    Virtio 省去了纯模拟的异常捕捉, 即 Guest OS 直接和 QEMU 的 I/O 模块**直接通信**

![Difference between Pure-software and Virtio](011807259737673.jpg)

Virtio 的完整虚机流程

![Virtio 的完整流程](011809286761592.jpg)

- KVM 通过中断的方式通知 QEMU 去获取数据, 放到 Virtio Queue 中
- KVM 再通知 Guest 去 virtio 中取数据

##### Virtio 的 Linux 下实现

- 前端驱动: 客户机(Guest OS)中安装的驱动程序模块
- 后端驱动: 在 QEMU 中实现, 调用主机上的物理设备, 或者完全由软件实现
- virtio 层: 虚拟队列接口, 从概念上连接 前端和后端; 根据需要使用不同数量的队列, 如 virtio-net 使用两个队列, virtio-block 使用一个队列; 该队列使用 virtio-ring 虚拟
- virtio-ring: 实现虚拟队列的环形缓冲区

Linux 内核中实现的前端驱动:

- 块设备(磁盘等)
- 网络设备
- PCI 设备
- 气球驱动设备(动态管理客户机 *内存* 使用情况)
- 控制台驱动程序

当 Guest OS 使用某个 virtio 设备时, 对应的驱动会被加载

示例: virtio-net

![virtio-net](022013482737327.jpg)

特点

- 多个虚机共享主机网卡 eth0
- QEMU 使用标准 tun/tap 将虚机的网络桥接到主机网卡上
- 每个虚机看起来有一个直接连接到主机 PCI 总线的私有 virtio 网络设备
- 需要在虚机中安装 virtio 驱动

![virtio-net 流程](022011246173106.jpg)

Virtio 优缺点:
- 优点: 更高的 I/O 性能, 几乎和原生系统相近
- 缺点: 客户端必须安装特定的 virtio 驱动, 难以兼容旧版 Linux(< 2.6.24), 部分 Windows 需要安装特定驱动

##### Vhost-net(kernel-level virtio server)

将 Virtio-net 的后端处理任务放在内核空间中执行

virtio-net 与 vhost-net 对比

![对比](022016033364356.jpg)

##### virtio-balloon(Linux-2.6.27 qemu-kvm-0.13)

针对内存的 Ballooning 技术

- 当宿主机内存紧张, 请求客户机回收部分内存; 甚至可能回收利用已分配给客户机的部分内存
- 当客户机内存不足, 释放部分主机内存到客户机


##### RadHat 的多队列 Virtio

多队列 virtio-net 提供一个随着虚机的虚拟 CPU 增加而增强网络性能的方法;
同时多队列 virtio-net 会相应增加 CPU 的负担, 可能会降低 **对外数据流** 的性能


### Tun/Tap

虚拟网卡驱动
- tun - 虚拟的点对点设备
- tap - 虚拟的以太网设备

通过虚拟 *设备文件* 实现用户态和核心态的数据交互

![Tun/Tap 数据收发](image004.jpg)

#### 驱动程序
1. 网卡驱动: 接收来自 TCP/IP 协议栈的网络分包并发送 OR 将接收到的网络分包传给协议栈处理
2. 字符设备驱动: 将网络分包在内核与用户态间传送, 模拟物理链路的数据接收和发送

与真实网卡设备的区别:
1. 获取的数据不来自物理链路, 而是来自于用户区, Tun/Tap 设备驱动通过 **字符设备文件** 来实现数据从用户区获取
2. 发送时通过字符设备发送到用户区, 再由用户区程序通过其他渠道发送

#### 发送过程:

1. 目标数据 -> `hard_start_xmit` --> `tun_net_xmit`
2. skb(?) 将会被加入 skb 链表, 唤醒被阻塞的使用 Tun/Tap 设备字符驱动读数据的进程
3. --> `tun_chr_read` 读取 skb 链表, 发送到目标用户区

#### 接受过程:

1. 目标数据 -> Tun/Tap 设备字符驱动 --> `tun_chr_write` --> `tun_get_user`(从用户区接收数据)
2. 将数据存入 skb 中, 然后调用 `netif_rx(skb)` 将 skb 送给 tcp/ip 协议栈处理


## KVM I/O 设备直接分配和 SR-IOV

### PCI/PCI-E 设备直接分配给虚机 (PCI Pass-through)

![PCI/PCI-E 的区别](041136014418109.jpg)

PCI-E 直接连接在 IOMMU 上, PCI 连接在一个 IO Hub 上, 所以 PCI 卡的性能没有 PCI-E 高

主要的 PCI 设备
- Network cards(wired or wireless)
- SCSI adapters
- Bus controllers: USB, PCMICIA, I2C, FireWire, IDE
- Graphics and Video cards
- Sound cards

#### PCI/PCIe Pass-through 原理

- KVM 支持客户机以 **独占方式** 访问主机的 PCI/PCIe 设备, 客户机对该设备的 I/O 交互操作和实际的物理设备操作完全一样, 不需要或者很少需要 KVM 的参与
- 几乎所有的 PCI/PCIe 设备都支持直接分配, 除了 *显卡* 除外
- 一般的 SATA 或者 SAS 等硬盘的控制器都是直接接入 PCI 或者 PCIe 总线的, 所以也可以将硬盘作为普通的 PCI 设备直接分配给客户机, 这时候分配的是 SATA 或者 SAS 的 **控制器**

设备直接分配的优势和不足

优势
- 在执行 I/O 操作时大量减少或者避免 VM-Exit(进入 root operation VMX) 陷入到 Hypervisor 中, 极大提高了性能
- VT-d 克服了 virtio 兼容性不好, CPU 使用频率高的问题

不足:
- 一块主板允许添加的 PCI/PCIe 设备是有限的, 大量使用 VT-d 独立分配设备给客户机, 硬件设备数量增加, 导致硬件投资成本高
- VT-d 直接分配的做法, 导致其动态迁移功能受限, 不过能通过 *热插拔* 或者 *libvirt* 等工具解决

解决方案:
- 对网络性能要求高的客户机使用 VT-d 直接分配, 其他使用 纯模拟 或者 virtio 实现多个客户机共享一个设备
- 对于 网络 I/O, 选择 SR-IOV 使一个网卡产生多个独立的虚拟网卡, 在单独分配到每一个客户机

### SR-IOV 设备分配

单根 I/O 虚拟化, SR-IOV 使的一个单一的功能单元能看起来像多个独立的物理设备; 一个带有 SR-IOV 功能的物理设备能被配置为多个功能单元

SR-IOV 使用两种功能
- 物理功能(Physical Functions-PF): 完整的带有 SR-IOV 能力的 PCIe 设备; PF 能像 PCI 设备那样被发现, 管理和配置
- 虚拟设备(Virtual Functions-VF): 简单的 PCIe 功能, 只能处理 I/O; 每个 VF 都是从 PF 分离出来的; 一个 PF 最多能被虚拟成 **有限制数目** 的 VF, 供虚拟机使用

网卡 SR-IOV 例子:

![1](041217128487605.jpg)
![2](011818595515631.jpg)

#### SR-IOV 条件

1. 需要 CPU 支持 Intel VT-x 和 VT-D(或者 AMD 的 SVM 和 IOMMU)
2. 需要支持 SR-IOV 规范的设备 (Intel 的中高端网卡等)
3. 需要 QEMU/KVM 支持

#### 优势和不足

优势:
1. 真正实现设备共享(多个客户机共享一个 SR-IOV 设备的物理端口)
2. 接近原生性能
3. 相比 VT-D, SR-IOV 可以使用更少的设备来支持更多的客户机, 提高数据中心的空间利用率

不足:
1. 对设备有依赖(需要支持 SR-IOV 规范的设备)
2. 不方便动态迁移客户机: 由于此时虚机直接使用主机上的物理设备, 即虚机的迁移(migiration)和保存(save)目前都不支持

![虚拟化方式比较](041757191911525.jpg)
![虚拟化方式比较](041757434104956.jpg)

Virtio 和 Pass-Through 的比较

![Virtio 和 Pass-Through 比较](030648412277872.jpg)


#### 综合结论

- I/O 设备尽量使用准虚拟化(virtio 和 vhost_net)
- 如果需要实施迁移, 不能使用 SR-IOV
- 有更高的 I/O 要求, 且不需要实时迁移的, 可以使用 SR-IOV

## Libvirt

### Libvirt 提供了什么

- 统一 稳定 开放的源代码的应用程序接口(API) 守护进程(libvirtd)和管理工具(virsh)
- 通过一种基于驱动程序的架构来实现对 Hypervisor 的支持

## Nova 通过 libvirt 管理 QEMU/KVM 虚机

### Libvirt 在 OpenStack 架构中的位置

![Nova 调用 libvirt 以及其他 API](111010352073441.jpg)

### Nova 中的 libvirt 使用

- 创建虚机
- 虚机的声明周期管理
- 添加和删除连接到别的网络的网卡 (interface)
- 添加和删除 Cinder 卷 (volume)

## 通过 libvirt 作 QEMU/KVM 快照和 Nova 实例的快照

### QEMU/KVM 快照

#### 概念

快照种类:

1. 磁盘快照 -- 磁盘内容在某个时间点上被保存, 用于将来恢复

- 在一个运行中的系统上, 某个磁盘快照可能是 **崩溃一致性**(例如机器突然掉电时的数据状态), 而不是 **完整一致性**, 需要通过文件检查(fcsk 等)对快照进行一致性检查
- 对一个非运行中的虚机, 如果上次虚机关闭时磁盘是完整一致的, 那么其被快照的磁盘快照也是完整一致的

磁盘快照类型

- 内部快照: 使用单个 Qcow2 的文件来保存快照和快照之后的改动; libvirt 的默认行为, 支持完整的 创建/回滚/删除 操作, 只支持 Qcow2 格式的磁盘镜像文件, 且过程慢
- 外部快照: 快照为一个只读文件, 快照的修改在另一个 Qcow2 文件中; 能操作各种格式的磁盘镜像文件; 其结果是形成一个 Qcow2 的文件链: original <- snap1 <- snap2 <- snap3; [详细文章](http://wiki.libvirt.org/page/I_created_an_external_snapshot,_but_libvirt_won't_let_me_delete_or_revert_to_it)

2. 内存状态(虚机状态)

只保持内存和虚机使用的其他资源的状态. 如果虚机状态在恢复之前没有被修改, 那么虚机会持续一个持续的状态; 否则可能导致 **数据 corruption**

3. 系统还原点(system checkpoint)

虚机的所有磁盘快照和内存状态快照的集合, 可用于 **恢复** 完整的系统状态(类似于系统休眠)

0. 关于崩溃一致性

- 尽量避免在虚机 I/O 繁忙时作快照
- VMware 装一个 tools -- 一个 PV driver, 可以在制造快照的时候 *挂起系统*
- KVM -- QEMU Guest Agent

快照还能分为 Live snapshot 和 Clod snapshot

- Live snapshot: 系统 *运行状态* 下的快照
- Cold snapshot: 系统 *停止状态* 下的快照

#### 制造 snapshot 的各个 API

| snapshot     | 做快照的 libvirt API                                                   | 从快照中恢复的 libvirt API |                                       |
| --           | --                                                                     | --                         | --                                    |
| 磁盘快照     | virDomainSnapshotCreateXML(flags=VIR_DOMAIN_SNAPSHOT_CREATE_DISK_ONLY) | virDmainRevertToSnapshot   | virsh snapshot-create/snapshot-revert |
| 内存状态快照 | virDomainSave                                                          | virDomainRestore           |                                       |
|              | virDomainSaveFlags                                                     | virDomainRestoreFlags      | virsh save /restore                   |
|              | virDomainManagedSave                                                   | virDomainCreate            |                                       |
|              |                                                                        | virDomainCreateWithFlags   |                                       |
| 系统检查点   | virDomainSnapshotCreateXML                                             | virDomainRevertToSnapshot  | virsh snapshot-create/snapshot-revert |


各个 API 工作情况:

1. virDomainSnapshotCreateXML(virDomainPtr domain, const char * xmlDesc, unsigned int flags)

| flags                                | 处于运行状态                                                                                                                                                               | 处于停止状态         |
| 0                                    | 创建系统检查点, 包括磁盘状态和内存状态(内存内容等)                                                                                                                         | 保持关机时的磁盘状态 |
| VIR_DOMAIN_SNAPSHOT_CREATE_LIVE      | 快照期间, 虚机不会 paused; 这将增加内存 dump file 的大小, 但是减少了系统停机时间; 部分 Hypervisor 只在做外部的系统检查点时才设置该 flags, 这意味着普通快照还是需要停机(??) |                      |
| VIR_DOMAIN_SNAPSHOT_CREATE_DISK_ONLY | 只做指定磁盘的快照; 对应运行着的虚机, 磁盘快照可能是不完整的(类似突然掉电的情况)                                                                                           | 只做指定磁盘的快照   |

- 运行中的虚机: 通过 QEMU Monitor 做快照, 磁盘镜像文件必须是 Qcow2; 此时虚机的 CPU 被停止, 快照结束后被重启
- 停止的虚机: 通过 QEMU-img 操作所有磁盘镜像文件

2. virDomainSave API

| virDomainSave        | 该方法会 suspend 一个运行中的虚机, 然后保存内容到一个文件中; 成功调用后, domain 将不会处于 running 状态, 需要通过 virDomainRestore 来恢复虚机                                                                                                                                         |
| virDomainSaveFlags   | 类似于 virDomainSave API , 一些 Hypervisor 在调用该方法前需要调用 virDomainBlockJobAbort 方法停止 block copy 操作                                                                                                                                                                     |
| virDomainManagedSave | 类似 virDomainSave API, 区别在于 libvirt 将其内存保存在一个受 libvirt 管理的文件中, 有助于 libvirt 可以一直跟踪 snapshot 的状态; 当调用 virDomainCreate/virDomainCreateWithFlags 重启该 Domain 时, libvirt 会直接使用该受管文件(, 而不是一个空白文件), 这样就可以 restore 该 snapshot |

3. snapshot-create-as

| <不使用额外参数>                                                    | 使用磁盘和内存的内部快照                                                                       |
| --memspec snapshot=external --diskspec vda,snapshot=external        | 磁盘和内存的外部快照, 虚机需要暂停                                                             |
| --live --memspec snapshot=external --diskspec vda,snapshot=external | 创建系统检查点(包括磁盘和内存的快照), 且虚机不会被暂停(会暂停, 但是暂停时间比不使用 --live 短) |
| --disk-only                                                         | 创建所有或者部分磁盘的外部快照                                                                 |

4. 外部快照的删除 -- workaround

### OpenStack 的快照

OpenStack Snapshot 可分为以下几种:

#### 对 Nova Instance 进行快照

- 对从镜像文件中启动的虚机做快照
  - 将运行中的虚机的 Root Disk 做成 image, 然后上传到 Glance
  - Live Snapshot: 对满足特定条件(QEMU 1.3+ Libvirt 1.0.0+ source_format not in ('lvm', 'rbd') and not CONF.ephemeral_storage_encryption.enabled and not CONF.workarounds.disable_libvirt_livesnapshot, 以及能正常调用 libvit.blockJobAbort)的虚机, 会进行 Live snapshot. Live Snapshot 允许用户在虚机处于运行状态时不停机做快照
  - Cold Snapshot: 对不能做 Live snapshot 的虚机做 Cold snapshot, 此时必须首先 Pause 虚机

- 对从卷启动的虚机做快照
  - 对虚机的每个挂载的 Volume 调用 Cinder API 做 Snapshot
  - Snapshot 出的 **Metadata** 会保存到 Glance 里面, 但是不会有 snapshot 的 **Image** 上传到 Glance
  - 这个 Snapshot 会出现在 Cinder 的数据库中 , 对 cinder API 可见

#### 对卷做快照
- 调用 cinder driver API, 对 Backend 中的 Volume 进行 Snapshot
- 这个 Snapshot 会出现在 Cinder 的数据库中 , 对 cinder API 可见


### 从镜像文件启动的 Nova 虚机做快照

严格得说, Nova 虚机的快照, 并不是对虚机做完整的快照, 而是对 **虚机的启动盘(root disk -- vda/hda)** 做快照生成 Qcow2 格式文件, 并传到 Glance, 仅用于方便使用快照生成的镜像来部署新的虚机, 以下式两种快照格式

#### Nova Live Snapshot -- 不停机快照

- 找到 虚机的 Root Disk(Vda OR Hda)
- 在 `CONF.libvirt.snapshot_dirctory` 指定的文件夹 (default /var/lib/nova/instances/snapshots) 中创建临时文件夹, 在其中创建一个 uuid 文件名的 Qcow2 格式的 delta 文件, 该文件的 backing file 和 root disk 文件夹的 backing file 相同
- 调用 `virDomainGetXMLDesc` 来保存 domain 的 xml 配置
- 调用 `virDomainBlockJobAbort` 来停止对 root disk 的活动块的操作
- 调用 `virDomainUndefine` 来将 domain 变为 transimit 类型的(因为 BlockRebase API 不能针对 Persistent domain 调用)
- 调用 `virDomainBlockRebase` 来将 root disk image 文件中不同的数据拷贝到 delta disk file 中(此时有可能应用正在向该磁盘写数据)
- Nova 每隔 0.5 秒调用 `virDomainBlockJobInfo` API 检查上一步骤是否结束
- 拷贝结束, 调用 `virDomainBlockJobAbort` 来终止数据拷贝
- 调用 `virDomainDefineXML` 将 domain 由 transimisit 改回 persistent
- 调用 qemu-img convert 命令将 delta image 文件和 blcking file 变回一个 Qcow2 文件
- 将 image 的元数据和 Qcow2 文件传到 Glance 中

对应于命令行顺序是:

```shell
$ qemu-img create -f qcow2 -o backing_file=/var/lib/nova/instances/_base/ed39541b2c77cd7b069558570fa1dff4fda4f678,size=21474836480 /var/lib/nova/instances/snapshots/tmpzfjdJS/7f8d11be9ff647f6b7a0a643fad1f030.delta

$ virsh blockjob <domain> <path> [--abort] [--async] [--pivot] [--info] [<bandwidth>]

$ qemu-img convert -f [qcow2 | raw | vmdk | vdi] -o dest_fmt # 将 backing file 的 qcow2 image 转换为 不带 backing file 的 flag image
```

```C
int virDomainBlockRebase(virDomainPtr dom, const char * disk, const char * base, unsigned long bandwidth, unsigned int flags)
```
该 API 从 backing 文件中拷贝数据, 或者拷贝整个 backing 文件到 @base 文件

Nova 中的调用方式:

```C
domain.blockRebase(disk_path, disk_delta, 0,
    libvirt.VIR_DOMAIN_BLOCK_REBASE_COPY |
    libvirt.VIR_DOMAIN_BLOCK_REBASE_REUSE_EXT |
    libvirt.VIR_DOMAIN_BLOCK_REBASE_SHALLOW)
```

- 默认, 该 API 会拷贝整个 @disk 文件到 @base 文件
- `VIR_DOMAIN_BLOCK_REBASE_SHALLOW`  -- 只拷贝差异数据 (top data) 因为 @disk 和 @base 使用相同的 backing 文件
- `VIR_DOMAIN_BLOCK_REBASE_REUSE_EXT` -- 使用已存在的 @base 文件(因为 Nova 会预先创建好这文件)

#### Nova Cold Snapshot -- 停机快照

当虚机不在运行中/不满足 live snapshot 条件的情况下, Nova 则执行 Cold snapshot:

- 虚机处于 running OR paused:
  - detach PCI devices
  - detach SR-IOV devices
  - 调用 `virDomainManagedSave` API 来将虚机 suspend 并且将内存状态保存到硬盘文件

- 调用 qemu-img convert 命令将 root disk 的镜像文件转化为 **相同格式** 的镜像文件
- 调用 `virDomainCreateWithFlags` API 将虚机变为初始状态
- 将在步骤 1 中卸载的 PCI 和 SR-IOV 设备重新挂载回来
- 将元数据和 Qcow2 文件传到 Glance 中


### 当前 Nova Snapshot 的局限

- Nova Snapshot 其实只是提供一种 **创造系统盘镜像** 的方法; 不支持回滚到快照点, 只能通过该快照镜像重新创建一个新快照
- 在虚机是从 image-boot 的时候, 只对 **系统盘** 进行快照, 不支持内存快照, 不支持系统还原点
- (?) Live Snapshot 需要用户进行一致性操作
- 由于是通过 image 进行保存的, 只支持虚拟机内置(全量)快照, 不支持外置(增量)快照
- 从 image boot 的虚机的快照以 image 方式保存到 Glance 中, 并非以 Cinder 卷方式保存
- 过程较长(存储快照 -> 抽取元数据(?) -> 上传到 Glance), 网络开销大

Nova 不支持虚机快照的[讨论](http://www.gossamer-threads.com/lists/openstack/dev/8678)结果:

- Nova snapshot 是一种虚拟化技术, 而不是云计算平台的功能
- Openstack 需要支持多种虚拟化的技术, 某些虚拟化技术实现这种功能比较困难
- 创建的 VM state snapshot 会面临 cpu feature 不兼容的问题
- 目前 libvirt 对 QEMU/KVM 虚机的外部快照的支持还不完善, 即使更新到最新的 libvirt 版本, 兼容性也较差

## 使用 libvirt 迁移 QEMU/KVM 虚机和 Nova 虚机

### QEMU/KVM 迁移的概念

迁移分为系统整体的迁移和某个工作负载的迁移; 静态迁移和动态迁移, 其中静态牵引由明显的一段时间客户机中的服务不可用

静态迁移:

- 关闭客户机, 复制硬盘镜像到另一台宿主机然后恢复启动; 但是不能保留客户机中运行的工作负载
- 需要迁移间的宿主机共享 **存储系统**, 同时保持客户机迁移前的内存状态和工作负载

动态迁移的关键:

- 保证客户机的 **内存, 硬盘存储和网络连接** 在迁移到目地主机后保持不变
- 迁移过程中, 服务暂停时间短

### 迁移效率的衡量

动态迁移的应用场景: 负载均衡, 解除硬件依赖, 节约能源, 异地迁移

- 整体迁移时间
- 服务器停机时间
- 迁移前后对服务性能的影响

### KVM 迁移操作

#### 静态迁移

- saveVM

- loadVM

#### 动态迁移

在源宿主机和目地宿主机共享存储系统时:
- 迁移开始时, 客户端依然在宿主机上运行, 同时 客户机的内存页被传输到目地主机上
- QEMU/KVM 监控并记录下迁移过程中所有 *已被传输的内存页的任何修改* , 并在所有内存页都传输完成后即开始传输在前面过程中 *内存页的更改内容*
- 如果剩余的内存数量能够在一个时间周期内完成传输, QEMU/KVM 会关闭源宿主机上客户机, 再将剩余的数据传输到目地主机, 最后传输过来的内存内容再目地宿主机上恢复客户机的运行状态
- 迁移完成, 补全(网桥等)剩余的配置

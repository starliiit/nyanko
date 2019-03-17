---
title: "LKD Recap"
date: 2019-03-17T23:31:44+08:00
draft: false
tags: [linux]
description: ""
---

本书的定位是 OS 教科书到源码分析书之间的过渡书籍，介绍了进程管理、中断处理、内存管理、文件系统几个部分的设计和接口。

This is an index.

## Overview

- 操作系统的组件

  - kernel
  - device driver
  - boot loader
  - command shell
  - basic file & system utils

  本书介绍的是 kernel。

- 典型的 kernel 组件

  - interrupt handlers
  - process scheduler
  - memory manage system
  - file system
  - other services like networking/IPC

  本书主要介绍了前 4 部分。

## 进程管理

- 数据结构 `task_struct`
- 标志符 `pid_t`
- 进程与线程之间的区别（以 Linux 内核的视角）
- 内核在 process context 中引用当前进程的宏 `current`
- 进程主要的三种状态
- `fork` 的实现（`clone` 系统调用，flags）
- 内核线程与用户线程的区别
- 进程终止的过程（`EXIT_ZOMBIE` 状态下进程占用的资源）

## CPU 调度

- 优先级：nice 值、rtprio 值
- CFS 完全公平调度器（perfect multitasking 的近似，nice 作为权值
- scheduler class
  - `SCHED_NORMAL`: CFS
  - `SCHED_FIFO`: FIFO
  - `SCHED_RR`: Round-Robin
- 调度相关数据结构：`sched_entry`
- 虚拟运行时间：`sched_entry.vruntime`
- 所有 runnable 进程的 `vruntime` 保存在一棵红黑树中
- 调度器入口函数：`schedule`
- 进程睡眠的过程（信号唤醒、伪唤醒）
- 进程唤醒的过程
- 上下文切换的过程 `context_switch`
  1. `swtich_mm` 修改内存映射
  2. `swtich_to` 修改处理器状态
- 进程切换的时机
  - `need_resched` 标志
  - user preemption: 内核完成系统调用，返回用户空间
  - kernel preemption: 一个内核线程被另一个内核线程抢占（中断处理程序）
- 处理器 affinity: `task_struct.cpus_allowed`
- `sched_yield` 自愿让出 CPU

## 系统调用

- 内核的 `sys_call_table` 中保存所有系统调用号
- 系统调用通过软中断实现，每次调用系统调用都会切换到内核态执行软中断处理程序
- 参数传递：调用号放入 eax，参数放到 ebx, ecx, edx, esi, edi 寄存器，使用 `int 0x80` 触发软中断
- 参数中的 flags：为了保持向后兼容性，同时能够添加新特性

## 中断处理

处理中断时要做的事情可能很多，但是中断处理必须暂时屏蔽中断，因此必须快速结束，整个中断处理分为两部分：top-half 和 bottom-half。

### top-half

- 中断传入内核的过程:设备->中断控制器->处理器引脚->处理器 IRQ line->内核 `do_IRQ` 函数
- 多个设备可以共享同一 IRQ line
- Interrupt 与 Exception 的区别
- Interrupt Context：不允许 sleep，因为中断处理程序没有对应的进程，无法再次调度
- top-half 在屏蔽其他中断的情况下运行
- 实现中断处理程序的接口
  - `request_irq, free_irq` 注册/删除中断处理函数
  - `static irqreturn_t intr_handler(int irq, void *dev)` 中断处理函数格式
- 中断处理程序在一个处理器上执行时，该处理器上对应 IRQ line 被屏蔽（新的请求是排队还是丢弃？）
- 中断控制接口
  - 禁止/开启当前处理器上的中断
  - 屏蔽 IRQ 线
  - 检查自己是否处于 Interrupt Context

### bottom-half

上半部分的中断处理程序在运行时会屏蔽当前 IRQ 线，如果指定了 `IRQF_DISABLED` 则会屏蔽当前处理器上所有 IRQ 线。降低了系统的响应能力。因此我们希望中断处理程序结束得越快越好，不是非常 time-critical 的工作就放到下半部分中执行。

bottom-half 实现机制：

- 软中断：同类型的软中断可以在不同处理器上运行
- tasklet：不同类型 tasklet 可以在不同处理器上运行，同类型 tasklet 不能在不同处理器上运行
- work queue：由一个内核线程在 process context 中运行

三种机制的实时性从高到低，根据需要自由选择使用哪种机制。

- 网络子系统需要的实时性最高，因此使用软中断。
- 一般驱动程序使用 tasklet。
- 需要调用一些可能会睡眠的函数，使用 work queue。

## 同步机制

- 造成并发执行的原因
  - 中断
  - 软中断、tasklet
  - 内核抢占
  - 进程睡眠
  - 多处理器同时执行代码
- 避免死锁的方法

### Linux 同步机制

- 原子操作：通过 `atomic_t` 实现
- spin_lock 自旋锁：
  - 同一时刻，至多被一个可执行线程持有。
  - 试图获取锁，如果该锁已经被其他进程持有，进行忙等待，直到其他进程释放锁。
  - 在同一个处理器上重复加锁会导致死锁（考虑普通进程、中断处理程序、软中断、tasklet之间的抢占关系）
  - 使用前考虑等待时间，应比两次 context_switch 短
- 读写自旋锁：读者优先
- semaphore 信号量 ：进程会睡眠，因此不能在中断处理程序(top half & bottom half)中使用
- 读写信号量
- mutex：轻量的二值信号量
- completion variable：类似信号量
- BKL：大内核锁，全局自旋锁，deprecated
- 顺序锁：读者很多，写者很少，写优先（例如系统时钟节拍数 `jiffies`）
- 比锁更轻量：对于处理器私有数据，不会被其他处理器共享，只需屏蔽抢占，不需要加锁

## 内存管理

- 内存管理基本单位：页。这是由硬件 MMU 决定的。
- 数据结构：`struct page`
- 内存区域划分：`struct zone`
  - `ZONE_DMA`
  - `ZONE_NORMAL`
  - `ZONE_HIGHMEM`
- 32 位内核虚拟地址空间布局
- 申请内存的接口：
  - 申请页面: `alloc_pages, __get_free_pages`
  - 申请以字节为单位的块: `kmalloc, vmalloc`
- 申请内存时的 flag：`gfp_mask`
  - 行为修饰符
  - 区修饰符
  - 类型标志（行为与区的组合）
- SLAB 对象高速缓存系统
- 处理器私有数据

## 虚拟地址空间

进程的虚拟地址空间由内存区域(memory area)组成，有多种类型：

- 可执行文件代码
- 可执行文件已初始化的全局变量
- 未初始化的全局变量（BSS）
- C 库、动态链接库的代码段、数据段、BSS
- 内存映射文件
- 共享内存段
- 匿名的内存映射（如 malloc 分配的内存）

### 核心数据结构

- `mm_struct`：表示一个进程的虚拟地址空间
- `vm_area_struct`：表示在一个地址空间内一个连续的虚拟地址区间。具有独立的访问权限。
- vma 访问权限标志：
  - `VM_READ, VM_WRITE, VM_EXEC, VM_SHARED` 等等
- 使用 vma 的接口：
  - 搜索: `find_vma*` 系列
  - 创建映射：`do_mmap`，可以创建文件映射或者匿名映射
  - 删除映射：`o_mummap`
- 页表：保存虚拟地址与物理地址之间的映射关系
	- 为什么三级页表能够节省内存空间？
- 每个进程拥有自己的页表
- 硬件 TLB 是页表的缓存

## 虚拟文件系统

主要数据结构：

- `file_system_type`

	每个 file_system_type 对应一种文件系统类型，例如 ext3, ext4, procfs 等。每种类型可以对应多个实例
  
- `super_block`, `vfsmount`
	
	表示一个被挂载的文件系统实例(a.k.a. mounted filesystem descriptor)。superblock 通常保存在磁盘上每个分区的开始处。在挂载文件系统时，内核会从磁盘上读取 superblock，superblock 中保存的是整个文件系统的 inode。从磁盘上读入之后，内核在内存中维护一个 super_block 结构，内存中的结构会定期与磁盘同步。

- `inode`

	表示一个实际的文件(的元数据)。(a.k.a. unqiue file or direcotry descriptor)
  inode 保存文件除了实际内容之外的信息
  -  权限/所有者/访问时间
  - 在磁盘上的 block 地址列表
  - inode number：在一个文件系统内是唯一的
  - i_flock, i_wait: 锁列表和等待队列
  - i_op:inode 相关操作

- `dentry`

	每个文件都有一个 inode，也都有一个 dentry。内核在进行路径遍历时根据文件路径字符串现场创建 dentry。dentry 的存在纯粹是为了建立一个目录结构，在访问文件时进行快速遍历，在磁盘上没有与之相对应的数据

- `file`
  
	file 表示内存中一个被打开的文件，包含进程与文件交互的信息

- `files_struct`
  
	在 task_struct 中存在一个 `struct files_struct *files`。

	其中含有一个 `struct file __rcu * fd_array[NR_OPEN_DEFAULT];`
	
	保存进程打开的所有文件。

- `fs_struct`
  
	task_struct 的 `fs` 成员。表示进程所在的文件系统，含有 `root, pwd` 成员表示进程的根目录和工作目录。

- `namespace`

## 块设备

支持随机寻址的设备。

- 块：软件最小可寻址单元，一般为 512B/1KB/4KB 等，小于内存页面大小

- 描述块与内存页之间映射的数据结构：`buffer_head`
	
	包含了该块的物理地址（设备名，物理偏移）和内存地址（页面地址，页面内偏移），以及一些控制信息，如引用计数、块状态（是否脏、是否被锁）等
	
- 描述一次 I/O 操作的数据结构：`bio`
	
	将一次 I/O 操作分割为地址链表和操作函数。块地址使用 `bio_vec` 结构表示，
	
	它只包含`<page, offset, len>` 信息。每次 I/O 操作，就是使用操作函数对这个链表进行遍历。
	
	Q：一个 bio 描述的是究竟是什么操作？一个磁盘块只能对应一个页面，一次 bio 中有多个页面，因此应该是多个磁盘块。但是在 bio 结构中好像只有页面地址的链表，却没有块设备地址的链表啊。物理上的块是不是连续的呢？
	
- I/O 调度策略
  - Linus 电梯
  - Deadline 调度
  - 预测 I/O 调度
  - CFQ 调度
  - 空操作调度


## 页面缓存

- 三种写策略
- 缓存回收：双链 LRU
- 数据结构：`address_space`，维护一个缓存节点与多个虚拟页面之间的映射
- 同步：`flusher` 线程
- 回写策略：
  - 空闲内存阈值
  - 脏页驻留时间阈值
  - sync/fsync 系统调用

## 设备

### 设备类型

主要三种：

- 块设备：支持随机寻址
- 字符设备：仅提供数据流式访问（存在一个当前位置指针）
- 网络设备：不通过设备节点，而是通过 socket 访问

其他的次要类型：

- 杂项设备（miscdev）
- 伪设备：如 /dev/random, /dev/urandom, /dev/null, /dev/zero 等

### 统一设备模型

描述设备在系统中的拓扑结构，能够将系统中所有设备使用树的形式表示。最初是为了构造设备树，从叶节点向根遍历，按照正确的顺序关闭电源。

核心数据结构：

- `kobject`：具有 `parent` 指针，用来形成树状结构。
- `kobj_type`：定义了 `kobject` 的基本操作和属性。
- `kset`：将多个 `kobject` 联系在一起的容器。

### sysfs

基于 `kobject` 实现的在内存中的虚拟文件系统，包含了系统中所有设备的拓扑结构。每个设备`kobject` 在 sysfs 中被表示成为一个目录，目录中的文件就是 `kobject` （设备）的属性。

对设备目录中的文件进行读写可以实现对设备信息的查询和修改。sysfs 实际上提供了内核与用户空间交流的接口。
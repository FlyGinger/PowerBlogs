---
title: DPDK Programmer's Guide笔记
date: 2023-08-04 16:49:12
categories: DPDK
tags: [DPDK,Linux]
---

> 本文基于DPDK 22.11.2。

## Environment Abstraction Layer

EAL（Environment Abstraction Layer）负责访问低级资源，比如硬件和内存空间。它为应用和库提供了一个通用接口，隐藏了环境特定的细节。

### Linux用户空间下的EAL

在Linux用户空间环境中，DPDK应用利用`pthread`库，以用户空间应用的身份运行。EAL利用`mmap()`在`hugetlbfs`中进行物理内存分配，然后将内存暴露给DPDK服务层，比如`mempool`库。

#### 初始化和核心启动

部分初始化是由`glibc`的`start`函数完成的，然后就进入了DPDK应用的`main()`函数。核心的初始化和启动是通过`rte_eal_init()`函数完成的，`rte_eal_init()`中使用`pthread_self()`、`pthread_create()`、`pthread_setaffinity_np()`创建执行单元，并分配它到特定的逻辑核。

![EAL在Linux用户态环境中初始化](DPDK-Programmer-s-Guide笔记/linuxapp_launch.svg)

内存空间、ring、内存池、LPM表和哈希表等对象的初始化也是在main逻辑核，是整个应用初始化的一部分。这些对象的创建和初始化函数不是线程安全的，但完成初始化之后，这些对象本身是线程安全的。

``` plaintext
rte_eal_init():
  lib/eal/include/rte_eal.h
  lib/eal/linux/eal.c
```

#### 内存映射发现和内存预留

EAL使用大页完成大块连续物理内存的分配，并且EAL提供了在这块连续内存中预留命名内存空间的API。

> DPDK作为用户空间应用框架，使用DPDK的软件需要处理的是虚拟地址。但是，硬件不能理解用户空间虚拟地址，而是使用IO地址。IO地址可能是物理地址（Physical Addresses，PA），也可能是IO虚拟地址（IO Virtual Addresses，IOVA）。
>
> DPDK不区分PA和IOVA，而是统一称作IOVA。但是，DPDK会区分PA直接作为IOVA（IOVA as PA），还是IOVA匹配用户空间虚拟地址（IOVA as VA）这两种情况。
>
> ![IOVA as PA模式](DPDK-Programmer-s-Guide笔记/deep-dive-into-iova-fig01-813747.png)
>
> IOVA as PA模式下，分配给DPDK的所有内存区域的IOVA都是实际的PA，并且虚拟内存的布局与物理内存的布局相同。IOVA as PA的优势在于适用于所有硬件，并且与内核空间之间也很适配（因为物理地址到内核空间虚拟地址的映射是直接映射）。IOVA as PA的缺点之一在于需要root权限，否则无法获取内存区域的真实物理地址。缺点之二是物理内存的布局会影响虚拟内存的布局。如果物理内存碎片化比较严重，那么虚拟内存也会是相同程度的碎片化，从而影响DPDK应用对虚拟内存的使用。
>
> ![IOVA as VA模式](DPDK-Programmer-s-Guide笔记/deep-dive-into-iova-fig03-813747.png)
>
> IOVA as VA模式下，物理内存会被重新排列，以匹配虚拟内存空间的布局。DPDK通过内核提供的功能完成这一操作，而内核则是通过IOMMU来完成物理内存重映射的。IOVA as VA的优点之一就是无需root权限，之二就是避免物理内存的碎片化影响到虚拟内存。IOVA as VA的缺点在于需要IOMMU的支持。
>
> VFIO（Virtual Function I/O）是内核基础设置，可以把设备寄存器和中断暴露给用户空间应用，并且可以利用IOMMU建立IOVA映射。

DPDK的内存子系统有两种模式，动态模式和传统模式。

动态模式下，DPDK使用的大页数量会随着DPDK应用需求而增减。这种模式下分配的内存不能保证是IOVA连续的。如果需要连续的多页IOVA，建议对所有的物理设备使用VFIO驱动，或使用传统模式。动态模式下也可以使用`-m`或`--socket-mem`指定预分配内存的大小，预分配的内存在运行时不会被释放。动态模式下可以使用`--single-file-segments`命令行参数把多个内存页放在同一个文件中，以满足例如用户空间vhost等应用对于页文件描述符数量的限制。可以使用`--socket-limit`命令行参数来限制DPDK应用可以使用的内存的最大数量。

传统模式下，EAL会在启动时预留全部内存，然后将其重排为IOVA连续的一大块，并且不会在运行时申请或释放大页。需要使用`--legacy-mem`EAL命令行参数启用传统模式，如果不用`-m`或`--socket-mem`指定大小，那么DPDK将使用所有可用的内存大页。

Linux下可以使用`hugetlbfs`中的文件或者匿名映射来获取大页。使用多进程时必须使用`hugetlbfs`，因为多个进程需要映射到同一个大页。EAL会在`--huge-dir`指定的目录中创建`--file-prefix`指定前缀的文件。匿名映射模式虽然不支持多进程，但是优点在于不需要root权限。

### 多线程

DPDK通常会在每个核心上绑定一个`pthread`来避免任务切换导致的性能损失，这可以提供显著的性能提升，但缺少灵活性，而且也不总是高效的。

`lcore`代表EAL线程，在Linux实现中就是`pthread`。`lcore`由EAL创建和管理，并负责完成`rte_eal_remote_launch`发出的工作指令。可以使用`--lcores`命令行参数来设置`lcore`的核心亲和性，从而间接完成绑核操作。

`rte_ring`支持多生产者多消费者队列，但是是非抢占式的，即同一个`ring`同时只能有一个线程进行入队或者出队的操作。如果某个线程A正在进行入队操作，另一个线程B也想要对同一个队列进行入队操作，则B只能等待；如果此时线程A被更高优先级的线程抢占，那么有可能会发生死锁。

> [Linux CPU调度概览](https://man7.org/linux/man-pages/man7/sched.7.html)

POSIX API定义了一种概念：异步信号安全函数（async-signal-safe function），指的是可以安全地在信号处理函数中使用的函数。许多DPDK函数不是可重入的（non-reentrant），因此不是异步信号安全函数。

> [异步信号安全函数](https://man7.org/linux/man-pages/man7/signal-safety.7.html)
>
> [可重入](https://en.wikipedia.org/wiki/Reentrancy_(computing))
>
> 举例来说，函数A执行过程中，被更高优先级的任务B（例如中断、信号处理）抢占，而在任务B中又调用了函数A。如果A在这种情况下也能保证正确性，那么A就是可重入的。

### `malloc`

EAL提供了`malloc` API用于分配任意大小的内存。通常来说，`malloc` API不应该在数据面中使用，因为它比基于池（pool-based）的API慢，因为分配和释放过程中使用了锁。

`malloc`库中有两个主要数据结构：

- `malloc_heap`，用于跟踪每个`socket`的空闲空间，是一个双向链表。
- `malloc_elem`，分配和空闲空间跟踪的基本元素，是链表中的节点。

`malloc_heap`中有以下关键字段：

- `lock`：用于控制同步访问。由于`malloc_heap`中存放着跟踪空闲空间的链表，因此需要防止多个线程同时对链表进行操作。
- `free_head`：指向第一个空闲空间链表节点。
- `first`：指向`malloc_heap`中第一个`malloc_elem`。
- `last`：指向`malloc_heap`中最后一个`malloc_elem`。

![`malloc_heap`和`malloc_elem`举例](DPDK-Programmer-s-Guide笔记/malloc_heap.svg)

由于在动态模式下大页是在运行时向系统申请和释放的，所以相邻的`malloc_elem`对应的实际内存不一定相邻。并且，横跨多页的`malloc_elem`中的页也可能不是IOVA连续的，每个`malloc_elem`只能保证其中的内存是VA连续的。

`malloc_elem`有两种用途，其一是作为空闲或已分配内存的header，其二是作为header padding。

`malloc_elem`中有以下关键字段：

- `heap`：指向持有该`malloc_elem`的`malloc_heap`的指针。
- `prev`、`next`：指向前一个和后一个元素。当前元素被释放时，可以查看是否可以和前后元素进行合并。
- `free_list`：指向前一个和后一个空闲节点。
- `state`：可选值有`FREE`、`BUSY`、`PAD`，表示空闲、已使用或者作为padding。
- `dirty`：仅在`state == FREE`时有效，表示内存中的内容不是全部置零的。`dirty`被置位的情况只会发生在使用`--huge-unlink=never`命令行参数的时候。
- `pad`：保存了padding的长度。
- `size`：包含header在内的数据块总长度。

> 当`hugetlbfs`中的文件或文件的一部分首次在全系统范围映射时，内核会清空其中的数据，以防止数据泄露。EAL会在启动时删除现有后备文件（backing file）并重新创建，然后再进行映射，以保证数据被清空。
>
> 清空内存占了大页映射总耗时的95%以上，因此可以使用`--huge-unlink=never`让EAL直接使用现有的后备文件以及其中的数据，从而跳过清空内存的过程。通常`--huge-unlink=never`用于加快应用重启的速度。

当`malloc_heap`中没有足够的空间满足分配请求时，EAL会像系统申请更多内存。任何申请新页的请求都必须经过主进程。如果主进程不活跃，那么无法申请任何新内存。主进程负责决定什么应该被映射、什么不应该被映射。每个次进程都有自己的本地内存映射，但它们不能修改这些映射，只能从主进程拷贝内存映射到本地内存映射。

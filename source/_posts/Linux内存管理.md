---
title: Linux内存管理
date: 2023-06-27 16:57:25
categories: 学习笔记
tags: [Linux, 内存, 透明大页]
---

## 基本概念

> <https://www.kernel.org/doc/html/next/admin-guide/mm/concepts.html>

### 虚拟内存

计算机系统的物理内存是有限的，而且不一定是连续的。虚拟内存可以屏蔽物理内存的这些细节，让每个程序都认为自己可以访问一段独立、连续、完整的内存空间。

使用虚拟内存技术时，CPU需要将虚拟地址翻译为物理地址。物理内存是按页组织的，不同架构上内存页的大小可能不同，典型值是4K。物理页和虚拟页之间存在一对多的映射关系，这种映射关系保存在页表之中。

页表可以是多级结构。最低级页表中保存了最终的物理地址，高级页表中保存的是低一级页表的物理地址。CPU中有一个寄存器，保存了指向最高级页表的指针。

当CPU翻译虚拟地址时，会按从高到低的顺序访问页表。在页表的每一级，截取虚拟地址的高位作为索引，低位传入下一级页表。最终剩下的最低几位代表该地址在这一页内存中的位置。

### 大页

对于CPU来说，内存访问是很慢的，将虚拟地址翻译为物理地址需要多次访问内存。为了提高性能，有了TLB（Translation Lookaside Buffer）的设计，它是页表的缓存。

许多CPU架构允许高级页表直接映射至物理内存（而不是低一级页表）。比如x86，可以在二级和三级页表中使用2M甚至1G的大页。使用大页能够显著地减少TLB miss。

在Linux中，有两种方式使用大页。第一种是HugeTLB文件系统，也称hugetlbfs，它是一种内存文件系统。hugetlbfs需要用户配置具体哪些内存被映射至大页。

另一种是THP（Transparent HugePages），它无需用户配置。

### 匿名内存

匿名内存（anonymous memory）代表没有对应文件系统的内存，与之对应的是文件和设备映射内存等。通常堆、栈都属于匿名内存。

### 回收（Reclaim）

在系统的生命周期中，单个物理页可能被用于存储不同类型的数据，例如内核的内部数据结构、DMA缓存、从文件系统中读取的数据、用户空间进程分配的内存等。Linux会根据用途区别对待物理页。

某些页起缓存的作用，其中的数据可从其他位置（例如硬盘）再次获取，还有些页中的数据可以被交换出内存，这两类内存页可以随时释放。可以随时释放的页是可回收（reclaimable）页。可回收页主要是页缓存（page cache）和匿名内存，其中页缓存是内存中对硬盘数据的缓存。

在内存基本没被使用时，Linux系统中的大多数内存分配请求能够立刻被满足。当内存占用达到low watermask时，内存分配请求会唤醒`kswapd`后台驻留程序，异步地对可回收页进行回收。当内存占用达到min watermask后，内存分配请求需要等待系统同步地回收其他内存页，以满足该请求对内存的需求，这称为直接回收。

### 压缩（Compaction）

内存压缩会把内存空间（zone）中较低部分的内存页挪到较高部分，使得内存空间较低部分有连续的大块儿可用内存。

压缩可以通过`kcompactd`后台驻留程序异步地完成，也可以由内存分配请求触发，同步地完成。

## 透明大页

> <https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html>

### 目标

透明大页目前仅支持匿名内存、tmpfs和shmem。tmpfs是一种基于内存的文件系统；shmem是一种进程间通信的方式。

透明大页主要在两方面提升了应用程序的效率。第一是，页错误每次处理的内存大小更大了，从而大幅减少了进入和退出内核态的次数。第二与TLB miss相关，分为两个方面。首先是TLB miss的惩罚更低了，因为页表层级变少了。其次是TLB miss数量降低了，因为每个TLB条目都对应了更大的一片内存。

透明大页可以在系统范围内开启，也可以只为某些任务开启，甚至可以单独为任务内存空间的一部分开启。除非透明大页被完全禁用，否则`khugepaged`会扫描内存，然后把连续的普通页转换为大页。

透明大页的行为可以通过sysfs接口或者madvise和prctl系统调用来控制。

### sysfs

透明大页可以完全禁用（通常为了debug），或只在`MADV_HUGEPAGE`区域开始（通常用于内存有限的系统中），或者完全开启。

``` bash
echo always >/sys/kernel/mm/transparent_hugepage/enabled
echo madvise >/sys/kernel/mm/transparent_hugepage/enabled
echo never >/sys/kernel/mm/transparent_hugepage/enabled
```

有时分配透明大页的请求不能被立刻满足，此时可以选择等待，或者直接回收，或者回退到使用普通页。为了分配大页而整理内存需要占用CPU资源，所以我们当然希望物超所值，即分配了大页带来的好处超过占用的CPU资源。

透明大页的`defrag`配置有以下选项：

- `awlays`：同步地分配透明大页，即无法立即分配时阻塞，执行回收和压缩动作，从而完成分配。
- `defer`：异步地分配透明大页，即唤醒`kswapd`、`kcompactd`来异步地完成回收和压缩，然后通过`khugepaged`分配透明大页。
- `defer+madvise`：在`MADV_HUGEPAGE`区域使用`awlays`策略，其他区域使用`defer`策略。
- `madvise`：在`MADV_HUGEPAGE`区域使用`awlays`策略，其他区域使用`never`策略。此为默认配置。
- `never`：无法立即分配时回退到使用普通页。

``` bash
echo always >/sys/kernel/mm/transparent_hugepage/defrag
echo defer >/sys/kernel/mm/transparent_hugepage/defrag
echo defer+madvise >/sys/kernel/mm/transparent_hugepage/defrag
echo madvise >/sys/kernel/mm/transparent_hugepage/defrag
echo never >/sys/kernel/mm/transparent_hugepage/defrag
```

内核默认在匿名内存的读页错误发生时使用零页（zero page），但是这也是可配置的。

``` bash
echo 0 >/sys/kernel/mm/transparent_hugepage/use_zero_page
echo 1 >/sys/kernel/mm/transparent_hugepage/use_zero_page
```

通过以下命令可以查看透明大页的尺寸。

``` bash
cat /sys/kernel/mm/transparent_hugepage/hpage_pmd_size
```

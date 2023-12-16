---
title: TCMalloc：线程缓存内存分配【译】
author: june
date: 2023-12-16
category: go
layout: post
---

原文：《[TCMalloc : Thread-Caching Malloc](https://google.github.io/tcmalloc/design.html)》

### Motivation

TCMalloc is a memory allocator designed as an alternative to the system default allocator that has the following characteristics:
* Fast, uncontended allocation and deallocation for most objects. Objects are cached, depending on mode, either per-thread, or per-logical-CPU. Most allocations do not need to take locks, so there is low contention and good scaling for multi-threaded applications.
* Flexible use of memory, so freed memory can be reused for different object sizes, or returned to the OS.
* Low per object memory overhead by allocating “pages” of objects of the same size. Leading to space-efficient representation of small objects.
* Low overhead sampling, enabling detailed insight into applications memory usage.

TCMalloc 是一个内存分配器，旨在替代系统默认分配器，具有以下特征：
* 大多数对象是快速、无竞争的分配和释放。对象的缓存取决于不同的模式（按线程或按逻辑CPU模式）大多数分配不需要获取锁，因此对于多线程应用，分配具备低竞争性和易扩展性。
* 灵活使用内存，因此释放的内存可以重新用于不同的对象大小，或返回给操作系统。
* 通过分配相同大小的对象“页”，降低每个对象的内存开销。实现小对象的空间高效表示。
* 低开销采样，可以详细了解应用程序内存使用情况。

### Usage

You use TCMalloc by specifying it as the `malloc` attribute on your binary rules in Bazel.

您可以通过将TCMalloc指定为Bazel中二进制规则的`malloc`属性来使用它。

### Overview

The following block diagram shows the rough internal structure of TCMalloc（下面的框图展示了TCMalloc的大致内部结构）:

![tcmalloc_f1](/assets/post/go/tcmalloc_f1.png "tcmalloc_f1")

We can break TCMalloc into three components. The front-end, middle-end, and back-end. We will discuss these in more details in the following sections. A rough breakdown of responsibilities is:
* The front-end is a cache that provides fast allocation and deallocation of memory to the application.
* The middle-end is responsible for refilling the front-end cache.
* The back-end handles fetching memory from the OS.

我们可以将TCMalloc分为三个部分。前端、中端、后端。我们将在以下部分中更详细地讨论这些内容。职责的粗略细分是：
* 前端是一个缓存，为应用程序提供快速的内存分配和释放。
* 中端负责重新填充前端缓存。
* 后端处理从操作系统获取内存。

Note that the front-end can be run in either per-CPU or legacy per-thread mode, and the back-end can support either the hugepage aware pageheap or the legacy pageheap.

请注意，前端可以在每CPU或传统每线程模式下运行，后端可以支持巨页感知页堆或传统页堆。

### The TCMalloc Front-end

The front-end handles a request for memory of a particular size. The front-end has a cache of memory that it can use for allocation or to hold free memory. This cache is only accessible by a single thread at a time, so it does not require any locks, hence most allocations and deallocations are fast.

前端处理对特定大小的内存的请求。前端有一个内存缓存，可用于分配或保存空闲内存。该缓存同一时刻只能由一个线程访问，因此不需要任何锁，因此大多数分配和释放速度都很快。

The front-end will satisfy any request if it has cached memory of the appropriate size. If the cache for that particular size is empty, the front-end will request a batch of memory from the middle-end to refill the cache. The middle-end comprises the CentralFreeList and the TransferCache.

如果前端具有适当大小的缓存内存，它将满足任何请求。如果该特定大小的缓存为空，前端会向中端请求一批内存来重新填充缓存。中端包括CentralFreeList和TransferCache。

If the middle-end is exhausted, or if the requested size is greater than the maximum size that the front-end caches handle, a request will go to the back-end to either satisfy the large allocation, or to refill the caches in the middle-end. The back-end is also referred to as the PageHeap.

如果中端耗尽，或者请求的大小大于前端缓存处理的最大大小，则请求将转到后端以满足大分配，或者重新填充中端的缓存中端。后端也称为页堆PageHeap。

There are two implementations of the TCMalloc front-end:
* Originally it supported per-thread caches of objects (hence the name Thread Caching Malloc). However, this resulted in memory footprints that scaled with the number of threads. Modern applications can have large thread counts, which result in either large amounts of aggregate per-thread memory, or many threads having minuscule per-thread caches.
* More recently TCMalloc has supported per-CPU mode. In this mode each logical CPU in the system has its own cache from which to allocate memory. Note: On x86 a logical CPU is equivalent to a hyperthread.

TCMalloc前端有两种实现：
* 最初它支持对象的线程缓存（因此称为“Thread Caching Malloc”）。然而，这会导致内存占用随着线程数量的增加而增加。现代应用程序可能具有大量线程数，这会导致大量的线程聚合内存，或者许多线程具有极小的线程缓存。
* 最近，TCMalloc支持CPU模式。在此模式下，系统中的每个逻辑CPU都有自己的缓存，可从中分配内存。注意：在x86上，逻辑CPU相当于超线程。

The differences between per-thread and per-CPU modes are entirely confined to the implementations of malloc/new and free/delete.

线程和CPU模式之间的差异完全局限于malloc/new和free/delete的实现。

### Small and Large Object Allocation

Allocations of “small” objects are mapped onto one of 60-80 allocatable size-classes. For example, an allocation of 12 bytes will get rounded up to the 16 byte size-class. The size-classes are designed to minimize the amount of memory that is wasted when rounding to the next largest size-class.

“小”对象的分配被映射到60-80个可分配size-class之一。例如，12字节的分配将向上取整16字节的size-class。size-class旨在最大限度地减少舍入到下一个最大size-class时浪费的内存量。

When compiled with `__STDCPP_DEFAULT_NEW_ALIGNMENT__ <= 8`, we use a set of sizes aligned to 8 bytes for raw storage allocated with `::operator new`. This smaller alignment minimizes wasted memory for many common allocation sizes (24, 40, etc.) which are otherwise rounded up to a multiple of 16 bytes. On many compilers, this behavior is controlled by the `-fnew-alignment=...` flag. When `__STDCPP_DEFAULT_NEW_ALIGNMENT__` is not specified (or is larger than 8 bytes), we use standard 16 byte alignments for `::operator new`. However, for allocations under 16 bytes, we may return an object with a lower alignment, as no object with a larger alignment requirement can be allocated in the space.

当使用`__STDCPP_DEFAULT_NEW_ALIGNMENT__ <= 8`进行编译时，我们使用一组与8字节对齐的大小来分配通过`::operator new`分配的原始存储。这种较小的对齐方式最大限度地减少了许多常见分配大小（24、40等）的内存浪费，否则这些分配大小将四舍五入为16字节的倍数。在许多编译器上，此行为由`-fnew-alignment=...`标志控制。当未指定`__STDCPP_DEFAULT_NEW_ALIGNMENT__`时（或大于8字节），我们对`::operator new`使用标准16字节对齐。但是，对于16字节以下的分配，我们可能会返回具有较低对齐方式的对象，因为无法在该空间中分配具有较大对齐方式要求的对象。

When an object of a given size is requested, that request is mapped to a request of a particular size-class using the `SizeMap::GetSizeClass()` function, and the returned memory is from that size-class. This means that the returned memory is at least as large as the requested size. Allocations from size-classes are handled by the front-end.

当请求给定大小的对象时，使用`SizeMap::GetSizeClass()`函数将该请求映射到特定size-class的请求，并且返回该size-class的内存。这意味着返回的内存至少与请求的大小一样大。size-class的分配由前端处理。

Objects of size greater than the limit defined by `kMaxSize` are allocated directly from the backend. As such they are not cached in either the front or middle ends. Allocation requests for large object sizes are rounded up to the TCMalloc page size.

大小大于`kMaxSize`定义的限制的对象直接从后端分配。因此，它们不会缓存在前端或中端。大对象大小的分配请求将向上舍入为TCMalloc 页面大小。

### Deallocation

When an object is deallocated, the compiler will provide the size of the object if it is known at compile time. If the size is not known, it will be looked up in the pagemap. If the object is small it will be put back into the front-end cache. If the object is larger than kMaxSize it is returned directly to the pageheap.

当对象被释放时，如果在编译时已知该对象的大小，编译器将提供该对象的大小。如果尺寸未知，则会在页面映射中查找。如果对象很小，则会被放回到前端缓存中。如果对象大于kMaxSize，则直接返回到页堆。

### Per-CPU Mode

In per-CPU mode a single large block of memory is allocated. The following diagram shows how this slab of memory is divided between CPUs and how each CPU uses a part of the slab to hold metadata as well as pointers to available objects.

在CPU模式下，分配一个大内存块。下图显示了该内存块如何在CPU之间划分，以及每个CPU如何使用该内存块的一部分来保存元数据以及指向可用对象的指针。

![tcmalloc_f2](/assets/post/go/tcmalloc_f2.png "tcmalloc_f2")

Each logical CPU is assigned a section of this memory to hold metadata and pointers to available objects of particular size-classes. The metadata comprises one /header/ block per size-class. The header has a pointer to the start of the per-size-class array of pointers to objects, as well as a pointer to the current, dynamic, maximum capacity and the current position within that array segment. The static maximum capacity of each per-size-class array of pointers is determined at start time by the difference between the start of the array for this size-class and the start of the array for the next size-class.

每个逻辑CPU都被分配了该内存的一部分来保存元数据和指向特定size-classe的可用对象的指针。元数据包含每个size-classe的一个/头部/块。头部有一个指向每个size-classe对象指针数组开头的指针，以及指向该数组段中当前、动态、最大容量和当前位置的指针。每个size-classe的指针数组的静态最大容量在开始时由该size-classe的数组开头与下一个size-classe的数组开头之间的差异确定。

At runtime the maximum number of items of a particular size-class that can be stored in the per-cpu block will vary, but it can never exceed the statically determined maximum capacity assigned at start up.

在运行时，每个CPU块中可以存储的特定大小类别的最大数量会有所不同，但它永远不会超过启动时分配的静态确定的最大容量。

When an object of a particular size-class is requested it is removed from this array, when the object is freed it is added to the array. If the array is exhausted the array is refilled using a batch of objects from the middle-end. If the array would overflow, a batch of objects are removed from the array and returned to the middle-end.

当请求特定size-class的对象时，它将从该数组中移除，当该对象被释放时，它将加回该数组中。如果数组耗尽，则使用中端的一批对象重新填充数组。如果数组溢出，则从数组中移除一批对象并返回到中端。

The amount of memory that can be cached is limited per-cpu by the parameter `MallocExtension::SetMaxPerCpuCacheSize`. This means that the total amount of cached memory depends on the number of active per-cpu caches. Consequently machines with higher CPU counts can cache more memory.

每个CPU可以缓存的内存大小由参数`MallocExtension::SetMaxPerCpuCacheSize`限制。这意味着缓存内存总量取决于每个CPU的活跃缓存数量。因此，CPU数量较多的机器可以缓存更多内存。

To avoid holding memory on CPUs where the application no longer runs, `MallocExtension::ReleaseCpuMemory` frees objects held in a specified CPU’s caches.

为了避免不再运行应用程序的CPU持有内存，`MallocExtension::ReleaseCpuMemory`会释放指定CPU缓存中保存的对象。

Within a CPU, the distribution of memory is managed across all the size-classes so as to keep the maximum amount of cached memory below the limit. Notice that it is managing the maximum amount that can be cached, and not the amount that is currently cached. On average the amount actually cached should be about half the limit.
在CPU内，跨所有size-class管理内存分布，以便将最大缓存内存大小保持在限制以下。请注意，它管理的是可以缓存的最大数量，而不是当前缓存的数量。平均而言，实际缓存的数量应约为限制的一半。

The maximum capacity is increased when a size-class runs out of objects, and when fetching more objects, it also considers increasing the capacity of the size-class. It can increase the capacity of the size-class up until the total memory (for all size-classes) that the cache could hold reaches the per-cpu limit or until the capacity of that size-class reaches the hard-coded size limit for that size-class. If the size-class has not reached the hard-coded limit, then in order to increase the capacity it can steal capacity from another size-class on the same CPU.

当某个size-class的对象用完时，最大容量会增加，当获取更多的对象时，也会考虑增加该size-class的容量。它可以增加大小类别的容量，直到缓存可以容纳的总内存（对于所有大小类别）达到每个 CPU 的限制，或者直到该size-class的容量达到硬编码的大小限制那个尺寸等级。如果size-class尚未达到硬编码限制，那么为了增加容量，它可以从同一CPU上的另一个size-class窃取容量。

### Restartable Sequences and Per-CPU TCMalloc

To work correctly, per-CPU mode relies on restartable sequences (man rseq(2)). A restartable sequence is just a block of (assembly language) instructions, largely like a typical function. A restriction of restartable sequences is that they cannot write partial state to memory, the final instruction must be a single write of the updated state. The idea of restartable sequences is that if a thread is removed from a CPU (e.g. context switched) while it is executing a restartable sequence, the sequence will be restarted from the top. Hence the sequence will either complete without interruption, or be repeatedly restarted until it completes without interruption. This is achieved without using any locking or atomic instructions, thereby avoiding any contention in the sequence itself.

为了正常工作，CPU模式依赖于可重新启动序列 (man rseq(2))。可重新启动的序列只是一个（汇编语言）指令块，很大程度上类似于经典函数。可重启序列的一个限制是它们不能将部分状态写入内存，最终指令必须是更新状态的单次写入。可重启序列的想法是，如果一个在执行可重启序列的线程从CPU中移除（例如上下文切换），则该序列将从顶部重新启动。因此，该序列要么不间断地完成，要么重复重新启动，直到不间断地完成。这是在不使用任何锁或原子指令的情况下实现的，从而避免了序列本身的任何竞争。

The practical implication of this for TCMalloc is that the code can use a restartable sequence like TcmallocSlab_Internal_Push to fetch from or return an element to a per-CPU array without needing locking. The restartable sequence ensures that either the array is updated without the thread being interrupted, or the sequence is restarted if the thread was interrupted (for example, by a context switch that enables a different thread to run on that CPU).

这对于TCMalloc的实际意义是，代码可以使用可重新启动的序列（如TcmallocSlab_Internal_Push）从每个 CPU 数组中获取或返回元素，而无需锁。可重新启动的序列可确保在线程不中断的情况下更新数组，或者在线程被中断时重新启动序列（例如，通过允许不同线程在该CPU上运行的上下文切换）。

Additional information about the design choices and implementation are discussed in a specific design doc for it.

有关设计选择和实现的其他信息将在其特定的设计文档中讨论。

## Legacy Per-Thread mode

In per-thread mode, TCMalloc assigns each thread a thread-local cache. Small allocations are satisfied from this thread-local cache. Objects are moved between the middle-end into and out of the thread-local cache as needed.

在线程模式下，TCMalloc为每个线程分配一个线程本地缓存。小分配可以通过该线程本地缓存来满足。根据需要，对象在中端之间移入和移出线程本地缓存。

A thread cache contains one singly linked list of free objects per size-class (so if there are N size-classes, there will be N corresponding linked lists), as shown in the following diagram.

线程缓存包含每个size-class的一个空闲对象的单链表（因此，如果有N个size-class，就会有N个相应的链表），如下图所示。

![tcmalloc_f3](/assets/post/go/tcmalloc_f3.png "tcmalloc_f3")

On allocation an object is removed from the appropriate size-class of the per-thread caches. On deallocation, the object is prepended to the appropriate size-class. Underflow and overflow are handled by accessing the middle-end to either fetch more objects, or to return some objects.

分配时，对象将从线程缓存的恰当的size-class中移除。释放时，对象会被添加到恰当的size-class头部。下溢和溢出是通过访问中端来处理的，或者获取更多对象，或者返回一些对象。

The maximum capacity of the per-thread caches is set by the parameter `MallocExtension::SetMaxTotalThreadCacheBytes`. However it is possible for the total size to exceed that limit as each per-thread cache has a minimum size KMinThreadCacheSize which is usually 512KiB. In the event that a thread wishes to increase its capacity, it needs to scavenge capacity from other threads.

每个线程缓存的最大容量由参数`MallocExtension::SetMaxTotalThreadCacheBytes`设置。但是，总大小可能会超过该限制，因为每个线程缓存都有一个最小大小KMinThreadCacheSize，通常为512KiB。如果线程希望增加其容量，则需要从其他线程中清除容量。

### Runtime Sizing of Front-end Caches

It is important for the size of the front-end cache free lists to adjust optimally. If the free list is too small, we’ll need to go to the central free list too often. If the free list is too big, we’ll waste memory as objects sit idle in there.

对于前端缓存空闲列表的大小进行优化调整非常重要。如果空闲列表太小，我们就需要过于频繁地访问中央空闲列表。如果空闲列表太大，我们就会浪费内存，因为对象会闲置在那里。

Note that the caches are just as important for deallocation as they are for allocation. Without a cache, each deallocation would require moving the memory to the central free list.

请注意，缓存对于释放和分配同样重要。如果没有缓存，每次释放都需要将内存移动到中央空闲列表。

Per-CPU and per-thread modes have different implementations of a dynamic cache sizing algorithm.
* In per-thread mode the maximum number of objects that can be stored is increased up to a limit whenever more objects need to be fetched from the middle-end. Similarly the capacity is decreased when we find that we have cached too many objects. The size of the cache is also reduced should the total size of the cached objects exceed the per-thread limit.
* In per-CPU mode the capacity of the free list is increased depending on whether we are alternating between underflows and overflows (indicating that a larger cache might stop this alternation). The capacity is reduced when it has not been grown for a time and may therefore be over capacity.

CPU和线程模式具有不同的动态缓存大小调整算法实现：
* 在线程模式下，每当需要从中端获取更多对象时，可以存储的最大对象数量就会增加到限制数。同样，当我们发现缓存了太多对象时，容量就会减少。如果缓存对象的总大小超过每个线程的限制，缓存的大小也会减少。
* 在CPU模式下，空闲列表的容量会根据我们是否在下溢和溢出之间交替而增加（表明较大的缓存可能会阻止这种交替）。当容量一段时间没有增长时，容量就会减少，因此可能会超出容量。

### TCMalloc Middle-end

The middle-end is responsible for providing memory to the front-end and returning memory to the back-end. The middle-end comprises the Transfer cache and the Central free list. Although these are often referred to as singular, there is one transfer cache and one central free list per size-class. These caches are each protected by a mutex lock - so there is a serialization cost to accessing them.

中端负责向前端提供内存，并向后端返回内存。中端包括传输缓存和中央空闲列表。尽管这些通常被称为单一的，但每个size-class都有一个传输缓存和一个中央空闲列表 这些缓存均受互斥锁保护 - 因此访问它们会产生序列化成本。

### Transfer Cache

When the front-end requests memory, or returns memory, it will reach out to the transfer cache.

当前端请求内存或返回内存时，它将访问传输缓存。

The transfer cache holds an array of pointers to free memory, and it is quick to move objects into this array, or fetch objects from this array on behalf of the front-end.

传输缓存保存一个指向可用内存的指针数组，可以快速地将对象移动到该数组中，或者代表前端从该数组中获取对象。

The transfer cache gets its name from situations where one CPU (or thread) is allocating memory that is deallocated by another CPU (or thread). The transfer cache allows memory to rapidly flow between two different CPUs (or threads).

传输缓存得名于一个CPU（或线程）正在分配由另一CPU（或线程）释放的内存的情况。传输缓存允许内存在两个不同的CPU（或线程）之间快速流动。

If the transfer cache is unable to satisfy the memory request, or has insufficient space to hold the returned objects, it will access the central free list.

如果传输缓存无法满足内存请求，或者没有足够的空间来保存返回的对象，它将访问中央空闲列表。

### Central Free List

The central free list manages memory in “spans”, a span is a collection of one or more “TCMalloc pages” of memory. These terms will be explained in the next couple of sections.

中央空闲列表以“spans”的形式管理内存，一个spans是一个或多个“TCMalloc pages”的集合。这些术语将在接下来的几节中进行解释。

A request for one or more objects is satisfied by the central free list by extracting objects from spans until the request is satisfied. If there are insufficient available objects in the spans, more spans are requested from the back-end.

对一个或多个对象的请求由中央空闲列表通过从spans中提取对象来满足，直到请求得到满足。如果spans中可用对象不足，则向后端请求更多跨度。

When objects are returned to the central free list, each object is mapped to the span to which it belongs (using the pagemap) and then released into that span. If all the objects that reside in a particular span are returned to it, the entire span gets returned to the back-end.

当对象返回到中央空闲列表时，每个对象都被映射到它所属的spans（使用页面映射），然后释放到该spans。如果驻留在特定spans中的所有对象都返回给它，则整个spans都会返回到后端。

### Pagemap and Spans

The heap managed by TCMalloc is divided into pages of a compile-time determined size. A run of contiguous pages is represented by a Span object. A span can be used to manage a large object that has been handed off to the application, or a run of pages that have been split up into a sequence of small objects. If the span manages small objects, the size-class of the objects is recorded in the span.

TCMalloc管理的堆被划分为编译时确定大小的页面。span对象表示一系列连续的页面。span可用于管理已移交给应用程序的大对象，或一系列页面被划分为一系列小对象。如果span管理小对象，则对象的Size-Class会记录在Span中。

The pagemap is used to look up the span to which an object belongs, or to identify the size-class for a given object.

pagemap用于查找对象所属的span，或识别给定对象的size-class。

TCMalloc uses a 2-level or 3-level radix tree in order to map all possible memory locations onto spans.

TCMalloc使用2级或3级基数树将所有可能的内存位置映射到span上。

The following diagram shows how a radix-2 pagemap is used to map the address of objects onto the spans that control the pages where the objects reside. In the diagram span A covers two pages, and span B covers 3 pages.

下图显示了如何使用radix-2 pagemap将对象地址映射到控制对象所在页面的span上。在图中，**span A**涵盖两页，**span B**涵盖3页。

![tcmalloc_f4](/assets/post/go/tcmalloc_f4.png "tcmalloc_f4")

Spans are used in the middle-end to determine where to place returned objects, and in the back-end to manage the handling of page ranges.

Span在中端用于确定返回对象的放置位置，在后端用于管理页面范围的处理。


### Storing Small Objects in Spans

A span contains a pointer to the base of the TCMalloc pages that the span controls. For small objects those pages are divided into at most $ 2^{16} $ objects. This value is selected so that within the span we can refer to objects by a two-byte index.

span包含指向该span控制的TCMalloc页基址的指针。对于小对象，这些页面最多分为$ 2^{16} $个对象。选择该值是为了在span内我们可以通过两字节索引引用对象。

This means that we can use an unrolled linked list to hold the objects. For example, if we have eight byte objects we can store the indexes of three ready-to-use objects, and use the forth slot to store the index of the next object in the chain. This data structure reduces cache misses over a fully linked list.

这意味着我们可以使用展开的链表来保存对象。例如，如果我们有8字节对象，我们可以存储3个可用对象的索引，并使用第四个槽来存储链中下一个对象的索引。这种数据结构减少了完全链接列表上的缓存未命中率。

The other advantage of using two byte indexes is that we’re able to use spare capacity in the span itself to cache four objects.

使用两个字节索引的另一个优点是我们能够使用span本身的备用容量来缓存四个对象。

When we have no available objects for a size-class, we need to fetch a new span from the pageheap and populate it.

当我们没有可用的size-class对象时，我们需要从页堆中获取新的span并填充它。

### TCMalloc Page Sizes

TCMalloc can be built with various “page sizes” . Note that these do not correspond to the page size used in the TLB of the underlying hardware. These TCMalloc page sizes are currently 4KiB, 8KiB, 32KiB, and 256KiB.

TCMalloc可以使用各种“页面大小”构建。请注意，这些与底层硬件的TLB中使用的页面大小并不对应。这些TCMalloc页面大小当前为 4KiB、8KiB、32KiB 和 256KiB。

A TCMalloc page either holds multiple objects of a particular size, or is used as part of a group to hold an object of size greater than a single page. If an entire page becomes free it will be returned to the back-end (the pageheap) and can later be repurposed to hold objects of a different size (or returned to the OS).

TCMalloc页面要么保存特定大小的多个对象，要么用作大小大于单个页面的对象的一部分。如果整个页面变得空闲，它将被返回到后端（页面堆），并且稍后可以重新调整用途以保存不同大小的对象（或返回到操作系统）。

Small pages are better able to handle the memory requirements of the application with less overhead. For example, a half-used 4KiB page will have 2KiB left over versus a 32KiB page which would have 16KiB. Small pages are also more likely to become free. For example, a 4KiB page can hold eight 512-byte objects versus 64 objects on a 32KiB page; and there is much less chance of 64 objects being free at the same time than there is of eight becoming free.

小页面能够以更少的开销更好地处理应用程序的内存需求。例如，使用一半的4KiB页面将剩余2KiB，而32KiB页面将剩余16KiB。小页面也更有可能变得免费。例如，4KiB页面可以容纳8个512字节的对象，而32KiB页面可以容纳64个对象；并且64个对象同时空闲的可能性比8个对象同时空闲的可能性要小得多。

Large pages result in less need to fetch and return memory from the back-end. A single 32KiB page can hold eight times the objects of a 4KiB page, and this can result in the costs of managing the larger pages being smaller. It also takes fewer large pages to map the entire virtual address space. TCMalloc has a pagemap which maps a virtual address onto the structures that manage the objects in that address range. Larger pages mean that the pagemap needs fewer entries and is therefore smaller.

大页面可以减少从后端获取和返回内存的需要。单个32KiB页面可以容纳4KiB页面对象的八倍，这可能会导致管理较大页面的成本更小。映射整个虚拟地址空间也需要更少的大页面。TCMalloc有一个页面映射，它将虚拟地址映射到管理该地址范围内的对象的结构上。较大的页面意味着页面映射需要较少的条目，因此较小。

Consequently, it makes sense for applications with small memory footprints, or that are sensitive to memory footprint size to use smaller TCMalloc page sizes. Applications with large memory footprints are likely to benefit from larger TCMalloc page sizes.

因此，对于内存占用较小或对内存占用大小敏感的应用程序来说，使用较小的TCMalloc页面大小是有意义的。内存占用较大的应用程序可能会受益于较大的TCMalloc页大小。

### TCMalloc Backend

The back-end of TCMalloc has three jobs:
* It manages large chunks of unused memory.
* It is responsible for fetching memory from the OS when there is no suitably sized memory available to fulfill an allocation request.
* It is responsible for returning unneeded memory back to the OS.

TCMalloc的后端有3个工作：
* 它管理大量未使用的内存。
* 当分配请求没有合适大小的内存满足时，它负责从操作系统获取内存。
* 它负责将不需要的内存返回给操作系统。

here are two backends for TCMalloc:
* The Legacy pageheap which manages memory in TCMalloc page sized chunks.
* The hugepage aware pageheap which manages memory in chunks of hugepage sizes. Managing memory in hugepage chunks enables the allocator to improve application performance by reducing TLB misses.

TCMalloc有两个后端：
* 传统页堆以TCMalloc页大小的块管理内存。
* 大页感知页堆以大页大小的块形式管理内存。管理大页块中的内存使分配器能够通过减少TLB未命中来提高应用程序性能。

### Legacy Pageheap

The legacy pageheap is an array of free lists for particular lengths of contiguous pages of available memory. For `k < 256`, the `k`th entry is a free list of runs that consist of `k` TCMalloc pages. The `256`th entry is a free list of runs that have length `>= 256` pages:

传统页堆是可用内存的特定长度的连续页的空闲列表数组。对于`k < 256`，第`k`个条目是由`k`个TCMalloc页组成的空闲运行列表。第`256`个条目是长度`>= 256`页的空闲运行列表：

![tcmalloc_f5](/assets/post/go/tcmalloc_f5.png "tcmalloc_f5")

An allocation for `k` pages is satisfied by looking in the `k`th free list. If that free list is empty, we look in the next free list, and so forth. Eventually, we look in the last free list if necessary. If that fails, we fetch memory from the system `mmap`.

通过查看第`k`个空闲列表来满足`k`个页面的分配。如果该空闲列表为空，我们将查找下一个空闲列表，依此类推。最后，如果有必要，我们会查看最后一个空闲列表。如果失败，我们从系统`mmap`中获取内存。

If an allocation for `k` pages is satisfied by a run of pages of length `> k` , the remainder of the run is re-inserted back into the appropriate free list in the pageheap.

如果`k`个页面的分配由长度`> k`的页面运行满足，则该运行的剩余部分将重新插入到页堆中适当的空闲列表中。

When a range of pages are returned to the pageheap, the adjacent pages are checked to determine if they now form a contiguous region, if that is the case then the pages are concatenated and placed into the appropriate free list.

当一系列页面返回到页堆时，将检查相邻页面以确定它们现在是否形成连续区域，如果是这种情况，则将这些页面连接起来并放入适当的空闲列表中。

### Hugepage Aware Allocator

The objective of the hugepage aware allocator is to hold memory in hugepage size chunks. On x86 a hugepage is 2MiB in size. To do this the back-end has three different caches:
* The filler cache holds hugepages which have had some memory allocated from them. This can be considered to be similar to the legacy pageheap in that it holds linked lists of memory of a particular number of TCMalloc pages. Allocation requests for sizes of less than a hugepage in size are (typically) returned from the filler cache. If the filler cache does not have sufficient available memory it will request additional hugepages from which to allocate.
* The region cache which handles allocations of greater than a hugepage. This cache allows allocations to straddle multiple hugepages, and packs multiple such allocations into a contiguous region. This is particularly useful for allocations that slightly exceed the size of a hugepage (for example, 2.1 MiB).
* The hugepage cache handles large allocations of at least a hugepage. There is overlap in usage with the region cache, but the region cache is only enabled when it is determined (at runtime) that the allocation pattern would benefit from it.

大页感知分配器的目标是将内存保存在大页大小的块中。在x86上，大页面的大小为2MiB。为此，后端具有三个不同的缓存：
* 填充缓存保存着大页，这些大页已经分配了一些内存。这可以被认为与传统页堆类似，因为它保存特定数量的TCMalloc页的内存链表。大小小于大页的分配请求（通常）从填充缓存返回。如果填充缓存没有足够的可用内存，它将请求额外的大页来进行分配。
* 处理大于大页的分配的区域缓存。此缓存允许分配跨越多个大页，并将多个此类分配打包到连续区域中。这对于稍微超过大页大小（例如 2.1MiB）的分配特别有用。
* 大页缓存处理至少一个大页的大量分配。与区域缓存的使用存在重叠，但区域缓存仅在确定（在运行时）分配模式将从中受益时才启用。

Additional information about the design choices made in HPAA are discussed in a specific design doc for it.

有关HPAA中的设计选择的更多信息将在其特定的设计文档中讨论。

### Caveats

TCMalloc will reserve some memory for metadata at start up. The amount of metadata will grow as the heap grows. In particular the pagemap will grow with the virtual address range that TCMalloc uses, and the spans will grow as the number of active pages of memory grows. In per-CPU mode, TCMalloc will reserve a slab of memory per-CPU (typically 256 KiB), which, on systems with large numbers of logical CPUs, can lead to a multi-mebibyte footprint.

TCMalloc将在启动时为元数据保留一些内存。元数据的数量将随着堆的增长而增长。特别是，页面映射将随着TCMalloc使用的虚拟地址范围而增长，并且span将随着内存活动页面数量的增长而增长。在每CPU模式下，TCMalloc将为每个CPU保留一块内存（通常为256KiB），在具有大量逻辑CPU的系统上，这可能会导致多兆字节的占用空间。

It is worth noting that TCMalloc requests memory from the OS in large chunks (typically 1 GiB regions). The address space is reserved, but not backed by physical memory until it is used. Because of this approach the VSS of the application can be substantially larger than the RSS. A side effect of this is that trying to limit an application’s memory use by restricting VSS will fail long before the application has used that much physical memory.

值得注意的是，TCMalloc以大块（通常为1GiB区域）向操作系统请求内存。地址空间被保留，但在使用之前不受物理内存支持。由于这种方法，应用程序的VSS可能比RSS大得多。这样做的副作用是，在应用程序使用那么多物理内存之前，尝试通过限制VSS来限制应用程序的内存使用将会失败。

Don’t try to load TCMalloc into a running binary (e.g., using JNI in Java programs). The binary will have allocated some objects using the system malloc, and may try to pass them to TCMalloc for deallocation. TCMalloc will not be able to handle such objects.

不要尝试将TCMalloc加载到正在运行的二进制文件中（例如，在Java程序中使用JNI）。二进制文件将使用系统malloc分配一些对象，并可能尝试将它们传递给TCMalloc进行释放。TCMalloc将无法处理此类对象。
# [SPDK/NVMe存储技术分析\]003 - NVMeDirect论文

```
说明： 之所以要翻译这篇论文，是因为参考此论文可以很好地理解SPDK/NVMe的设计思想。
```

**[NVMeDirect: A User-space I/O Framework for Application-specific Optimization on NVMe SSDs](https://www.usenix.org/conference/hotstorage16/workshop-program/presentation/kim)**

**NVMeDirect: 面向基于NVMe固态硬盘存储应用优化的一种用户空间I/O框架**

```
Hyeong-Jun Kim, Sungkyunkwan University, hjkim@csl.skku.edu
Young-Sik Lee,  KAIST,                   yslee@calab.kaist.ac.kr
Jin-Soo Kim,    Sungkyunkwan University, jinsookim@skku.edu
```

**Abstract | 摘要**

The performance of storage devices has been increased significantly due to emerging technologies such as Solid State Drives (SSDs) and Non-Volatile Memory Express (NVMe) interface. However, the complex I/O stack of the kernel impedes utilizing the full performance of NVMe SSDs. The application-specific optimization is also difficult on the kernel because the kernel should provide generality and fairness.
新兴技术如固态硬盘(SSD)和NVMe接口的发展，使存储设备的性能得到大幅度的提升。但是，操作系统内核I/O过于复杂，于是阻碍了NVMe接口的固态硬盘发挥其全部的性能。在内核中，面向应用程序的优化实现起来也很困难，因为内核就应该提供通用性和公平性。

In this paper, we propose a user-level I/O framework which improves the performance by allowing user applications to access commercial NVMe SSDs directly without any hardware modification. Moreover, the proposed framework provides flexibility where user applications can select their own I/O policies including I/O completion method, caching, and I/O scheduling. Our evaluation results show that the proposed framework outperforms the kernel-based I/O by up to 30% on microbenchmarks and by up to 15% on Redis.
在本论文中，我们提出了一个用户级的I/O架构，允许用户应用程序直接访问商用的NVMe固态硬盘，而无需做任何硬件修改。而且，该框架给用户提供了灵活性，用户应用程序可以选择自己的I/O策略，包括I/O完成方法、缓存和I/O调度。评估结果表明，我们的提出的框架优于基于内核的I/O，在微基准测试中性能提升高达30%，应用到Redis上性能提升高达15%。

## **1 Introduction | 概述**

The new emerging technologies are making a remarkable progress in the performance of storage devices. NAND flash-based Solid State Drives (SSDs) are being widely adopted on behalf of hard disk drives (HDDs). The next-generation non-volatile memory such as 3D XPoint[8] promises the next step for the storage devices. In accordance with the improvement in the storage performance, the new NVMe (Non-Volatile Memory Express) interface has been standardized to support high performance storage based on the PCI Express (PCIe) interconnect.
新兴技术使得存储设备在性能上取得了显著的进步。基于NAND闪存的SSD正作为硬盘驱动器(HDD)的代表被广泛地采用。存储设备的下一步发展将是新一代的NVM如3D XPoint。为了提高存储性能，新的NVMe接口已经标准化，用来支持基于PCIe的高性能存储设备。

As storage devices are getting faster, the overhead of the legacy kernel I/O stack becomes noticeable since it has been optimized for slow HDDs. To overcome this problem, many researchers have tried to reduce the kernel overhead by using the polling mechanism[5, 13] and eliminating unnecessary context switching[11, 14].
存储设备变得越来越快，传统的内核I/O栈的开销变得异常明显，虽然内核针对较慢的硬盘做过一些优化。为了解决这一问题，许多研究人员试图通过使用轮询机制来减少内核开销，消除不必要的上下文切换。

However, kernel-level I/O optimizations have several limitations to satisfy the requirements of user applications. First, the kernel should be general because it provides an abstraction layer for applications, managing all the hardware resources. Thus, it is hard to optimize the kernel without loss of generality. Second, the kernel cannot implement any policy that favors a certain application because it should provide fairness among applications. Lastly, the frequent update of the kernel requires a constant effort to port such application-specific optimizations.
然而，如果要满足用户应用需求的话，内核级的I/O优化受到了一些限制。首先，内核就应该是通用的，因为它为应用提供了一个抽象层，负责管理所有硬件资源。因此，对内核进行优化而不损失通用性是很困难的。其次，内核不能实现任何的只支持某个应用的策略，因为内核应该给多个应用程序提供公平性。最后，内核的频繁更新需要持续不断的投入来支持这种面向某个应用所做的优化。

In this sense, it would be desirable if a user-space I/O framework is supported for high-performance storage devices which enables the optimization of the I/O stack in the user space without any kernel intervention. In particular, such a user-space I/O framework can have a great impact on modern data-intensive applications such as distributed data processing platforms, NoSQL systems, and database systems, where the I/O performance plays an important role in the overall performance. Recently, Intel released a set of tools and libraries for accessing NVMe SSDs in the user space, called SPDK [7]. However, SPDK only works for a single user and application because it moves the whole NVMe driver from the kernel to the user space.
从这个意义上讲，如果在没有任何内核干预的情况下，有一个用户空间I/O框架支持高性能存储设备，能够优化用户空间I/O栈，那么这个框架就是极好的。特别地，这样一个用户空间I/O框架能够对现代数据密集型应用(例如：分布式数据处理平台，NoSQL系统和数据库系统)产生重大的影响。在这些应用中，I/O性能在整体性能中扮演者着重要的角色。近年来，英特尔发布了称之为SPDK的一套工具和库，用来在用户空间访问NVMe固态硬盘。然而，SPDK只适用于单个用户和单一应用，因为它把整个NVMe驱动从内核空间移动到了用户空间。

## **2 Background | 背景**

NVM Express (NVMe) is a high performance and scalable host controller interface for PCIe-based SSDs [1]. The notable feature of NVMe is to offer multiple queues to process I/O commands. Each I/O queue can manage up to 64K commands and a single NVMe device supports up to 64K I/O queues. When issuing an I/O command, the host system places the command into the submission queue and notify the NVMe device using the doorbell register. After the NVMe device processes the I/O command, it writes the results to the completion queue and raises an interrupt to the host system. NVMe enhances the performance of interrupt processing by MSI/MSI-X and interrupt aggregation. In the current Linux kernel, the NVMe driver creates a submission queue and a completion queue per core in the host system to avoid locking and cache collision.
NVMe是一个针对基于PCIe的固态硬盘的高性能的、可扩展的主机控制器接口。NVMe的显著特征是提供多个队列来处理I/O命令。单个NVMe设备支持多达64K个I/O队列,每个I/O队列可以管理多达64K个命令。当发出一个I/O命令的时候，主机系统将命令放置到提交队列(SQ)，然后使用门铃寄存器(DB)通知NVMe设备。当NVMe设备处理完I/O命令之后，设备将处理结果写入到完成队列(CQ)，并引发一个中断通知主机系统。NVMe使用MSI/MSI-X和中断聚合来提高中断处理的性能。在当前的Linux内核中，NVMe驱动程序在每一个CPU核(core)上创建一个提交队列(SQ)和完成队列(CQ)，以避免锁和缓存冲突。

## **3 NVMeDirect I/O Framework | NVMeDirect I/O框架**

### **3.1 Design | 设计**

We develop a user-space I/O framework called NVMeDirect to fully utilize the performance of NVMe SSDs while meeting the diverse requirements from user applications. Figure 1 illustrates the overall architecture of our NVMeDirect framework.
我们开发了一个叫做NVMeDirect的用户空间的I/O框架，该框架充分利用了NVMe固态硬盘的性能，同时能够满足用户应用多样化的需求。图1描述了NVMeDirect框架的总体框架。

The Admin tool controls the kernel driver with the root privilege to manage the access permission of I/O queues. When an application requests a single I/O queue to NVMeDirect, the user-level library calls the kernel driver. The kernel first checks whether the application is allowed to perform user-level I/Os. And then it creates the required submission queue and the completion queue, and maps their memory regions and the associated doorbell registers to the user-space memory region of the application. After this initialization process, the application can issue I/O commands directly to the NVMe SSD without any hardware modification or help from the kernel I/O stack. NVMeDirect offers the notion of I/O handles to send I/O requests to NVMe queues. A thread can create one or more I/O handles to access the queues and each handle can be bound to a dedicated queue or a shared queue. According to the characteristics of the workloads, each handle can be configured to use different features such as caching, I/O scheduling, and I/O completion. The major APIs supported by the NVMeDirect framework are summarized in Table 1. NVMeDirect also provides various wrapper APIs that correspond to NVMe commands such as read, write, flush, and discard.
管理工具使用root权限来控制内核驱动程序从而管理I/O队列的访问权限。当应用向NVMeDirect请求一个I/O队列的时候，用户级别的库函数调用内核驱动。内核首先检查该应用是否被允许执行用户级别的I/O，然后创建所必需的提交队列(SQ)和完成队列(CQ)，并将它们的内存区域和相关的门铃寄存器映射到应用程序的用户空间内存区域。初始化过程完成后，应用程序可以直接给NVMe固态硬盘发I/O命令,无需修改任何硬件或求助于内核I/O栈。NVMeDirect提供了I/O句柄的概念，用来给NVMe队列发送I/O请求。线程可以创建一个或多个I/O句柄来访问队列，每个句柄都可以绑定到一个专用的队列或共享的队列上。根据工作负载特性，可以为每个句柄都配置使用不同的特性，例如缓存、I/O调度和I/O完成。表1总结了NVMeDirect框架支持的主要的API。NVMeDirect还对NVMe命令提供各种包装API，例如read, write, flush, 和discard。

![img](https://images2017.cnblogs.com/blog/1264595/201710/1264595-20171025150010285-1416173100.png)

Separating handles from queues enables flexible grouped I/O policies among multiple threads and makes it easy to implement differentiated I/O services. Basically, a single I/O queue is bound to a single handle as Thread A in Figure 1. If a thread needs to separate read requests from write requests to avoid read starvation due to bulk writes, it can bind multiple queues to a single handle as Thread B in Figure 1. It is also possible to share a single queue among multiple threads as Thread C and D in Figure 1.
从队列中分离出句柄可以在多个线程之间实现灵活的分组I/O策略，并易于实现差异化的I/O服务。总的来说，单个的I/O队列绑定到单个句柄上，例如图1中的线程A。如果一个线程需要将读请求与写请求分开，以避免由于批量写入而导致的读饥饿问题，那么可以将多个队列绑定到一个单独的句柄上，例如图1中的线程B。还可以在多个线程之间共享单个队列，例如图1中的线程C和D。

NVMeDirect also offers block cache, I/O scheduler, and I/O completion thread components for supporting diverse I/O policies. Applications can mix and match these components depending on their needs. Block cache manipulates the memory for I/O in 4KB unit size similar to the page cache in the kernel. Since the NVMe interface uses the physical memory addresses for I/O commands, the memory in the block cache utilizes pretranslated physical addresses to avoid address translation overhead. I/O scheduler issues I/O commands for asynchronous I/O operations or when an I/O request is dispatched from the block cache. Since the interrupt-based I/O completion incurs context switching and additional software overhead, it is not suitable for high performance and low latency I/O devices [13]. However, if an application is sensitive to bandwidth rather than to latency, polling is not efficient due to the significant increase in the CPU load. Thus, NVMeDirect utilizes a dedicated polling thread with dynamic polling period control based on the I/O size or a hint from applications to avoid unnecessary CPU usage.
NVMeDirect还提供了块缓存(block cache)，I/O调度器和I/O完成线程组件以支持多种I/O策略。应用程序可以根据自己的需要混合和适配这些组件。类似于内核页面缓存， 块缓存以4KB单元大小操纵内存I/O。由于NVMe接口使用物理内存地址发送I/O命令，位于块缓存的内存采用预翻译物理地址以避免地址转换开销。I/O调度器发出I/O命令，要么为异步I/O操作，要么从块缓存中发出I/O请求。由于基于中断的I/O完成会导致上下文切换和额外的软件开销，因此不适合高性能和低延迟的I/O设备。但是，如果应用对带宽敏感而不那么在乎时延的话，那么轮询效率就不高了，因为显著地增加了CPU负载。因此，NVMeDirect利用专用的轮询线程与应用的足迹避免了不必要的CPU占用，注意这个专用的轮询线程具有基于I/O的大小的动态轮询周期控制机制。

![img](https://images2017.cnblogs.com/blog/1264595/201710/1264595-20171025150110394-13994373.png)

### **3.2 Implementation | 实现**

The NVMeDirect framework is composed of three components: queue management, admin tool, and user-level library. The queue management and the admin tool are mostly implemented in the NVMe kernel driver, and user-level library is developed as a shared library.
NVMeDirect框架由三个组件构成：队列管理，管理工具和用户级别的函数库。其中，队列管理和管理工具大部分实现都在NVMe内核驱动中，而用户级别的函数库是一个共享函数库。

We implement the queue management module in the NVMe driver of the Linux kernel 4.3.3. At the initialization stage, the admin tool notifies the access privileges and the number of available queues to the queue management module with ioctl(). When an application requests to create a queue, the queue management module checks the permission and creates a pair of submission and completion queues. The module maps the kernel memory addresses of the created queues and the doorbell to the user's address space using dma_common_mmap() to make them visible for the user-level library. The module also exports the memory addresses via the proc file system. Then, the user-level library can issue I/O commands by accessing the memory addresses of queues and doorbell registers.
我们在Linux 内核4.3.3上实现了NVMe驱动的队列管理模块。在初始化阶段，管理工具使用ioctl()给队列管理模块发通知，告知访问权限和可用的队列数。当应用请求创建一个队列的时候，队列管理模块检查权限，然后创建一个包含提交队列(SQ)和完成队列(CQ)的队列对(QP)。队列管理模块使用dma_common_mmap()将创建的队列的内核空间的内存地址和门铃寄存器映射到用户态地址空间，这样用户级别的函数库就可以看见已经创建的队列和门铃寄存器。队列管理模块也使用proc文件系统，将内存地址导出。那么，用户级别的函数库就可以通过访问队列地址和门铃寄存器发送I/O命令。

The I/O completion thread is implemented as a standalone thread to check the completion of I/O using polling. Multiple completion queues can share a single I/O completion thread or a single completion queue can use a dedicated thread to check the I/O completion. The polling period can be adjusted dynamically depending on the I/O characteristics of applications. Also, an application can explicitly set the polling period of the specific queue using nvmed_set_param(). The I/O completion thread uses usleep() to adjust the polling period.
I/O完成线程是作为一个独立的线程来实现的，它使用轮询去检查I/O的完成。多个完成队列(CQ)可以共享一个I/O完成线程，或单个完成队列可以使用专有的完成线程去检查I/O完成情况。轮询周期可以根据应用程序的I/O特性动态地做出调整。此外，应用程序可以调用nvmed_set_param()显式地设置在特定队列上的轮询周期。I/O完成现成使用usleep()来调整轮询周期。

## **4 Evaluation | 评估**

We compare the I/O performance of NVMeDirect with the original kernel-based I/O with asynchronous I/O support (Kernel I/O) and SPDK using the Flexible IO Tester (fio) benchmark [3]. For all the experiments, we use a Linux machine equipped with a 3.3GHz Intel Core i7 CPU and 64GB of memory running Ubuntu 14.04. All the performance evaluations are performed on a commercial Intel 750 Series 400GB NVMe SSD.
通过使用fio benchmark测试，我们比较了NVMeDirect, Kernel I/O和SPDK的I/O性能。在所有的试验中，我们使用的都是运行Ubuntu 14.04的Linux机器(CPU: 3.3GHz Intel Core i7, 内存：64GB)。所有的性能评估都是在商用的Intel 750系列的NVMe固态硬盘(400GB)上完成的。

### **4.1 Baseline Performance | 性能基线**

Figure 2 depicts the IOPS of random reads (Figure 2a) and random writes (Figure 2b) on NVMeDirect, SPDK, and Kernel I/O varying the queue depth with a single thread. When the queue depth is sufficiently large, the performance of random reads and writes meets or exceeds the performance specification of the device on both NVMeDirect, SPDK, and Kernel I/O. However, NVMeDirect achieves higher IOPS compared to Kernel I/O until the IOPS is saturated. This is because NVMeDirect avoids the overhead of the kernel I/O stack by supporting direct accesses between the user application and the NVMe SSD. As shown in Figure 2a, we can see that our framework records the near maximum performance of the device with the queue depth of 64 for random reads, while Kernel I/O has 12% less IOPS in the same configuration. In Figure 2b, when NVMeDirect achieves the maximum device performance, Kernel I/O shows 20% less IOPS. SPDK shows the same trend as NVMeDirect because it also accesses the NVMe SSD directly in the user space.
图2描述了在NVMeDirect, SPDK, 和Kernel I/O上随机读(图2a)，随机写(图2b)的IOPS, 使用单个线程和多个队列深度。当队列深度足够大，随机读写的性能达到或超过设备指定的性能指标，无论是NVMeDirect，还是SPDK以及Kernel I/O。然而，NVMeDirect比Kernel I/O获得了更高的IOPS，直到IOPS达到饱和状态。这是因为NVMeDirect支持用户应用直接访问NVMe固态硬盘，避免了Kernel I/O栈的开销。如图2a所示，我们可以看到，当队列深度为64的时候，NVMeDirect框架在随机读的时候达到了近乎最大的性能。而在同样的配置上，Kernel I/O的IOPS少了12%。在图2b中，NVMeDirect在随机写的时候达到了设备的最大的性能，而Kernel I/O的IOPS少了20%。SPDK显示的趋势跟NVMeDirect是一样的，因为它也在用户空间直接访问NVMe固态硬盘。

![img](https://images2017.cnblogs.com/blog/1264595/201710/1264595-20171025151655566-2097419923.png)

### **4.2 Impact of the Polling Period | 轮询周期产生的影响**

Figure 3 shows the trends of IOPS (denoted by lines) and CPU utilization (denoted by bars) when we vary the polling period per I/O size. The result is normalized to the IOPS achieved when the polling is performed without delay for each I/O size. We can notice that a significant performance degradation occurs in a certain point for each I/O size. For instance, if the I/O is 4KB in size, it is better to shorten the polling period as much as possible because the I/O is completed very quickly. In case of 8KB and 16KB I/O sizes, there is no significant slow-down, even though the polling is performed once in 70μs and 200μs, respectively. At the same time, we can reduce the CPU utilization due to the polling thread to 4% for 8KB I/Os and 1% for 16KB I/Os. As mentioned in Section 3.1, we use the adaptive polling period control based on this result to reduce the CPU utilization associated with polling.
图3显示了IOPS趋势(线状图)和CPU的利用率(柱状图)，当我们针对每一个I/O大小使用不同的轮询周期的时候。当对每一个I/O大小进行没有任何延迟的轮询的时候，获得的IOPS会归一化到某一点。我们不难注意到，对每一个I/O大小来说，在某几个点性能严重下降。例如：当I/O大小为4KB的时候，最好尽可能地缩短轮询周期，因为I/O能够非常快速地完成。当I/O大小为8KB和16KB的时候，性能并没有明显的下降，即使轮询周期在70μs和200μs之间摆动。与此同时，我们可以降低CPU利用率，对8KB的I/O来说降低4%, 对16KB的I/O来说降低1%。正如第3.1节所提到的，我们使用基于此结果的自适应轮询周期控制机制来降低与轮询相关的CPU利用率。

![img](https://images2017.cnblogs.com/blog/1264595/201710/1264595-20171025152122223-538450327.png)

### **4.3 Latency Sensitive Application | 时延敏感型应用**

The low latency is particularly effective on the latency sensitive application such as key-value stores. We evaluate NVMeDirect on one of the latency sensitive applications, Redis, which is an in-memory key value store mainly used as database, cache, and message broker [2]. Although Redis is an in-memory database, Redis writes logs for all write commands to provide persistence. This makes the write latency be critical to the performance of Redis. To run Redis on NVMeDirect, we added 6 LOC (lines of code) for initialization and modified 12 LOC to replace POSIX I/O interface with the corresponding NVMeDirect interface with the block cache. We use the total 10 clients with workload-A in YCSB [6], which is an update heavy workload.
对诸如键值对存储之类的对时延敏感的应用来说，低延迟尤其重要。我们使用了一种叫做Redis的对时延敏感的应用来对NVMeDirect进行评估。Redis是一种基于内存的键值对存储，主要用于数据库，缓存和消息代理。虽然Redis是一种内存数据库，但是Redis将所有写命令的日志写入到磁盘以提供持久性。这使得写时延对Redis的性能影响至为关键。为了在NVMeDirect上运行Redis, 我们给Redis的初始化部分增加了6行代码，修改了12行代码，用与块缓存相关的NVMeDirect接口替代了POSIX I/O接口。我们使用了YSCB文中的workload-A, 包含了10个客户端，这样的工作负载很大。

Table 2 shows the throughput and latency of Redis on Kernel I/O and NVMeDirect. NVMeDirect improves the overall throughput by about 15% and decreases the average latency by 13% on read and by 20% on update operations. This is because NVMeDirect reduces the latency by eliminating the software overhead of the kernel.
表2显示了在Kernel I/O和NVMeDirect上Redis的吞吐量和时延。NVMeDirect提高了总的吞吐量约15%，降低了读平均时延13%和更新操作时延20%。这是因为NVMeDirect通过消除内核软件开销从而降低了时延。

![img](https://images2017.cnblogs.com/blog/1264595/201710/1264595-20171025152310269-1915887936.png)

### **4.4 Differentiated I/O Service | 差异化的I/O服务**

I/O classification and boosting the prioritized I/Os is important to improve the application performance such as writing logs in database systems [9, 10]. NVMeDirect can provide the differentiated I/O service easily because the framework can apply different I/O policies to the application using I/O handles and multiple queues.
I/O分类和增强I/O优先级对于提高应用程序性能非常重要，例如在数据库系统中写入日志。NVMeDirect可以很容易地提供差异化的I/O服务，因为此框架可以通过使用I/O句柄和多个队列来将不同的I/O策略应用于不同的应用程序。

We perform an experiment to demonstrate the possible I/O boosting scheme in NVMeDirect. To boost specific I/Os, we assign a prioritized thread to a dedicated queue while the other threads share a single queue. For the case of non-boosting mode, each thread has its own queue in the framework. Figure 4 illustrates the IOPS of Kernel I/O and two I/O boosting modes of NVMeDirect while running the fio benchmark. The benchmark runs four threads including one prioritized thread and each thread performs random writes with a queue depth of 4. As shown in the result, the prioritized thread with a dedicated queue on NVMeDirect outperforms the other threads remarkably. In the case of Kernel I/O, all threads have the similar performance because there is no mechanism to support prioritized I/Os. This result shows that NVMeDirect can provide the differentiated I/O service without any software overhead.
我们做了一个实验来论证NVMeDirect这一I/O性能提升方案。为了增强特定的I/O，我们给优先级高的线程分配一个专用队列，而其他线程则共享一个队列。对于非增强模式，每个线程在框架中都有自己的队列。图4说明了Kernel I/O和NVMeDirect双I/O在fio基准测试的结果。基准测试运行四个线程，其中包括一个优先级线程，每个线程执行随机写，队列深度为4。结果表明，使用专用队列的优先级高的线程的性能明显优于其他线程。在Kernel I/O的情况下，所有线程都具有相似的性能，因为没有支持优先级I/O的机制。这一结果表明，NVMeDirect可以提供差异化的I/O服务，而不需要额外的软件开销。

![img](https://images2017.cnblogs.com/blog/1264595/201710/1264595-20171025152427723-799235239.png)

## **5 Evaluation | 评估**

There have been several studies for optimizing the storage stack as the hardware latency is decreased to a few milliseconds. Shin et al. [11] present a low latency I/O completion scheme based on the optimized low level hardware abstraction layer. Especially, optimizing I/O path minimizes the scheduling delay caused by the context switch. Yu et al. [14] propose several optimization schemes to fully exploit the performance of fast storage devices. The optimization includes polling I/O completion, eliminating context switches, merging I/O, and double buffering. Yang et al. [13] also present that the polling method for the I/O completion delivers higher performance than the traditional interrupt-driven I/O.
由于硬件时延减少到了几个毫秒，于是业界针对存储栈优化进行了研究。 Shin et al. [11]提出了一种基于优化的底层硬件抽象层的低时延I/O完成方案。特别地，优化I/O路径最小化了因为上下文切换引起的调度时延。Yu et al. [14]提出了几种优化方案，以充分利用快速存储设备的性能。优化包括对I/O完成进行轮询、消除上下文切换、合并I/O和双缓冲。Yang et al. [13]还指出I/O完成轮询方法比传统的中断驱动I/O性能更高。

Since the kernel still has overhead in spite of several optimizations, researchers have tried to utilize direct access to the storage devices without involving the kernel. Caulfield et al. [5] present user-space software stacks to further reduce storage access latencies based on their special storage device, Moneta [4]. Likewise, Volos et al. [12] propose a flexible file-system architecture that exposes the storage-class memory to user applications to access storage without kernel interaction. These approaches are similar to the proposed NVMeDirect I/O framework. However, their studies require special hardware while our framework can run on any commercial NVMe SSDs. In addition, they still have complex modules to provide general file system layer which is not necessary for all applications.  
尽管有一些优化，但是内核开销依然存在，于是研究人员试图直接访问存储设备而不经过内核。Caulfield et al. [5]提出用户空间软件栈以进一步降低存储访问的时延，基于他们的特殊存储设备(Moneta)。类似地，Volos et al. [12]提出了一个灵活的文件系统架构，它将存储类内存暴露给用户应用，在没有内核交互的情况下直接访问存储。这些方法与我们提出的NVMeDirect I/O框架类似。然而，他们的研究需要特定的硬件，而我们的框架可以运行在任何商用的NVMe固态硬盘上。此外，他们仍然有复杂的模块来提供通用文件系统层，而文件系统层对于所有应用程序来说都不是必需的。

NVMeDirect is a research outcome independent of SPDK [7] released by Intel. Although NVMeDirect is conceptually similar to SPDK, NVMeDirect has following differences. First, NVMeDirect leverages the kernel NVMe driver for control-plane operations, thus existing applications and NVMeDirect-enabled applications can share the same NVMe SSD. In SPDK, however, the whole NVMe SSD is dedicated to a single process who has all the user-level driver code. Second, NVMeDirect is not intended to be just a set of mechanisms to allow user-level direct accesses to NVMe SSDs. Instead, NVMeDirect also aims to provide a large set of I/O policies to optimize various data-intensive applications according to their characteristics and requirements.
NVMeDirect是一个独立于英特尔公布的SPDK的研究结果。虽然NVMeDirect在概念上与SPDK相似，但NVMeDirect有以下不同。首先，NVMeDirect利用内核NVMe驱动做控制平面的操作，因此，现有的应用和启用了NVMeDirect的应用可以共享同一块NVMe固态硬盘。然而在SPDK中，整块NVMe固态硬盘是专属于某一个过程，该进程拥有所有的用户级别的驱动代码。其次，NVMeDirect不只是一套允许用户直接访问NVMe固态硬盘的机制。它还提供大量的I/O策略，用来优化各种数据密集型应用，根据应用自身的特点和需求。

## **6 Conclusion | 结论**

We propose a user-space I/O framework, NVMeDirect, to enable the application-specific optimization on NVMe SSDs. Using NVMeDirect, user-space applications can access NVMe SSDs directly without any overhead of the kernel I/O stack. In addition, the framework provides several I/O policies which can be used selectively by the demand of applications. The evaluation results show that NVMeDirect is a promising approach to improve application performance using several user-level I/O optimization schemes.
我们提出了一个用户空间的I/O框架，NVMeDirect，用来对面向NVMe固态硬盘存储的应用做优化。使用NVMeDirect, 用户态的应用程序能够直接访问NVMe固态硬盘，而不需要任何内核I/O栈开销。此外，该框架还提供了可根据应用需求有选择地使用的一些I/O策略。评估结果表明，通过使用多用户级别的I/O优化方案，NVMeDirect不失为一种有效地提高应用程序性能的方法。

Since NVMeDirect does not interact with the kernel during the I/Os, it cannot provide enough protection normally enforced by the file system layer. In spite of this, we believe NVMeDirect is still useful for many data-intensive applications which are deployed in a trusted environment. As future work, we plan to investigate ways to protect the system against illegal memory and storage accesses. We are also planning to provide user-level file systems to support more diverse application scenarios. NVMeDirect is available as opensource at https://github.com/nvmedirect.
因为NVMeDirect在I/O过程中不与内核交互，所以它不能提供通常由文件系统层提供的足够的访问保护。虽然如此，我们依然相信NVMeDirect对很多数据密集型应用是有用的，这些应用部署在可信任的环境中。未来我们计划研究保护系统免受非法内存/存储访问的方法。我们还计划提供用户级别的文件系统，以支持更加多样化的应用场景。关于NVMeDirect的源代码，请访问https://github.com/nvmedirect。

**Acknowledgements | 鸣谢**

We would like to thank the anonymous reviewers and our shepherd, Nisha Talagala, for their valuable comments. This work was supported by Samsung Research Funding Center of Samsung Electronics under Project Number SRFC-TB1503-03.
请允许我们对匿名审稿人和指导老师Nisha Talagala表示感谢，感谢他们提出的宝贵意见。三星电子的三星研究基金中心为这项工作提供了支持和帮助，项目编号是SRFC-TB1503-03。

**References | 参考文献**

[1] NVM Express Overview. http://www.nvmexpress.org/about/nvm-express-overview/.
[2] Redis. http://redis.io/.
[3] AXBOE, J. Flexible IO tester. http://git.kernel.dk/?p=fio.git;a=summary.
[4] CAULFIELD, A. M., DE, A., COBURN , J., MOLLOW , T. I., GUPTA , R. K., AND SWANSON, S.Moneta: A high-performance storage array architecture for next-generation, non-volatile memories. In Proc. MICRO (2010), pp. 385-395.
[5] CAULFIELD, A. M., MOLLOV, T. I., EISNER , L. A., DE, A., COBURN, J., AND SWANSON , S. Providing safe, user space access to fast, solid state disks. In Proc. ASPLOS (2012), pp. 387-400.
[6] COOPER , B. F., SILBERSTEIN, A., TAM , E., RAMAKRISHNAN ,R., AND SEARS, R. Benchmarking cloud serving systems with YCSB. In Proc. SOCC (2010), pp. 143-154.
[7] INTEL. Storage performance development kit. https://01.org/spdk.
[8] INTEL, AND MICRON. Intel and Micron Produce Break-through Memory Technology. http://newsroom.intel.com/community/intel_newsroom/blog/2015/07/28/intel-and-micron-produce-breakthrough-memory-technology, 2015.
[9] KIM, S., KIM, H., KIM , S.-H., LEE, J., AND JEONG, J. Request-oriented durable write caching for application performance. In Proc. USENIX ATC(2015), pp. 193-206.
[10] LEE, S.-W., MOON, B., PARK, C., KIM, J.-M., AND KIM, S.-W. A case for flash memory SSD in enterprise database applications. In Proc. SIGMOD (2008), pp. 1075-1086. [11] SHIN, W., CHEN, Q., OH, M., EOM, H., AND YEOM, H. Y. OS I/O path optimizations for flash solid-state drives. In Proc. USENIX ATC (2014), pp. 483-488.
[12] VOLOS, H., NALLI, S., PANNEERSELVAM, S., VARADARAJAN, V., SAXENA, P., AND SWIFT, M. M. Aerie: Flexible file-system interfaces to storage-class memory. In Proc. EuroSys (2014).
[13] YANG, J., MINTURN, D. B., AND HADY, F. When poll is better than interrupt. In Proc. FAST (2012), p. 3.
[14] YU, Y. J., SHIN, D.I., SHIN, W., SONG, N. Y., CHOI, J. W., KIM, H. S., EOM, H., AND YEOM, H. Y. Optimizing the block I/O subsystem for fast storage devices. ACM Transactions on Computer Systems (TOCS) 32, 2 (2014), 6.





**转载自：  作者** [vlhn](https://home.cnblogs.com/u/vlhn/)   **本文链接**https://www.cnblogs.com/vlhn/p/7727591.html
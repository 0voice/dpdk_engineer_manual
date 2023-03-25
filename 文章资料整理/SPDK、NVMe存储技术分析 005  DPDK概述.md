# [SPDK/NVMe存储技术分析\]005  DPDK概述

注： 之所以要中英文对照翻译下面的文章，是因为SPDK严重依赖于DPDK的实现。

[Introduction to DPDK: Architecture and Principles](https://blog.selectel.com/introduction-dpdk-architecture-principles/)

## 1. **DPDK概论：体系结构与实现原理**

![img](https://images2017.cnblogs.com/blog/1264595/201710/1264595-20171030160624355-289495642.png)

Linux network stack performance has become increasingly relevant over the past few years. This is perfectly understandable: the amount of data that can be transferred over a network and the corresponding workload has been growing not by the day, but by the hour.
这几年以来，Linux网络栈的性能变得越来越重要。这很好理解，因为伴随着时间的向前推移，可以通过网络来传输的数据量和对应的工作负载在大幅度地向上增长。

Not even the widespread use of 10 GE network cards has resolved this issue; this is because a lot of bottlenecks that prevent packets from being quickly processed are found in the Linux kernel itself.
即便广泛使用10GbE网卡也解决不了这一性能问题（用重庆话说就是，10GbE的网卡mang起儿子家整，然并卵），因为在Linux内核中，阻止数据包被快速地处理掉的瓶颈实在是太多啦。

There have been many attempts to circumvent these bottlenecks with techniques called kernel bypasses (a short description can be found [here](https://blog.cloudflare.com/kernel-bypass/)). They let you process packets without involving the Linux network stack and make it so that the application running in the user space communicates directly with networking device. We’d like to discuss one of these solutions, the Intel [DPDK](http://dpdk.org/) (Data Plane Development Kit), in today’s article.
尝试绕过这些瓶颈的技术有很多，统称为kernel bypass(简短的描述戳这里)。kernel bypass技术让编程人员处理数据包，而不卷入Linux网络栈，在用户空间中运行的应用程序能够直接与网络设备打交道。在本文中，我们将讨论众多的kernel bypass解决方案中的一种，那就是Intel的DPDK(数据平面开发套件)。

A lot of posts have already been published about the DPDK and in a variety of languages. Although many of these are fairly informative, they don’t answer the most important questions: How does the DPDK process packets and what route does the packet take from the network device to the user?
来自多国语言的与有关DPDK的文章很多。虽然信息量已经很丰富了，但是并没有回答两个重要的问题。问题一：DPDK是如何处理数据包的？问题二：从网络设备到用户程序，数据包使用的路由是什么？

Finding the answers to these questions was not easy; since we couldn’t find everything we needed in the official documentation, we had to look through a myriad of additional materials and thoroughly review their sources. But first thing’s first: before talking about the DPDK and the issues it can help resolve, we should review how packets are processed in Linux.
找到上面的两个问题的答案不是一件容易的事情。由于我们无法在官方文档中找到我们所需要的所有东西，我们不得不查阅大量的额外资料，并彻底审查资料来源。但是最为首要的是：在谈论DPDK可以帮助我们解决问题之前，我们应该审阅一下数据包在Linux中是如何被处理的。

## 2. **Processing Packets in Linux: Main Stages | Linux中的数据包处理的主要几个阶段**

When a network card first receives a packet, it sends it to a receive queue, or RX. From there, it gets copied to the main memory via the DMA (Direct Memory Access) mechanism.
当网卡接收数据包之后，首先将其发送到接收队列(RX)。在那里，数据包被复制到内存中，通过直接内存访问(DMA)机制。

Afterwards, the system needs to be notified of the new packet and pass the data onto a specially allocated buffer (Linux allocates these buffers for every packet). To do this, Linux uses an interrupt mechanism: an interrupt is generated several times when a new packet enters the system. The packet then needs to be transferred to the user space.
接下来，系统需要被通知到，有新的数据包来了，然后系统将数据传递到一个专门分配的缓冲区中去（Linux为每一个数据包都分配这样的特别缓冲区）。为了做到这一点，Linux使用了中断机制：当一个新的数据包进入系统时，中断多次生成。然后，该数据包需要被转移到用户空间中去。

One bottleneck is already apparent: as more packets have to be processed, more resources are consumed, which negatively affects the overall system performance.
在这里，存在着一个很明显的瓶颈：伴随着更多的数据包需要被处理，更多的资源将被消耗掉，这无疑对整个系统的性能将产生负面的影响。

As we've already said, these packets are saved to specially allocated buffers - more specifically, the [sk_buff struct](http://lxr.free-electrons.com/source/include/linux/skbuff.h). This struct is allocated for each packet and becomes free when a packet enters the user space. This operation consumes a lot of bus cycles (i.e. cycles that transfer data from the CPU to the main memory).
我们在前面已经说过，这些数据包被保存在专门分配的缓冲区中-更具体地说就是sk_buff结构体。系统给每一个数据包都分配一个这样的结构体，一但数据包到达用户空间，该结构体就被系统给释放掉。这种操作消耗大量的总线周期（bus cycle即是把数据从CPU挪到内存的周期）。

There is another problem with the sk_buff struct: the Linux network stack was originally designed to be compatible with as many protocols as possible. As such, metadata for all of these protocols is included in the sk_buff struct, but that’s simply not necessary for processing specific packets. Because of this overly complicated struct, processing is slower than it could be.
与sk_buff struct密切相关的另一个问题是：设计Linux网络协议栈的初衷是尽可能地兼容更多的协议。因此，所有协议的元数据都包含在sk_buff struct中，但是，处理特定的数据包的时候这些（与特定数据包无关的协议元数据）根本不需要。因而处理速度就肯定比较慢，由于这个结构体过于复杂。

Another factor that negatively affects performance is context switching. When an application in the user space needs to send or receive a packet, it executes a system call. The context is switched to kernel mode and then back to user mode. This consumes a significant amount of system resources.
对性能产生负面影响的另外一个因素就是上下文切换。当用户空间的应用程序需要发送或接收一个数据包的时候，执行一个系统调用。这个上下文切换就是从用户态切换到内核态，（系统调用在内核的活干完后）再返回到用户态。这无疑消耗了大量的系统资源。

To solve some of these problems, all Linux kernels since version 2.6 have included [NAPI](https://wiki.linuxfoundation.org/networking/napi) (New API), which combines interrupts with requests. Let’s take a quick look at how this works.
为了解决上面提及的所有问题，Linux内核从2.6版本开始包括了NAPI，将请求和中断予以合并。接下来我们将快速地看一看这是如何工作的。

The network card first works in interrupt mode, but as soon as a packet enters the network interface, it registers itself in a poll queue and disables the interrupt. The system periodically checks the queue for new devices and gathers packets for further processing. As soon as the packets are processed, the card will be deleted from the queue and interrupts are again enabled.
首先网卡是在中断模式下工作。但是，一旦有数据包进入网络接口，网卡就会去轮询队列中注册，并将中断禁用掉。系统周期性地检查新设备队列，收集数据包以便做进一步地处理。一旦数据包被系统处理了，系统就将对应的网卡从轮询队列中删除掉，并再次启用该网卡的中断（即将网卡恢复到中断模式下去工作）。

This has been just a cursory description of how packets are processed. A more detailed look at this process can be found in [an article series](https://www.privateinternetaccess.com/blog/2016/01/linux-networking-stack-from-the-ground-up-part-1/) from Private Internet Access. However, even a quick glance is enough to see the problems slowing down packet processing. In the next section, we’ll describe how these problems are solved using DPDK.
这只是对数据包如何被处理的粗略的描述。有关数据包处理过程的详细描述请参见Private Internet Access的系列文章。然而，就是这么一个快速一瞥也足以让我们看到数据包处理被减缓的问题。在下一节中，我们将描述使用DPDK后，这些问题是如何被解决掉的。

## 3. **DPDK: How It Works | DPDK 是如何工作的**

**General Features | 一般特性**

Let's look at the following illustration:
让我们来看看下面的插图：

![img](https://images2017.cnblogs.com/blog/1264595/201710/1264595-20171030161858668-1918350931.png)

On the left you see the traditional way packets are processed, and on the right - with DPDK. As we can see, the kernel in the second example doesn’t step in at all: interactions with the network card are performed via special drivers and libraries.
如图所示，在左边的是传统的数据包处理方式，在右边的则是使用了DPDK之后的数据包处理方式。正如我们看到的一样，右边的例子中，内核根本不需要介入，与网卡的交互是通过特殊的驱动和库函数来进行的。

If you've already read about DPDK or have ever used it, then you know that the ports receiving incoming traffic on network cards need to be unbound from Linux (the kernel driver). This is done using the dpdk_nic_bind (or dpdk-devbind) command, or ./dpdk_nic_bind.py in earlier versions.
如果你已经读过DPDK或者已经使用过DPDK，那么你肯定知道网卡接收数据传入的网口需要从Linux内核驱动上去除绑定（松棒）。用dpdk_nic_bind(或dpdk-devbind)命令就可以完成松绑，早期的版本中使用dpdk_nic_bind.py。

How are ports then managed by DPDK? Every driver in Linux has bind and unbind files. That includes network card drivers:
DPDK是如何管理网口的?每一个Linux内核驱动都有bind和unbind文件。 当然包括网卡驱动：

```
ls /sys/bus/pci/drivers/ixgbe
bind  module  new_id  remove_id  uevent  unbind
```

To unbind a device from a driver, the device's bus number needs to be written to the unbind file. Similarly, to bind a device to another driver, the bus number needs to be written to its bind file. More detailed information about this can be found [here](https://lwn.net/Articles/143397/).
从内核驱动上给一个设备松绑，需要把设备的bus号写入unbind文件。类似地，将设备绑定到另外一个驱动上，同样需要将bus号写入bind文件。更多详细信息请参见这里。

The DPDK installation instructions [tell us](http://dpdk.org/doc/guides-16.04/linux_gsg/build_dpdk.html#loading-modules-to-enable-userspace-io-for-dpdk) that our ports need to be managed by the vfio_pci, igb_uio, or uio_pci_generic driver. (We won’t be geting into details here, but we suggested interested readers look at the following articles on kernel.org: [1](https://www.kernel.org/doc/Documentation/vfio.txt) and [2](https://www.kernel.org/doc/htmldocs/uio-howto/about.html).)
DPDK的安装指南告诉我们ports需要被vfio_pci, igb_uio或uio_pci_generic驱动管理。(细节这里就不谈了，但建议有兴趣的读者阅读kernel.org上的文章：1和2)

These drivers make it possible to interact with devices in the user space. Of course they include a kernel module, but that's just to initialize devices and assign the PCI interface.
有了这些驱动程序，就可以在用户空间与设备进行交互。当然，它们包含了一个内核模块，但是该内核模块只负责设备初始化和分配PCI接口。

All further communication between the application and network card is organized by the DPDK poll mode driver (PMD). DPDK has poll mode drivers for all supported network cards and virtual devices.
在应用程序与网卡之间的所有的进一步的通信，都是有DPDK的轮询模式驱动(PMD)负责组织的。DPDK对所有网卡和虚拟设备都支持轮询模式驱动(PMD)。

The DPDK also requires hugepages be configured. This is required for allocating large chunks of memory and writing data to them. We can say that hugepages does the same job in DPDK that DMA does in traditional packet processing.
大内存页(hug pages)的配置对DPDK来说是必须的。这是因为需要分配大块内存并向大块内存中写入数据。可以这么说，数据包处理的活，传统的方式是使用直接内存访问(DMA)来干，而DPDK使用大内存页(huge pages)来完成。

We'll discuss all of its nuances in more detail, but for now, let's go over the main stages of packet processing with the DPDK:
我们将讨论更多的细节。但是现在，让我们浏览一下使用DPDK做数据包处理的几个主要阶段：

1. Incoming packets go to a ring buffer (we'll look at its setup in the next section). The application periodically checks this buffer for new packets.
   传入的数据包被放到环形缓冲区(ring buffer)中去。应用程序周期性检查这个缓冲区(ring buffer)以获取新的数据包。
2. If the buffer contains new packet descriptors, the application will refer to the DPDK packet buffers in the specially allocated memory pool using the pointers in the packet descriptors.
   如果ring buffer包含有新的数据包描述符，应用程序就使用数据包描述符所包含的指针去做处理，该指针指向的是DPDK数据包缓冲区，该缓冲区位于专门的内存池中。
3. If the ring buffer does not contain any packets, the application will queue the network devices under the DPDK and then refer to the ring again.
   如果ring buffer中不包含任何数据包描述符，应用程序就会在DPDK中将网络设备排队，然后再次指向ring。

Let's take a closer look at the DPDK's internal structure.
接下来我们将近距离地深入到DPDK的内部结构中去。

**EAL: Environment Abstraction | 环境抽象层**

The [EAL](http://dpdk.org/doc/guides-16.04/prog_guide/env_abstraction_layer.html), or Environment Abstraction Layer, is the main concept behind the DPDK.
环境抽象层(EAL)，是位于DPDK背后的主要概念。

The EAL is a set of programming tools that let the DPDK work in a specific hardware environment and under a specific operating system. In the official DPDK repository, libraries and drivers that are part of the EAL are saved in the rte_eal directory.
EAL是一套编程工具，允许DPDK在特定的硬件环境和特定的操作系统下工作。在官方的DPDK仓库中，库和驱动是EAL的一部分，被保存在rte_eal目录。

Drivers and libraries for Linux and the BSD system are saved in this directory. It also contains a set of header files for various processor architectures: ARM, x86, TILE64, and PPC64.
为Linux和BSD系统写的库和驱动就保存在这个目录。同时还包含了一系列针对不同的处理器架构的头文件，不同的处理器包括ARM, x86, TILE64和PPC64。

We access software in the EAL when we compile the DPDK from the source code:
在从源代码编译DPDK的时候，就会访问到EAL中的软件：

```
make config T=x86_64-native-linuxapp-gcc
```

One can guess that this command will compile DPDK for Linux in an x86_64 architecture.
一猜上面的命令就是为Linux的x86_64架构编译DPDK。

The EAL is what binds the DPDK to applications. All of the applications that use the DPDK (see [here](http://dpdk.org/doc/guides-16.04/prog_guide/index.html) for examples) must include the EAL's header files.
将DPDK与应用程序捆绑到一起的就是EAL。所有使用DPDK的应用程序(例子戳这里)必须包含EAL的头文件。

The most commonly of these include:
其中最常见的包括：

- rte_lcore.h -- manages processor cores and sockets; 管理处理器核和socket；
- rte_memory.h -- manages memory; 管理内存；
- rte_pci.h -- provides the interface access to PCI address space; 提供访问PCI地址空间的接口；
- rte_debug.h -- provides trace and debug functions (logging, dump_stack, and more); 提供trace和debug函数(logging, dump_stack, 和更多)；
- rte_interrupts.h -- processes interrupts. 中断处理。

More details on this structure and EAL functions can be found in the [official documentation](http://dpdk.org/doc/guides/prog_guide/env_abstraction_layer.html).
有关EAL功能与结构的更多详细信息请参见官方文档。

## 4. **Managing Queues: rte_ring | 队列管理**

As we've already said, packets received by the network card are sent to a ring buffer, which acts as a receiving queue. Packets received in the DPDK are also sent to a queue implemented on the rte_ring library. The library's description below comes from information gathered from the developer's guide and comments in the source code.
我们在前面已经说过了，网卡接收到的数据包被发送到环形缓冲区(ring buffer)，该环形缓冲区充当接收队列的角色。DPDk接收到的数据包也被发送到用rte_ring函数库实现的队列中去。注意下面的函数库描述拉源于开发指南和源代码注释。

The rte_ring was developed from the [FreeBSD ring buffer](https://svnweb.freebsd.org/base/release/8.0.0/sys/sys/buf_ring.h?revision=199625&view=markup). If you look at the [source code](http://www.dpdk.org/browse/dpdk/plain/lib/librte_ring/rte_ring.c), you'll see the following comment: Derived from FreeBSD's bufring.c.
rte_ring是来自于FreeBSD的ring buffer。如果你阅读源代码，就会看见后面的注释: 来自于FreeBSD的bufring.c。

The queue is a lockless ring buffer built on the [FIFO](https://en.wikipedia.org/wiki/FIFO) (First In, First Out) principle. The ring buffer is a table of pointers for objects that can be saved to the memory. Pointers can be divided into four categories: prod_tail, prod_head, cons_tail, cons_head.
DPDK的队列是一个无锁的环形缓冲区，基于FIFO（先进先出原理）构建。ring buffer本质上是一张表，表里的每一个元素是可以保存在内存中的对象的指针。指针分为4类： prod_tail, prod_head, cons_tail, 和cons_head。

Prod is short for producer, and cons for consumer. The **producer** is the process that writes data to the buffer at a given time, and the consumer is the process that removes data from the buffer.
prod是producer(生产者)的缩写，而cons是consumer(消费者)的缩写。生产者（producer）是在给定的时间之内将数据写入缓冲区的进程，而消费者(consumer)是从缓冲区中读走数据的进程。

The **tail** is where writing takes place on the ring buffer. The place the buffer is read from at a given time is called the **head**.
tail（尾部）是写入环形缓冲区的地方，而在给定的时间内从环形缓冲区读取数据的地方称之为head（头部）。

The idea behind the process for adding and removing elements from the queue is as follows: when a new object is added to the queue, the ring->prod_tail indicator should end up pointing to the location where ring->prod_head previously pointed to.
在给队列添加一个元素和从队列中移除一个元素的过程中， 藏在其背后的思想是：当一个新的对象被添加到队列中，rihg->prod_tail应该最终指向ring->prod_head在之前指向的位置。

This is just a brief description; a more detailed account of how the ring buffer scripts work can be found in the [developer's manual](http://dpdk.org/doc/guides-16.04/prog_guide/ring_lib.html) on the DPDK site.
这里只是做一个简短的描述。有关ring buffer是如何编排其工作的详细说明请参见DPDK网站的开发者手册。

This approach has a number of advantages. Firstly, data is written to the buffer extremely quickly. Secondly, when adding or removing a large number of objects from the queue, cache misses occur much less frequently since pointers are saved in a table.
这一方法有很多优点。首先，将数据写入缓冲区非常快。其次，当给队列添加大量对象或者从队列中移除大量对象时，cache未命中发生的频率要低得多，因为保存在表中的是对象的指针。

The drawback to DPDK's ring buffer is its fixed size, which cannot be increased on the fly. Additionally, much more memory is spent working with the ring structure than in a linked queue since the buffer always uses the the maximum number of pointers.
DPDK的ring buffer的缺点是ring buffer的长度是固定的，不能够在运行时间动态地修改。另外，与链式队列相比，在ring结构中使用的内存比较多，因为ring buffer总是使用支持的对象指针数量的最大值。

## 5. **Memory Management: rte_mempool | 内存管理**

We mentioned above that DPDK requires hugepages. The installation instructions recommend creating 2MB hugepages.
在上面我们有提到DPDK需要使用大内存页。安装说明建议创建2MB的大内存页。

These pages are combined in segments, which are then divided into zones. Objects that are created by applications or other libraries, like queues and packet buffers, are placed in these zones.
这些大内存页合并为段，然后分割成zone。应用程序或者其他库(比如队列和数据包缓冲区)创建的对象被放置在这些zone中。

These objects include memory pools, which are created by the rte_mempool library. These are fixed size object pools that use rte_ring for storing free objects and can be identified by a unique name.
这些对象包括通过rte_mempool库创建的内存池。这些对象池的大小是固定的，使用rte_ring存储自由对象，能够用一个独一无二的名称来标识。

Memory alignment techniques can be implemented to improve performance.
内存对齐技术能够被用来提高性能。

Even though access to free objects is designed on a lockless ring buffer, consumption of system resources may still be very high. As multiple cores have access to the ring, a [compare-and-set](https://en.wikipedia.org/wiki/Compare-and-swap) (CAS) operation usually has to be performed each time it is accessed.
尽管访问自由对象被设计在一个无锁的ring buffer中，但是系统资源消耗可能还是很大。 由于这个ring被多个核访问，在每一次访问ring的时候，通常不得不执行CAS原子操作。

To prevent bottlenecking, every core is given an additional local cache in the memory pool. Using the locking mechanism, cores can fully access the free object cache. When the cache is full or entirely empty, the memory pool exchanges data with the ring buffer. This gives the core access to frequently used objects.
为了防止瓶颈发生，在内存池中给每一个CPU核配备额外的本地缓存。通过使用锁机制，多个CPU核能够完全访问自由对象缓存。当缓存满了或者完全空了，内存池与ring buffer进行数据交换。这使得CPU核能够访问那些被频繁使用的对象。

## 6. **Buffer Management: rte_mbuf | 缓冲区管理**

In the Linux network stack, all network packets are represented by the the sk_buff data structure. In DPDK, this is done using the rte_mbuf struct, which is described in the [rte_mbuf.h](http://dpdk.org/doc/api/rte__mbuf_8h_source.html) header file.
在Linux网络栈中，所有的网络数据包用sk_buff结构体表示。而在DPDK中，使用rte_mbuf结构体来表示网络数据包，该结构体的描述位于头文件rte_mbuf.h中。

The buffer management approach in DPDK is reminiscent of the approach used in FreeBSD: instead of one big sk_buff struct, there are many smaller rte_mbuf buffers. The buffers are created before the DPDK application is launched and are saved in memory pools (memory is allocated by rte_mempool).
DPDK的缓冲区管理方法让人不禁联想到FreeBSD的做法：使用很多较小的rte_buf缓冲区来代替一个大大的sk_buff结构体。在DPDK应用程序被启动之前，这些缓冲区就创建好了，它们被保存在内存池中(内存是通过rte_mempool来分配的)。

In addition to its own packet data, each buffer contains metadata (message type, length, data segment starting address). The buffer also contains pointers for the next buffer. This is needed when handling packets with large amounts of data. In cases like these, packets can be combined (as is done in FreeBSD; more detailed information about this can be found [here](https://www.bsdcan.org/2004/papers/NetworkBufferAllocation.pdf)).
除了包含它自己的包数据之外，每一个buffer还包含了元数据(消息类型，消息长度，数据段起始地址)。与此同时buffer也包含指向下一个buffer的指针，在处理大量的数据包的时候这是必须的。在这种情况下，数据包可以合并在一起（合并工作是由FreeBSD完成的，更多详细信息请参见这里）。

## 7. **Other Libraries: General Overview | 鸟瞰其他库**

In previous sections, we talked about the most basic DPDK libraries. There's a great deal of other libraries, but one article isn't enough to describe them all. Thus, we'll be limiting ourselves to just a brief overview.
在前面的章节中，我们谈到了最基本的DPDK库。其实DPDK库还有很多，用一篇文章将所有库都描述到是不可能的。因此，我们只是做一个简短的概述罢了。

With the [LPM library](http://dpdk.org/doc/guides/prog_guide/lpm_lib.html), DPDK runs the [Longest Prefix Match](https://en.wikipedia.org/wiki/Longest_prefix_match) (LPM) algorithm, which can be used to forward packets based on their IPv4 address. The primary function of this library is to add and delete IP addresses as well as to search for new addresses using the LPM algorithm.
在LPM库中，DPDK运行LPM(匹配最长前缀)算法，用于转发基于IPv4地址的数据包。这个库的主要功能就是添加和删除IP地址，同时使用LPM算法寻找新的地址。

A similar function can be performed for IPv6 addresses using the [LPM6 library](http://dpdk.org/doc/guides/prog_guide/lpm6_lib.html).
类似地，使用LPM6库处理IPv6地址。

Other libraries offer similar functionality based on hash functions. With [rte_hash](http://dpdk.org/doc/guides/prog_guide/hash_lib.html), you can search through a large record set using a unique key. This library can be used for classifying and distributing packets, for example.
其他库基于hash函数提供类似的功能。例如：使用rte_hash, 可以通过使用一个独一无二的key来搜索大记录集。这个库可用来分类和分发数据包。

The [rte_timer library](http://dpdk.org/doc/guides/prog_guide/timer_lib.html) lets you execute functions asynchronously. The timer can run once or periodically.
rte_timer库允许执行异步函数调用。定时器可以运行一次，也可以周期性地运行。

## 8. **Conclusion | 总结**

In this article we went over the internal device and principles of DPDK. This is far from comprehensive though; the subject is too complex and extensive to fit in one article. So sit tight, we will continue this topic in a future article, where we'll discuss the practical aspects of using DPDK.
本文我们讨论了DPDK的内部设备和原理。但这很不全面，因为该主题过于复杂和宽泛，以至于不能在一篇文章中讲述清楚。所以，我们将在未来的文章中继续讨论这一话题，讨论使用DPDK所遇到的实际问题。

We'd be happy to answer your questions in the comments below. And if you've had any experience using DPDK, we'd love to hear your thoughts and impressions.
我们非常乐意回答你在评论中提出的问题。如果你有任何使用DPDK的经验，请跟我们分享你的想法与感受。

For anyone interested in learning more, please visit the following links:
如有兴趣学习更多，请访问下面的链接：

- http://dpdk.org/doc/guides/prog_guide/ — a detailed (but confusing in some places) description of all the DPDK libraries;
- https://www.net.in.tum.de/fileadmin/TUM/NET/NET-2014-08-1/NET-2014-08-1_15.pdf — a brief overview of DPDK's capabilities and comparison with other frameworks (netmap and PF_RING);
- http://www.slideshare.net/garyachy/dpdk-44585840 — an introductory presentation to DPDK for beginners;
- http://www.it-sobytie.ru/system/attachments/files/000/001/102/original/LinuxPiter-DPDK-2015.pdf — a presentation explaining the DPDK structure.

[Andrej Yemelianov](https://blog.selectel.com/author/yemelianov/) 24 November 2016 Tags: DPDK, linux, network, network stacks, packet processing





**转载自：**  **作者** [vlhn](https://home.cnblogs.com/u/vlhn/)    **本文链接**https://www.cnblogs.com/vlhn/p/7754940.html
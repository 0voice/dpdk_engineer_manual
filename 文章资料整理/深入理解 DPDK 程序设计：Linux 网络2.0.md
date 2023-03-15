# 深入理解 DPDK 程序设计：Linux 网络2.0

## **一、网络IO的处境和趋势**

从我们用户的使用就可以感受到网速一直在提升，而网络技术的发展也从1GE/10GE/25GE/40GE/100GE的演变，从中可以得出单机的网络IO能力必须跟上时代的发展。

### **1. 传统的电信领域**

IP层及以下，例如路由器、交换机、防火墙、基站等设备都是采用硬件解决方案。基于专用网络处理器（NP），有基于FPGA，更有基于ASIC的。但是基于硬件的劣势非常明显，发生Bug不易修复，不易调试维护，并且网络技术一直在发展，例如2G/3G/4G/5G等移动技术的革新，这些属于业务的逻辑基于硬件实现太痛苦，不能快速迭代。传统领域面临的挑战是急需一套软件架构的高性能网络IO开发框架。

### **2. 云的发展**

私有云的出现通过网络功能虚拟化（NFV）共享硬件成为趋势，NFV的定义是通过标准的服务器、标准交换机实现各种传统的或新的网络功能。急需一套基于常用系统和标准服务器的高性能网络IO开发框架。

### **3. 单机性能的飙升**

网卡从1G到100G的发展，CPU从单核到多核到多CPU的发展，服务器的单机能力通过横行扩展达到新的高点。但是软件开发却无法跟上节奏，单机处理能力没能和硬件门当户对，如何开发出与时并进高吞吐量的服务，单机百万千万并发能力。即使有业务对QPS要求不高，主要是CPU密集型，但是现在大数据分析、人工智能等应用都需要在分布式服务器之间传输大量数据完成作业。这点应该是我们互联网后台开发最应关注，也最关联的。



## 二、Linux + x86网络IO瓶颈

根据经验，在C1（8核）上跑应用每1W包处理需要消耗1%软中断CPU，这意味着单机的上限是100万PPS（Packet Per Second）。从TGW（Netfilter版）的性能100万PPS，AliLVS优化了也只到150万PPS，并且他们使用的服务器的配置还是比较好的。假设，我们要跑满10GE网卡，每个包64字节，这就需要2000万PPS。

**注**：以太网万兆网卡速度上限是1488万PPS，因为最小帧大小为84B

《Bandwidth, Packets Per Second, and Other Network Performance Metrics》

100G是2亿PPS，即每个包的处理耗时不能超过50纳秒。而一次Cache Miss，不管是TLB、数据Cache、指令Cache发生Miss，回内存读取大约65纳秒，NUMA体系下跨Node通讯大约40纳秒。所以，即使不加上业务逻辑，即使纯收发包都如此艰难。我们要控制Cache的命中率，我们要了解计算机体系结构，不能发生跨Node通讯。

从这些数据，我希望可以直接感受一下这里的挑战有多大，理想和现实，我们需要从中平衡。**问题都有这些**：

- 传统的收发报文方式都必须采用硬中断来做通讯，每次硬中断大约消耗100微秒，这还不算因为终止上下文所带来的Cache Miss。
- 数据必须从内核态用户态之间切换拷贝带来大量CPU消耗，全局锁竞争。
- Linux协议栈处理路径长，多核扩展性不足，系统调用开销大。
- 内核工作在多核上，为可全局一致，即使采用Lock Free，也避免不了锁总线、内存屏障带来的性能损耗。
- 从网卡到业务进程，经过的路径太长，有些其实未必要的，例如netfilter框架，这些都带来一定的消耗，而且容易Cache Miss。



## 三、DPDK的基本原理

从前面的分析可以得知IO实现的方式、内核的瓶颈，以及数据流过内核存在不可控因素，这些都是在内核中实现，内核是导致瓶颈的原因所在，要解决问题需要绕过内核。所以主流解决方案都是旁路网卡IO，绕过内核直接在用户态收发包来解决内核的瓶颈。

Linux社区也提供了旁路机制Netmap，官方数据10G网卡1400万PPS，但是Netmap没广泛使用。其原因有几个：

1. Netmap需要驱动的支持，即需要网卡厂商认可这个方案。
2. Netmap仍然依赖中断通知机制，没完全解决瓶颈。
3. Netmap更像是几个系统调用，实现用户态直接收发包，功能太过原始，没形成依赖的网络开发框架，社区不完善。

那么，我们来看看发展了十几年的DPDK，从Intel主导开发，到华为、思科、AWS等大厂商的加入，核心玩家都在该圈子里，拥有完善的社区，生态形成闭环。早期，主要是传统电信领域3层以下的应用，如华为、中国电信、中国移动都是其早期使用者，交换机、路由器、网关是主要应用场景。但是，随着上层业务的需求以及DPDK的完善，在更高的应用也在逐步出现，尤其当前云计算领域（网络吞吐量巨大），已经成为云网络主要的核心技术之一。





![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nEHYY1vNIQOKf8iaKxv341VnrBWspc3BnjqK3WzFBI7Tq3sDiaUwKRlhQtmPrjNQJWn7ru3AY8OhgA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

DPDK旁路原理



左边是内核协议栈：

 **网卡 -> 驱动 -> 协议栈 -> Socket接口 -> 业务**

右边是DPDK的方式（基于UIO（Userspace I/O）旁路数据）：

**网卡 -> DPDK轮询模式-> DPDK基础库 -> 业务**

用户态的好处是易用开发和维护，灵活性好。并且Crash也不影响内核运行，鲁棒性强。

而DPDK不光是bypass 内核协议栈，还无所不及地采用各种手段，把凡是能够影响的网络IO性能的瓶颈点都做了极致的优化。

![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6lxib9nibKIEcFsmyfzMvlwGjZx9wericicssXRy2xs5EsXQj6qX5IvVAsQRjNPibFyyibkN3q7SJiaZMN0g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

DPDK架构

**核心模块**

环境抽象层

环境抽象层 (EAL) 提供了一个通用接口，该接口对应用程序和库隐藏了环境细    节。EAL 提供的服务是：

- DPDK 加载和启动
- 支持多进程和多线程执行类型
- 核心关联/分配程序
- 系统内存分配/解除分配
- 原子/锁操作
- 时间参考
- PCI总线访问
- 跟踪和调试功能
- CPU特性识别
- 中断处理
- 报警操作
- 内存管理（malloc）



### 环管理器 (librte_ring)

环形结构在有限大小的表中提供了一个无锁的多生产者、多消费者 FIFO API。它比无锁队列有一些优势；更容易实施，适应批量操作，速度更快。环由内存池管理器 (librte_mempool) 使用，并可用作核心和/或逻辑核心上连接在一起的执行块之间的通用通信机制。

### 内存池管理器 (librte_mempool)

内存池管理器负责分配内存中的对象池。池由名称标识并使用环来存储空闲对象，它提供了一些其他可选服务，例如每核对象缓存和对齐助手，以确保填充对象以在所有 RAM 通道上均匀分布它们。

### 网络数据包缓冲区管理 (librte_mbuf)

mbuf 库提供了创建和销毁缓冲区的功能，DPDK 应用程序可以使用这些缓冲区来存储消息缓冲区。消息缓冲区在启动时创建并存储在内存池中，使用 DPDK 内存池库。该库提供了一个 API 来分配/释放 mbuf，操作用于承载网络数据包的数据包缓冲区。

### 定时器管理器 (librte_timer)

该库为 DPDK 执行单元提供定时器服务，提供异步执行功能的能力。它可以是周期性的函数调用，也可以是一次性调用。它使用环境抽象层 (EAL) 提供的计时器接口来获取精确的时间参考，并且可以根据需要在每个内核的基础上启动。

###  以太网轮询模式驱动程序架构

DPDK 包括用于 1 GbE、10 GbE 和 40 GbE 的轮询模式驱动程序 (PMD)，以及半虚拟化的 virtio 以太网控制器，这些控制器旨在在没有异步、基于中断的信号机制的情况下工作。

数据包转发算法支持

DPDK 包括哈希（librte_hash）和最长前缀匹配（LPM，librte_lpm）库，以支持相应的数据包转发算法。

### librte_net

librte_net 库是 IP 协议定义和便利宏的集合。它基于 FreeBSD* IP 堆栈中的代  码，包含协议编号（用于 IP 标头）、IP 相关宏、IPv4/IPv6 标头结构以及 TCP、UDP 和 SCTP 标头结构。

**优化技术**

- PMD用户态驱动，使用无中断方式直接操作网卡的接收和发送队列；
- 采用HugePage减少TLB Miss；
- DPDK采用向量SIMD指令优化性能；
- CPU亲缘性和独占；
- 内存对齐：根据不同存储硬件的配置来优化程序，确保对象位于不同channel和rank的起始地址，这样能保证对象并并行加载，性能也能够得到极大的提升；
- Cache对齐，提高cache访问效率：
- NUMA亲和，提高numa内存访问性能；
- 减少进程上下文切换：保证活跃进程数目不超过CPU个数；减少堵塞函数的调用，尽量采样无锁数据结构；
- 利用空间局部性，采用预取Prefetch，在数据被用到之前就将其调入缓存，增加缓存命中率；
- 充分挖掘网卡的潜能：借助现代网卡支持的分流（RSS, FDIR）和卸载（TSO，chksum）等特性；

**DPDK支持的CPU体系架构**

x86、ARM、PowerPC（PPC）

**DPDK支持的网卡列表**

https://core.dpdk.org/supported



## **四、DPDK的基石UIO**

为了让驱动运行在用户态，Linux提供UIO机制。使用UIO可以通过read感知中断，通过mmap实现和网卡的通讯。

UIO（Userspace I/O）是运行在用户空间的I/O技术。Linux系统中一般的驱动设备都是运行在内核空间，而在用户空间用应用程序调用即可，而UIO则是将驱动的很少一部分运行在内核空间，而在用户空间实现驱动的绝大多数功能！使用UIO可以避免设备的驱动程序需要随着内核的更新而更新的问题.通过UIO的运行原理图可以看出,用户空间下的驱动程序比运行在内核空间的驱动要多得多,UIO框架下运行在内核空间的驱动程序所做的工作比较简单。





UIO原理：



![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nEHYY1vNIQOKf8iaKxv341Vib5u3YKhHzsdXDMP3FUeV9MQJzggSCP0kjNC3ibxgtibGfwuDxJfeLZiaA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)





要开发用户态驱动有几个步骤：

1.开发运行在内核的UIO模块（分配和记录设备需要的资源和注册uio设备），因为硬中断只能在内核处理；

2.通过/dev/uioX读取中断；

3.通过mmap和外设共享内存，实现零拷贝；



## 五、DPDK核心优化：PMD

UIO旁路了内核，主动轮询去掉硬中断，DPDK从而可以在用户态做收发包处理。带来Zero Copy、无系统调用的好处，同步处理减少上下文切换带来的Cache Miss。

运行在PMD的Core会处于用户态CPU100%的状态



![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nEHYY1vNIQOKf8iaKxv341Vy1F4ycm3jddzxZQ3Uyw1P0JehpvtTVX0lINoncNAqNj5fbeyxDZlqw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



网络空闲时CPU长期空转，会带来能耗问题。所以，DPDK推出Interrupt DPDK模式。

Interrupt DPDK：



![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nEHYY1vNIQOKf8iaKxv341VAuZyFoUZ8Kuicc19dywasS0clYmNqshZVAkvzMosgbOYDEHmwXs6CXw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图片引自David Su/Yunhong Jiang/Wei Wang的文档《Towards Low Latency Interrupt Mode DPDK》

它的原理和NAPI很像，就是没包可处理时进入睡眠，改为中断通知。并且可以和其他进程共享同个CPU Core，但是DPDK进程会有更高调度优先级。



## 六、DPDK的高性能代码实现

**1. 采用HugePage减少TLB Miss**





![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6laZIYYWK4xiaZPmRb3WHQAQSnWM6bq1xGJCVZRzX4WeoVSwgaKvM0UbgPmB6uaxg8WoJibqCDE2vFQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

默认下Linux采用4KB为一页，页越小内存越大，页表的开销越大，页表的内存占用也越大。CPU有TLB（Translation Lookaside Buffer）成本高所以一般就只能存放几百到上千个页表项。如果进程要使用64G内存，则64G/4KB=16000000（一千六百万）页，每页在页表项中占用16000000 * 4B=62MB。如果用HugePage采用2MB作为一页，只需64G/2MB=2000，数量不在同个级别。



![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6laZIYYWK4xiaZPmRb3WHQAQVh6a5nt0fZYXIsCaLt0BEPIwpGMgJy6ZbukobdnBAiblmYSbaugFZcg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



而DPDK采用HugePage，在x86-64下支持2MB、1GB的页大小，几何级的降低了页表项的大小，从而减少TLB-Miss。并提供了内存池（Mempool）、MBuf、无锁环（Ring）、Bitmap等基础库。根据我们的实践，在数据平面（Data Plane）频繁的内存分配释放，必须使用内存池，不能直接使用rte_malloc，DPDK的内存分配实现非常简陋，不如ptmalloc。



**2. SNA（Shared-nothing Architecture）**

![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6laZIYYWK4xiaZPmRb3WHQAQSbcqtAWKRABFHpHQj21YDdd07GhGIjbHpcib7N507cjXA8XDbYuJic3w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



软件架构去中心化，尽量避免全局共享，带来全局竞争，失去横向扩展的能力。NUMA体系下不跨Node远程使用内存。

**3. SIMD（Single Instruction Multiple Data）**

![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6laZIYYWK4xiaZPmRb3WHQAQoeicdzGnZt0V6Nn3ZFMOvVuqFibsyr28LyHpf2NgYv5hL18eHdXKo1fQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



从最早的mmx/sse到最新的avx2，SIMD的能力一直在增强。DPDK采用批量同时处理多个包，再用向量编程，一个周期内对所有包进行处理。比如，memcpy就使用SIMD来提高速度。SIMD在游戏后台比较常见，但是其他业务如果有类似批量处理的场景，要提高性能，也可看看能否满足。



**4. 不使用慢速API**

这里需要重新定义一下慢速API，比如说gettimeofday，虽然在64位下通过vDSO已经不需要陷入内核态，只是一个纯内存访问，每秒也能达到几千万的级别。但是，不要忘记了我们在10GE下，每秒的处理能力就要达到几千万。所以即使是gettimeofday也属于慢速API。DPDK提供Cycles接口，例如rte_get_tsc_cycles接口，基于HPET或TSC实现。

在x86-64下使用RDTSC指令，直接从寄存器读取，需要输入2个参数，比较常见的实现：

![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabkw1RIoghHwDYunXWBMfDib7woDfnC2TWV9WYXMeTLQqgjp6MoGicJ8ibdI0yQFWjFYHmuzicibGYaALuA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这么写逻辑没错，但是还不够极致，还涉及到2次位运算才能得到结果，我们看看DPDK是怎么实现：

![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabkw1RIoghHwDYunXWBMfDib7j1CpJZqvSSzxsx4dceca1pv31CPxJNTjpTS1GZiccEs9kr7X9KdhFyg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

巧妙的利用C的union共享内存，直接赋值，减少了不必要的运算。但是使用tsc有些问题需要面对和解决



\1) CPU亲和性，解决多核跳动不精确的问题

\2) 内存屏障，解决乱序执行不精确的问题

\3) 禁止降频和禁止Intel Turbo Boost，固定CPU频率，解决频率变化带来的失准问题



**5. 编译执行优化**

**1) 分支预测**

**![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6laZIYYWK4xiaZPmRb3WHQAQ2RRW48DWNicW3McEwvBd0dWRt2KQxMuJsWaYYTyibO5EbAw4CicKMTIcQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)**

现代CPU通过pipeline、superscalar提高并行处理能力，为了进一步发挥并行能力会做分支预测，提升CPU的并行能力。遇到分支时判断可能进入哪个分支，提前处理该分支的代码，预先做指令读取编码读取寄存器等，预测失败则预处理全部丢弃。我们开发业务有时候会非常清楚这个分支是true还是false，那就可以通过人工干预生成更紧凑的代码提示CPU分支预测成功率。

![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabkw1RIoghHwDYunXWBMfDib7hMyQFKicCBqCwflpk8Kia5cgdEYdPRggxraDWpMs0d8YhFxNbfZg8dSg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**2) CPU Cache预取**

![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6laZIYYWK4xiaZPmRb3WHQAQuv3VykP0Q2a3uD1JJaAGTwNKcVAX1lrrN3nvXyibf6PickbkBoGlECtQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Cache Miss的代价非常高，回内存读需要65纳秒，可以将即将访问的数据主动推送的CPU Cache进行优化。比较典型的场景是链表的遍历，链表的下一节点都是随机内存地址，所以CPU肯定是无法自动预加载的。但是我们在处理本节点时，可以通过CPU指令将下一个节点推送到Cache里。

API文档：https://doc.dpdk.org/api/rte__prefetch_8h.html

![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabkw1RIoghHwDYunXWBMfDib7Al0voVBmmMBgdibCrpxCyFOsib2IKvU1oTRn8c1WJQxWKKibicqBmiaopXg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**3) 内存对齐**



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6laZIYYWK4xiaZPmRb3WHQAQxpWZIVzfn42UiaBNiab08edCicylnM8BOicGGgsCiahHzaKQhNlUGzUW51A/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



内存对齐有2个好处：

3.1 避免结构体成员跨Cache Line，需2次读取才能合并到寄存器中，降低性能。结构体成员需从大到小排序和以及强制对齐。

参考《Data alignment: Straighten up and fly right》

![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabkw1RIoghHwDYunXWBMfDib7E9icZhIKymL5MBfzOeTmxSYYOm3bMZ20CW57GmNUib1KsnV4SCPDYVHg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

3.2 多线程场景下写产生False sharing，造成Cache Miss，结构体按Cache Line对齐



![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6laZIYYWK4xiaZPmRb3WHQAQTKjlnBNVtsaWW2fYZ7NlxZqlbmFn4JdvDibVpJWxOKxUvaILAicQZmVw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabkw1RIoghHwDYunXWBMfDib7vLfkvHGlsVWZ8PKVXUHnQDk5K9sfjOSrl2kWewx30LicLicFXTNW5zKA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



**4) 常量优化**

常量相关的运算的编译阶段完成。比如C++11引入了constexp，比如可以使用GCC的__builtin_constant_p来判断值是否常量，然后对常量进行编译时得出结果。举例网络序主机序转换

![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabkw1RIoghHwDYunXWBMfDib78yVUu1F5pmGE68DMfjglJlFsheFLwxC21bniaUl1H2kTDFoTicmArQuw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

其中rte_constant_bswap32的实现

![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabkw1RIoghHwDYunXWBMfDib7pNbPpicm4EtTKYXjSAaLFI8VSeRHJxBR8KOZtGhXgx7ZnbXmyBz9nhQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**5）使用CPU指令**

现代CPU提供很多指令可直接完成常见功能，比如大小端转换，x86有bswap指令直接支持了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/4amQNRETabkw1RIoghHwDYunXWBMfDib7mQ8Nr8F7hm9rrCb9fIjPYokRTojliaJDSOib97L8OIaicVpyMJTkviab3g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



这个实现，也是GLIBC的实现，先常量优化、CPU指令优化、最后才用裸代码实现。毕竟都是顶端程序员，对语言、编译器，对实现的追求不一样，所以造轮子前一定要先了解好轮子。

Google开源的cpu_features可以获取当前CPU支持什么特性，从而对特定CPU进行执行优化。高性能编程永无止境，对硬件、内核、编译器、开发语言的理解要深入且与时俱进。

## 七、DPDK生态

对我们互联网后台开发来说DPDK框架本身提供的能力还是比较裸的，比如要使用DPDK就必须实现**TCP/IP协议栈**（ARP，IP，TCP/UDP, socket等）这些基础功能，有一定上手难度。如果要更高层的业务使用，还需要**用户态的协议栈**支持。不建议直接使用DPDK。



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6laZIYYWK4xiaZPmRb3WHQAQYNgKqgl4PtHVchtYvzLx8GWzhYvmyBlRLXoQic4Z8tu8MVdq1BrnNGw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

应用

目前生态完善，社区强大（一线大厂支持）的应用层开发项目是**FD.io**（The Fast Data Project），有思科开源支持的VPP，比较完善的协议支持，ARP、VLAN、Multipath、IPv4/v6、MPLS等。用户态传输协议UDP/TCP有TLDK。从项目定位到社区支持力度算比较靠谱的框架。

Fd.io: The Universal Dataplane

**FD.io**（快速数据 - 输入/输出）是多个项目和库的集合，用于扩展基于数据平面开发套件 (DPDK) 的应用，以在通用硬件平台上支持灵活、可编程和可组合的服务。FD.io 为软件定义基础设施开发人员社区提供了一个登陆站点，其中包含多个项目：





![图片](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6lxib9nibKIEcFsmyfzMvlwGjqx8OwSPNIFDrkvzmRZlhvKZ7ATVqaXxmXofsNF9htPSw0jjkVkwlkw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

促进基于软件的数据包处理的创新，以创建适用于许多架构（x86、ARM、 PowerPC）和部署环境（裸机、VM、容器）。



腾讯云开源的**F-Stack**也值得关注一下，开发更简单，直接提供了POSIX接口。





![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6lxib9nibKIEcFsmyfzMvlwGjVR6gxr05omH7uVibHiahhB9lBCibvTTf4Yb7jxHgDShy3QMialmxx1E2Aw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**F-Stack是一个基于DPDK的开源高性能网络框架**，具有以下特点：

1. 网卡满载时可以达到的超高网络性能：1000万并发连接，500万RPS，100万CPS。

2. 移植FreeBSD 11.01用户空间堆栈，提供完整的堆栈功能，并删减了大量无关功能。这大大提高了网络性能。

3. 支持Nginx、Redis等成熟应用。服务可以轻松使用 F-Stack。

4. 易于扩展的多进程架构。

5. 提供微线程接口。各种有状态应用程序可以轻松使用 F-Stack 来获得高性能，而无需处理复杂的异步逻辑。

6. 提供 Epoll/Kqueue 接口，允许多种应用轻松使用 F-Stack。

   

**Seastar**也很强大和灵活，内核态和DPDK都随意切换，也有自己的传输协议Seastar Native TCP/IP Stack支持，但是目前还未看到有大型项目在使用Seastar，可能需要填的坑比较多。



![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6lxib9nibKIEcFsmyfzMvlwGjXeyOfibJd2KvFDE4nRLN9qE9nZsN28EMq80kiaIRcSIfoicX77rB0NrMA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



 Seastar 目前专注于高吞吐量、低延迟的 I/O 密集型应用程序。

- Pedis : Redis 兼容的数据结构存储
- Scylla：NoSQL 列存储数据库，以 10 倍的吞吐量与 Apache Cassandra 兼容
- Seastar HTTPD：网络服务器
- Seastar Memcached：Memcache 键值存储的快速服务器

**Open vSwitch (OVS)**高性能开源虚拟交换机, 可以利用DPDK这些功能绕过 Linux 内核OVS 处理,增强OVS的IO性能，官方数据显示，可以提高9倍以上的性能提升.



![图片](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6lxib9nibKIEcFsmyfzMvlwGjYh2iaAaI2PW7fESYUP66F1EJYoCXqy12BWkt1bRFXf8iatOrObBN2Vfg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## **八、高性能编程技术和代码优化**

性能评估公式

![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6khRXP0ZqI4HGicVpY2zd4iajFqFWVzibz3h88cFYCqIoVvJiboqibuwSPIRbNLr4BgGHhOEPr57UY03Aw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- IPP 表示代码的复杂程度，IPC 表示代码执行的效率

- 2G主频CPU处理10G网卡达到线速•2GHz/14.8Mpps =134 clock cycles

  

**1.用户空间轮询**

- 减少中断带来开销；
- 减少系统调用带来开销；
- 零拷贝减少内存拷贝的时间；
- 轮询 Polling,busy looping 提供了I/O批量处理的可能性；
- 避免中断性能瓶颈是DPDK提升数据包处理速度的关键之一；



**2.多核CPU性能优化**

- RSS硬件队列；

- CPU独占：独占CPU资源，减少调度影响，提高系统性能；

- CPU绑定：减少CPU上下文切换，提高系统性能；

- 中断亲和 : 中断负载均衡，减轻其他CPU负担，提高系统性能；

- 进程亲和：减少CPU上下文切换，提高系统性能；

- 中断隔离：减少中断对CPU调度影响，提高系统性能；

- Per CPU：Per-CPU是基于空间换时间的方法, 让每个CPU都有自己的私有数据段(放在L1中),并将一些变量私有化到 每个CPU的私有数据段中. 单个CPU在访问自己的私有数据段时, 不需要考虑其他CPU之间的竞争问题,也不存在同步的问题.  注意只有在该变量在各个CPU上逻辑独立时才可使用；

  

**3. 锁优化**

- 无锁数据结构，将并发最大化；

- Per-CPU设计，尽量避免资源竞争；

- 采用RCU机制，读写共享数据可以无锁并行；

  [  深入理解RCU|核心原理](http://mp.weixin.qq.com/s?__biz=MzAxNDI5NzEzNg==&mid=2651163409&idx=1&sn=bed98c68ebeb7ffdfb9c91febd0f5713&chksm=80645a4eb713d3580db29964e90026e2390ef5e2e77a0f2e9164433e8008bb0d3440bff0c972&scene=21#wechat_redirect)

- spinlock，采用非阻塞锁，防止上下文切换导致cache miss；

- 采用CAS原子操作(Compare and Swap)进行无锁设计；



**4.批量处理**

- 轮询机制允许一次接收或发送多个报文；
- 批量处理分摊的接收或发送操作本身的开销；
- 绝大部分报文需要做相同或相似的计算处理，意味着相同的指令会被反复地执行，报文的批量计算分摊了函数调用的上下文切换，堆栈的初始化等等开销，同时大大减少了l1i cache miss
- 对于某一个函数，l1icache miss 仅仅发生在第一个报文被处理的时刻
- 批量计算提供了更好的代码优化可能性（数据预取，多重循环等）



**5.Cache优化**

- CPU的速度远远大于RAM的速度
- 程序在运行时具有局部性规律

​    时间局部性，很快还会访问

​    空间局部性，相邻也会访问

- 不同级别cache速度差异 L1 > L2 > L3
- 减少Cache Miss是提升性能的关键
- Cache 层次化结构



![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6khRXP0ZqI4HGicVpY2zd4iajewvrA9qibw0TUSpQXypiaY1qGCyyopia1t80gwlHQXlYo7qy8bFHJHhNA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

 

- iCacheMiss 常常被忽略

​    更优的代码，编译器优化选项

​    更小的代码尺寸

​    更好的代码布局- 分支预测

- 代码布局影响iCache命中率



![图片](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6khRXP0ZqI4HGicVpY2zd4iajvMfIO7qfjr72PP8dlyeyev1DP7UdWJggxbHgCwiajB5M56Y87qTSc9A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​      B 段代码较少会被调用

- Cache一致性问题

  原则是避免多个核访问同一个内存地址或数据结构

  在数据结构上，每个核都有独立的数据结构

  多个核访问同一个网卡：每个核都创建单独的接收队列和发送队列




**6. 代码优化技巧**

- Cache Line 对齐，减少dCache miss， 避免伪共享；
- 数据预取，减少dCache miss， prefetch 指令；
- 分支预测，优化代码布局， 提高CPU流水线效率；‍‍‍‍‍‍
- 函数内联，减少函数调用开销；
- CPU扩展指令集SIMD：sse,avx，减少指令数量，最大化的利用一级缓存访存的带宽；
- 多重循环处理报文，更好地优化CPU流水线；
- 编译器优化；



‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍原文链接：https://mp.weixin.qq.com/s/SCIvXCghKyXCiT-KednDjw
# DPDK源码分析之DPDK基础概览

本文主要介绍一下DPDK这项技术的基础概览，包括什么是DPDK，为什么有它存在的必要，它的框架是怎样的，使用了哪些技术实现，DPDK的应用场景有哪些，最后在centos7服务器上实装一个dpdk环境做一个简单的数据包收发的小实验。

## 1.网络基础知识

### 1.1什么是路由器?

提供**路由（host到host之间的最佳传输路径）与转发**（路由器输⼊端的数据包移送⾄适当的输出端）两种重要机制的三层网络节点设备。

![img](https://pic1.zhimg.com/80/v2-640e3f22440dc01d2274e7d49c831710_720w.webp)

### 1.2. 什么是交换机?

交换机按部署方式分为二层交换机，三层交换机，四层交换机和七层交换机。

二层交换机工作在数据链路层，根据**MAC进行交换机端口选择**，转发通过ASIC (Application specific Integrated Circuit)芯片实现。总体的工作流程如下：

主机A通过ARP协议广播得到主机B的Mac地址，接着主机A以主机B的Mac地址和IP地址发送网络包，中间设备交换机根据Mac地址查找Mac-端口映射表，将数据包发送到对应的端口中，如果没有找到记录则会发送广播包，询问Mac地址并将响应包的端口归属进行新增记录。

三层交换机利用**IP交换技术**，在不同子网的两个主机通过映射关系直接硬件交换，而避开路由器，目的就是减轻路由器的负担。

四层交换机和七层交换机主要通过流的会话，传输层端口号以及协议内容，做出更智能的**负载均衡**决定。

### 1.3. 什么是网关?

网关就是网络的入口和出口，用于**定义网络的边界**，对于局域网来说网关就是路由器。

按照功能划分，最常见的就是**传输网关，用于在2个网络间建立传输连接；其次还有应用网关是在使用不同[数据格式](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/%E6%95%B0%E6%8D%AE%E6%A0%BC%E5%BC%8F)间翻译数据的系统**。典型的应用网关接收一种格式的输入，将之翻译， 然后以新的格式发送。

### 1.4. 什么是NFV?

NFV技术的诞生源于互联网发展迅速，原来运营商的架构存在很多痛点：能固化的运营设施与业务的灵活适配、稳定的通信标准与新业务的快速部署、粗粒度的网络资源调整与用户/业务资源的按需分配能力。

NFV技术，它使得专有的网络功能能够运行在通用x86架构硬件上的虚拟化网络功能（Virtual Network Function，VNF）中，即**网络设备和网络功能解耦**，为电信运营商和互联网服务商提供了一种灵活的业务部署手段和高效的组网方案，可以支持固移网络和 IDC 中 NAT（Network Address Translation，网络地址转换）、DPI（Deep Packet Inspection，深度分组检测）、EPC （Evolved Packet Core，演进分组核心网）、防火墙（Firewall）等各类业务功能的广域灵活部署与网元弹性扩展。

![img](https://pic3.zhimg.com/80/v2-ba4a05163f4a62dae5320879d68d5226_720w.webp)

一个NFV的标准架构包括NFV infrastructure(NFVI)，MANO(Management and Orchestration)和VNFs。

![img](https://pic2.zhimg.com/80/v2-a6c9d8f3f1cb43e87a8e570fc9cdc9dd_720w.webp)

> NFVI：提供VNF的运行环境，包括所需的硬件及软件。硬件包括计算、网络、存储资源；软件主要包括Hypervisor、网络控制器、存储管理器等工具，NFVI将物理资源虚拟化为虚拟资源，供VNF使用。
> VNF：包括VNF和EMS，VNF网络功能，EMS为单元管理系统，对VNF的功能进行配置和管理。一般情况下，EMS和VNF是一一对应的。
> VIM：NFVI管理模块，通常运行于对应的基础设施站点中，主要功能包括：资源的发现、虚拟资源的管理分配、故障处理等，为VNF运行提供资源支持。
> VNFM：VNF管理模块，主要对VNF的生命周期（实例化、配置、关闭等）进行控制，一般情况下与VNF一一对应。
> NFVO：NS生命周期的管理模块，同时负责协调NS、组成NS的VNFs以及承载各VNF的虚拟资源的控制和管理。
> OSS/BSS：服务提供商的管理功能，不属于NFV框架内的功能组件，但NFVO需要提供对OSS/BSS的接口。

### 1.5. 什么是SDN？

SDN技术路线强调网络设备的控制面可直接编程，将网络服务从底层硬件设备中抽象出来，开发可编程的控制平面，硬件仍负责转发，数据中心使用较广。

SDN架构可分为基础设施层、控制层和应用层。

- 基础设施层：主要为转发设备，实现转发功能，例如数据中心交换机。
- 控制层：由SDN控制软件组成，可通过标准化协议与转发设备进行通信，实现对基础设施层的控制。
- 应用层：常见的有基于[OpenStack](https://link.zhihu.com/?target=https%3A//info.support.huawei.com/info-finder/encyclopedia/zh/OpenStack.html)架构的云平台。另外，也可以基于OpenStack构建用户自己的云管理平台。

SDN使用北向和南向应用程序接口（API）来进行层与层之间的通信，分为北向API和南向API。北向API负责应用层和控制层之间的通信，南向API负责基础设施层和控制层之间的通信。

![img](https://pic4.zhimg.com/80/v2-7faf9e62e0d44c10db6c895ade17eda3_720w.webp)

![img](https://pic1.zhimg.com/80/v2-ca1fd0d36ca0f1e749614557c55f4ba4_720w.webp)

**SDN抽象物理网络资源（交换机、路由器等），并将决策转移到虚拟网络控制平面**。控制平面决定将流量发送到哪里，而硬件继续引导和处理流量，无需依赖标准的硬件设备。**NFV的目标是将所有物理网络资源进行虚拟化**，允许网络在不添加更多设备的情况下增长，这依赖于标准的硬件设备。

## 2.为什么要有DPDK

大部分的目标平台都是以Intel为架构的多核处理器，在IA上，网络数据包处理远早于DPDK而存在。从商业版的Windows到开源的Linux操作系统，所有跨主机通信几乎都会涉及网络协议栈以及底层网卡驱动对于数据包的处理。然而，在应对如今的万兆带宽的高速网络处理，现在架构的操作系统逐渐产生瓶颈，甚至在2010年前采用IA处理器的用户会得出这样一个结论，那就是**IA不适合做高速包处理**。

我们先以Linux为例，传统网络设备驱动包处理的动作可以概括如下：

❑数据包到达网卡设备。

❑网卡设备依据配置进行DMA操作。

❑网卡发送中断，唤醒处理器。

❑驱动软件填充读写缓冲区数据结构

❑数据报文达到内核协议栈，进行高层处理。

❑如果最终应用在用户态，数据从内核搬移到用户态。

❑如果最终应用在内核态，在内核继续进行。

综上所述，从软件结构上看，报文的收发需要经过物理网卡驱程、宿主机内核网络协议栈、内核态虚拟交换层、虚拟机（VM）网卡驱程、虚拟机内核态网络协议栈、虚拟机用户态App等多个转发通道，**存在着海量系统中断、内核上下文切换、内存复制、虚拟化封装/解封等大量CPU费时操作过程。**

随着芯片技术与高速网络接口技术的一日千里式发展，报文吞吐需要高达10Gbit/s的端口处理能力，市面上已经出现大量的25Gbit/s、40Gbit/s甚至100Gbit/s高速端口，但这么大的数据流量，早期的Linux和服务器根本无法处理。

## 3.什么是DPDK

data-plane-development-kit，数据平面的开发套件，可以极大提高数据处理性能和吞吐量，为数据平面应用程序提供更多时间。

DPDK并非是凭空使用了什么神秘的技术，而是多年的工程优化迭代和最佳实践的融合。

### 3.1轮询技术

为了减少中断处理开销，DPDK使用了轮询技术来处理网络报文。网卡收到报文后，直接将报文保存到处理器缓存中（有DDIO（Direct Data I/O）技术的情况下），或者内存中（没有DDIO技术的情况下），并设置报文到达的标志位。应用软件则周期性地轮询报文到达的标志位，检测是否有新报文需要处理。整个过程中完全没有中断处理过程

### 3.2巨页技术

DPDK则利用巨页技术，所有的内存都是从巨页里分配，实现对内存池（Mempool）的管理，并预先分配好同样大小的mbuf，供每一个数据分组使用。使用了巨页表功能后，一个TLB表项可以指向更大的内存区域，这样可以大幅减少TLB miss的发生。

### 3.3CPU亲和性

DPDK工作在用户态，线程的调度仍然依赖内核。利用线程的CPU亲和绑定的方式，特定任务可以被指定只在某个核上工作。好处是可避免线程在不同核间频繁切换，核间线程切换容易导致因cache miss和cache write back造成的大量性能损失。

### 3.4用户态驱动

户态驱动，在这种工作方式下，既规避了不必要的内存拷贝又避免了系统调用，实现了零拷贝

## 3.5软件调优多核间访问避免跨cache line共享

**（6）挖掘最新的IA硬件指令集**

## 4.DPDK框架

DPDK主要模块分解如下图所示。它大量利用了有助于包处理的软硬件特性，如大页、缓存行对齐、线程绑定、预取、NUMA、IA最新指令的利用、Intel®DDIO、内存交叉访问等。

![img](https://pic3.zhimg.com/80/v2-152d7d6a50e29f90a795cca3819ac24a_720w.webp)

### 4.1核心部件库-core Libraries

该模块构成的运行环境建立在Linux上，通过环境抽象层（EAL）的运行环境进行初始化，包括巨页内存分配、内存/缓冲区/队列分配与无锁操作、CPU亲和性绑定等；其次，EAL实现了对操作系统内核与底层网卡I/O操作的屏蔽（I/O旁路了内核及其协议栈），为DPDK应用程序提供了一组调用接口，通过UIO或VFIO技术将PCI设备地址映射到用户空间，方便了应用程序的调用，避免了网络协议栈和内核切换造成的处理时延。另外，核心部件还包括创建适合报文处理的内存池、缓冲区分配管理、内存复制、定时器、环型缓冲区管理等。

### 4.2平台相关模块-platform

其内部模块主要包括KNI、能耗管理以及IVSHMEM接口。其中，KNI模块主要通过kni.ko模块将数据报文从用户态传递给内核态协议栈处理，以便用户进程使用传统的Socket接口对相关报文进行处理；能耗管理则提供了一些API，应用程序可以根据分组接收速率动态调整处理器频率或进入处理器的不同休眠状态；另外，IVSHMEM模块提供了虚拟机与虚拟机之间，或者虚拟机与主机之间的零复制共享内存机制，当DPDK程序运行时，IVSHMEM模块会调用核心部件库API，把几个巨页映射为一个IVSHMEM设备池，并通过参数传递给QEMU，这样，就实现了虚拟机之间的零复制内存共享。

### 4.3轮询模式驱动模块-PMD-natives & virtual

PMD相关API实现了在轮询方式下进行网卡报文收发，避免了常规报文处理方法中因采用中断方式造成的响应时延，极大提升了网卡收发性能。此外，该模块还同时支持物理和虚拟化两种网络接口，从仅仅支持Intel网卡，发展到支持Cisco、Broadcom、Mellanox、Chelsio等整个行业生态系统以及基于KVM、VMware、XEN等虚拟化网络接口。

### 4.4Classify库

支持精确匹配（Exact Match）、最长匹配（LPM）和通配符匹配（ACL），提供常用包处理的查表操作

### 4.5QoS库

提供网络服务质量相关组件，如限速（Meter）和调度（Sched）

## 5.DPDK库函数

### 5.1. EAL库

EAL环境抽象层用于获取底层资源（硬件或内存空间）。EAL提供了一个通用接口来屏蔽应用和库的环境特殊性，同时也负责初始化工作分配资源（内存空间，PCI设备，时钟，控制等）。

EAL主要提供以下几种典型的服务：

- DPDK的装载和启动
- 内核亲和性与分配
- 系统内存预留
- PCI地址抽象化
- 跟踪与调试功能
- 原子计数
- CPU特征识别
- 中断处理
- 告警功能

在装载和启动层面，Linux用户空间环境下，DPDK通过使用pthread库运行一个类似用户空间的应用，PCI设备信息和地址空间的获取通过/sys下的内核接口以及内核模块如（igb_uio）来实现。

在内存层面，EAL通过使用mmap（）函数在巨页表内进行物理内存分配，通过巨页内存分配提高性能，

在亲和性方面，DPDK服务初始化后，EAL通过pthread亲和性设置来实现应用实例的核绑定。

> [DPDK系列之三：Linux UIO技术在DPDK的应用_cloudvtech的博客-CSDN博客_dpdk uio](https://link.zhihu.com/?target=https%3A//blog.csdn.net/cloudvtech/article/details/80359834%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164126003916781683920755%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D%26request_id%3D164126003916781683920755%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~baidu_landing_v2~default-4-80359834.pc_search_es_clickV2%26utm_term%3Digb_uio%26spm%3D1018.2226.3001.4187)

### 2. Ring库

dpdk使用ring来管理队列，它有如下几个特性：

- 先进先出
- 一个ring用全局唯一的名字来标识
- ring有一个阈值，当被设置时，一旦入队达到阈值，生产者就会接到通知
- 并非全无锁，有若干次CAS操作

> [dpdk介绍系列之ring_潜心修行，独立思考-CSDN博客_dpdk ring 用法](https://link.zhihu.com/?target=https%3A//blog.csdn.net/me_blue/article/details/77678868)

### 3. Mempool库

dpdk的mempool是预先分配好大小的内存池，它以名字加以区分，每个内存池通过注册函数都有独立的内存申请和释放函数，对象的处理函数默认采用无锁ring队列实现。其结构如下：

![img](https://pic3.zhimg.com/80/v2-3fa4e74bc2334135f92edd6526920f5a_720w.webp)

- 每个core独立的对象缓存区local_cache，用来减少多核访问造成的冲突

![img](https://pic4.zhimg.com/80/v2-1dc1331df3ba929c6da5d24d85554d9b_720w.webp)

- 根据命令行配置的内存通道或rank，将对象进行内存对齐填充

> [DPDK 内存池rte_mempool实现（二十三）_bob的博客-CSDN博客_dpdk内存池](https://link.zhihu.com/?target=https%3A//blog.csdn.net/qq_20817327/article/details/106459262%3Fspm%3D1001.2101.3001.6650.1%26utm_medium%3Ddistribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.pc_relevant_paycolumn_v2%26depth_1-utm_source%3Ddistribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.pc_relevant_paycolumn_v2%26utm_relevant_index%3D2)

### 4. mbuf库

mbuf库用来分配和释放缓冲区，这部分缓冲区是存储在mempool中，用来存储进程和网络接口间互相传递的用户数据。dpdk采用一个大小固定的单一内存缓冲区来存储数据分组，这种方法的优势是对整个内存只有一对分配和释放操作。

```c
/* mbuf的头部信息 */
这个结构用来描述mbuf跟具体的内容无关
struct m_hdr {
       struct  mbuf *mh_next;        /* 指向链中下一个mbuf的指针 */
       struct  mbuf *mh_nextpkt;     /* 指向下一个链的指针 */
       int     mh_len;               /* mbuf中数据的长度（不包括头部） */
       char    *mh_data;             /* 指向数据区的指针 */
       short   mh_type;              /* mbuf的数据类型，如MT_DATA*/
       short   mh_flags;             /* mbuf标识，具体定义见下 */
};

//头部填充数据
rte_pktmbuf_prepend()

//尾部填充数据
rte_pktmbuf_append()

//头部移除数据
rte_pktmbuf_adj()

//尾部移除数据
rte_pktmbuf_trim()
```

### 5. PMD库

dpdk提供1Gbit/s，10Gbit/s，40Gbit/s和虚拟网卡驱动的PMD，轮询模式驱动

PMD由API组成，运行在用户空间，可以配置设备以及对应的队列，一个PMD可以无中断的直接访问Rx和Tx，进而快速的接收，处理和转发数据分组。

dpdk对于数据包的处理有两种情况：

- rtc模式

通过指定端口的rx ring接收数据包，并在当前逻辑核处理，处理后通过指定端口tx转发出去，该模式的特性就是同一时刻只处理一个数据分组

- pipleline模式

多核协作模式，一个核负责收报文，一个核负责发报文，一个核或几个核负责处理报文，多个核之间采用ring队列进行数据交换。

### 6. IVSHMEM库

Ivshmem是虚拟机内部共享内存的pci设备。虚拟机之间实现内存共享是把内存映射成guest内的pci设备来实现的。

dpdk提供Ivshmem库用于虚拟机内快速共享零拷贝数据。

此库使用一个命令将几个巨页映射为一个单一IVSHMEM设备，同时将一个元数据文件映射出一个IVSHMEM段，用来区分DPDK和非DPDK的IVSHMEM设备:

![img](https://pic2.zhimg.com/80/v2-0580b927353779dbc98d103aaf7dced9_720w.webp)

> [《深入浅出DPDK》读书笔记（十四）：DPDK应用篇（DPDK与网络功能虚拟化：NFV、VNF、IVSHMEM、Virtual BRAS“商业案例”](https://link.zhihu.com/?target=https%3A//blog.csdn.net/Rong_Toa/article/details/109370979%3Fops_request_misc%3D%26request_id%3D%26biz_id%3D%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~es_rank~default-1-109370979.pc_search_es_clickV2%26utm_term%3DIVSHMEM%2Bdpdk%26spm%3D1018.2226.3001.4187)

### 7. Timer库

为DPDK执行单元提供计时器功能，用于保证执行体异步回调功能

定时器结构主要包含一个特殊的状态机，此状态机用于定时器状态切换（包括定时器停止、增加、运行、配置等），且为所属者（逻辑核ID）唯一的状态机。

1）STOPPED：没有所属者，也不存在于列表中。

2）CONFIG：属于一个核，而且不能被另外的核修改，也许存在一个列表中，也许不存在，这主要依赖于前期状态。

3）PENDING：属于一个核，存在于一个列表内。

4）RUNNING：属于一个核，不能被另外的核修改，存在于一个列表内。

当定时器处于CONFIG或RUNNING状态时，不允许重置或停止一个定时器；当修改定时器状态时，CAS指令会保证定时器状态能够自动修改。

为了提高性能，每一个核独自维护着那些没有过期的定时器列表，这个列表的数据结构是跳表*skiplist*

### 8. LPM库

这个库主要实现了最长前缀匹配算法，目的是查找路由表确定报文转发的下一条。

### 9. Hash库

增删改查，hash算法为Cuckoo 算法

![img](https://pic3.zhimg.com/80/v2-2d16e5a3813dd3a7fbc5745d766b975e_720w.webp)

## 6.DPDK命令行启动参数

每个DPDK应用程序都要设定对应的DPDK目标环境的抽象层（EAL）库，EAL库提供一些选项给DPDK应用程序使用，选项列表在下面列出：./rte-app -c COREMASK [-n NUM] [-b <domain：bus：devid.func>] \[--Socket-mem=MB，...] [-m MB] [-r NUM] [-v] [--file-prefix] \[--proc-type <primary|secondary|auto>] [-- xen-dom0]EAL选项说明如下。

- -c COREMASK：16位进制掩码，用于指定使用的CPU核的编号。核的编号可以在不同平台间变化，但是需要事先确定。
- -n NUM：每个处理器插槽内存通道数。
- -b <domain:bus:devid.func>：端口名单，防止EAL使用特定的PCI设备（允许使用多个-b选项）。（4）--use-device：指定使用的以太网设备。
- --socket-mem：从巨页中给特定Socket分配的内存。
- -m MB：巨页分配的内存大小。建议使用--Socket-mem来代替该选项。
- -r NUM：memory rank的个数。
- -v：启动时显示版本信息。
- --huge-dir：hugetlbfs的安装目录。
- --file-prefix：巨页文件名的前缀文本。
- --proc-type：流程实例的类型。
- --xen-dom0：运行在XEN Domain0上无hugetlbfs的应用程序。
- --vmware-tsc-map：使用VMware的TSC视图替代本地RDTSC。
- --base-virtaddr：指定虚拟地址。
- --vfio-intr：指定要使用的VFIO中断类型。

**其中-c和-n选项是必选的，其他是可选的。**

## 7.DPDK安装及收发包实验

整体环境拓扑：

![img](https://pic4.zhimg.com/80/v2-2d483b2a3fdee417c6040c8dff3bb213_720w.webp)

网络不好的可以先设置一下镜像源：

```text
1、首先备份系统自带yum源配置文件/etc/yum.repos.d/CentOS-Base.repo

　　mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

2、进入yum源配置文件所在的文件夹

　　cd /etc/yum.repos.d/

3、下载163的yum源配置文件到上面那个文件夹内

　　CentOS7

　　　　wget http://mirrors.163.com/.help/CentOS7-Base-163.repo

　　CentOS6

　　　　wget http://mirrors.163.com/.help/CentOS6-Base-163.repo

　　CentOS5

　　　　wget http://mirrors.163.com/.help/CentOS5-Base-163.repo

4、运行yum makecache生成缓存

　　yum makecache

5、这时候再更新系统就会看到以下mirrors.163.com信息

　　yum -y update
```

首先进行tcpreplay发包工具安装：

```text
yum -y install tcpreplay
```

其次进行DPDK安装：

```text
export RTE_SDK=/home/nidps/dpdk-stable-20.11.2
export PKG_CONFIG_PATH=/usr/local/lib64/pkgconfig
export LD_LIBRARY_PATH=/usr/local/lib64

pip3 install meson ninja
yum install -y numactl numactl-devel

git clone git://dpdk.org/dpdk-stable
cd dpdk-stable
git checkout 20.11
meson build
cd build
ninja
ninja install

检查是否安装成功：
pkg-config --modversion libdpdk
20.11.3
```

内页内存配置：

```text
yum install libhugetlbfs
vim /etc/default/grub 
GRUB_CMDLINE_LINUX尾部追加transparent_hugepage=never default_hugepagesz=2M hugepagesz=2M hugepages=1024 
grub2-mkconfig -o /boot/grub2/grub.cfg

echo 4096 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 4096 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages

验证一下：
[root@bogon ~]# cat /proc/meminfo |grep -i HugePages
AnonHugePages:   1695744 kB
HugePages_Total:    8192
HugePages_Free:     7986
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

DPDK网卡绑定：

```text
./dpdk-devbind.py  --status
先看一眼状态：
[root@bogon usertools]# ./dpdk-devbind.py --status
Network devices using kernel driver
===================================
0000:1a:00.0 'Ethernet Connection X722 for 10GbE SFP+ 37d0' if=eno1 drv=i40e unused=igb_uio *Active*
0000:1a:00.1 'Ethernet Connection X722 for 10GbE SFP+ 37d0' if=eno2 drv=i40e unused=igb_uio *Active*
0000:1a:00.2 'Ethernet Connection X722 for 1GbE 37d1' if=eno3 drv=i40e unused=igb_uio *Active*
0000:af:00.0 'I350 Gigabit Network Connection 1521' if=eno5 drv=igb unused=igb_uio *Active*
0000:af:00.3 'I350 Gigabit Network Connection 1521' if=eno4 drv=igb unused=igb_uio *Active*

我们把eno4下电再运行绑定：
ifconfig eno4 down

#加载驱动
modprobe uio
git clone http://dpdk.org/git/dpdk-kmods
make
 rmmod igb_uio.ko
insmod igb_uio.ko
cp igb_uio.ko /lib/modules/XXX系统内核号
depmod
modprobe igb_uio

#绑网卡
 ./dpdk-devbind.py  --bind=igb_uio 0000:af:00.3

再看一下状态：
[root@bogon usertools]# ./dpdk-devbind.py --status

Network devices using DPDK-compatible driver
============================================
0000:1a:00.3 'Ethernet Connection X722 for 1GbE 37d1' drv=igb_uio unused=i40e
```

开始发包测试：

```text
单包发送
tcpreplay -o --verbose -i eno5 1.pcap
一次性全发
tcpreplay --loop -i eno5 1.pcap
```

通过tcpreplay在网卡5发包，因为网卡4和5直连，所以dpdk可以收到报文，成了~：

![img](https://pic4.zhimg.com/80/v2-664a8e67329be98f57b6a078c3b4dd9b_720w.webp)

![img](https://pic4.zhimg.com/80/v2-f4ef608f802091dfbf778651422e4fc7_720w.webp)

多线程调试

```text
 info thread
 thread 4
 set scheduler-locking on
 where
 set scheduler-locking off
```

## 8.Reference

[dpdk应用基础 唐宏 / 柴卓原 / 任平 / 人民邮电出版社 / 2016-8 / 49](https://link.zhihu.com/?target=https%3A//book.douban.com/subject/26880617/)

[深入浅出DPDK朱河清 / 梁存铭 / 胡雪焜 / 机械工业出版社 / 2016-5 / 69](https://link.zhihu.com/?target=https%3A//book.douban.com/subject/26798275/)

[小枣君：NFV和SDN之间到底有什么关系？](https://zhuanlan.zhihu.com/p/109404523)

[什么是SDN？SDN和NFV有什么区别？ - 华为](https://link.zhihu.com/?target=https%3A//info.support.huawei.com/info-finder/encyclopedia/zh/SDN.html)

[NFV基本概念_Shallow的博客-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/xili2532/article/details/97624315%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522163539163116780262572030%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D%26request_id%3D163539163116780262572030%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~hot_rank-1-97624315.pc_search_result_control_group%26utm_term%3DNFV%26spm%3D1018.2226.3001.4187)

[如何旁路内核协议栈 - rebeca8 - 博客园 (cnblogs.com)](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/zafu/p/9003392.html)

[修改CentOS默认yum源为国内yum镜像源 - Tsubasa0769 - 博客园 (cnblogs.com)](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/Tsubasa0769/p/10728161.html)

[DPDK-20.11.1版本在Centos8上安装和测试_mseaspring的博客-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/mseaspring/article/details/116178602)

[gcc 升级 9 - 大哥超帅 - 博客园 (cnblogs.com)](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/funkboy/articles/13472098.html)

[dpdk-20.11 编译和安装_choumin的专栏-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/choumin/article/details/120884484)

[Meson构建系统（一）_u010074726的博客-CSDN博客_meson](https://link.zhihu.com/?target=https%3A//blog.csdn.net/u010074726/article/details/108695256)

[【DPDK】一步一个坑：从下载到 Helloworld_Iulia-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/Q93068/article/details/110581617)

[大页内存的使用和配置_liyu123__的博客-CSDN博客_查看大页内存](https://link.zhihu.com/?target=https%3A//blog.csdn.net/liyu123__/article/details/83539348)

[DPDK应用示例指南简介(汇总) - 叨陪鲤 - 博客园 (cnblogs.com)](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/s2603898260/p/14624233.html)

[Tcpreplay与DPDK的收发包测试实验（草稿，未完整）_yeshankuangrenaaaaa的博客-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/yeshankuangrenaaaaa/article/details/83748187)

[DPDK — IGB_UIO，与 UIO Framework 进行交互的内核模块 - 不言不语技术 - 博客园 (cnblogs.com)](https://zhuanlan.zhihu.com/p/426628853/edit#IGB_UIO_8)

[DPDK网卡驱动加载、绑定和解绑_aischang-CSDN博客_dpdk解绑网卡](https://link.zhihu.com/?target=https%3A//blog.csdn.net/zhangmingcai/article/details/82423535)

[DPDK-20.11.1版本在Centos8上安装和测试 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/368045406)

[网关_百度百科 (baidu.com)](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/%E7%BD%91%E5%85%B3/98992%3Ffr%3Daladdin)

[NFV概述_嚴 帅的博客-CSDN博客_nfv是什么功能](https://link.zhihu.com/?target=https%3A//blog.csdn.net/weixin_43997530/article/details/110262287)

[DPDK系列之三：Linux UIO技术在DPDK的应用_cloudvtech的博客-CSDN博客_dpdk uio](https://link.zhihu.com/?target=https%3A//blog.csdn.net/cloudvtech/article/details/80359834%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164126003916781683920755%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D%26request_id%3D164126003916781683920755%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~baidu_landing_v2~default-4-80359834.pc_search_es_clickV2%26utm_term%3Digb_uio%26spm%3D1018.2226.3001.4187)

[异步实现方式一：异步回调_Hahaha_Val的博客-CSDN博客_异步请求回调](https://link.zhihu.com/?target=https%3A//blog.csdn.net/Hahaha_Val/article/details/79642678%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164129490716780366565805%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D%26request_id%3D164129490716780366565805%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-79642678.pc_search_es_clickV2%26utm_term%3D%E5%BC%82%E6%AD%A5%E5%9B%9E%E8%B0%83%26spm%3D1018.2226.3001.4187)

原文链接：https://zhuanlan.zhihu.com/p/426628853  原文作者：于顾而言
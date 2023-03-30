# Intel DPDK 简介（上篇）

## 1.Overview

​     Intel DPDK： Intel Data Plane Development Kit

​    [www.intel.com/go/dpdk](http://www.intel.com/go/dpdk)

- Intel 开发在IA架构上用于高性能网络数据面处理的基础库。
- 针对IA 处理器的数据包处理性能进行了大量优化。
- 绕过Linux内核，在应用层开发高性能网络包处理程序。
- DPDK不是网络协议栈，不提供二层，三层转发功能，不具备防火墙ACL功能，但通过DPDK可以轻松开发出上述功能。
- 采用BSD开源协议                                                   ![img](https://nos.netease.com/cloud-website-bucket/20180907132657256c7c5e-b7dd-40bb-adcb-0ccf5d99f240.jpg)

### 1.1Performance limitation factors

使用linux socket进行开发存在如下一些性能瓶颈：

1. cpu瓶颈：调度，上下文切换，中断，系统调用
2. 内存瓶颈：为每个packet分配释放内存，复杂的sk_buffer结构，

​     copy_from/to_user，TLB miss

1. 冗长而非必要的内核协议栈路径

下图为sk_buffer结构：                                                                     ![img](https://nos.netease.com/cloud-website-bucket/20180907132724021da70e-b2b4-4fa3-8190-c60f7340f863.jpg)



该图是linux内核协议栈草图：                             ![img](https://nos.netease.com/cloud-website-bucket/20180907132747e1020be3-9f84-4ae4-a017-d1302a86d96e.jpg)



结论：标准的数据面只是用于通用环境

### 1.2DPDK优化

1. 每个应用程序线程通过Pthead亲和性绑定到特定的硬件线程。程序在该线程所在socket上分配分配内存。

2. 重载malloc，所有内存分配基于hugepage，减少tlb miss
3. 实现对象缓冲池mempool，对象着色，确保所有对象均匀分配到cpu cache上。
4. 网络包结构采用mbuf。mbuf 在程序启动时预先从mempool中分配。
5. Lockless ring队列，所有对象均采用此设计进行连接
6. 针对Intel 网卡的Poll mode driver，无中断机制，能够高速收发报。数据包进出应用程序中实现了零拷贝。 驱动采用run-to-completion机制，数据包收发处理都在一个硬件线程上完成。    同时还支持：virtio NIC pmd，libpcap/ring based pmd link bonding pmd
7. 流分类：实现了针对Intel架构的优化(Intel SSE)查找hash /lpm/ACL 库

一个Ipv4 LPM查看仅需一次内存访问。

1. 提供QOS Framework,packets Framework.
2. The Intel ® DPDK IVSHMEM library facilitates fast zero-copy data sharing among virtual machines (host-to-guest or guest-to-guest) by means of QEUMU’s IVSHMEM mechanism.

## 2.Development

### 2.1Required：

​     kernel version >=2.6.33

​     glibc >= 2.7

​     kernel configuration :

​          Uio support

​          Hugetlbfs support

​          …

编译安装：

### 2.2目录结构：

​     app/ config/ examples/ lib/ LICENSE.GPL LICENSE.LGPL  Makefile mk/ scripts/ tools/

​     lib:Intel dpdk库源码

​     config,tools,mk,scipts:架构相关配置，makefiles，脚本

​     examples 目录提供了大量例程：           ![img](https://nos.netease.com/cloud-website-bucket/20180907132812a51e1c80-c168-4939-b3b6-399e7ce722ed.jpg)



### 2.3编译:

​     make install T=i686-native-linuxapp-gcc

​     T指定编译目标环境,格式：ARCH-MACHINE-EXECENV-TOOLCHAIN

​     ARCH can be: i686, x86_64

​     MACHINE can be: native, ivshmem

​     EXECENV can be: linuxapp, bsdapp

​     TOOLCHAIN can be: gcc, icc

### 2.4编译examples：

​     设置环境变量RTE_SDK为DPDK安装目录 RTE_TARGET指定目标环境

​     进入example目录

​     \# export RTE_SDK=$HOME/DPDK

​     \# export RTE_TARGET=x86_64-native-linuxapp-gcc

​     \# make

### 2.5驱动加载：

​     sudo modprobe uio

​     sudo insmod kmod/igb_uio.ko

### 2.6解绑默认网卡驱动

​     通过DPDK提供的工具解绑网卡默认驱动，并绑定DPDK驱动

​     ./tools/dpdk_nic_bind.py –u eth1

​     ./tools/dpdk_nic_bind.py --bind=igb_uio 04:00.1 /eth1        

### 2.7预留hugepages

​     \# echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

​     \# mkdir /mnt/huge 

​     \# mount -t hugetlbfs nodev /mnt/huge 

### 2.8开始测试

​     ./build/l2fwd [EAL options] -- -p PORTMASK [-q NQ] -T PERIOD 

​     EAL 参数:                            ![img](https://nos.netease.com/cloud-website-bucket/20180907132832ef101b92-a2d0-4956-8c06-6fcff06608e4.jpg)



例程执行：                                                                         ![img](https://nos.netease.com/cloud-website-bucket/20180907132852f2320a98-1666-4171-af44-7e3f7d323d40.gif)



## 3.Core components

环境抽象层： Environment Abstraction Layer

内存管理：      rte_malloc

Ring队列管理：    rte_ring

内存池管理：     rte_mempool

数据包缓存管理：   rte_mbuf

轮询模式网卡驱动：  PMD

算法库：       rte_hash，rte_lpm

IP stack库及工具：  rte_net

定时器、日志及调试功能 .etc      

下图是DPDK应用的架构图： 

![img](https://nos.netease.com/cloud-website-bucket/201809071329177c3ce318-c2a3-4690-9bc6-2578b8ffafa7.jpg)



各组件之间依赖关系：

![img](https://nos.netease.com/cloud-website-bucket/201809071329308ae3be23-7e6d-4f5b-9b39-2a466628eeee.jpg)

相关库的描述如下：

 ![img](https://nos.netease.com/cloud-website-bucket/20180907132942c4ed1953-afad-4910-b070-9475aa1e3c3f.jpg)

## 4.EAL: Environment Abstraction Layer 

环境适应层（EAL）提供了一套API，主要用于系统的初始化，获取核心数量和线程等配置信息， 发现外围设备互连（PCI）设备，设置超大页面，留出与高速缓存相应的内存、缓冲区和描述符环，初始化轮询模式驱动程序（PMD） ，并在其他核心上生成线程，后续任务交由这些线程去做。

EAL 层的主要工作：

\- Intel DPDK loading and launching

\- Support for multi-process and multi-thread execution types

\- Core affinity / assignment procedures

\- System memory allocation / de-allocation

\- Atomic / lock operations

\- Time reference

\- PCI bus access

\- Trace and debug functions

\- CPU feature identification

\- Interrupt handling

\- alarm operations

DPDK应用程序在开始执行时，都需要调用EAL层初始化函数：rte_eal_init(argc, argv)

该函数执行流程如下：

 ![img](https://nos.netease.com/cloud-website-bucket/20180907132956e27ac785-43ed-4ecc-9709-ecd26a75d8a6.jpg)



EAL 初始化函数中内存初始化主要工作：

**eal_hugepage_info_init()**

1. 通过/proc/mounts获取hugetlbfs mount 路径
2. 通过/sys/kernel/mm/hugepages获取hugepage信息

**rte_eal_memory_init()->**rte_eal_hugepage_init

1. 获取所有预留hugepage的物理地址并按物理地址进行排序
2. 根据物理物理地址，虚拟地址，soket_id等将hugpages组合成memseg
3. 将所有memseg信息在所有dpdk程序间共享

**rte_eal_memzone_init**

1. Memzone是内存分配的基本单元，mempool,malloc_heap在需要内存时，都会执行rte_memzone_reserve操作
2. rte_memzone_reserve 从memseg中分配一块内存出来

DPDK应用程序所分配的内存都是在hugepage上分配，以上函数通过hugetlbfs,将预留的hugepage按照物理地址连续性，是否在相同socket等性质，组装成 memseg，后续当应用程序使用内存时，就会从这些memseg中，根据需要分配内存的大小来划分一块称为 memzone的区域，供应用程序使用。
DPDK应用程序主要的内存分配操作有两个，一个是通过malloc函数分配内存，另一种是分配mbuf。两者底层都使用了memzone。

![img](https://nos.netease.com/cloud-website-bucket/20180907133011c9a7ffae-5043-4c2a-ab50-17b83e42189c.jpg)

## 5.rte_malloc

提供malloc-like API在memzone上分配任意大小的内存。DPDK应用程序分配内存时，不是在程序栈上分配，而是通过其自身实现的malloc，在hugepage上分配。

接口：

void * rte_malloc(const char *type, size_t size, unsigned align)

void * rte_malloc_socket(const char *type, size_t size, unsigned align, int socket)

1. 分配的内存对其到align*2^n
2. 前者在当前core所在的socket的local memory上分配内存，后者可以指定的socket上分配。
3. malloc_heap 维护当前free space
4. malloc_elem 跟踪内存划分出的内存元素
5. 分配时，从一开始的memzone 分裂出两个malloc_elem，返回尾部elem

​     split_elem(elem, new_elem)

1. 释放时，会检查前后是否都已经free，如果是则合并

​     join_elem(elem, next)

​    

该机制对内存的管理类似内核伙伴关系算法，freehead用于管理空闲空间，按空闲空间大小分为不同等级。当需要内存时，就从适合的级别寻找空闲空间，从尾部将该空间分为两部分，后半部供应用程序使用，前半部放入级别较低的空心空间队列。 当释放内存时，会顺便检查其伙伴是否已经空闲，如果是，则将其合并，放入级别更高的空闲列表。

![img](https://nos.netease.com/cloud-website-bucket/20180907133027c897b705-6a04-4858-8592-e1f136dfa4ac.jpg)







原文链接：https://sq.sf.163.com/blog/article/196389338897940480  原文作者：猪小花1号
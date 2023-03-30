# Intel DPDK 简介（下篇）

## 1.rte_ring

DPDK应用使用ring来管理各种对象。ring是一个FIFO队列，大小固定，采用无锁机制。

可以批量进出队列。具体描述如下：

- FIFO
- Maximum size is fixed, the pointers are stored in a table
- Lockless implementation
- Multi-consumer or single-consumer dequeue
- Multi-producer or single-producer enqueue
- Bulk dequeue - Dequeues the specified count of objects if successful; otherwise

​     fails

- Bulk enqueue - Enqueues the specified count of objects if successful; otherwise

​     fails

- Burst dequeue - Dequeue the maximum available objects if the specified count

​     cannot be fulfilled

- Burst enqueue - Enqueue the maximum available objects if the specified count

​     cannot be fulfilled

实现原理：

​     以如队为例，对ring的操作需要在一个循环里。

​     1 prod_head = ring->prod_head 首先将ring结构中的prod_head保存到本地。

​     2 然后执行CAS指令：Compare and swap(CAS) : 如果本地prod_head不等于ring_prod_head,失败，回到1，失败说明之前正有其它线程也在操作该队列，且先成功执行了CAS。

​     3 否则，将ring->prod_head设置为下一个元素的位置prod_next 该设置过程是CAS的一部分，CAS执行成功，就会原子的更改ring结构中的prod_head.这样如果有另一个线程执行CAS操作，就会失败。执行成功的线程就可以将数据放到它本地prod_head 到prod_next之间了。

![img](https://nos.netease.com/cloud-website-bucket/20180907133056dc726055-59ad-4375-8d20-9b9c33939886.jpg)



有两个线程都获取了ring的prod_head.

![img](https://nos.netease.com/cloud-website-bucket/20180907133105417ce5cd-c081-49a2-9b03-c96e92b365c5.jpg)



core1 CAS成功，ring结构的prod_head变为了core1的prod_next.

core2失败，重新执行步骤1，此时它获取到的 prod_head就跟core1不同了。



 ![img](https://nos.netease.com/cloud-website-bucket/20180907133209e159408e-b71e-4149-a134-d6782cef0e24.jpg)

core1 将自己的数据放入prod_head和prod_next之间，如 obj4，没有竞争。

core2 放入它自己的prod_head和prod_next之间，如obj5，也没有竞争。

![img](https://nos.netease.com/cloud-website-bucket/2018090713320145976d9e-4544-4b26-9222-12e25e7fd34c.jpg)



当队列元素修改结束，需要修改prod_tail 来表示队列的结束位置。

比较线程本地prod_head与ring结构中的prod_tail，相等方可移动prod_tail。

![img](https://nos.netease.com/cloud-website-bucket/20180907133152246ac42f-15d8-45eb-ad5b-239fafcffa04.jpg)

也就是core1现移动，core2后移动。该过程当然也是循环操作，如果core2

先修改好了obj5,它会一直循环，直到ring的prod_tail与其prod_head相等。

## 2.rte_mempool

mempool是固定大小对象的分配器，类似linux内核slab。将很多对象比如mbuf，缓存在队列中，分配是从pool中取出使用，用完后在归还给pool。

- 从EAL memzone分配内存，创建对象数目固定的pool。
- 不支持resizing。
- 采用rte_ring管理，支持lockless。
- Per-core cache。



![img](https://nos.netease.com/cloud-website-bucket/20180907133230ce3d654d-2e94-4a57-9c81-a440e6c4ba71.jpg)

mempool有两种队列，首先per-cpu队列，不同线程在自己核心处理自己的队列，互不干涉。当per-cpu队列满了，需要放回pool中时，采用ring的bulk入队机制。所以效率是很高的。

## 3.Poll Mode Driver

DPDK PMD采用UIO机制，其特点概括来说：用户层驱动、轮询、0拷贝。

详细特性描述：

- The Intel ® DPDK includes 1 Gigabit, 10 Gigabit and 40 Gigabit and para virtualized virtio Poll Mode Drivers.
- a PMD accesses the RX and TX descriptors directly without any interrupts (with the exception of Link Status Change interrupts) to quickly receive, process and deliver packets in the user‘s application.
- To avoid any unnecessary interrupt processing overhead, the execution environment must not use any asynchronous notification mechanisms.
- Avoiding lock contention is a key issue in a multi-core environment. To address this issue, PMDs are designed to work with per-core private resources as much as possible.
- Non-Uniform Memory Access (NUMA)
- Enable the rte_eth_tx_burst/ rte_eth_rx_burst function to take advantage of burst-oriented hardware features (prefetch data in cache, use of NIC head/tail registers)
- Vector PMD uses Intel ® SIMD instructions to optimize packet I/O
- Support SR-IOV mode and VMDq mode of operation in a virtualized environment
- Poll Mode Driver for Emulated Virtio NIC
- A libpcap-based PMD (librte_pmd_pcap) that reads and writes packets using libpcap, - both from files on disk, as well as from physical NIC devices using standard Linux kernel drivers.
- In addition to Poll Mode Drivers (PMDs) for physical and virtual hardware, Intel® DPDK also includes a pure-software library that allows physical PMD’s to be bonded together to create a single logical PMD.Currently the Link Bonding PMD library supports 4 modes of operation： Round-Robin (Mode 0)， Active Backup (Mode 1)， Balance XOR (Mode 2)， Broadcast (Mode 3)

PMD 采用*UIO*（Userspace I/O）用户空间的I/O技术。驱动内核部分负责将网卡PCI资源映射到用户空间，驱动用户空间部分，负责轮询网卡PCI配置空间内容，进行数据包的收发工作。由于PCI配置空间内容都映射到了用户空间，所以用户空间驱动可以通过网卡寄存器状态来判断是否有数据包到来等。

## 4.Flow Classification

DPDK 支持常用的流分类算法，如hash,LPM。而且针对Intel CPU的特性进行了优化。

PMD将数据包收集到用户态，用户态通过这些算法，处理自己感兴趣的数据包。针对不同需求，所使用的算法也不相同。l2转发一般会用到hash算法，而L3则一般会需要LPM算法。

-   The Intel® DPDK provides a Hash Library for creating hash table for fast lookup. hashes: [hash](http://dpdk.org/doc/api/rte__hash_8h.html), [jhash](http://dpdk.org/doc/api/rte__jhash_8h.html), [FBK hash](http://dpdk.org/doc/api/rte__fbk__hash_8h.html), [CRC hash](http://dpdk.org/doc/api/rte__hash__crc_8h.html)
-   The Intel ® DPDK LPM library component implements the Longest Prefix Match (LPM) table search method for 32-bit keys that is typically used to find the best route match in IP forwarding applications
-   The Intel® DPDK provides an Access Control library that gives the ability to classify an input packet based on a set of classification rules.

## 5.Qos Framework

DPDK提供了Qos框架。

![img](https://nos.netease.com/cloud-website-bucket/2018090713324668802006-c2a7-4292-a606-ecde99cf545d.jpg)



该框架将数据包的处理抽象成一个pipeline。数据从管道一端流入，经过各种Block的处理后，从另一端流出。这些block所应用到的算法可能是前边讲的flow Classification算法，也可能是一些负载均衡算等。简要描述如下：

 ![img](https://nos.netease.com/cloud-website-bucket/2018090713325848b1f967-eda3-439c-970a-2d8b52a1f1e6.jpg)

## 6.DPDK性能

## ![img](https://nos.netease.com/cloud-website-bucket/201809071333097c93de50-742a-42e8-9651-2ca3f91791a0.jpg)

## 7.Extension

扩展介绍了一些已经开始使用DPDK的项目或公司简介。

### 7.1DPDK-OVS

基于dpdk的Open vSwitch。

1. Standard Open vSwitch has control plane (ovs-vswitchd and ovsdb-server) running in user space, while data plane stays in kernel.
2. It is a fork of Open vSwitch with data plane implemented in user space with DPDK frame work. It also has a modified QEMU which supports DPDK version of vHost implementation and Inter-VM shared Memory (IVSHMEM) PCI device.
3. There are two approaches for using DPDK acceleration in DPDK. One is the openvswitch fork from intel, called dpdk-ovs the other is done directly in openvswitch with a different approach from intel.

参考链接:

  -http://git.openvswitch.org/cgi[-](http://git.openvswitch.org/cgi-bin/gitweb.cgi?p=openvswitch;a=blob_plain;f=INSTALL.DPDK;hb=HEAD)[bin/gitweb.cgip=openvswitch;a=blob_plain;f=INSTALL.DPDK;hb=HEAD](http://git.openvswitch.org/cgi-bin/gitweb.cgi?p=openvswitch;a=blob_plain;f=INSTALL.DPDK;hb=HEAD)

  \- https://github.com/01org/dpdk-ovs

  -https://www.mail-archive.com/discuss@openvswitch.org/msg10957.html

Ovs架构：

![img](https://nos.netease.com/cloud-website-bucket/201809071333297185e9a7-7053-460b-974d-1325cdb09077.jpg)

使用DPDK后，所有的OVS模块都在用户态实现：



 ![img](https://nos.netease.com/cloud-website-bucket/201809071333359d281442-9a77-4729-9b50-13d1d1dba5c1.jpg)

### 7.2风河流量分析引擎

将数据包从输入数据包流中提取出来，并使用数据流类库将其分为不同的流量。流分析引擎还可用来识别与单个数据包和流量通信有关的协议和应用程序。流分析引擎的流量信息被转发到其它 INP 内部或外部网络单元，并可用于优先化与高价值应用或用户相关联的流量。

![img](https://nos.netease.com/cloud-website-bucket/20180907133347c122e0b3-7517-4f98-ad4d-46fb88430c03.jpg)





### 7.3腾讯宙斯盾团队

1. 在软件层面看来，首先要解决的就是把所有流量都收上来，采用操作系统标准的协议栈显然是不现实的，在不同的时期，宙斯盾团队先后采用了内核挂钩，libpcap，libpfring，专用硬件以及DPDK的解决方案。![img](https://nos.netease.com/cloud-website-bucket/2018090713340794014a93-3c1e-488c-bec2-37b01518a0d4.jpg)



1. 大眼系统能有效检测四层DDoS攻击和七层DDoS攻击

![img](https://nos.netease.com/cloud-website-bucket/201809071334150748300e-2556-468d-bf4b-46116b6c8540.jpg) ~~~~![img](https://nos.netease.com/cloud-website-bucket/201809071334164bcb18ea-5aff-4b67-8107-8f70b9bf5544.jpg)~~~~

 

1. http://security.tencent.com/index.php/blog/msg/40
2. http://security.tencent.com/index.php/blog/msg/67







原文链接：https://sq.sf.163.com/blog/article/196390567405387776  原文作者：猪小花1号
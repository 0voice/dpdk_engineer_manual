DPDK内核NIC接口

# DPDK内核NIC接口

## 一. 缘由

还是继续学习DPDK相关知识点，了解一下DPDK的虚拟设备的用法和实现。

## 二. 介绍

DPDK内核NIC接口（KNI）允许用户态应用程序访问linux*控制面板。

DPDK KNI的优势：

a.比Linux TUN/TAP接口要快（通过消除系统调用copy_to_user()/copy_from_user()）

b.允许用ethtoll,ifconfig和tcpdump管理DPDK端口

c.允许与内核协议栈交互

使用DPDK内核NIC接口的应用程序的组件如下所示：

![img](https://pic2.zhimg.com/80/v2-3775f8e9783691efbcc78f4d87f90cd9_720w.webp)

### 1. DPDK KNI内核模块

这个KNI内核加载模块提供两种类型的设备:

KNI设备：

--创建网络设备（通过ioctl调用）

--维护所有KNI实例共享的内核线程上下文（模拟网络驱动程序的RX端）

--对于单个内核线程模式，在所有KNI实例之间位置一个内核线程上下文

--对于多内核线程模式，为每个KNI实例位置一个内核线程上下文；

网络设备：

--通过实现由struct net_device定义的诸如netdev_ops，header_ops，ethtool_ops之类的几个操作来提供Net功能，包括支持DPDK mbufs和FIFO。

--这个结构名由用户空间提供。

--这个MAC地址可以是真实NIC MAC，也可以随机生成。

### 2. KNI创建和销毁

KNI接口通过DPDK应用程序动态创建。接口名称和FIFO细节由应用程序调用ioctl调用提供（rte_kni_device_info结构）：

--接口名称

--FIFO中与内存区域相一致的物理地址

--Mbuf 内存池的细节，物理地址和虚拟地址（通过计算Mbuf指针偏移计算）

--PCI 信息

--CPU 核亲和性

rte_kni_common.h源码中有相关细节。

这物理地址将重新映射到内核地址空间并且与KNI上下文分开存储。

内核RX线程（单线程和多线程模式）的亲和力由force_bind和core_id配置参数控制。

这个KNI接口能够在DPDK应用创建后动态的删除。此外，所有未删除的KNI接口将在其他设备的释放操作中被删除（当DPDK应用程序关闭时）。

### 3. DPDK mbuf流

为了让最少的代码运行在内核空间，mbuf内存池仅仅通过用户空间去管理。内核模块能处理mbufs，但是所有的分配和释放操作将仅仅由DPDK应用操作处理。

![img](https://pic2.zhimg.com/80/v2-ec5eb11986a376fe0d98bdf9b9f984e5_720w.webp)

### 4. 案例：包进入

在DPDK RX侧，在RX线程上下文通过PMD去分配mbuf。这个线程在rx_q(FIFO)入队mbuf。KNI线程将轮询rx_q的所有KNI活动设备。如果一个mbuf出队，mbuf将转化为sk_buff，然后通过netif_rx()发送给网络协议栈。这个出队mbuf必须被释放，因此mbuf同样的指针将放入free_q（FIFO）队列。

### 5. 案例：包出去

对预DPDK应用出队必须首先出现一些mbufs队列，然后再内核侧创建mbuf cache。数据包是从linux 网络协议栈中接收，通过调用 kni_net_tx() 回调。这个mbuf出队（因为是cache，不用等待）然后填空数据到sk_buff。这个sk_buff要释放，并且这个mbuf入tx_q FIFO队列。

DPDK TX线程出队mbuf，然后发送到PMD（通过 rte_eth_tx_burst()）。然后将mbuf放回cache。

### 6. Ethtool

Ethtool是一个linux指定的工具，每个网络设备会支持相关回调函数的操作，当前用igb/ixgb修改linux驱动去实现ethtool。ethtool不支持i40e和vms。

### 7. Link状态和MTU改变

链路状态和MTU盖面是网络接口指定的操作，通过ifconfg。这个请求的初始化在内核侧完成（ifconfig进程的上下文）而相关处理由用户态空间应用处理。这个应用polls这个请求，调用这个应用程序处理，然后返回相应到内核空间。

应用处理程序可以在创建界面时注册，也可以在运行时明确注册/未注册。他提供多进程方案的灵活性（KNI在主进程中创建，但回调在次要进程中处理）。 约束是单个进程可以注册和处理请求。

原文链接：[https://blog.csdn.net/pangyemen](
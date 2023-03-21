# DPDK Mempool 库原理（学习笔记） 

# 1 前置知识点学习（了解）

从CPU到实际的存储节点，依据层级划分：Channel > DIMM > Rank > Chip > Bank > Row /Column

[![clipboard](https://img2020.cnblogs.com/blog/1926214/202005/1926214-20200517030823911-869650075.png)](https://img2020.cnblogs.com/blog/1926214/202005/1926214-20200517030823042-149025103.png)

## 1.1 channel

CPU到内存的通路是channel，每个channel对应一个CPU的内存控制器，每个channel可以配有多个DIMM。

双通道：CPU外核或北桥有两个内存控制器，每个控制器控制一个内存通道。理论上内存带宽增加一倍。

四通道同理。

## 1.2 DIMM

全称Dual-Inline-Memory-Modules（双列直插式存储模块），是目前最常见的内存模块（ 可以理解为内存条）。

以前的主机是直接将存储芯片（chip）插在主板上的，然后发展出SIMM（Single In-line Memory Module），将多个chip焊在一片电路板上，成为内存模块，再将它插到主板上。

## 1.3 Rank

DIMM上一部分或所有chip组成一个rank（64bit），因此内存至少需要有16片4bit的chip或者8bit的chip（不存在4bit和8bit芯片混搭的情况）。

内存控制器只允许CPU每次与内存进行一组64bits的数据交换，对应的就是一个rank。rank也可以理解为连接到同一个CS（chip select）的一组chip。

rank分类：

- Single-Rank（1R），要动用到DIMM上所有的chip，这些chip由同一个片选信号控制。
- Double-Rank（2R），产生2个64位rank，由2个片选信号控制，这2个片选信号是交错的，不争抢内存总线。
- Quad-Rank（4R），产生4个64位rank,由4个片选信号控制，这4个片选信号是交错的，不争抢内存总线。

在地址选择时，只有当片选信号有效时，此片所连的地址线才有效。

## 1.4 chip

内存条上的黑色芯片就是chip，提供4bit/8bit/16bit/32bit的数据，提供4bit的芯片记作x4，提供8bit的芯片记作x8。

## 1.5 其他

再往下的bank、Row /Column这里可以暂时不用关心了，通过上文的示意图中了解一下就行。

# 2 DPDK Mempool 库

内存池是一个具有固定大小的对象分配器。 在DPDK中，它由名称唯一标识，并且使用mempool handler来存储空闲对象。 默认的mempool handler是基于ring的。它提供了一些可选的服务，例如“per-core缓存”和“内存对齐”，内存对齐能确保对象被填充，以在所有DRAM或DDR3通道上均匀分布。

这个库由 Mbuf Library 使用。

## 2.1 Cookies保护字段

在调试模式中(CONFIG_RTE_LIBRTE_MEMPOOL_DEBUG is enabled)，将在块的开头和结尾处添加cookies。 分配的对象包含保护字段，以帮助调试缓冲区溢出。

## 2.2 Stats统计信息

在调试模式中(CONFIG_RTE_LIBRTE_MEMPOOL_DEBUG is enabled)，从池中获取、释放的统计信息存放在mempool结构体中。 为了避免并发访问统计计数器，统计信息是per-lcore的。

## 2.3 内存对齐约束

根据X86架构上的硬件内存配置，可以通过在对象之间添加特定的填充来极大地提高性能。目的是确保每个对象的起始位置被均匀的分布在不同的channel和rank上，以便实现所有通道的负载均衡。

当执行L3转发或流分类时，对于包缓冲区尤其如此。只访问前64个字节，因此可以通过将对象的开始地址分布在不同的通道中来提高性能。

DIMM上的rank数目是可访问DIMM完整数据位宽的独立DIMM集合的数量。 由于他们共享相同的路径，因此rank不能被同时访问。 DIMM上的DRAM芯片的物理布局不一定与rank数目相关。

当运行app时，EAL命令行选项提供了添加内存通道和rank数目的能力。

注：命令行必须始终指定处理器的内存通道数目。

不同DIMM架构的对齐示例如下两张图所示 。在例子中，我们假设包是16个64字节的块（在实际应用中这是不正确的）。

例1：Two Channels and Quad-ranked DIMM Example

![img](https://dpdk-docs.readthedocs.io/en/latest/_images/memory-management.svg)

例2：Three Channels and Two Dual-ranked DIMM Example

Intel® 5520芯片组有三个通道，因此，在大多数情况下，对象之间不需要填充。(除了大小为n x 3 x 64B的块)

![img](https://dpdk-docs.readthedocs.io/en/latest/_images/memory-management2.svg)

当创建一个新池时，用户可以指定使用此功能。

> 我的疑问：
>
> 这里的例子是从dpdk官网拷贝过来的，我有个疑问，如果有谁知道麻烦给我留言。
>
> 例2中的pkt0根据图上看明明是starts at channel 0， rank0，为什么图上标注的是rank1？如果从图上看，这里只保证了起始于不同channel，但仍是同一个rank而且是同一个DIMM。根据原文描述，pkt只要不是n x 3 x 64B的大小就不需要填充，感觉只是保证起始在不同channel就可以了。这个问题搜索了很久没有满意的答案。

## 2.4 本地缓存

在CPU使用率方面，由于每个访问需要compare-and-set (CAS)操作，所以多核访问内存池的空闲缓冲区成本比较高。 为了避免对内存池ring的访问请求太多，内存池分配器可以维护per-core cache，并通过实际内存池中具有较少锁定的缓存对内存池ring执行批量请求。 通过这种方式，每个core都可以访问自己空闲对象的缓存（带锁）， 只有当缓存填充时，内核才需要将某些空闲对象重新放回到缓冲池ring，或者当缓存空时，从缓冲池中获取更多对象。

虽然这意味着一些buffer可能在某些core的缓存上处于空闲状态，但是core可以无锁访问其自己的缓存提供了性能上的提升。

缓存由一个小型的per-core表及其长度组成。可以在创建池时启用/禁用此缓存。

缓存大小的最大值是静态配置，并在编译时定义的(CONFIG_RTE_MEMPOOL_CACHE_MAX_SIZE)。

![img](https://dpdk-docs.readthedocs.io/en/latest/_images/mempool.svg)

不同于per-lcore内部缓存，应用程序可以通过接口 rte_mempool_cache_create() ， rte_mempool_cache_free() 和 rte_mempool_cache_flush() 创建和管理外部缓存。 这些用户拥有的缓存可以被显式传递给 rte_mempool_generic_put() 和 rte_mempool_generic_get() 。 接口 rte_mempool_default_cache() 返回默认内部缓存。 与默认缓存相反，用户拥有的高速缓存可以由非EAL线程使用。

## 2.5 Mempool handlers

这允许外部存储子系统，如外部硬件存储管理系统和软件存储管理与DPDK一起使用。

mempool handler包括两方面：

- 添加新的mempool操作代码。这是通过添加mempool ops代码，并使用 MEMPOOL_REGISTER_OPS 宏来实现的。
- 使用新的API调用 rte_mempool_create_empty() 及 rte_mempool_set_ops_byname() 用于创建新的mempool，并制定用户要使用的操作。

在同一个应用程序中可能会使用几个不同的mempool处理。 可以使用 rte_mempool_create_empty() 创建一个新的mempool，然后用 rte_mempool_set_ops_byname() 将mempool指向相关的 mempool处理回调（ops）结构体。

传统的应用程序可能会继续使用旧的 rte_mempool_create() API调用，它默认使用基于ring的mempool处理。 这些应用程序需要修改为新的mempool处理。

对于使用 rte_pktmbuf_create() 的应用程序，有一个配置设置(RTE_MBUF_DEFAULT_MEMPOOL_OPS)，允许应用程序使用另一个mempool处理。

## 2.6 用例

需要高性能的所有分配器应该使用内存池实现。 以下是一些使用实例：

- Mbuf Library
- Environment Abstraction Layer
- 任何需要在程序中分配固定大小对象，并将被系统持续使用的应用程序



**引用参考资料：**

1）dpdk官方文档：http://doc.dpdk.org/guides-20.02/prog_guide/mempool_lib.html

2）RAM结构与原理：https://www.techbang.com/posts/18381-from-the-channel-to-address-computer-main-memory-structures-to-understand

3）DDR扫盲——single rank与dual-rank：https://www.sohu.com/a/168446287_781333



原创作者 [一觉醒来写程序](https://home.cnblogs.com/u/realjimmy/) 本文地址：https://www.cnblogs.com/realjimmy/p/12903372.html
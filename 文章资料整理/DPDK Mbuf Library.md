# DPDK Mbuf Library

## 1.Mbuf库

Mbuf库提供了分配和释放缓冲区（mbufs）的功能，DPDK应用程序可以使用这些mbufs来存储消息缓冲。 消息缓冲存储在内存池中，使用Mempool库。

数据结构rte_mbuf通常用来承载网络数据包buffers，但它实际上可以是任何数据(控制数据、事件……)。rte_mbuf头部结构尽可能小，目前仅使用个cache line，最常用的字段位于第一个cache line。

## 2.数据包缓冲区的设计

为了存储数据包数据（包括协议头），考虑了两种方法：

- 在单个存储buffer中嵌入元数据，后面跟着数据包数据固定大小区域
- 为元数据和数据包数据分别使用独立的存储buffer。

第一种方法的优点是它只需要一个操作即可分配/释放数据包的整个内存。 另一方面，第二种方法更加灵活，可以将元数据结构的分配与数据包数据的buffer分配完全分开。

DPDK选择了第一种方法。 元数据包含控制信息，例如消息类型，长度，到数据开头的偏移量以及用于允许缓冲区链接的附加mbuf结构的指针。

用于承载网络数据包的消息缓冲可以处理需要由多个缓冲区存储来保证据包完整的情况。许多mbuf组成的巨型帧就是这种情况，这些mbuf通过它们的next字段链接在一起。

对于新分配的mbuf，消息缓冲区中数据开始的区域为缓冲区开始之后的RTE_PKTMBUF_HEADROOM字节，该字节与缓存对齐。 消息缓冲区可用于在系统中不同实体之间承载控制信息、数据包、事件等。 消息缓冲也可以使用其buffer指针来指向其他消息缓冲或其他数据结构。

例1：只有一段的mbuf

![image-20221216152914389](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221216152914389.png)

例2：有三段的mbuf

![image-20221216152935987](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221216152935987.png)

Buffer管理器实现了一组相当标准的Buffer访问功能来操纵网络数据包。

## 3.存储在内存池中的缓冲区

缓冲区管理器使用Mempool库来申请buffer。 因此它确保数据包头均衡分布在channels和ranks上并进行L3处理（channel和rank的概念查看mempool库里的介绍）。 mbuf包含一个字段，用于表示它从哪个池中申请出来。当调用rte_pktmbuf_free（m）时，mbuf将被其所在的内存池回收。

## 4.构造函数

数据包mbuf构造函数由API提供。rte_pktmbuf_init()函数初始化mbuf结构中的某些字段，一旦创建，这些字段就不会被用户修改（mbuf类型，源池，缓冲区起始地址等）。此函数在池创建时作为rte_mempool_create()函数的回掉函数给出。

## 5.分配和释放mbufs

分配一个新mbuf需要用户指定从哪个池中申请。对于任何新分配的mbuf，它包含一个长度为0的段。缓冲区到数据的偏移量被初始化，以便使得buffer具有一些字节headroom（RTE_PKTMBUF_HEADROOM字节）。

释放mbuf意味着将其返回到其所在的内存池。当mbuf的内容存储在一个池中（作为一个空闲的mbuf）时，mbuf的内容不会被修改。 由构造函数初始化的字段不需要在mbuf分配时重新初始化。

释放包含多个段的数据包mbuf时，所有这些段都会被释放并返回其所在的内存池。

## 6.操作mbufs

该库提供了一些用于处理数据包mbuf中的数据的功能。 例如：

- 获取数据长度
- 获取指向数据开始的指针
- 在数据之前添加数据
- 数据后追加数据
- 删除缓冲区开头的数据（rte_pktmbuf_adj（））
- 删除缓冲区末尾的数据（rte_pktmbuf_trim（））有关详细信息，请参阅《 DPDK API参考》。

## 7.元数据

部分信息由网络驱动程序检索并存储在mbuf中，以使得处理更加简单。 例如，VLAN，RSS哈希结果（请参阅轮询模式驱动程序）和一个指示校验和是否由硬件计算的标志。

mbuf还包含输入端口（它来自何处）以及链中mbuf段的数量。

对于链接的mbuf，只有链的第一个mbuf存储此元信息。

例如，IEEE1588数据包的时间戳机制，VLAN标记和IP校验和计算在RX端就是这种情况。

在TX端，如果应用程序支持，则也可以将某些处理委托给硬件。 例如，PKT_TX_IP_CKSUM标志允许卸载IPv4校验和的计算。

以下示例说明了如何在封装了vxlan的tcp数据包上配置不同的TX卸载：out_eth/out_ip/out_udp/vxlan/in_eth/in_ip/in_tcp/payload

- **计算out_ip的校验和:**

```
mb->l2_len = len(out_eth)
mb->l3_len = len(out_ip)
mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM
set out_ip checksum to 0 in the packet
```

配置DEV_TX_OFFLOAD_IPV4_CKSUM支持在硬件计算。

- **计算out_ip 和 out_udp的校验和:**

```
mb->l2_len = len(out_eth)
mb->l3_len = len(out_ip)
mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM | PKT_TX_UDP_CKSUM
set out_ip checksum to 0 in the packet
set out_udp checksum to pseudo header using rte_ipv4_phdr_cksum()
```

配置DEV_TX_OFFLOAD_IPV4_CKSUM 和 DEV_TX_OFFLOAD_UDP_CKSUM支持在硬件上计算。

- **计算in_ip的校验和:**

```
mb->l2_len = len(out_eth + out_ip + out_udp + vxlan + in_eth)
mb->l3_len = len(in_ip)
mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM
set in_ip checksum to 0 in the packet
```

这以情况1类似，但是l2_len不同。 配置DEV_TX_OFFLOAD_IPV4_CKSUM支持硬件计算。 注意，只有外部L4校验和为0时才可以工作。

- **计算in_ip 和 in_tcp的校验和:**

```
mb->l2_len = len(out_eth + out_ip + out_udp + vxlan + in_eth)
mb->l3_len = len(in_ip)
mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM | PKT_TX_TCP_CKSUM
在报文中设置in_ip校验和为0
使用rte_ipv4_phdr_cksum()将in_tcp校验和设置为伪头
```

这与情况2类似，但是l2_len不同。 配置DEV_TX_OFFLOAD_IPV4_CKSUM 和 DEV_TX_OFFLOAD_TCP_CKSUM支持硬件实现。 注意，只有外部L4校验和为0才能工作。

- **segment inner TCP:**

```
mb->l2_len = len(out_eth + out_ip + out_udp + vxlan + in_eth)
mb->l3_len = len(in_ip)
mb->l4_len = len(in_tcp)
mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CKSUM | PKT_TX_TCP_CKSUM | PKT_TX_TCP_SEG;
在报文中设置in_ip校验和为0
将in_tcp校验和设置为伪头部，而不使用IP载荷长度
```

配置DEV_TX_OFFLOAD_TCP_TSO支持硬件实现。 注意，只有L4校验和为0时才能工作。

- **计算out_ip, in_ip, in_tcp的校验和:**

```
mb->outer_l2_len = len(out_eth)
mb->outer_l3_len = len(out_ip)
mb->l2_len = len(out_udp + vxlan + in_eth)
mb->l3_len = len(in_ip)
mb->ol_flags |= PKT_TX_OUTER_IPV4 | PKT_TX_OUTER_IP_CKSUM  | PKT_TX_IP_CKSUM |  PKT_TX_TCP_CKSUM;
设置 out_ip 校验和为0
设置 in_ip 校验和为0
使用rte_ipv4_phdr_cksum()设置in_tcp校验和为伪头部
```

配置DEV_TX_OFFLOAD_IPV4_CKSUM, DEV_TX_OFFLOAD_UDP_CKSUM 和 DEV_TX_OFFLOAD_OUTER_IPV4_CKSUM支持硬件实现。

Flage标记的意义在mbuf API文档(rte_mbuf.h)中有详细描述。 更多详细信息还可以参阅testpmd 源码(特别是csumonly.c)。

## 8.直接及间接 Buffers

直接缓冲区是完全独立且独立的缓冲区。 间接缓冲区的行为类似于直接缓冲区，但事实上，缓冲区指针和其中的数据偏移量引用了另一个直接缓冲区中的数据。 这在需要复制或分段数据包的情况下很有用，因为间接缓冲区提供跨越多个缓冲区重用相同数据包数据的手段。

当使口 rte_pktmbuf_attach() 函数将缓冲区附加到直接缓冲区时，该缓冲区变成间接缓冲区。每个缓冲区有一个引用计数器字段，每当直接缓冲区附加一个间接缓冲区时，直接缓冲区上的应用计数器递增。 类似的，每当间接缓冲区被分裂时，直接缓冲区上的引用计数器递减。 如果生成的引用计数器为0，则直接缓冲区将被释放，因为它不再使用。

处理间接缓冲区时需要注意几件事情。 首先，间接缓冲区从不附加到另一个间接缓冲区。 尝试将缓冲区A附加到间接缓冲区B（且B附加到C上了），rte_pktmbuf_attach()将自动把A附加到C上，从而有效地克隆了B。其次，为了使缓冲区变成间接缓冲区，其引用计数必须等于1，也就是说它不能被另一个间接缓冲区引用。最后，不能将间接缓冲区重新链接到直接缓冲区（除非这个间接缓冲区已经被分离了）。

虽然可以使用推荐的rte_pktmbuf_attach（）和rte_pktmbuf_detach（）函数直接调用附加/分离操作， 但建议使用更高级的rte_pktmbuf_clone（）函数，该函数负责正确初始化间接缓冲区，并可以进行克隆具有多个段的缓冲区。

由于间接缓冲区不应该实际保存任何数据，因此应该配置间接缓冲区的内存池以指示减少内存消耗。可以在几个示例应用程序中找到用于间接缓冲区的内存池初始化示例（以及用于间接缓冲区的用例示例），例如IPv4组播示例应用程序。

## 9.调试

在调试模式 (CONFIG_RTE_MBUF_DEBUG使能)下，mbuf库的功能在任何操作之前执行完整性检查(如缓冲区检查、类型错误等)。

## 10.用例

所有网络应用程序都应该使用mbufs来传输网络数据包。



**参考：**

dpdk官方编程指南：http://doc.dpdk.org/guides-20.02/prog_guide/mbuf_lib.html



原创：[一觉醒来写程序](https://home.cnblogs.com/u/realjimmy/)   本文链接：https://www.cnblogs.com/realjimmy/p/12914237.html


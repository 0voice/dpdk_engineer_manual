# 数据平面开发套件（DPDK）中的Vhost / Virtio的配置和性能

## 1.Vhost / Virtio简介

**Vhost / virtio是一种半虚拟化的设备抽象接口规范，已广泛应用于QEMU \*和基于内核的虚拟机（KVM）。当它在来宾操作系统中用作前端驱动程序时，通常称为virtio；当在主机中用作后端驱动程序时，则称为vhost。与在主机上进行纯软件输入/输出（I / O）仿真相比，virtio可以实现更好的性能，并广泛用于数据中心。Linux \*内核提供了相应的设备驱动程序，分别是virtio-net和vhost-net。**

![img](https://pic4.zhimg.com/80/v2-194b17fbd979fe66be4fdb762dfd55db_720w.webp)

virtio-net和vhost-net

**为了帮助提高数据吞吐性能，**[数据平面开发套件（DPDK）](https://link.zhihu.com/?target=http%3A//dpdk.org/)提供了用户状态轮询模式驱动程序（PMD）virtio-pmd和用户状态实现vhost-user。

![img](https://pic1.zhimg.com/80/v2-804a19982abd88a74af60057dd8d91f4_720w.webp)

用户状态轮询模式驱动程序（PMD）virtio-pmd和用户状态实现vhost-user



本文将介绍如何使用DPDK代码示例配置和使用vhost / virtio，**testpmd**。列出了每个虚拟主机/虚拟Rx / Tx路径的性能编号。



![img](https://pic1.zhimg.com/80/v2-252c40f4d1186f089a2798775de58ff0_720w.webp)

virtio的典型应用场景





![img](https://pic1.zhimg.com/80/v2-353dde6222b474d68863ef97dd81e840_720w.webp)

virtio的典型应用场景



## 2.接收和发送路径

在DPDK的vhost / virtio中，为不同的用户场景提供了三个Rx（接收）和Tx（发送）路径。可合并路径是为大数据包Rx / Tx设计的，是纯I / O转发的向量路径，如果没有给出参数，则不可合并路径是默认路径。

## 3.**可合并路径**

使用此接收路径的优势在于，虚拟主机可以将可用环中的独立mbuf组织到一个链表中，以接收较大大小的数据包。这是最广泛采用的路径，也是最近几个月DPDK开发团队的性能优化重点。该路径使用的接收和发送功能配置如下：



1. eth_dev->tx_pkt_burst = &virtio_xmit_pkts;
2. eth_dev->rx_pkt_burst = &virtio_recv_mergeable_pkts;



在虚拟主机和QEMU之间的连接协商期间，可以通过设置标志VIRTIO_NET_F_MRG_RXBUF来启动路径。虚拟主机用户默认情况下支持此功能，并且在QEMU中启用此功能的命令如下：



1. qemu-system-x86_64 -name vhost-vm1
2. …..-device virtio-net-pci,mac=52:54:00:00:00:01,netdev=mynet1,mrg_rxbuf=on \
3. ……



DPDK将根据VIRTIO_NET_F_MRG_RXBUF标志选择相应的Rx功能 ：



1. if (vtpci_with_feature(hw, VIRTIO_NET_F_MRG_RXBUF))
2. eth_dev->rx_pkt_burst = &virtio_recv_mergeable_pkts;
3. else
4. eth_dev->rx_pkt_burst = &virtio_recv_pkts;
5. 
6. 



可合并路径与其他两个路径之间的区别在于，只要启用可合并，rte_eth_txconf-> txq_flags的值就不会受到影响。

## 3.**向量路径**

该路径利用处理器中的单个指令，多个数据（SIMD）指令来矢量化数据Rx / Tx动作。在纯I / O数据包转发方案中，它具有更好的性能。该路径的接收和发送功能如下：



1. eth_dev->tx_pkt_burst = virtio_xmit_pkts_simple;
2. eth_dev->rx_pkt_burst = virtio_recv_pkts_vec;



使用此路径的要求包括：

1. 平台处理器应支持相应的指令集。x86平台应支持[Streaming SIMD Extensions 3（SSE3），](https://link.zhihu.com/?target=https%3A//www.intel.com/content/www/us/en/support/processors/000005779.html)可以通过DPDK中的rte_cpu_get_flag_enabled（RTE_CPUFLAG_SSE3）进行检查。ARM *平台应支持NEON *，可以通过DPDK中的rte_cpu_get_flag_enabled（RTE_CPUFLAG_NEON）进行检查。

2. Rx端的可合并路径应关闭，这可以通过DPDK中的功能进行检查。关闭此功能的命令如下：

3. 1. qemu-system-x86_64 -name vhost-vm1
   2. …..
   3. -device virtio-net-pci,mac=52:54:00:00:00:01,netdev=mynet1,mrg_rxbuf=off \
   4. ……



1. 未启用“卸载”功能，包括VLAN卸载，SCTP校验和卸载，UDP校验和卸载以及TCP校验和卸载。
2. rte_eth_txconf-> txq_flags需要设置为1。例如，在DPDK提供的**testpmd**示例中，我们可以使用以下命令在虚拟机中配置virtio设备：

```text
testpmd -c 0x3 -n 4 -- -i --txqflags=0xf01
```

可以看出向量路径的功能是相对有限的，这就是为什么它没有成为DPDK性能优化的重点的原因。

## 4.**不可合并的路径**

不可合并的路径很少使用。这是它的接收和发送路径：

1. eth_dev->tx_pkt_burst = &virtio_xmit_pkts;
2. eth_dev->rx_pkt_burst = &virtio_recv_pkts;



应用此路径需要以下配置：

1. 沿Rx方向可合并关闭。
2. rte_eth_txconf-> txq_flags需要设置为0。例如，在**testpmd中**，我们可以使用以下命令在虚拟机中配置virtio设备：

```text
#testpmd -c 0x3 -n 4 -- -i --txqflags=0xf00
```

## 5.**PVP路径性能比较**

使用Physical-VM-Physical（PVP）测试比较了不同DPDK vhost / virtio接收和传输路径的性能。在此测试中，使用**testpmd**生成vhost-user端口，如下所示：

```text
testpmd -c 0xc00 -n 4 --socket-mem 2048,2048 --vdev'eth_vhost0，iface = vhost-net，queues = 1'--i --nb-cores = 1
```

在VM中，**testpmd**用于控制virtio设备。测试方案如下图所示。主要目的是在虚拟化环境中测试南北数据转发功能。Ixia公司*流量发生器发送64字节的数据包发送到以太网卡以每秒10个千兆比特的线速度，**testpmd**在物理机调用虚拟主机用户来转发数据包到虚拟机，并且**testpmd**在虚拟机调用virtio-user将数据包发送回物理计算机，最后发送回Ixia。其发送路径如下：

```
IXIA→NIC port1→Vhost-user0→Virtio-user0→NIC port1→IXIA
```

## 6.**I/O转发吞吐量**

将DPDK 17.05与I / O转发配置一起使用，不同路径的性能如下：

![img](https://pic4.zhimg.com/80/v2-bc00caf50969e796965ee2f14609d42b_720w.webp)

I / O转发测试结果



在纯I / O转发的情况下，矢量路径具有最佳吞吐量，比可合并的吞吐量高出近15％。

## 7.**Mac转发吞吐量**

在MAC转发配置下，不同路径的转发性能如下：

![img](https://pic3.zhimg.com/80/v2-fb7c22d985d45564f9f60508733ceb0e_720w.webp)

MAC转发测试结果

在这种情况下，这三个路径的性能几乎相同。我们建议使用可合并路径，因为它提供了更多功能。

## 8.**PVP MAC转发吞吐量**

下图显示了从DPDK 16.07开始在x86平台上PVP MAC的转发性能趋势。由于可合并路径的应用场景更广，DPDK工程师从那时开始对其进行了优化，并且PVP的性能在此路径上提高了近20％。

![img](https://pic1.zhimg.com/80/v2-035bc82c181e78304595d4c2596693c0_720w.webp)

PVP MAC转发测试结果

注意：DPDK 16.11的性能下降主要是由于添加的新功能（例如vhost xstats和间接描述符表）带来的开销。

### 8.1**测试台信息**

CPU：英特尔®至强®CPU E5-2680 v2 @ 2.80GHz

操作系统：Ubuntu 16.04

## 9.关于作者

姚雷是英特尔的软件工程师。他主要负责与DPDK虚拟化测试相关的工作。

## 10.告示

性能测试中使用的软件和工作负载可能仅针对英特尔微处理器的性能进行了优化。性能测试（例如SYSmark *和MobileMark *）是使用特定的计算机系统，组件，软件，操作和功能进行测量的。任何这些因素的任何变化都可能导致结果变化。您应该查阅其他信息和性能测试，以帮助您全面评估您的预期购买，包括该产品与其他产品组合时的性能。

欲了解更多信息，请访问[HTTP ](https://link.zhihu.com/?target=http%3A//www.intel.com/performance)[：// ](https://link.zhihu.com/?target=http%3A//www.intel.com/performance)[www.intel.com/performance](https://link.zhihu.com/?target=http%3A//www.intel.com/performance)。

英特尔技术的功能和优势取决于系统配置，并且可能需要启用硬件，软件或服务才能激活。性能因系统配置而异。请与您的系统制造商或零售商联系，或在[intel.com上](https://link.zhihu.com/?target=http%3A//www.intel.com/)了解更多信息。

## 11.相关阅读

《[Linux虚拟化KVM-Qemu分析（八）之virtio初探](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/113819423)》

《[Linux虚拟化KVM-Qemu分析（九）之virtio设备](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/113819437)》

《[virtio 网络的演化](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/113819506)》

《[使用DPDK打开Open vSwitch（OvS） *概述](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/108747601)》

《[Open vSwitch（OVS）文档](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/109005008)》

《[《深入浅出DPDK》读书笔记（十五）：DPDK应用篇（Open vSwitch（OVS）中的DPDK性能加速）](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/109371440)》

《[virtio 网络的演化：原始virtio ＞ vhost-net(内核态) ＞ vhost-user(DPDK) ＞ vDPA](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/113819506)》

《[Virtio原理简介](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/115287131)》

《[Virtio、Vhost、Vhost-user介绍](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/115287676)》



原文链接：[https://rtoax.blog.csdn.net/art](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/115417265%3Fspm%3D1001.2101.3001.6650.6%26utm_medium%3Ddistribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-6-115417265-blog-102830343.pc_relevant_3mothn_strategy_and_data_recovery%26depth_1-utm_source%3Ddistribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-6-115417265-blog-102830343.pc_relevant_3mothn_strategy_and_data_recovery%26utm_relevant_index%3D13)  原文作者：rtoax
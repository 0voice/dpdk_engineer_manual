# DPDK源码分析之l2fwd

## 1.什么是L2转发

2层转发，即对应OSI模型中的数据链路层，该层以Mac帧进行传输，运行在2层的比较有代表性的设备就是交换机了。

当交换机收到数据时，它会检查它的目的MAC地址，然后把数据从目的主机所在的接口转发出去。

交换机之所以能实现这一功能，是因为交换机内部有一个MAC地址表，MAC地址表记录了网络中所有MAC地址与该交换机各端口的对应信息。某一数据帧需要转发时，交换机根据该数据帧的目的MAC地址来查找MAC地址表，从而得到该地址对应的端口，即知道具有该MAC地址的设备是连接在交换机的哪个端口上，然后交换机把数据帧从该端口转发出去。

1.交换机根据收到数据帧中的源MAC地址建立该地址同交换机端口的映射，并将其写入MAC地址表中。
2.交换机将数据帧中的目的MAC地址同已建立的MAC地址表进行比较，以决定由哪个端口进行转发。
3.如数据帧中的目的MAC地址不在MAC地址表中，则向所有端口转发。这一过程称为泛洪（flood）。
4.接到广播帧或组播帧的时候，它立即转发到除接收端口之外的所有其他端口。

## 2.DPDK-l2fwd做了什么

该实例中代码写死的网卡为promiscuous混杂模式，我的虚拟机的两个网卡是直连的，拓扑如下：

![img](https://pic2.zhimg.com/80/v2-4d0bac811e8490e601fc70344280b7c1_720w.webp)

因此，dpdk-l2fwd中，port 0收到包会转发给port 1，port 1收到包也会转发给相邻端口port 0，下图port 0混杂模式收到29694508个包然后会把这些包都sent给port 1，port 1同样收到其他包后也会转发给port 0。

因此，dpdk l2 fwd这个例子展示了两个网卡在mac层成功的转发了数据包，后续我们会阅读源码并调试程序来看，dpdk是如何实现这一功能的。

![img](https://pic3.zhimg.com/80/v2-7922c7178bcb1d96b737173d0297778a_720w.webp)

## 3.Pktgen安装

pktgen-dpdk是用于对DPDK进行高速数据包测试的工具

使用的命令行参数如下：

```c
-c  : 用于指定运行程序的CPU内核掩码。
-n  : 用来指定内存通道。
-s  : 如果你想用pktgen发送pcap文件  例如 [-s P:PCAP_file]  -s 0 : 1.pcap    0表示在第0个网卡，1.pcap及文件名
-P  : 启动所有网卡，并进入混杂模式，想指定特定网卡 用 -p mask
-m  : 指定lcore和port的映射关系 [1].0, [2].1 core1->port0 core2->port2
```

由于pktgen是基于dpdk进行开发的，因此选取的pktgen的版本和dpdk的版本一定是相互兼容的，我这边版本如下：

```c
pktgen version:
pktgen-dpdk-pktgen-21.03.1

dpdk version:
20.11.4-rc1
```

安装过程严格按照install.md即可：

```c
//1. 先升级一下gcc版本
sudo yum install centos-release-scl
sudo yum install devtoolset-7-gcc*
scl enable devtoolset-7 bash
which gcc 
gcc -version
//2. 安装libpcap
yum install libpcap
yum install dnf-plugins-core
yum install libpcap-devel
//3. 环境变量设置
export RTE_SDK=<DPDKinstallDir>
export RTE_TARGET=x86_64-native-linux-gcc
//4. make
```

这样就可以通过参数配置，指定port去发包了，下面我通过port0发包10000pkts/s，由于port0和port1直连，可以看到port 1收包10000pkts/s。

```text
//core0为主控负责命令行接受，流量显示，消息调度
//core1->port0 core2->port1
./pktgen -l 0-2 -n 3 -- -P -m "[1].0, [2].1"
set 0 dst mac 00:0c:29:93:e6:be
set 0 count 10000
start 0
```

![img](https://pic4.zhimg.com/80/v2-351eba2e59f6e3f52e8458a42c723b73_720w.webp)

![img](https://pic4.zhimg.com/80/v2-65c71ee36da15a5d5f4e7ad127c3455b_720w.webp)

## 4.源码阅读

![img](https://pic2.zhimg.com/80/v2-1aa7d3b7ac957b11c85769f2dac01801_720w.webp)

## 5.无情GDB

gdb并打印一些关键信息，加深l2fwd源码理解

### 5.1l2fwd_parse_args

解析l2fwd转发的一些命令行配置，我这边配置的是set args -- -q 1 -p 0x3即：

![img](https://pic3.zhimg.com/80/v2-def8f69fc005b2b537c0ab87e6bfdd26_720w.webp)

l2fwd_rx_queue_per_lcore：每个逻辑核负责处理一个rx队列，后续可以看到网卡配置时一个网卡配置一个rx队列和tx队列，因此这个参数可以理解为一个逻辑核负责处理一个网卡。

l2fwd_enabled_port_mask：可用的的网卡port的掩码

timer_period：多长时间将统计信息输出到stdout中，缺省为10s

### 5.2转发端口配置

两两一组互为转发，因为我这边就两个port0和port1：

![img](https://pic4.zhimg.com/80/v2-dfdeba868066bbf09d5836753c9dcf7f_720w.webp)

port 0 转给port 1, port 1 转给port 0

这里面用到了rte_eth_devices[portid]->data.owner.id与RTE_ETH_DEV_NO_OWNER进行比较，代表该接管的网卡还没有被使用。

### 5.3网卡接管

我这边是虚拟机网卡E1000，因此是在eal初始化时调用的是eth_em_dev*init，内容很多后面用到哪个在详细看一下*，这边就是dpdk通过igb_uio用户态驱动接管网卡，并对其进行一些参数的初始化。

```c
//pmd驱动一系列函数，包括设备的启动，停止，混杂模式，广播模式，MTU设置以及Mac地址设置
eth_dev->dev_ops = &eth_em_ops;
eth_dev->rx_queue_count = eth_em_rx_queue_count;
//DD位（Descriptor Done Status）用于标志标识一个描述符buf是否可用。
eth_dev->rx_descriptor_done   = eth_em_rx_descriptor_done;
eth_dev->rx_descriptor_status = eth_em_rx_descriptor_status;
eth_dev->tx_descriptor_status = eth_em_tx_descriptor_status;
//收包函数
//1、网卡使用DMA写Rx FIFO中的Frame到Rx Ring Buffer中的mbuf，设置desc的DD为1
//2、网卡驱动取走mbuf后，设置desc的DD为0，更新RDT
eth_dev->rx_pkt_burst = (eth_rx_burst_t)&eth_em_recv_pkts;
//发包函数
eth_dev->tx_pkt_burst = (eth_tx_burst_t)&eth_em_xmit_pkts;
//发包准备函数，offload设置校验以及检验和
eth_dev->tx_pkt_prepare = (eth_tx_prep_t)&eth_em_prep_pkts;
//mac地址字符串
eth_dev->data->mac_addrs = rte_zmalloc("e1000", RTE_ETHER_ADDR_LEN * hw->mac.rar_entry_count, 0);
```

![img](https://pic2.zhimg.com/80/v2-92340cbfcebabfae2ee6a3e898236725_720w.webp)

### 5.4为逻辑核分配port

根据之前配置的core最大可以处理几个网卡port，我这边是1所以，每个core赋值一个网卡，可以看到n_rx_port都是1

![img](https://pic3.zhimg.com/80/v2-dc7f20477d20433144c84b4e442efbd6_720w.webp)

### 5.5**rte_pktmbuf_pool_create**

这个mbuf pool主要是给网卡接收数据包提供mbuf的，换句话说网卡通过DMA收到数据需要把数据包通过DMA传送到一块内存，正是这个mbuf pool中的内存。这里会创建一个mbuf的内存池，每个muf的大小为（ sizeof(struct rte_mbuf) + priv_size + data_room_size ，总共有nb_mbufs个mbuf。

```c
//收队列数量+发队列数量+一次最大收报文数量+核数*它的cache中报文的数量
nb_mbufs = RTE_MAX(nb_ports * (nb_rxd + nb_txd + MAX_PKT_BURST + nb_lcores * MEMPOOL_CACHE_SIZE), 8192U);
```

pktbuf pool创建使用了下面几个函数，我们逐一debug:

**rte_mempool_create_empty**

->**rte_mempool_populate_default**

->**rte_mempool_obj_iter**

- **rte_mempool_create_empty**

在memzone申请内存，这块内存包含sizeof（struct rte_mempool），每个逻辑核的cache size大小，以及私有数据的大小。然后会创建一个空的mempool头指针指向这块内存，并挂载到rte_mempool_tailq中，这个头包含sizeof（struct rte_mempool）以及每个逻辑核的cache大小，然后会在memzone申请所有pool内元素所需要的内存，这个内存地址赋给mp指针。总结如下：

![img](https://pic3.zhimg.com/80/v2-2b773d40f378f1947c1122b9ad4c4e66_720w.webp)

debug如下：

![img](https://pic2.zhimg.com/80/v2-ed613dab1c1e995dddde10dba5ff0cb5_720w.webp)

![img](https://pic1.zhimg.com/80/v2-672cd3bdcbbf2030e6a4a7a3ef6a149c_720w.webp)

![img](https://pic1.zhimg.com/80/v2-acc63d6b4630270f5e92f6a72455cd68_720w.webp)

![img](https://pic4.zhimg.com/80/v2-8ffeed220df556164335eb65a48dbdc3_720w.webp)

![img](https://pic2.zhimg.com/80/v2-ca5b9f83266403bd4d0e54aa32bebfe5_720w.webp)

- **rte_mempool_populate_default**

ring队列默认为多生产者多消费者模式，eal初始化时会生成一个ring队列table，用于规定每种ring队列的一些函数操作，包括元素入队，出队，遍历等。

为mp创建一个ring队列，后续会存mbuf的指针。mp->flags |= MEMPOOL_F_POOL_CREATED

为mp每一个元素分配空间，这里面会做一个page-aligned address的操作如下：

![img](https://pic3.zhimg.com/80/v2-0de6fbd0b5d43949d9f259bbaacb0926_720w.webp)

然后寻找最大的连续页面进行分配元素，将这些元素指针加入到mp->elmlist中，分配的内存块信息记录在mp->memlist中，并把这些mbuf指针入队ring，然后会继续寻找连续页面分配元素直到elt_size。总结如下：

![img](https://pic2.zhimg.com/80/v2-7d21e9582a16e0e55803f85112ddaf45_720w.webp)

debug如下：

![img](https://pic1.zhimg.com/80/v2-e691b28424228acc6a76ea1996e9c210_720w.webp)

![img](https://pic2.zhimg.com/80/v2-a8b0180b7a165c069a0dd3ab73442cf5_720w.webp)

![img](https://pic2.zhimg.com/80/v2-6ad492889d8ff6a2870ba7d49b8bf939_720w.webp)

- **rte_mempool_obj_iter**

初始化mbuf信息，包括所属内存池、缓存起始地址等。

### 5.6网卡启动

rte_eth_dev_info_get获取对应port的网卡信息，包括网卡驱动，发送和接收队列个数，mtu值以及rx和tx的hw descriptor（后序给dma 用的， 里面包括了 内存搬运的起始地址 结束地址 什么的，dma 分析这个结构体完成数据传输。）

![img](https://pic1.zhimg.com/80/v2-f936869c69e41db47b73ac1eb3b20c20_720w.webp)

然后会通过eth_dev_rx_queue_config为dev->data->tx_queues和dev->data->rx_queues分配指向队列的指针，这里面代码写死了，1个接收队列1个发送队列，然后会调用setup函数对收发队列进行初始化。

**收队列初始化：**

网卡驱动的rx_queue_setup函数，由于我是虚拟网卡e1000，所以这边调用的是eth_igb_rx_queuesetup，这个函数主要分配了2个队列，sw_ring和rx_ring，以及网卡相关的寄存器设置。

rx_ring包含了E1000_MAX_RING_DESC个网卡描述符，里面含有dma传输的报文数据地址以及报文头部地址，rss hash值以及报文的校验和，长度等等。

sw_ring包含了 nb_desc个struct igb_rx_entry，也就是mbuf地址。

> rx_ring主要存储报文数据的物理地址，物理地址供网卡DMA使用，也称为DMA地址（硬件使用物理地址，将报文copy到报文物理位置上）。sw_ring主要存储报文数据的虚拟地址，虚拟地址供应用使用（软件使用虚拟地址，读取报文）。

发队列初始化：

和接收队列类似，调用eth_em_tx_queue_setup。

**启动网卡前：**

设置报文发送失败的回调函数，这个实例是失败计数并free的操作，rte_eth_tx_buffer_count_callback.

**启动网卡：**

我这边调用的是e1000的pmd驱动，eth_em_start，函数极其复杂，原谅太菜的我过早的遇到了它，不过这边有个函数比较关键em_alloc_rx_queue_mbufs。它将sw_ring队列和用户的mbuf pool内存池关联，并设置rx_ring的dma地址。总结如下：

![img](https://pic2.zhimg.com/80/v2-3c10bdca7ef132a1e823bf38849af649_720w.webp)

debug如下：

![img](https://pic2.zhimg.com/80/v2-8018feb6bc10ae576ed1efcf73320951_720w.webp)

### 5.7lcore_worker启动

主核负责统计各个网卡port收发包的数量

其他子核负责读取对应的网卡port的rx queue，然后如果有数据包的话就，就循环转发到配置的另一个网卡上。

代码极其简单，关键收发包api：

```text
(*dev->tx_pkt_burst)(dev->data->tx_queues[queue_id], tx_pkts, nb_pkts);

(*dev->rx_pkt_burst)(dev->data->rx_queues[queue_id], rx_pkts, nb_pkts);
```

![img](https://pic3.zhimg.com/80/v2-e04e2eeef273eac46cdaf35daa95d592_720w.webp)

![img](https://pic3.zhimg.com/80/v2-a1b72d35026637676cf3523f57bbc53e_720w.webp)

![img](https://pic4.zhimg.com/80/v2-a4a5d555c0836ea36d03c14bf2e0582f_720w.webp)

我这边就一个核，所以它接收后会自己发出去。

## 6.Reference

[dpdk应用基础 (豆瓣)](https://link.zhihu.com/?target=https%3A//book.douban.com/subject/26880617/)

[理解物理网卡、网卡接口、内核、IP等属性的关系-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/qq_41679358/article/details/107344448%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164154113016780271552947%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D%26request_id%3D164154113016780271552947%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-2-107344448.pc_search_result_cache%26utm_term%3D%E7%BD%91%E5%8D%A1%E7%89%A9%E7%90%86%E7%AB%AF%E5%8F%A3%26spm%3D1018.2226.3001.4187)

[交换机-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/wdirdo/article/details/99705861%3Futm_medium%3Ddistribute.pc_relevant.none-task-blog-2~default~baidujs_utm_term~default-0.pc_relevant_paycolumn_v2%26spm%3D1001.2101.3001.4242.1%26utm_relevant_index%3D3)

[dpdk多队列机制-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/zengxiaosen/article/details/79846714)

[混杂模式和非混杂模式-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/lengye7/article/details/71191291)

[DPDK L2FWD使用 - 简书](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/cec11b3f4fb3)

[在CentOS中升级gcc4.8到gcc5并修改默认设置 - it610.com](https://link.zhihu.com/?target=https%3A//www.it610.com/article/1282311418737082368.htm)

[dpdk环境搭建+创建dpdk项目，并连接dpdk库_linggang_123的博客-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/linggang_123/article/details/114137361)

[DPDK PKTGEN使用 - 简书](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/fa7d9f2c0f55)

[DPDK 示例之 L2FWD - 知乎](https://zhuanlan.zhihu.com/p/363030706)

[DPDK-Pktgen的使用](https://link.zhihu.com/?target=https%3A//blog.csdn.net/Gerald_Jones/article/details/103645108)

[网卡基础概念扫盲-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/ScilogyHunter/article/details/107368303%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164163394916780274143025%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D%26request_id%3D164163394916780274143025%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-107368303.first_rank_v2_pc_rank_v29%26utm_term%3D%E7%BD%91%E5%8D%A1%26spm%3D1018.2226.3001.4187)

[交换机的工作原理-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/weixin_42048417/article/details/82386767%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164164261016780274165367%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D%26request_id%3D164164261016780274165367%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-82386767.first_rank_v2_pc_rank_v29%26utm_term%3D%E4%BA%A4%E6%8D%A2%E6%9C%BA%E5%8E%9F%E7%90%86%E5%9B%BE%26spm%3D1018.2226.3001.4187)

[c/c++ signal(信号)解析-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/sysleo/article/details/95984946%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164172701416781683983766%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D%26request_id%3D164172701416781683983766%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-95984946.first_rank_v2_pc_rank_v29%26utm_term%3Dsignal%26spm%3D1018.2226.3001.4187)

[dpdk网卡收发包分析-ChinaUnix博客](https://link.zhihu.com/?target=http%3A//blog.chinaunix.net/uid-28541347-id-5785122.html)

[DPDK总结网卡初始化-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/hz5034/article/details/88367518%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164173170816780265415442%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D%26request_id%3D164173170816780265415442%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~baidu_landing_v2~default-1-88367518.first_rank_v2_pc_rank_v29%26utm_term%3Dstruct%2Brte_eth_dev%2B%26spm%3D1018.2226.3001.4187)

[网卡offload功能介绍-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/u012247418/article/details/117715650%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164180475216780274165105%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D%26request_id%3D164180475216780274165105%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-2-117715650.pc_search_result_cache%26utm_term%3Dtx%2Boffload%26spm%3D1018.2226.3001.4187)

[dpdk 网卡队列初始化 + 收发包 - tycoon3 - 博客园 (cnblogs.com)](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/dream397/p/13608829.html)

[内存池之rte_mempool-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/jeawayfox/article/details/106116123)

[DPDK内存管理-mempool、mbuf-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/qq_15437629/article/details/78149983%3Futm_medium%3Ddistribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-3.control%26depth_1-utm_source%3Ddistribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-3.control)

[DPDK数据包与内存专题-mempool内存池 - AISEED - 博客园 (cnblogs.com)](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/yhp-smarthome/p/6687175.html)

[dpdk mbuf之概念理解_ych的专栏-CSDN博客_mbuf](https://link.zhihu.com/?target=https%3A//blog.csdn.net/yaochuh/article/details/88207553%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164212465516780265410184%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D%26request_id%3D164212465516780265410184%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-1-88207553.pc_search_result_cache%26utm_term%3Drte_pktmbuf_init%26spm%3D1018.2226.3001.4187)

[DPDK 网卡收包流程_RToax-CSDN博客_dpdk多队列收包](https://link.zhihu.com/?target=https%3A//blog.csdn.net/Rong_Toa/article/details/109400686%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522164214560216780274143460%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D%26request_id%3D164214560216780274143460%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-1-109400686.pc_search_result_cache%26utm_term%3Dsw_ring%2B%26spm%3D1018.2226.3001.4187)

[搞懂Linux零拷贝，DMA_RToax-CSDN博客](https://link.zhihu.com/?target=https%3A//rtoax.blog.csdn.net/article/details/108825666)

 原文链接： https://zhuanlan.zhihu.com/p/454346807 原文作者：于顾而言


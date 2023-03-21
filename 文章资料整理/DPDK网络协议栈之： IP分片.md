# DPDK网络协议栈之： IP分片

## 1.初始化EAL环境

先rte_eal_init(argc, argv)初始化EAL环境，确定绑定的核数，队列数。解析parse_args(argc, argv)参数，如-c是确定虚拟的内核数。



## 2.获取网卡的数目

通过函数rte_eth_dev_count()来获取网卡的数目。



## 3. 获取核的个数

通过函数rte_lcore_count()获取核的个数



## 4.初始化内存池

初始化内存池时通过遍历lcore来获取每个lcore上的socketid，在通过socket id来创建socket_direct_pool[socket]内存空间。



## 5.初始化所有的网卡

1.将网卡跟对应的lcore绑定起来，定义一个全局的结构体数组，然后用一个结构体指针qconf指向各个网卡的结构体地址（qconf =&lcore_queue_conf[rx_lcore_id];），当rx_core_id大于设定的最大core数或者qconf->n_rx_queue ==1 时，rx_lcore_id++；qconf会指向会改变。



2.获取内核的socket id，发送队列n_tx_queue等于网卡的数，通过函数ret =rte_eth_dev_configure(portid, 1, (uint16_t)n_tx_queue,

&port_conf)配置接受队列和发送队列的个数(接收队列只能有一个，发送队列可以有多个)，

dev = &rte_eth_devices[port_id];定义了一个全局变量rte_eth_devices主要是配置物理层接口，设置网卡驱动。配置网卡驱动时，会检查初始化的网卡id是否有效，通过函数(*dev->dev_ops->dev_infos_get)(dev,&dev_info)来获取网卡信息，从而判断我们设置的接受和发送队列是否大于了网卡支持的最大队列数。检查完后设置新的接收和发送队列，rte_eth_dev_rx_queue_config(dev, nb_rx_q)，dev->data->rx_queues= rte_zmalloc("ethdev->rx_queues",

sizeof(dev->data->rx_queues[0]) * nb_queues,

RTE_CACHE_LINE_SIZE);为接受队列开辟空间存储数据，同时记录开辟队列的个数，发送队列是一样的。



3.初始化接收队列

从内存池内将数据存到接收队列中。



## 6.遍历接收队列，从接收队列读取数据然后发送数据。

### 6.1 从接收队列里面读取数据

从对应的网卡的接收队列里面读取数据，通过函数rte_eth_rx_burst(portid, 0, pkts_burst,

MAX_PKT_BURST)将数据读到结构体数组structrte_mbuf *pkts_burst[MAX_PKT_BURST]。

### 6.2 将数据进行缓存和发送

在发送数据前，先会进行缓存，缓存会进行两次，然后将缓存的数据包发送，最后将没有缓存的数据包发送到下一层。发送数据包时，会判断是IPv4的包还是Ipv6的数据包，

**缓存的处理流程如下：**

判定完后，在根据包的长度是否大于MTU 1500,如果大于就会分片，分片的包长度需要是8的倍数，同时也会检查pkts_out的空间是否足够大能到存下整个分片的包，如果不够就会返回–EINVAL。

函数rte_ipv4_fragment_packet是处理分片的流程的，发送数据的时候会建立直接存储区和间接存储区，先建立一个直接存储让后将要发送的数据包通过函数rte_pktmbuf_attach将数据不断地往间接存储区拷贝数据，直到将所有数据读完，直接存储区和间接存储区是通过链表链接起来的。链表存储着要发送的数据,函数返回的是包的个数。





原文链接：[https://blog.csdn.net/pangyemen](https://link.zhihu.com/?target=https%3A//blog.csdn.net/pangyemeng/article/details/73017230)

原文作者：庞叶蒙
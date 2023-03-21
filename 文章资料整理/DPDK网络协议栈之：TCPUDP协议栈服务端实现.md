# DPDK网络协议栈之：TCP/UDP协议栈服务端实现

## 1.介绍

为了对协议栈了解更深入一些，借助dpdk-19.11实现一个简易协议栈。

关于dpdk的介绍同学们可以看看这篇：https://blog.csdn.net/hjlogzw/article/details/124561294?spm=1001.2014.3001.5502

先来张架构图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/ac8ba9e03ec449c7a7cdfe602ffacb03.png)

本项目要完成的工作如下：

dpdk相关配置
实现协议栈，主要针对TCP与UDP，包含三次握手、四次挥手
实现服务端socket系统调用api：socket、bind、listen、accept、recv、send、recvfrom、sendto、recv、send、close
实现epoll多线程

## 2.DPDK配置

dpdk，主要针对网卡设备进行收发数据包，同时负责组装和解析数据包，发送给应用程序如TCP或UDP进行处理。

### 2.1 dpdk环境初始化

```cpp
if(rte_eal_init(argc, argv) < 0)
	    rte_exit(EXIT_FAILURE, "Error with EAL init\n");	

pstMbufPoolPub = rte_pktmbuf_pool_create("MBUF_POOL_PUB", D_NUM_MBUFS, 0, 0, 
        RTE_MBUF_DEFAULT_BUF_SIZE, rte_socket_id());
if(pstMbufPoolPub == NULL)
{
	printf("rte_errno = %x, errmsg = %s\n", rte_errno, rte_strerror(rte_errno));
	return -1;
}
```

### 2.2 端口及收发队列配置

主要设置网卡的发送和接收队列。

```cpp
struct rte_eth_conf stPortConf =    // 端口配置信息
{
    .rxmode = {.max_rx_pkt_len = 1518 }   // RTE_ETHER_MAX_LEN = 1518
};
    
uiPortsNum = rte_eth_dev_count_avail(); 
if (uiPortsNum == 0) 
	rte_exit(EXIT_FAILURE, "No Supported eth found\n");

rte_eth_dev_info_get(D_PORT_ID, &stDevInfo); 

  // 配置以太网设备
rte_eth_dev_configure(D_PORT_ID, iRxQueueNum, iTxQueueNum, &stPortConf);

iRet = rte_eth_rx_queue_setup(D_PORT_ID, 0 , 1024, rte_eth_dev_socket_id(D_PORT_ID), NULL, pstMbufPoolPub);
if(iRet < 0) 
   rte_exit(EXIT_FAILURE, "Could not setup RX queue!\n");

stTxConf = stDevInfo.default_txconf;
stTxConf.offloads = stPortConf.txmode.offloads;
iRet = rte_eth_tx_queue_setup(D_PORT_ID, 0 , 1024, rte_eth_dev_socket_id(D_PORT_ID), &stTxConf);
if (iRet < 0) 
	rte_exit(EXIT_FAILURE, "Could not setup TX queue\n");

if (rte_eth_dev_start(D_PORT_ID) < 0 )
	rte_exit(EXIT_FAILURE, "Could not start\n");
  
rte_eth_promiscuous_enable(D_PORT_ID);

// 设置IN、OUT队列，将网卡的收发数据包存入队列
pstRing->pstInRing = rte_ring_create("in ring", D_RING_SIZE, rte_socket_id(), RING_F_SP_ENQ | RING_F_SC_DEQ);
pstRing->pstOutRing = rte_ring_create("out ring", D_RING_SIZE, rte_socket_id(), RING_F_SP_ENQ | RING_F_SC_DEQ);
	
```

### 2.3 线程设置

dpdk具有cpu亲和性的特点，可以为每个线程绑定单独的cpu核心，本项目设置三个工作线程：网卡数据包分发、TCP应用程序、UDP应用程序，分别完成：对网卡来的数据包进行分发筛选、TCP应用层业务服务、UDP应用层业务服务。

```cpp
uiCoreId = rte_lcore_id();

uiCoreId = rte_get_next_lcore(uiCoreId, 1, 0);
rte_eal_remote_launch(pkt_process, pstMbufPoolPub, uiCoreId);

uiCoreId = rte_get_next_lcore(uiCoreId, 1, 0);
rte_eal_remote_launch(udp_server_entry, pstMbufPoolPub, uiCoreId);

uiCoreId = rte_get_next_lcore(uiCoreId, 1, 0);
rte_eal_remote_launch(tcp_server_entry, pstMbufPoolPub, uiCoreId);
```

主线程只负责将来自网卡的数据放入IN ring，将要发送给网卡的数据放入OUT ring，实现解耦。


```c++
while (1) 
{
    // rx
    iRxNum = rte_eth_rx_burst(D_PORT_ID, 0, pstRecvMbuf, D_BURST_SIZE);
    if(iRxNum > 0)
        rte_ring_sp_enqueue_burst(pstRing->pstInRing, (void**)pstRecvMbuf, iRxNum, NULL);
    
    // tx
    iTotalNum = rte_ring_sc_dequeue_burst(pstRing->pstOutRing, (void**)pstSendMbuf, D_BURST_SIZE, NULL);
	if(iTotalNum > 0)
	{
		iOffset = 0;
		while(iOffset < iTotalNum)
		{
			iTxNum = rte_eth_tx_burst(D_PORT_ID, 0, &pstSendMbuf[iOffset], iTotalNum - iOffset);
			if(iTxNum > 0)
				iOffset += iTxNum;
		}
	}
}

```

## 3.小结

本文主要对项目的架构进行介绍，以及DPDK的相关配置。





原文链接：https://blog.csdn.net/hjlogzw/article/details/124924951

原文作者：阿杰的小鱼塘


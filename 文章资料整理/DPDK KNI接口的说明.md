# DPDK KNI接口的说明

前言：在DPDK报文处理中，有些报文需要发送到内核协议栈进行处理，如GTP-C控制报文，如果报文数量较少，可以使用内核提供的TAP/TUN设备，但是鉴于这种设备使用的系统调用的方式，还涉及到`copy_to_user()`和`copy_from_user()`的开销，因此，提供了KNI接口用于改善用户态和内核态间报文的处理效率。

### KNI实现组成

KNI的实现由两部分组成，内核态的模块和用户态的模块。通过创建kni接口上下文，在内核态和用户态之间使用队列的方式，传递数据包的指针，从而避免了拷贝。

![img](https://images2015.cnblogs.com/blog/614525/201705/614525-20170526224652279-1344044339.png)

对于每个接口，要创建4个队列，实际的代码实现中，由于要支持一些工具的操作，如ifconfig的配置等，又创建了两个队列---`m_req_q`和`m_resp_q`；请求队列和回应队列用于配置。

> 使用KNI接口上下文的方式，大概是因为内核态和用户态的地址空间不一致。所以要进行地址的转换。

### KNI的使用解析

在使用解析这节，以例子中的代码进行说明：整体上，main.c仍旧遵循着DPDK的使用格局

- 1.进行EAL层的初始化
- 2.解析DPDK和应用层的参数
- 3.分配报文的mempool，为接收报文做准备
- 4.初始化以及启动接口
- 5.启动各个核上的线程
- 6.等待线程退出，进行最后的处理。

重点说明一下第4步，初始化和启动接口。其他的部分没有差别。

```c
/* Initialize KNI subsystem */
init_kni();
 
/* Initialise each port */
for (port = 0; port < nb_sys_ports; port++) {
	/* Skip ports that are not enabled */
	if (!(ports_mask & (1 << port)))
		continue;
	init_port(port);
 
	if (port >= RTE_MAX_ETHPORTS)
		rte_exit(EXIT_FAILURE, "Can not use more than "
			"%d ports for kni\n", RTE_MAX_ETHPORTS);
 
	kni_alloc(port);
}
```

从上面可以看出，主要的接口初始化包括了KNI的部分，在`init_port()`部分，和别的工程并没有什么不同（设置接口，设置队列，启动接口），所以这次重点看一下关于KNI部分的两个函数---`init_kni()`和`kni_alloc()`

首先来看`init_kni()`，这个函数主要对kni设备对应的queue等进行分配，需要注意的是一个物理接口可以对应1个或者多个kni设备，通过在命令行中配置，但推荐每个物理接口对应一个KNI设备，在文档上有如下的解释：

> For a physical NIC port, one thread reads from the port and writes to KNI devices, and another thread reads from KNI devices and writes the data unmodified to the physical NIC port. It is recommended to configure one KNI device for each physical NIC port. If configured with more than one KNI devices for a physical NIC port, it is just for performance testing

所以，在`init_kni()`中通过`nb_lcore_k`标志先判断了kni接口的数量

```c
/* Calculate the maximum number of KNI interfaces that will be used */
	for (i = 0; i < RTE_MAX_ETHPORTS; i++) {
		if (kni_port_params_array[i]) {
			num_of_kni_ports += (params[i]->nb_lcore_k ?
				params[i]->nb_lcore_k : 1);
		}
	}
```

然后就为每个kni接口分配一套queue了（多数情况是一个接口），--> `rte_kni_init()`中，先填充了`kni_memzone_pool`结构，然后为这个KNI接口依次分配`kni_ctx,tx_ring,rx_ring,alloc_ring`等。具体看一下`struct rte_kni_memzone_pool`这个结构体

```c
struct rte_kni_memzone_pool {
	uint8_t initialized : 1;            /**< Global KNI pool init flag */
 
	uint32_t max_ifaces;                /**< Max. num of KNI ifaces */
	struct rte_kni_memzone_slot *slots;        /**< Pool slots */
	rte_spinlock_t mutex;               /**< alloc/relase mutex */
 
	/* Free memzone slots linked-list */
	struct rte_kni_memzone_slot *free;         /**< First empty slot */
	struct rte_kni_memzone_slot *free_tail;    /**< Last empty slot */
};
```

在初始化过后，这个结构体管理着一个KNI接口的所有队列。

之后就进行kni设备的真正创建，通过IOCTL调用到内核创建。在IOCTL之前，先把两个结构体进行了填充，`struct rte_kni_ops ops`和`struct rte_eth_dev_info dev_info`，其中，如果一个物理接口上配置了多个kni接口，那么第一个kni接口是master，只有在master上才能进行设备的操作，暂时只有设置MTU和更改接口运行两种。
根据填充的这两个结构体，在`rte_kni_alloc()`中做了两件事：

- 填充rte_kni *ctx结构，这个结构代表了运行时的上下文，把kni初始化时分配的各种ring关联到这个上下文中，并对ring（FIFO）做初始化，这样，上下文地址就关联好了。

```c
	/* RX RING */
	mz = slot->m_rx_q;
	ctx->rx_q = mz->addr;
	kni_fifo_init(ctx->rx_q, KNI_FIFO_COUNT_MAX);
	dev_info.rx_phys = mz->phys_addr;
```

- 填充`rte_kni_device_info dev_info`结构体，正是根据这个结构体，发送到内核，创建设备。这个结构体主要填充的是物理地址，`dev_info.rx_phys = mz->phys_addr;`。

两个操作完了以后，设备的创建就完成了。皆大欢喜！

原创：[AISEED](https://home.cnblogs.com/u/yhp-smarthome/)    本文链接：https://www.cnblogs.com/yhp-smarthome/p/6910760.html
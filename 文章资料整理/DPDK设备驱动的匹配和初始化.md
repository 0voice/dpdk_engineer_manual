# DPDK设备驱动的匹配和初始化

前言：DPDK使用了UIO（用户空间I/O）的机制，跳过内核态的网卡驱动，转而使用用户态的收发包驱动，从驱动到内存和数据包，继而到数据包的处理，这是一个完整的收发包流程。这篇主要介绍设备驱动的初始化，和收发包的处理。所选代码以DPDK-17.02版本为依据。

------

数据包的驱动初始化是在`rte_eal_init()进行的`，总体上分为2个阶段进行。

- 1.第一阶段是`rte_eal_pci_init()`，主要是获取系统中的设备PCI。
- 2.第二阶段是`rte_eal_pci_probe()`,这个阶段做的事比较多，匹配对应的设备驱动，分配设备，并对设备进行初始化。

我们就按照这个顺序进行介绍。

<1>.先看`rte_eal_init()`这个函数，了解这一阶段的处理过程。

在函数中，调用了rte_eal_pci_scan()，来扫描系统目录里的PCI设备。默认的扫描目录是`#define SYSFS_PCI_DEVICES "/sys/bus/pci/devices"`,就是依次读取目录下的每一个文件名，解析PCI的地址信息，填充地址信息。

然后调用了`pci_scan_one()`来进行设备信息的填充，挂接设备。先分配了一个PCI设备结构，注意：此处分配的是PCI设备结构，并不是rte_eth_dev设备。前者标识一个PCI设备，后者标识一个网卡设备。然后依次读取每个PCI设备目录下的vendor,device等文件，填充刚分配出来的PCI设备结构。接下来使用`pci_get_kernel_driver_by_path()`获取设备驱动的类型。比如使用82599的网卡，就会看到类型为igb_uio，设置对应的驱动类型。

```c
if (!ret) {
	if (!strcmp(driver, "vfio-pci"))
		dev->kdrv = RTE_KDRV_VFIO;
	else if (!strcmp(driver, "igb_uio"))
		dev->kdrv = RTE_KDRV_IGB_UIO;
	else if (!strcmp(driver, "uio_pci_generic"))
		dev->kdrv = RTE_KDRV_UIO_GENERIC;
	else
		dev->kdrv = RTE_KDRV_UNKNOWN;
} else
	dev->kdrv = RTE_KDRV_NONE;
```

最后，把PCI设备挂在`pci_device_list`中：如果设备队列是空的，则直接挂上，如果不是空的，则按照PCI地址排序后挂接在队列中。

这样，第一阶段的工作就做完了，主要是扫描PCI的所有设备，填充设备信息，挂接在队列中。

<2>.`rte_eal_pci_probe()`函数进入了第二阶段的初始化。
先进行了一个设备参数类型的检查，`rte_eal_devargs_type_count()`，在这里又涉及到另一个变量---`devargs_list`，这个全局变量记录着哪些设备的PCI是在白或者黑名单里面，如果是在黑名单里，后面就不进行初始化。这个`devargs_list`的添加注册是在参数解析部分，-w,-b参数指定的名单。

然后依次遍历队列中的每个PCI设备，和devargs_list比较，查看是否有设备在列表中，如果在黑名单中，就不进行初始化。

之后调用`pci_probe_all_drivers()`对每个允许的设备进行初始化。

```c
TAILQ_FOREACH(dr, &pci_driver_list, next) {
	rc = rte_eal_pci_probe_one_driver(dr, dev);
	if (rc < 0)
		/* negative value is an error */
		return -1;
	if (rc > 0)
		/* positive value means driver doesn't support it */
		continue;
	return 0;
}
```

和每个注册的驱动进行比较，注册的驱动都挂接在`pci_driver_list`中，驱动的注册是通过下面的一段代码实现的

```c
#define RTE_PMD_REGISTER_PCI(nm, pci_drv) \
RTE_INIT(pciinitfn_ ##nm); \
static void pciinitfn_ ##nm(void) \
{\
	(pci_drv).driver.name = RTE_STR(nm);\
	rte_eal_pci_register(&pci_drv); \
} \
RTE_PMD_EXPORT_NAME(nm, __COUNTER__)
```

这里注意注册函数的类型为析构函数，gcc的补充。它是在main函数之前就执行的，所以，在main之前，驱动就已经注册好了。

```c
#define RTE_INIT(func) \
static void __attribute__((constructor, used)) func(void)
```

进一步查看的话，发现系统注册了这么几种类型的驱动：
(1).rte_igb_pmd
(2).rte_igbvf_pmd
(3).rte_ixgbe_pmd
.....

如rte_ixgbe_pmd驱动

```c
static struct eth_driver rte_ixgbe_pmd = {
.pci_drv = {
	.id_table = pci_id_ixgbe_map,
	.drv_flags = RTE_PCI_DRV_NEED_MAPPING | RTE_PCI_DRV_INTR_LSC,
	.probe = rte_eth_dev_pci_probe,
	.remove = rte_eth_dev_pci_remove,
},
.eth_dev_init = eth_ixgbe_dev_init,
.eth_dev_uninit = eth_ixgbe_dev_uninit,
.dev_private_size = sizeof(struct ixgbe_adapter),
};
```

其中的id_table表中就存放了各种支持的ixgbe设备的vendor号等详细信息。

接下来，自然的，如果匹配上了，就调用对应的驱动probe函数。进入`rte_eal_pci_probe_one_driver()`函数进行匹配。
当匹配成功后，对PCI资源进行映射--`rte_eal_pci_map_device()`，这个函数就不进行细细分析了。
最重要的地方到了，匹配成功后，就调用了`dr->probe`函数，对于ixgbe驱动，就是`rte_eth_dev_pci_probe()`函数，我们跳进去看看这个probe函数。
首先检查进程如果为RTE_PROC_PRIMARY类型的，那么就分配一个rte_eth_dev设备，调用`rte_eth_dev_allocate()`，分配可用的port_id,然后如果rte_eth_dev_data没有分 配，则一下子分配RTE_MAX_ETHPORTS个这个结构，这个结构描述了每个网卡的数据信息，并把对应port_id的rte_eth_dev_data[port_id]关联到新分配的设备上。

设备创建好了以后，就给设备的私有数据分配空间，

```c
eth_dev->data->dev_private = rte_zmalloc("ethdev private structure",
			  eth_drv->dev_private_size,
			  RTE_CACHE_LINE_SIZE);
```

然后填充设备的device,driver信息等。最后调用设备的初始化函数--`eth_drv->eth_dev_init`，这在ixgbe驱动中，是`eth_ixgbe_dev_init()`。

从这个初始化函数，进入最后的初始化环节。
要知道的一点是：在这个函数中，很多的工作肯定还是填充分配的设备结构体。先填充了设备的操作函数，以及非常重要的收发包函数

```c
eth_dev->dev_ops = &ixgbe_eth_dev_ops;
eth_dev->rx_pkt_burst = &ixgbe_recv_pkts;
eth_dev->tx_pkt_burst = &ixgbe_xmit_pkts;
eth_dev->tx_pkt_prepare = &ixgbe_prep_pkts;
```

再检查如果不是RTE_PROC_PRIMARY进程，则只要检查一下收发函数，并不进一步设置。
然后拷贝一下pci设备的相关信息

```c
rte_eth_copy_pci_info(eth_dev, pci_dev);
eth_dev->data->dev_flags |= RTE_ETH_DEV_DETACHABLE;
 
/* Vendor and Device ID need to be set before init of shared code */
hw->device_id = pci_dev->id.device_id;
hw->vendor_id = pci_dev->id.vendor_id;
hw->hw_addr = (void *)pci_dev->mem_resource[0].addr;
hw->allow_unsupported_sfp = 1;
```

接下来针对对应的设备，调用`ixgbe_init_shared_code()`根据`hw->device_id`来初始化特定的设备的MAC层操作函数集，ixgbe_mac_operations，如82599设备。

上面的操作都完成后，就可以调用`ixgbe_init_hw()`对硬件进行初始化了，初始化的函数在上一步MAC层函数操作集已经初始化。

然后重置设备的硬件统计，分配MAC地址，最后初始化一下各种过滤条件。

Done!!整个PCI驱动的匹配和初始化过程就完成了。



原创：[AISEED](https://home.cnblogs.com/u/yhp-smarthome/)    本文链接：https://www.cnblogs.com/yhp-smarthome/p/6690401.html
# DPDK 网卡驱动学习 

DPDK版本19.02

## 1.初始化：

```
/* Launch threads, called at application init(). */
int
rte_eal_init(int argc, char **argv)
{
    ...
    /* rte_eal_cpu_init() ->
     *     eal_cpu_core_id()
     *     eal_cpu_socket_id()
     * 读取/sys/devices/system/[cpu|node]
     * 设置lcore_config->[core_role|core_id|socket_id] */
    if (rte_eal_cpu_init() < 0) {
        rte_eal_init_alert("Cannot detect lcores.");
        rte_errno = ENOTSUP;
        return -1;
    }

    /* eal_parse_args() ->
     *     eal_parse_common_option() ->
     *         eal_parse_coremask()
     *         eal_parse_master_lcore()
     *         eal_parse_lcores()
     *     eal_adjust_config()
     * 解析-c、--master_lcore、--lcores参数
     * 在eal_parse_lcores()中确认可用的logical CPU
     * 在eal_adjust_config()中设置rte_config.master_lcore为0 (设置第一个lcore为MASTER lcore) */
    fctret = eal_parse_args(argc, argv);
    if (fctret < 0) {
        rte_eal_init_alert("Invalid 'command line' arguments.");
        rte_errno = EINVAL;
        rte_atomic32_clear(&run_once);
        return -1;
    }
    ...
    /* 初始化大页信息 */
    if (rte_eal_memory_init() < 0) {
        rte_eal_init_alert("Cannot init memory\n");
        rte_errno = ENOMEM;
        return -1;
    }
    ...
    /* eal_thread_init_master() ->
     *     eal_thread_set_affinity()
     * 设置当前线程为MASTER lcore
     * 在eal_thread_set_affinity()中绑定MASTER lcore到logical CPU */
    eal_thread_init_master(rte_config.master_lcore);
    ...
    /* rte_bus_scan() ->
     *     rte_pci_scan() ->
     *         pci_scan_one() ->
     *             pci_parse_sysfs_resource()
     *             rte_pci_add_device()
     * 遍历rte_bus_list链表，调用每个bus的scan函数，pci为rte_pci_scan()
     * 遍历/sys/bus/pci/devices目录，为每个DBSF分配struct rte_pci_device
     * 逐行读取并解析每个DBSF的resource，保存到dev->mem_resource[i]
     * 将dev插入rte_pci_bus.device_list链表 */
    if (rte_bus_scan()) {
        rte_eal_init_alert("Cannot scan the buses for devices\n");
        rte_errno = ENODEV;
        return -1;
    }

    /* pthread_create() ->
     *     eal_thread_loop() ->
     *         eal_thread_set_affinity()
     * 为每个SLAVE lcore创建线程，线程函数为eal_thread_loop()
     * 在eal_thread_set_affinity()中绑定SLAVE lcore到logical CPU */
    RTE_LCORE_FOREACH_SLAVE(i) {

        /*
         * create communication pipes between master thread
         * and children
         */
        /* MASTER lcore创建pipes用于MASTER和SLAVE lcore间通信（父子线程间通信） */
        if (pipe(lcore_config[i].pipe_master2slave) < 0)
            rte_panic("Cannot create pipe\n");
        if (pipe(lcore_config[i].pipe_slave2master) < 0)
            rte_panic("Cannot create pipe\n");

        lcore_config[i].state = WAIT; /* 设置SLAVE lcore的状态为WAIT */

        /* create a thread for each lcore */
        ret = pthread_create(&lcore_config[i].thread_id, NULL,
                     eal_thread_loop, NULL);
        ...
    }

    /*
     * Launch a dummy function on all slave lcores, so that master lcore
     * knows they are all ready when this function returns.
     */
    rte_eal_mp_remote_launch(sync_func, NULL, SKIP_MASTER);
    rte_eal_mp_wait_lcore();
    ...
    /* Probe all the buses and devices/drivers on them */
    /* rte_bus_probe() ->
     *     rte_pci_probe() ->
     *         pci_probe_all_drivers() ->
     *             rte_pci_probe_one_driver() ->
     *                 rte_pci_match()
     *                 rte_pci_map_device() ->
     *                     pci_uio_map_resource()
     *                 eth_ixgbe_pci_probe()
     * 遍历rte_bus_list链表，调用每个bus的probe函数，pci为rte_pci_probe()
     * rte_pci_probe()/pci_probe_all_drivers()分别遍历rte_pci_bus.device_list/driver_list链表，匹配设备和驱动
     * 映射BAR，调用驱动的probe函数，ixgbe为eth_ixgbe_pci_probe() */
    if (rte_bus_probe()) {
        rte_eal_init_alert("Cannot probe devices\n");
        rte_errno = ENOTSUP;
        return -1;
    }
    ...
}
```

在dpdk 16.11是没有bud这一层抽象的，直接通过rte_eal_initàrte_eal_pci_init调用pci设备的初始化。

**也就是 dpdk 16.11 只支持pci 这一种总线设备**。但是到了dpdk17.11引入了bus的概念。在rte_bus_scan 进行bus的初始化

```
/* Scan all the buses for registered devices */
int
rte_bus_scan(void)
{
    int ret;
    struct rte_bus *bus = NULL;

    TAILQ_FOREACH(bus, &rte_bus_list, next) {
        ret = bus->scan();
        if (ret)
            RTE_LOG(ERR, EAL, "Scan for (%s) bus failed.\n",
                bus->name);
    }

    return 0;
}
```

这个函数会调用rte_bus_list上注册的所有bus的scan函数，这些bus是通过rte_bus_register函数注册上去的，而宏RTE_REGISTER_BUS又是rte_bus_register的封装。

## 2.重要结构体

### 2.1rte_bus_list

```
struct rte_bus {
    TAILQ_ENTRY(rte_bus) next;   /**< Next bus object in linked list */
    const char *name;            /**< Name of the bus */
    rte_bus_scan_t scan;         /**< Scan for devices attached to bus */
    rte_bus_probe_t probe;       /**< Probe devices on bus */
    rte_bus_find_device_t find_device; /**< Find a device on the bus */
    rte_bus_plug_t plug;         /**< Probe single device for drivers */
    rte_bus_unplug_t unplug;     /**< Remove single device from driver */
    rte_bus_parse_t parse;       /**< Parse a device name */
    struct rte_bus_conf conf;    /**< Bus configuration */
};

TAILQ_HEAD(rte_bus_list, rte_bus);

#define    TAILQ_HEAD(name, type)                        \
struct name {                                \
    struct type *tqh_first;    /* first element */            \
    struct type **tqh_last;    /* addr of last next element */        \
}

/* 定义rte_bus_list */
struct rte_bus_list rte_bus_list =
    TAILQ_HEAD_INITIALIZER(rte_bus_list);
```

### 2.2注册pci bus

将rte_pci_bus插入rte_bus_list链表

```
struct rte_pci_bus {
    struct rte_bus bus;               /**< Inherit the generic class */
    struct rte_pci_device_list device_list;  /**< List of PCI devices */
    struct rte_pci_driver_list driver_list;  /**< List of PCI drivers */
};

/* 定义rte_pci_bus */
struct rte_pci_bus rte_pci_bus = {
    .bus = {
        .scan = rte_pci_scan,
        .probe = rte_pci_probe,
        .find_device = pci_find_device,
        .plug = pci_plug,
        .unplug = pci_unplug,
        .parse = pci_parse,
    },
    .device_list = TAILQ_HEAD_INITIALIZER(rte_pci_bus.device_list),
    .driver_list = TAILQ_HEAD_INITIALIZER(rte_pci_bus.driver_list),
};

RTE_REGISTER_BUS(pci, rte_pci_bus.bus);

#define RTE_REGISTER_BUS(nm, bus) \
RTE_INIT_PRIO(businitfn_ ##nm, 101); \ /* 声明为gcc构造函数，先于main()执行 */
static void businitfn_ ##nm(void) \
{\
    (bus).name = RTE_STR(nm);\
    rte_bus_register(&bus); \
}

void
rte_bus_register(struct rte_bus *bus)
{
    RTE_VERIFY(bus);
    RTE_VERIFY(bus->name && strlen(bus->name));
    /* A bus should mandatorily have the scan implemented */
    RTE_VERIFY(bus->scan);
    RTE_VERIFY(bus->probe);
    RTE_VERIFY(bus->find_device);
    /* Buses supporting driver plug also require unplug. */
    RTE_VERIFY(!bus->plug || bus->unplug);

    /* 将rte_pci_bus.bus插入rte_bus_list链表 */
    TAILQ_INSERT_TAIL(&rte_bus_list, bus, next);
    RTE_LOG(DEBUG, EAL, "Registered [%s] bus.\n", bus->name);
}
```



### 2.3设备初始化过程：

**pci 设备的初始化是通过以下路径完成的：rte_eal_init à rte_bus_scan ->rte_pci_scan**。而对应驱动的加载则是通过如下调用完成的：rte_eal_init->rte_bus_probe->rte_pci_probe完成的。

 

## 3.注册pci driver

将rte_ixgbe_pmd插入rte_pci_bus.driver_list链表

```
struct rte_pci_driver {
    TAILQ_ENTRY(rte_pci_driver) next;       /**< Next in list. */
    struct rte_driver driver;               /**< Inherit core driver. */
    struct rte_pci_bus *bus;                /**< PCI bus reference. */
    pci_probe_t *probe;                     /**< Device Probe function. */
    pci_remove_t *remove;                   /**< Device Remove function. */
    const struct rte_pci_id *id_table;    /**< ID table, NULL terminated. */
    uint32_t drv_flags;                     /**< Flags contolling handling of device. */
};

/* 定义rte_ixgbe_pmd */
static struct rte_pci_driver rte_ixgbe_pmd = {
    .id_table = pci_id_ixgbe_map,
    .drv_flags = RTE_PCI_DRV_NEED_MAPPING | RTE_PCI_DRV_INTR_LSC,
    .probe = eth_ixgbe_pci_probe,
    .remove = eth_ixgbe_pci_remove,
};

RTE_PMD_REGISTER_PCI(net_ixgbe, rte_ixgbe_pmd);

#define RTE_PMD_REGISTER_PCI(nm, pci_drv) \
RTE_INIT(pciinitfn_ ##nm); \ /* 声明为gcc构造函数，先于main()执行 */
static void pciinitfn_ ##nm(void) \
{\
    (pci_drv).driver.name = RTE_STR(nm);\
    rte_pci_register(&pci_drv); \
} \
RTE_PMD_EXPORT_NAME(nm, __COUNTER__)

void
rte_pci_register(struct rte_pci_driver *driver)
{
    /* 将rte_ixgbe_pmd插入rte_pci_bus.driver_list链表 */
    TAILQ_INSERT_TAIL(&rte_pci_bus.driver_list, driver, next);
    driver->bus = &rte_pci_bus;
}
```



## 4.eth_ixgbe_dev_init()

```
static int
eth_ixgbe_dev_init(struct rte_eth_dev *eth_dev)
{
    ...
    eth_dev->dev_ops = &ixgbe_eth_dev_ops; /* 注册ixgbe_eth_dev_ops函数表 */
    eth_dev->rx_pkt_burst = &ixgbe_recv_pkts; /* burst收包函数 */
    eth_dev->tx_pkt_burst = &ixgbe_xmit_pkts; /* burst发包函数 */
    eth_dev->tx_pkt_prepare = &ixgbe_prep_pkts;
    ...
    hw->device_id = pci_dev->id.device_id; /* device_id */
    hw->vendor_id = pci_dev->id.vendor_id; /* vendor_id */
    hw->hw_addr = (void *)pci_dev->mem_resource[0].addr; /* mmap()得到的BAR的虚拟地址 */
    ...
    /* ixgbe_init_shared_code() ->
     *     ixgbe_set_mac_type()
     *     ixgbe_init_ops_82599()
     * 在ixgbe_set_mac_type()中根据vendor_id和device_id设置hw->mac.type，82599为ixgbe_mac_82599EB
     * 根据hw->mac.type调用对应的函数设置hw->mac.ops，82599为ixgbe_init_ops_82599() */
    diag = ixgbe_init_shared_code(hw);
    ...
    /* ixgbe_init_hw() ->
     *     ixgbe_call_func() ->
     *         ixgbe_init_hw_generic() ->
     *             ixgbe_reset_hw_82599() ->
     *                 ixgbe_get_mac_addr_generic()
     * 得到网卡的mac地址 */
    diag = ixgbe_init_hw(hw);
    ...
    ether_addr_copy((struct ether_addr *) hw->mac.perm_addr,
            &eth_dev->data->mac_addrs[0]); /* 复制网卡的mac地址到eth_dev->data->mac_addrs */
    ...
}

static const struct eth_dev_ops ixgbe_eth_dev_ops = {
    .dev_configure        = ixgbe_dev_configure,
    .dev_start            = ixgbe_dev_start,
    ...
    .rx_queue_setup       = ixgbe_dev_rx_queue_setup,
    ...
    .tx_queue_setup       = ixgbe_dev_tx_queue_setup,
    ...
};
```



### 4.1eth_ixgbe_pci_probe()

```
static int eth_ixgbe_pci_probe(struct rte_pci_driver *pci_drv __rte_unused,
    struct rte_pci_device *pci_dev)
{
    return rte_eth_dev_pci_generic_probe(pci_dev,
        sizeof(struct ixgbe_adapter), eth_ixgbe_dev_init);
}

static inline int
rte_eth_dev_pci_generic_probe(struct rte_pci_device *pci_dev,
    size_t private_data_size, eth_dev_pci_callback_t dev_init)
{
    ...
    eth_dev = rte_eth_dev_pci_allocate(pci_dev, private_data_size);
    ...
    ret = dev_init(eth_dev); /* ixgbe为eth_ixgbe_dev_init() */
    ...
}

static inline struct rte_eth_dev *
rte_eth_dev_pci_allocate(struct rte_pci_device *dev, size_t private_data_size)
{
    ...
    /* rte_eth_dev_allocate() ->
     *     rte_eth_dev_find_free_port()
     *     rte_eth_dev_data_alloc()
     *     eth_dev_get() */
    eth_dev = rte_eth_dev_allocate(name);
    ...
    /* 分配private data，ixgbe为struct ixgbe_adapter */
    eth_dev->data->dev_private = rte_zmalloc_socket(name,
        private_data_size, RTE_CACHE_LINE_SIZE,
        dev->device.numa_node);
    ...
}

struct rte_eth_dev *
rte_eth_dev_allocate(const char *name)
{
    ...
    /* 遍历rte_eth_devices数组，找到一个空闲的设备 */
    port_id = rte_eth_dev_find_free_port();
    ...
    /* 分配rte_eth_dev_data数组 */
    rte_eth_dev_data_alloc();
    ...
    /* 设置port_id对应的设备的state为RTE_ETH_DEV_ATTACHED */
    eth_dev = eth_dev_get(port_id);
    ...
}
```

## 5.ixgbe_recv_pkts()

接收时回写：
1、网卡使用DMA写Rx FIFO中的Frame到Rx Ring Buffer中的mbuf，设置desc的DD为1
2、网卡驱动取走mbuf后，设置desc的DD为0，更新RDT

```
uint16_t
ixgbe_recv_pkts(void *rx_queue, struct rte_mbuf **rx_pkts,
        uint16_t nb_pkts)
{
    ...
    nb_rx = 0;
    nb_hold = 0;
    rxq = rx_queue;
    rx_id = rxq->rx_tail; /* 相当于ixgbe的next_to_clean */
    rx_ring = rxq->rx_ring;
    sw_ring = rxq->sw_ring;
    ...
    while (nb_rx < nb_pkts) {
        ...
        /* 得到rx_tail指向的desc的指针 */
        rxdp = &rx_ring[rx_id];
        /* 若网卡回写的DD为0，跳出循环 */
        staterr = rxdp->wb.upper.status_error;
        if (!(staterr & rte_cpu_to_le_32(IXGBE_RXDADV_STAT_DD)))
            break;
        /* 得到rx_tail指向的desc */
        rxd = *rxdp;
        ...
        /* 分配新mbuf */
        nmb = rte_mbuf_raw_alloc(rxq->mb_pool);
        ...
        nb_hold++; /* 统计接收的mbuf数 */
        rxe = &sw_ring[rx_id]; /* 得到旧mbuf */
        rx_id++; /* 得到下一个desc的index，注意是一个环形缓冲区 */
        if (rx_id == rxq->nb_rx_desc)
            rx_id = 0;
        ...
        rte_ixgbe_prefetch(sw_ring[rx_id].mbuf); /* 预取下一个mbuf */
        ...
        if ((rx_id & 0x3) == 0) {
            rte_ixgbe_prefetch(&rx_ring[rx_id]);
            rte_ixgbe_prefetch(&sw_ring[rx_id]);
        }
        ...
        rxm = rxe->mbuf; /* rxm指向旧mbuf */
        rxe->mbuf = nmb; /* rxe->mbuf指向新mbuf */
        dma_addr =
            rte_cpu_to_le_64(rte_mbuf_data_dma_addr_default(nmb)); /* 得到新mbuf的总线地址 */
        rxdp->read.hdr_addr = 0; /* 清零新mbuf对应的desc的DD，后续网卡会读desc */
        rxdp->read.pkt_addr = dma_addr; /* 设置新mbuf对应的desc的总线地址，后续网卡会读desc */
        ...
        pkt_len = (uint16_t) (rte_le_to_cpu_16(rxd.wb.upper.length) -
                      rxq->crc_len); /* 包长 */
        rxm->data_off = RTE_PKTMBUF_HEADROOM;
        rte_packet_prefetch((char *)rxm->buf_addr + rxm->data_off);
        rxm->nb_segs = 1;
        rxm->next = NULL;
        rxm->pkt_len = pkt_len;
        rxm->data_len = pkt_len;
        rxm->port = rxq->port_id;
        ...
        if (likely(pkt_flags & PKT_RX_RSS_HASH)) /* RSS */
            rxm->hash.rss = rte_le_to_cpu_32(
                        rxd.wb.lower.hi_dword.rss);
        else if (pkt_flags & PKT_RX_FDIR) { /* FDIR */
            rxm->hash.fdir.hash = rte_le_to_cpu_16(
                    rxd.wb.lower.hi_dword.csum_ip.csum) &
                    IXGBE_ATR_HASH_MASK;
            rxm->hash.fdir.id = rte_le_to_cpu_16(
                    rxd.wb.lower.hi_dword.csum_ip.ip_id);
        }
        ...
        rx_pkts[nb_rx++] = rxm; /* 将旧mbuf放入rx_pkts数组 */
    }
    rxq->rx_tail = rx_id; /* rx_tail指向下一个desc */
    ...
    nb_hold = (uint16_t) (nb_hold + rxq->nb_rx_hold);
    /* 若已处理的mbuf数大于上限（默认为32），更新RDT */
    if (nb_hold > rxq->rx_free_thresh) {
        ...
        rx_id = (uint16_t) ((rx_id == 0) ?
                     (rxq->nb_rx_desc - 1) : (rx_id - 1));
        IXGBE_PCI_REG_WRITE(rxq->rdt_reg_addr, rx_id); /* 将rx_id写入RDT */
        nb_hold = 0; /* 清零nb_hold */
    }
    rxq->nb_rx_hold = nb_hold; /* 更新nb_rx_hold */
    return nb_rx;
}
```



## 6.ixgbe_xmit_pkts()

```
uint16_t
ixgbe_xmit_pkts(void *tx_queue, struct rte_mbuf **tx_pkts,
        uint16_t nb_pkts)
{
    ...
    txq = tx_queue;
    sw_ring = txq->sw_ring;
    txr     = txq->tx_ring;
    tx_id   = txq->tx_tail; /* 相当于ixgbe的next_to_use */
    txe = &sw_ring[tx_id]; /* 得到tx_tail指向的entry */
    txp = NULL;
    ...
    /* 若空闲的mbuf数小于下限（默认为32），清理空闲的mbuf */
    if (txq->nb_tx_free < txq->tx_free_thresh)
        ixgbe_xmit_cleanup(txq);
    ...
    /* TX loop */
    for (nb_tx = 0; nb_tx < nb_pkts; nb_tx++) {
        ...
        tx_pkt = *tx_pkts++; /* 待发送的mbuf */
        pkt_len = tx_pkt->pkt_len; /* 待发送的mbuf的长度 */
        ...
        nb_used = (uint16_t)(tx_pkt->nb_segs + new_ctx); /* 使用的desc数 */
        ...
        tx_last = (uint16_t) (tx_id + nb_used - 1); /* tx_last指向最后一个desc */
        ...
        if (tx_last >= txq->nb_tx_desc) /* 注意是一个环形缓冲区 */
            tx_last = (uint16_t) (tx_last - txq->nb_tx_desc);
        ...
        if (nb_used > txq->nb_tx_free) {
            ...
            if (ixgbe_xmit_cleanup(txq) != 0) {
                /* Could not clean any descriptors */
                if (nb_tx == 0) /* 若是第一个包（未发包），return 0 */
                    return 0;
                goto end_of_tx; /* 若非第一个包（已发包），停止发包，更新发送队列参数 */
            }
            ...
        }
        ...
        /* 每个包可能包含多个分段，m_seg指向第一个分段 */
        m_seg = tx_pkt;
        do {
            txd = &txr[tx_id]; /* desc */
            txn = &sw_ring[txe->next_id]; /* 下一个entry */
            ...
            txe->mbuf = m_seg; /* 将m_seg挂载到txe */
            ...
            slen = m_seg->data_len; /* m_seg的长度 */
            buf_dma_addr = rte_mbuf_data_dma_addr(m_seg); /* m_seg的总线地址 */
            txd->read.buffer_addr =
                rte_cpu_to_le_64(buf_dma_addr); /* 总线地址赋给txd->read.buffer_addr */
            txd->read.cmd_type_len =
                rte_cpu_to_le_32(cmd_type_len | slen); /* 长度赋给txd->read.cmd_type_len */
            ...
            txe->last_id = tx_last; /* last_id指向最后一个desc */
            tx_id = txe->next_id; /* tx_id指向下一个desc */
            txe = txn; /* txe指向下一个entry */
            m_seg = m_seg->next; /* m_seg指向下一个分段 */
        } while (m_seg != NULL);
        ...
        /* 最后一个分段 */
        cmd_type_len |= IXGBE_TXD_CMD_EOP;
        txq->nb_tx_used = (uint16_t)(txq->nb_tx_used + nb_used); /* 更新nb_tx_used */
        txq->nb_tx_free = (uint16_t)(txq->nb_tx_free - nb_used); /* 更新nb_tx_free */
        ...
        if (txq->nb_tx_used >= txq->tx_rs_thresh) { /* 若使用的mbuf数大于上限（默认为32），设置RS */
            ...
            cmd_type_len |= IXGBE_TXD_CMD_RS;
            ...
            txp = NULL; /* txp为NULL表示已设置RS */
        } else
            txp = txd; /* txp非NULL表示未设置RS */
        ...
        txd->read.cmd_type_len |= rte_cpu_to_le_32(cmd_type_len);
    }
    ...
end_of_tx:
    /* burst发包的最后一个包的最后一个分段 */
    ...
    if (txp != NULL) /* 若未设置RS，设置RS */
        txp->read.cmd_type_len |= rte_cpu_to_le_32(IXGBE_TXD_CMD_RS);
    ...
    IXGBE_PCI_REG_WRITE_RELAXED(txq->tdt_reg_addr, tx_id); /* 将tx_id写入TDT */
    txq->tx_tail = tx_id; /* tx_tail指向下一个desc */
    ...
    return nb_tx;
}
```

## ![img](https://img2020.cnblogs.com/blog/1414775/202006/1414775-20200611132756771-1342548066.png)7.rte_eth_rx/tx_burst()

```
static inline uint16_t
rte_eth_rx_burst(uint8_t port_id, uint16_t queue_id,
         struct rte_mbuf **rx_pkts, const uint16_t nb_pkts)
{
    /* 得到port_id对应的设备 */
    struct rte_eth_dev *dev = &rte_eth_devices[port_id];
    ...
    /* ixgbe为ixgbe_recv_pkts() */
    int16_t nb_rx = (*dev->rx_pkt_burst)(dev->data->rx_queues[queue_id],
            rx_pkts, nb_pkts);
    ...
}

static inline uint16_t
rte_eth_tx_burst(uint8_t port_id, uint16_t queue_id,
         struct rte_mbuf **tx_pkts, uint16_t nb_pkts)
{
    /* 得到port_id对应的设备 */
    struct rte_eth_dev *dev = &rte_eth_devices[port_id];
    ...
    /* ixgbe为ixgbe_xmit_pkts */
    return (*dev->tx_pkt_burst)(dev->data->tx_queues[queue_id], tx_pkts, nb_pkts);
}
```

### 7.1从描述符中获取报文返回给应用层

首先根据应用层最后一次获取报文的位置，进而从描述符队列找到待被应用层接收的描述符。此时会判断描述符中的status_error是否已经打上了dd标记，有dd标记说明dma控制器已经把报文放到mbuf中了。这里解释下dd标记，当dma控制器将接收到的报文保存到描述符指向的mbuf空间时，由dma控制器打上dd标记，表示dma控制器已经把报文放到mbuf中了。应用层在获取完报文后，需要清除dd标记。

找到了描述符的位置，也就找到了mbuf空间。此时会根据描述符里面保存的信息，填充mbuf结构。例如填充报文的长度，vlanid, rss等信息。填充完mbuf后，将这个mbuf保存到应用层传进来的结构中，返回给应用层，这应用层就获取到了这个报文。

```
uint16_t eth_igb_recv_pkts(void *rx_queue, struct rte_mbuf **rx_pkts, uint16_t nb_pkts)
{
    while (nb_rx < nb_pkts)
    {
        //从描述符队列中找到待被应用层最后一次接收的那个描述符位置
        rxdp = &rx_ring[rx_id];
        staterr = rxdp->wb.upper.status_error;
        //检查状态是否为dd, 不是则说明驱动还没有把报文放到接收队列，直接退出
        if (! (staterr & rte_cpu_to_le_32(E1000_RXD_STAT_DD)))
        {
            break;
        };
        //找到了描述符的位置，也就从软件队列中找到了mbuf
        rxe = &sw_ring[rx_id];
        rx_id++;
        rxm = rxe->mbuf;
        //填充mbuf
        pkt_len = (uint16_t) (rte_le_to_cpu_16(rxd.wb.upper.length) - rxq->crc_len);
        rxm->data_off = RTE_PKTMBUF_HEADROOM;
        rxm->nb_segs = 1;
        rxm->pkt_len = pkt_len;
        rxm->data_len = pkt_len;
        rxm->port = rxq->port_id;
        rxm->hash.rss = rxd.wb.lower.hi_dword.rss;
        rxm->vlan_tci = rte_le_to_cpu_16(rxd.wb.upper.vlan);
        //保存到应用层
        rx_pkts[nb_rx++] = rxm;
    }
}
```

### 7.2从内存池中获取新的mbuf告诉dma控制器

当应用层从软件队列中获取到mbuf后， 需要重新从内存池申请一个mbuf空间，并将mbuf地址放到描述符队列中， 相当于告诉dma控制器，后续将收到的报文保存到这个新的mbuf中， 这也是狸猫换太子的过程。描述符是mbuf与dma控制器的中介，那dma控制器怎么知道描述符队列的地址呢？这在上一篇文章中已经介绍过了，将描述符队列的地址写入到了寄存器中，dma控制器通过读取寄存器就知道描述符队列的地址。

需要注意的是，将mbuf的地址保存到描述符中，此时会将dd标记给清0，这样dma控制器就认为这个mbuf里面的内容已经被应用层接收了，收到新报文后可以重新放到这个mbuf中。

```
uint16_t eth_igb_recv_pkts(void *rx_queue, struct rte_mbuf **rx_pkts, uint16_t nb_pkts)
{
    while (nb_rx < nb_pkts)
    {
        //申请一个新的mbuf
        nmb = rte_rxmbuf_alloc(rxq->mb_pool);
        //因为原来的mbuf被应用层取走了。这里替换原来的软件队列mbuf，这样网卡收到报文后可以放到这个新的mbuf
        rxe->mbuf = nmb;
        dma_addr = rte_cpu_to_le_64(RTE_MBUF_DATA_DMA_ADDR_DEFAULT(nmb));
        //将mbuf地址保存到描述符中，相当于高速dma控制器mbuf的地址。
        rxdp->read.hdr_addr = dma_addr;            //这里会将dd标记清0
        rxdp->read.pkt_addr = dma_addr;
    }
}
```





**原文地址：https://www.cnblogs.com/mysky007/p/11219593.html**
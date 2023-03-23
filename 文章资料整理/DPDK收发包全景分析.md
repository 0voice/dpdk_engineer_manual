# DPDK收发包全景分析

前言：DPDK收发包是基础核心模块，从网卡收到包到驱动把包拷贝到系统内存中，再到系统对这块数据包的内存管理，由于在处理过程中实现了零拷贝，数据包从接收到发送始终只有一份，对这个报文的管理在前面的mempool内存池中有过介绍。这篇主要介绍收发包的过程。

### 一、收发包分解

收发包过程大致可以分为2个部分

- 1.收发包的配置和初始化，主要是配置收发队列等。
- 2.数据包的获取和发送，主要是从队列中获取到数据包或者把数据包放到队列中。

### 二、收发包的配置和初始化

##### 收发包的配置

收发包的配置最主要的工作就是配置网卡的收发队列，设置DMA拷贝数据包的地址等，配置好地址后，网卡收到数据包后会通过DMA控制器直接把数据包拷贝到指定的内存地址。我们使用数据包时，只要去对应队列取出指定地址的数据即可。

收发包的配置是从`rte_eth_dev_configure()`开始的，这里根据参数会配置队列的个数，以及接口的配置信息，如队列的使用模式，多队列的方式等。

前面会先进行一些各项检查，如果设备已经启动，就得先停下来才能配置（这时应该叫再配置吧）。然后把传进去的配置参数拷贝到设备的数据区。

```
memcpy(&dev->data->dev_conf, dev_conf, sizeof(dev->data->dev_conf));
```

之后获取设备的信息，主要也是为了后面的检查使用：

```
(*dev->dev_ops->dev_infos_get)(dev, &dev_info);
```

这里的dev_infos_get是在驱动初始化过程中设备初始化时配置的(eth_ixgbe_dev_init())

```
eth_dev->dev_ops = &ixgbe_eth_dev_ops;
```

重要的信息检查过后，下面就是对发送和接收队列进行配置 先看接收队列的配置，接收队列是从`rte_eth_dev_tx_queue_config()`开始的 在接收配置中，考虑的是有两种情况，一种是第一次配置；另一种是重新配置。所以，代码中都做了区分。 (1)如果是第一次配置，那么就为每个队列分配一个指针。 (2)如果是重新配置，配置的queue数量不为0，那么就取消之前的配置，重新配置。 (3)如果是重新配置，但要求的queue为0，那么释放已有的配置。

发送的配置也是同样的，在`rte_eth_dev_tx_queue_config()`。

当收发队列配置完成后，就调用设备的配置函数，进行最后的配置。`(*dev->dev_ops->dev_configure)(dev)`，我们找到对应的配置函数，进入`ixgbe_dev_configure()`来分析其过程,其实这个函数并没有做太多的事。

在函数中，先调用了`ixgbe_check_mq_mode()`来检查队列的模式。然后设置允许接收批量和向量的模式

```
adapter->rx_bulk_alloc_allowed = true;
adapter->rx_vec_allowed = true;
```

接下来就是收发队列的初始化，非常关键的一部分内容，这部分内容按照收发分别介绍：

##### 接收队列的初始化

接收队列的初始化是从`rte_eth_rx_queue_setup()`开始的，这里的参数需要指定要初始化的port_id,queue_id,以及描述符的个数，还可以指定接收的配置，如释放和回写的阈值等。

依然如其他函数的套路一样，先进行各种检查，如初始化的队列号是否合法有效，设备如果已经启动，就不能继续初始化了。检查函数指针是否有效等。检查mbuf的数据大小是否满足默认的设备信息里的配置。

```
rte_eth_dev_info_get(port_id, &dev_info);
```

这里获取了设备的配置信息，如果调用初始化函数时没有指定rx_conf配置，就会设备配置信息里的默认值

```
dev_info->default_rxconf = (struct rte_eth_rxconf) {
        .rx_thresh = {
            .pthresh = IXGBE_DEFAULT_RX_PTHRESH,
            .hthresh = IXGBE_DEFAULT_RX_HTHRESH,
            .wthresh = IXGBE_DEFAULT_RX_WTHRESH,
        },
        .rx_free_thresh = IXGBE_DEFAULT_RX_FREE_THRESH,
        .rx_drop_en = 0,
    };
```

还检查了要初始化的队列号对应的队列指针是否为空，如果不为空，则说明这个队列已经初始化过了，就释放这个队列。

```
rxq = dev->data->rx_queues;
    if (rxq[rx_queue_id]) {
        RTE_FUNC_PTR_OR_ERR_RET(*dev->dev_ops->rx_queue_release,
                    -ENOTSUP);
        (*dev->dev_ops->rx_queue_release)(rxq[rx_queue_id]);
        rxq[rx_queue_id] = NULL;
    }
```

最后，调用到队列的setup函数做最后的初始化。

```
ret = (*dev->dev_ops->rx_queue_setup)(dev, rx_queue_id, nb_rx_desc,
                          socket_id, rx_conf, mp);
```

对于ixgbe设备，rx_queue_setup就是函数`ixgbe_dev_rx_queue_setup()`，这里就是队列最终的初始化咯

依然是先检查，检查描述符的数量最大不能大于IXGBE_MAX_RING_DESC个，最小不能小于IXGBE_MIN_RING_DESC个。

接下来的都是重点咯： <1>.分配队列结构体，并填充结构

```
rxq = rte_zmalloc_socket("ethdev RX queue", sizeof(struct ixgbe_rx_queue),
                 RTE_CACHE_LINE_SIZE, socket_id);
```

填充结构体的所属内存池，描述符个数，队列号，队列所属接口号等成员。 <2>.分配描述符队列的空间，按照最大的描述符个数进行分配

```
rz = rte_eth_dma_zone_reserve(dev, "rx_ring", queue_idx,
                      RX_RING_SZ, IXGBE_ALIGN, socket_id);
```

接着获取描述符队列的头和尾寄存器的地址，在收发包后，软件要对这个寄存器进行处理。

```
rxq->rdt_reg_addr =
            IXGBE_PCI_REG_ADDR(hw, IXGBE_RDT(rxq->reg_idx));
        rxq->rdh_reg_addr =
            IXGBE_PCI_REG_ADDR(hw, IXGBE_RDH(rxq->reg_idx));
```

设置队列的接收描述符ring的物理地址和虚拟地址。

```
rxq->rx_ring_phys_addr = rte_mem_phy2mch(rz->memseg_id, rz->phys_addr);
    rxq->rx_ring = (union ixgbe_adv_rx_desc *) rz->addr;
```

<3>分配sw_ring，这个ring中存储的对象是`struct ixgbe_rx_entry`，其实里面就是数据包mbuf的指针。

```
rxq->sw_ring = rte_zmalloc_socket("rxq->sw_ring",
                      sizeof(struct ixgbe_rx_entry) * len,
                      RTE_CACHE_LINE_SIZE, socket_id);
```

以上三步做完以后，新分配的队列结构体重要的部分就已经填充完了，下面需要重置一下其他成员

```
ixgbe_reset_rx_queue()
```

先把分配的描述符队列清空，其实清空在分配的时候就已经做了，没必要重复做

```
for (i = 0; i < len; i++) {
        rxq->rx_ring[i] = zeroed_desc;
    }
```

然后初始化队列中一下其他成员

```
rxq->rx_nb_avail = 0;
rxq->rx_next_avail = 0;
rxq->rx_free_trigger = (uint16_t)(rxq->rx_free_thresh - 1);
rxq->rx_tail = 0;
rxq->nb_rx_hold = 0;
rxq->pkt_first_seg = NULL;
rxq->pkt_last_seg = NULL;
```

这样，接收队列就初始化完了。

##### 发送队列的初始化

发送队列的初始化在前面的检查基本和接收队列一样，只有些许区别在于setup环节，我们就从这个函数说起：`ixgbe_dev_tx_queue_setup()`。

在发送队列配置中，重点设置了`tx_rs_thresh`和`tx_free_thresh`的值。

然后分配了一个发送队列结构txq,之后分配发送队列ring的空间，并填充txq的结构体

```
txq->tx_ring_phys_addr = rte_mem_phy2mch(tz->memseg_id, tz->phys_addr);
    txq->tx_ring = (union ixgbe_adv_tx_desc *) tz->addr;
```

然后，分配队列的sw_ring,也挂载队列上。

重置发送队列

```
ixgbe_reset_tx_queue()
```

和接收队列一样，也是要把队列ring（描述符ring）清空，设置发送队列sw_ring,设置其他参数，队尾位置设置为0

```
txq->tx_next_dd = (uint16_t)(txq->tx_rs_thresh - 1);
txq->tx_next_rs = (uint16_t)(txq->tx_rs_thresh - 1);
 
txq->tx_tail = 0;
txq->nb_tx_used = 0;
/*
 * Always allow 1 descriptor to be un-allocated to avoid
 * a H/W race condition
 */
txq->last_desc_cleaned = (uint16_t)(txq->nb_tx_desc - 1);
txq->nb_tx_free = (uint16_t)(txq->nb_tx_desc - 1);
txq->ctx_curr = 0;
```

发送队列的初始化就完成了。

##### 设备的启动

经过上面的队列初始化，队列的ring和sw_ring都分配了，但是发现木有，DMA仍然还不知道要把数据包拷贝到哪里，我们说过，DPDK是零拷贝的，那么我们分配的mempool中的对象怎么和队列以及驱动联系起来呢？接下来就是最精彩的时刻了----建立mempool、queue、DMA、ring之间的关系。话说，这个为什么不是在队列的初始化中就做呢？

设备的启动是从`rte_eth_dev_start()`中开始的

```
diag = (*dev->dev_ops->dev_start)(dev);
```

进而，找到设备启动的真正启动函数：`ixgbe_dev_start()`

先检查设备的链路设置，暂时不支持半双工和固定速率的模式。看来是暂时只有自适应模式咯。

然后把中断禁掉，同时，停掉适配器

```
ixgbe_stop_adapter(hw);
```

在其中，就是调用了`ixgbe_stop_adapter_generic();`，主要的工作就是停止发送和接收单元。这是直接写寄存器来完成的。

然后重启硬件，ixgbe_pf_reset_hw()->ixgbe_reset_hw()->ixgbe_reset_hw_82599(),最终都是设置寄存器，这里就不细究了。之后，就启动了硬件。

再然后是初始化接收单元：`ixgbe_dev_rx_init()`

在这个函数中，主要就是设置各类寄存器，比如配置CRC校验，如果支持巨帧，配置对应的寄存器。还有如果配置了loopback模式，也要配置寄存器。

接下来最重要的就是为每个队列设置DMA寄存器，标识每个队列的描述符ring的地址，长度，头，尾等。

```
bus_addr = rxq->rx_ring_phys_addr;
IXGBE_WRITE_REG(hw, IXGBE_RDBAL(rxq->reg_idx),
        (uint32_t)(bus_addr & 0x00000000ffffffffULL));
IXGBE_WRITE_REG(hw, IXGBE_RDBAH(rxq->reg_idx),
        (uint32_t)(bus_addr >> 32));
IXGBE_WRITE_REG(hw, IXGBE_RDLEN(rxq->reg_idx),
        rxq->nb_rx_desc * sizeof(union ixgbe_adv_rx_desc));
IXGBE_WRITE_REG(hw, IXGBE_RDH(rxq->reg_idx), 0);
IXGBE_WRITE_REG(hw, IXGBE_RDT(rxq->reg_idx), 0);
```

这里可以看到把描述符ring的物理地址写入了寄存器，还写入了描述符ring的长度。

下面还计算了数据包数据的长度，写入到寄存器中.然后对于网卡的多队列设置,也进行了配置

```
ixgbe_dev_mq_rx_configure()
```

同时如果设置了接收校验和，还对校验和进行了寄存器设置。

最后，调用`ixgbe_set_rx_function()`对接收函数再进行设置，主要是针对支持LRO，vector,bulk等处理方法。

这样，接收单元的初始化就完成了。

接下来再初始化发送单元：`ixgbe_dev_tx_init()`

发送单元的的初始化和接收单元的初始化基本操作是一样的，都是填充寄存器的值，重点是设置描述符队列的基地址和长度。

```
bus_addr = txq->tx_ring_phys_addr;
IXGBE_WRITE_REG(hw, IXGBE_TDBAL(txq->reg_idx),
        (uint32_t)(bus_addr & 0x00000000ffffffffULL));
IXGBE_WRITE_REG(hw, IXGBE_TDBAH(txq->reg_idx),
        (uint32_t)(bus_addr >> 32));
IXGBE_WRITE_REG(hw, IXGBE_TDLEN(txq->reg_idx),
        txq->nb_tx_desc * sizeof(union ixgbe_adv_tx_desc));
/* Setup the HW Tx Head and TX Tail descriptor pointers */
IXGBE_WRITE_REG(hw, IXGBE_TDH(txq->reg_idx), 0);
IXGBE_WRITE_REG(hw, IXGBE_TDT(txq->reg_idx), 0);
```

最后配置一下多队列使用相关的寄存器：

```
ixgbe_dev_mq_tx_configure()
```

如此，发送单元的初始化就完成了。

收发单元初始化完毕后，就可以启动设备的收发单元咯：`ixgbe_dev_rxtx_start()` 先对每个发送队列的threshold相关寄存器进行设置，这是发送时的阈值参数，这个东西在发送部分有说明。

然后就是依次启动每个接收队列啦！

```
ixgbe_dev_rx_queue_start()
```

先检查，如果要启动的队列是合法的，那么就为这个接收队列分配存放mbuf的实际空间，

```
if (ixgbe_alloc_rx_queue_mbufs(rxq) != 0) 
{
    PMD_INIT_LOG(ERR, "Could not alloc mbuf for queue:%d",
             rx_queue_id);
    return -1;
}
```

在这里，你将找到终极答案--mempool、ring、queue ring、queue sw_ring的关系！

```
static int __attribute__((cold))
ixgbe_alloc_rx_queue_mbufs(struct ixgbe_rx_queue *rxq)
{
    struct ixgbe_rx_entry *rxe = rxq->sw_ring;
    uint64_t dma_addr;
    unsigned int i;
 
    /* Initialize software ring entries */
    for (i = 0; i < rxq->nb_rx_desc; i++) {
        volatile union ixgbe_adv_rx_desc *rxd;
        struct rte_mbuf *mbuf = rte_mbuf_raw_alloc(rxq->mb_pool);
 
        if (mbuf == NULL) {
            PMD_INIT_LOG(ERR, "RX mbuf alloc failed queue_id=%u",
                     (unsigned) rxq->queue_id);
            return -ENOMEM;
        }
 
        rte_mbuf_refcnt_set(mbuf, 1);
        mbuf->next = NULL;
        mbuf->data_off = RTE_PKTMBUF_HEADROOM;
        mbuf->nb_segs = 1;
        mbuf->port = rxq->port_id;
 
        dma_addr =
            rte_cpu_to_le_64(rte_mbuf_data_dma_addr_default(mbuf));
        rxd = &rxq->rx_ring[i];
        rxd->read.hdr_addr = 0;
        rxd->read.pkt_addr = dma_addr;
        rxe[i].mbuf = mbuf;
    }
 
    return 0;
}
```

看啊，真理就这么赤果果的在眼前啦，我都不知道该说些什么了！但还是得说点什么呀，不然就可以结束本文啦！ 我们看到，从队列所属内存池的ring中循环取出了nb_rx_desc个mbuf指针，也就是为了填充rxq->sw_ring。每个指针都指向内存池里的一个数据包空间。

然后就先填充了新分配的mbuf结构，最最重要的是填充计算了dma_addr

```
dma_addr = rte_cpu_to_le_64(rte_mbuf_data_dma_addr_default(mbuf));
```

然后初始化queue ring，即rxd的信息，标明了驱动把数据包放在dma_addr处。最后一句，把分配的mbuf“放入”queue 的sw_ring中，这样，驱动收过来的包，就直接放在了sw_ring中。

以上最重要的工作就完成了，下面就可以使能DMA引擎啦，准备收包。

```
hw->mac.ops.enable_rx_dma(hw, rxctrl);
```

然后再设置一下队列ring的头尾寄存器的值，这也是很重要的一点！头设置为0，尾设置为描述符个数减1，就是描述符填满整个ring。

```
IXGBE_WRITE_REG(hw, IXGBE_RDH(rxq->reg_idx), 0);
IXGBE_WRITE_REG(hw, IXGBE_RDT(rxq->reg_idx), rxq->nb_rx_desc - 1);
```

随着这步做完，剩余的就没有什么重要的事啦，就此打住！

接着依次启动每个发送队列：

发送队列的启动比接收队列的启动要简单，只是配置了txdctl寄存器，延时等待TX使能完成，最后，设置队列的头和尾位置都为0。

```
txdctl = IXGBE_READ_REG(hw, IXGBE_TXDCTL(txq->reg_idx));
txdctl |= IXGBE_TXDCTL_ENABLE;
IXGBE_WRITE_REG(hw, IXGBE_TXDCTL(txq->reg_idx), txdctl);
 
IXGBE_WRITE_REG(hw, IXGBE_TDH(txq->reg_idx), 0);
IXGBE_WRITE_REG(hw, IXGBE_TDT(txq->reg_idx), 0);
```

发送队列就启动完成了。

### 三、数据包的获取和发送

数据包的获取是指驱动把数据包放入了内存中，上层应用从队列中去取出这些数据包；发送是指把要发送的数据包放入到发送队列中，为实际发送做准备。

##### 数据包的获取

业务层面获取数据包是从`rte_eth_rx_burst()`开始的

```
int16_t nb_rx = (*dev->rx_pkt_burst)(dev->data->rx_queues[queue_id],
            rx_pkts, nb_pkts);
```

这里的dev->rx_pkt_burst在驱动初始化的时候已经注册过了，对于ixgbe设备，就是`ixgbe_recv_pkts()`函数。

在说收包之前，先了解网卡的DD标志，这个标志标识着一个描述符是否可用的情况：网卡在使用这个描述符前，先检查DD位是否为0，如果为0，那么就可以使用描述符，把数据拷贝到描述符指定的地址，之后把DD标志位置为1，否则表示不能使用这个描述符。而对于驱动而言，恰恰相反，在读取数据包时，先检查DD位是否为1，如果为1，表示网卡已经把数据放到了内存中，可以读取，读取完后，再把DD位设置为0，否则，就表示没有数据包可读。

就重点从这个函数看看，数据包是怎么被取出来的。

首先，取值`rx_id = rxq->rx_tail`,这个值初始化时为0，用来标识当前ring的尾。然后循环读取请求数量的描述符，这时候第一步判断就是这个描述符是否可用

```
staterr = rxdp->wb.upper.status_error;
if (!(staterr & rte_cpu_to_le_32(IXGBE_RXDADV_STAT_DD)))
    break;
```

如果描述符的DD位不为1，则表明这个描述符网卡还没有准备好，也就是没有包！没有包，就跳出循环。

如果描述符准备好了，就取出对应的描述符，因为网卡已经把一些信息存到了描述符里，可以后面把这些信息填充到新分配的数据包里。

下面就是一个狸猫换太子的事了，先从mempool的ring中分配一个新的“狸猫”---mbuf

```
nmb = rte_mbuf_raw_alloc(rxq->mb_pool);
```

然后找到当前描述符对应的“太子”---ixgbe_rx_entry *rxe

```
rxe = &sw_ring[rx_id];
```

中间略掉关于预取的操作代码，之后，就要用这个狸猫换个太子

```
rxm = rxe->mbuf;
rxe->mbuf = nmb;
```

这样换出来的太子rxm就是我们要取出来的数据包指针，在下面填充一些必要的信息，就可以把包返给接收的用户了

```
rxm->data_off = RTE_PKTMBUF_HEADROOM;
rte_packet_prefetch((char *)rxm->buf_addr + rxm->data_off);
rxm->nb_segs = 1;
rxm->next = NULL;
rxm->pkt_len = pkt_len;
rxm->data_len = pkt_len;
rxm->port = rxq->port_id;
 
pkt_info = rte_le_to_cpu_32(rxd.wb.lower.lo_dword.data);
/* Only valid if PKT_RX_VLAN_PKT set in pkt_flags */
rxm->vlan_tci = rte_le_to_cpu_16(rxd.wb.upper.vlan);
 
pkt_flags = rx_desc_status_to_pkt_flags(staterr, vlan_flags);
pkt_flags = pkt_flags | rx_desc_error_to_pkt_flags(staterr);
pkt_flags = pkt_flags |
    ixgbe_rxd_pkt_info_to_pkt_flags((uint16_t)pkt_info);
rxm->ol_flags = pkt_flags;
rxm->packet_type =
    ixgbe_rxd_pkt_info_to_pkt_type(pkt_info,
                       rxq->pkt_type_mask);
 
if (likely(pkt_flags & PKT_RX_RSS_HASH))
    rxm->hash.rss = rte_le_to_cpu_32(
                rxd.wb.lower.hi_dword.rss);
else if (pkt_flags & PKT_RX_FDIR) {
    rxm->hash.fdir.hash = rte_le_to_cpu_16(
            rxd.wb.lower.hi_dword.csum_ip.csum) &
            IXGBE_ATR_HASH_MASK;
    rxm->hash.fdir.id = rte_le_to_cpu_16(
            rxd.wb.lower.hi_dword.csum_ip.ip_id);
}
/*
 * Store the mbuf address into the next entry of the array
 * of returned packets.
 */
rx_pkts[nb_rx++] = rxm;
```

注意最后一句话，就是把包的指针返回给用户。

其实在换太子中间过程中，还有一件非常重要的事要做，就是开头说的，在驱动读取完数据包后，要把描述符的DD标志位置为0，同时设置新的DMA地址指向新的mbuf空间，这么描述符就可以再次被网卡硬件使用，拷贝数据到mbuf空间了。

```
dma_addr = rte_cpu_to_le_64(rte_mbuf_data_dma_addr_default(nmb));
rxdp->read.hdr_addr = 0;
rxdp->read.pkt_addr = dma_addr;
```

`rxdp->read.hdr_addr = 0;`一句中，就包含了设置DD位为0。

最后，就是检查空余可用描述符数量是否小于阀值，如果小于阀值，进行处理。不详细说了。

这样过后，收取数据包就完成啦！Done！

##### 数据包的发送

在说发送之前，先说一下描述符的回写（write-back），回写是指把用过后的描述符，恢复其重新使用的过程。在接收数据包过程中，回写是立马执行的，也就是DMA使用描述符标识包可读取，然后驱动程序读取数据包，读取之后，就会把DD位置0，同时进行回写操作，这个描述符也就可以再次被网卡硬件使用了。

但是发送过程中，回写却不是立刻完成的。发送有两种方式进行回写：

- 1.Updating by writing back into the Tx descriptor
- 2.Update by writing to the head pointer in system memory

第二种回写方式貌似针对的网卡比较老，对于82599，使用第一种回写方式。在下面三种情况下，才能进行回写操作：

- 1.TXDCTL[n].WTHRESH = 0 and a descriptor that has RS set is ready to be written back.
- 2.TXDCTL[n].WTHRESH > 0 and TXDCTL[n].WTHRESH descriptors have accumulated.
- 3.TXDCTL[n].WTHRESH > 0 and the corresponding EITR counter has reached zero. The timer expiration flushes any accumulated descriptors and sets an interrupt event(TXDW).

而在代码中，发送队列的初始化的时候，`ixgbe_dev_tx_queue_setup()`中

```
txq->pthresh = tx_conf->tx_thresh.pthresh;
txq->hthresh = tx_conf->tx_thresh.hthresh;
txq->wthresh = tx_conf->tx_thresh.wthresh;
```

pthresh,hthresh,wthresh的值，都是从tx_conf中配置的默认值，而tx_conf如果在我们的应用中没有赋值的话，就是采用的默认值：

```
dev_info->default_txconf = (struct rte_eth_txconf) {
    .tx_thresh = {
        .pthresh = IXGBE_DEFAULT_TX_PTHRESH,
        .hthresh = IXGBE_DEFAULT_TX_HTHRESH,
        .wthresh = IXGBE_DEFAULT_TX_WTHRESH,
    },
    .tx_free_thresh = IXGBE_DEFAULT_TX_FREE_THRESH,
    .tx_rs_thresh = IXGBE_DEFAULT_TX_RSBIT_THRESH,
    .txq_flags = ETH_TXQ_FLAGS_NOMULTSEGS |
            ETH_TXQ_FLAGS_NOOFFLOADS,
};
```

其中的wthresh就是0，其余两个是32.也就是说这种设置下，回写取决于RS标志位。RS标志位主要就是为了标识已经积累了一定数量的描述符，要进行回写了。

了解了这个，就来看看代码吧，从`ixgbe_xmit_pkts()`开始，为了看主要的框架，我们忽略掉网卡卸载等相关的功能的代码，主要看发送和回写

先检查剩余的描述符是否已经小于阈值，如果小于阈值，那么就先清理回收一下描述符

```
if (txq->nb_tx_free < txq->tx_free_thresh)
        ixgbe_xmit_cleanup(txq);
```

这是一个重要的操作，进去看看是怎么清理回收的：`ixgbe_xmit_cleanup(txq)`

取出上次清理的描述符位置，很明显，这次清理就接着上次的位置开始。所以，根据上次的位置，加上`txq->tx_rs_thresh`个描述符，就是标记有RS的描述符的位置，因为，tx_rs_thresh就是表示这么多个描述符后，设置RS位，进行回写。所以，从上次清理的位置跳过tx_rs_thresh个描述符，就能找到标记有RS的位置。

```
desc_to_clean_to = (uint16_t)(last_desc_cleaned + txq->tx_rs_thresh);
```

当网卡把队列的数据包发送完成后，就会把DD位设置为1，这个时候，先检查标记RS位置的描述符DD位，如果已经设置为1，则可以进行清理回收，否则，就不能清理。

接下来确认要清理的描述符个数

```
if (last_desc_cleaned > desc_to_clean_to)
		nb_tx_to_clean = (uint16_t)((nb_tx_desc - last_desc_cleaned) +
							desc_to_clean_to);
else
	nb_tx_to_clean = (uint16_t)(desc_to_clean_to -
					last_desc_cleaned);
```

然后，就把标记有RS位的描述符中的RS位清掉，确切的说，DD位等都清空了。调整上次清理的位置和空闲描述符大小。

```
txr[desc_to_clean_to].wb.status = 0;
 
/* Update the txq to reflect the last descriptor that was cleaned */
txq->last_desc_cleaned = desc_to_clean_to;
txq->nb_tx_free = (uint16_t)(txq->nb_tx_free + nb_tx_to_clean);
```

这样，就算清理完毕了！

继续看发送，依次处理每个要发送的数据包：

取出数据包，取出其中的卸载标志

```
ol_flags = tx_pkt->ol_flags;
 
/* If hardware offload required */
tx_ol_req = ol_flags & IXGBE_TX_OFFLOAD_MASK;
if (tx_ol_req) {
    tx_offload.l2_len = tx_pkt->l2_len;
    tx_offload.l3_len = tx_pkt->l3_len;
    tx_offload.l4_len = tx_pkt->l4_len;
    tx_offload.vlan_tci = tx_pkt->vlan_tci;
    tx_offload.tso_segsz = tx_pkt->tso_segsz;
    tx_offload.outer_l2_len = tx_pkt->outer_l2_len;
    tx_offload.outer_l3_len = tx_pkt->outer_l3_len;
 
    /* If new context need be built or reuse the exist ctx. */
    ctx = what_advctx_update(txq, tx_ol_req,
        tx_offload);
    /* Only allocate context descriptor if required*/
    new_ctx = (ctx == IXGBE_CTX_NUM);
    ctx = txq->ctx_curr;
}
```

这里卸载还要使用一个描述符，暂时不明白。

计算了发送这个包需要的描述符数量，主要是有些大包会分成几个segment,每个segment

```
nb_used = (uint16_t)(tx_pkt->nb_segs + new_ctx);
```

如果这次要用的数量加上设置RS之后积累的数量，又到达了tx_rs_thresh，那么就设置RS标志。

```
if (txp != NULL &&
        nb_used + txq->nb_tx_used >= txq->tx_rs_thresh)
/* set RS on the previous packet in the burst */
txp->read.cmd_type_len |=
    rte_cpu_to_le_32(IXGBE_TXD_CMD_RS);
```

接下来要确保用足够可用的描述符

如果描述符不够用了，就先进行清理回收，如果没能清理出空间，则把最后一个打上RS标志，更新队列尾寄存器，返回已经发送的数量。

```
if (txp != NULL)
        txp->read.cmd_type_len |= rte_cpu_to_le_32(IXGBE_TXD_CMD_RS);
 
    rte_wmb();
 
    /*
     * Set the Transmit Descriptor Tail (TDT)
     */
    PMD_TX_LOG(DEBUG, "port_id=%u queue_id=%u tx_tail=%u nb_tx=%u",
           (unsigned) txq->port_id, (unsigned) txq->queue_id,
           (unsigned) tx_id, (unsigned) nb_tx);
    IXGBE_PCI_REG_WRITE_RELAXED(txq->tdt_reg_addr, tx_id);
    txq->tx_tail = tx_id;
```

接下来的判断就很有意思了，

```
unlikely(nb_used > txq->tx_rs_thresh)
```

为什么说它奇怪呢？其实他自己都标明了unlikely,一个数据包会分为N多segment,多于txq->tx_rs_thresh（默认可是32啊），但即使出现了这种情况，也没做更多的处理，只是说会影响性能，然后开始清理描述符，其实这跟描述符还剩多少没有半毛钱关系，只是一个包占的描述符就超过了tx_rs_thresh,然而，并不见得是没有描述符了。所以，这时候清理描述符意义不明。

下面的处理应该都是已经有充足的描述符了，如果卸载有标志，就填充对应的值。不详细说了。

然后，就把数据包放到发送队列的sw_ring,并填充信息

```
m_seg = tx_pkt;
    do {
        txd = &txr[tx_id];
        txn = &sw_ring[txe->next_id];
        rte_prefetch0(&txn->mbuf->pool);
 
        if (txe->mbuf != NULL)
            rte_pktmbuf_free_seg(txe->mbuf);
        txe->mbuf = m_seg;
 
        /*
         * Set up Transmit Data Descriptor.
         */
        slen = m_seg->data_len;
        buf_dma_addr = rte_mbuf_data_dma_addr(m_seg);
        txd->read.buffer_addr =
            rte_cpu_to_le_64(buf_dma_addr);
        txd->read.cmd_type_len =
            rte_cpu_to_le_32(cmd_type_len | slen);
        txd->read.olinfo_status =
            rte_cpu_to_le_32(olinfo_status);
        txe->last_id = tx_last;
        tx_id = txe->next_id;
        txe = txn;
        m_seg = m_seg->next;
    } while (m_seg != NULL);
```

这里是把数据包的每个segment都放到队列sw_ring，很关键的是设置DMA地址，设置数据包长度和卸载参数。

一个数据包最后的segment的描述符需要一个EOP标志来结束。再更新剩余的描述符数：

```
cmd_type_len |= IXGBE_TXD_CMD_EOP;
txq->nb_tx_used = (uint16_t)(txq->nb_tx_used + nb_used);
txq->nb_tx_free = (uint16_t)(txq->nb_tx_free - nb_used);
```

然后再次检查是否已经达到了tx_rs_thresh，并做处理

```
if (txq->nb_tx_used >= txq->tx_rs_thresh) {
    PMD_TX_FREE_LOG(DEBUG,
            "Setting RS bit on TXD id="
            "%4u (port=%d queue=%d)",
            tx_last, txq->port_id, txq->queue_id);
 
    cmd_type_len |= IXGBE_TXD_CMD_RS;
 
    /* Update txq RS bit counters */
    txq->nb_tx_used = 0;
    txp = NULL;
} else
    txp = txd;
 
txd->read.cmd_type_len |= rte_cpu_to_le_32(cmd_type_len);
```

最后仍是做一下末尾的处理，更新队列尾指针。发送就结束啦！！

```
IXGBE_PCI_REG_WRITE_RELAXED(txq->tdt_reg_addr, tx_id);
txq->tx_tail = tx_id;
```

#### 总结：

可以看出数据包的发送和接收过程与驱动紧密相关，也与我们的配置有关，尤其是对于收发队列的参数配置，将直接影响性能，可以根据实际进行调整。对于收发虚拟化的部分，此文并未涉及，待后续有机会补充完整。

原创：[AISEED](https://home.cnblogs.com/u/yhp-smarthome/)    本文链接：https://www.cnblogs.com/yhp-smarthome/p/6705638.html
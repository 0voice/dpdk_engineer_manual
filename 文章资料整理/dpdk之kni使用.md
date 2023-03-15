# dpdk之kni使用

## 什么是kni

Kni(Kernel NIC Interface)内核网卡接口，是DPDK允许用户态和内核态交换报文的解决方案，模拟了一个虚拟的网口，提供dpdk的应用程序和linux内核之间通讯。kni接口允许报文从用户态接收后转发到linu协议栈去。 为什么要弄一个kni接口，虽然dpdk的高速转发性能很出色，但是也有自己的一些缺点，比如没有协议栈就是其中一项缺陷，当然也可能当时设计时就将没有将协议栈考虑进去，毕竟协议栈需要将报文转发处理，可能会使处理报文的能力大大降低。

当kni向linux发送报文时通过调用netif_rx()将报文送入linux协议栈，这其中需要将dpdk的mbuf转换成skb_buf。

当linux向kni端口发送报文时，调用回调函数kni_net_tx()，然后报文经过转换之后发送到端口上。

## kni优势

> 相较现存的Linux TUN/TAP接口更快的速度（消除了系统调用以及copy_to_user()/copy_from_user()内存拷贝的消耗）

> 允许标准Linux网络工具管理DPDK接口，如ethtool, ifconfig 和 tcpdump

> 提供到内核协议栈接口

## kni代码分析

和igb uio模块一样，kni模块分成内核以及用户态代码，内核模块在编译出来之后为rte_kni.ko，首先插入内核，dpdk提供了一个用户态的例子。首先看下kni内核模块代码,在kni_misc.c中，ko代码入口为

```
module_init(kni_init);
```

首先选择kni的线程模式，分为单线程还是多线程，所谓单线程是指所有的kni端口收发都由一个线程守护，多线程只是每一个kni端口分为由一个线程守护，这部分是在插入模块时带入参数选择。 接着调用注册函数misc_register，将kni注册为一个混杂设备。其中kni_misc结构体里面定义了该混杂设备的一些操作：

```
static struct miscdevice kni_misc = {
	.minor = MISC_DYNAMIC_MINOR,
	.name = KNI_DEVICE,
	.fops = &kni_fops,
};

static struct file_operations kni_fops = {
    .owner = THIS_MODULE,
    .open = kni_open,
    .release = kni_release,
    .unlocked_ioctl = (void *)kni_ioctl,
    .compat_ioctl = (void *)kni_compat_ioctl,
};
```

这里涉及的主要操作有kni_open，kni_release，以及kni_ioctl，分别对应几个函数。

### kni_open

kni_open时如果是单线程模式则会创建一个内核线程，并打开dev/kni，这个时候在host的dev下能看到kni文件夹

### kni_ioctl

kni_ioctl函数是与用户态通信的一个接口，主要是kni_ioctl_create函数。

ret = copy_from_user(&dev_info, (void *)ioctl_param, sizeof(dev_info));这条语句会拷贝从用户态传过来的消息，dev_info主要存放了虚拟kni网口的相关参数，接下来就会根据dev_info中的参数注册一个kni网口ret = register_netdev(net_dev);

这个函数完成创建，这样就虚拟出一个网口出来。其中165行是自己修改的，因为我发现按照文档提供的方法根本不能ping通报文，我将生成kni的mac地址修改成dpdk接管的网口mac即可贯通。原生态代码是随时生成一个mac。

### 杂项设备（misc device）

杂项设备也是在嵌入式系统中用得比较多的一种设备驱动。在 Linux 内核的include/linux目录下有Miscdevice.h文件，要把自己定义的misc device从设备定义在这里。其实是因为这些字符设备不符合预先确定的字符设备范畴，所有这些设备采用主编号10 ，一起归于misc device，其实misc_register就是用主标号10调用register_chrdev()的。

也就是说，misc设备其实也就是特殊的字符设备，可自动生成设备节点。

## dpdk mbuf与sk_buf转换

收包时需调用netif_receive_skb(skb)或netif_rx_ni(skb)通知内核处理包，发包时直接由内核调用ndo_start_xmit发出。

mbuf->sk_buf（收包）

```
/*
 * RX: normal working mode
 */
static void
kni_net_rx_normal(struct kni_dev *kni)
{
    unsigned ret;
    uint32_t len;
    unsigned i, num, num_rq, num_fq;
    struct rte_kni_mbuf *kva;
    struct rte_kni_mbuf *va[MBUF_BURST_SZ];
    void * data_kva;
 
    struct sk_buff *skb;
    struct net_device *dev = kni->net_dev;
 
    /* 每次收包的个数必须为rx_q和free_q的最小值且不超过MBUF_BURST_SZ */
 
    /* Get the number of entries in rx_q */
    num_rq = kni_fifo_count(kni->rx_q);
 
    /* Get the number of free entries in free_q */
    num_fq = kni_fifo_free_count(kni->free_q);
 
    /* Calculate the number of entries to dequeue in rx_q */
    num = min(num_rq, num_fq);
    num = min(num, (unsigned)MBUF_BURST_SZ);
 
    /* Return if no entry in rx_q and no free entry in free_q */
    if (num == 0)
        return;
 
    /* Burst dequeue from rx_q */
    ret = kni_fifo_get(kni->rx_q, (void **)va, num);
    if (ret == 0)
        return; /* Failing should not happen */
 
    /* mbuf转换为skb */
    /* Transfer received packets to netif */
    for (i = 0; i < num; i++) {
        /* mbuf kva */
        kva = (void *)va[i] - kni->mbuf_va + kni->mbuf_kva;
        len = kva->data_len;
        /* data kva */
        data_kva = kva->data - kni->mbuf_va + kni->mbuf_kva;
 
        skb = dev_alloc_skb(len + 2);
        if (!skb) {
            KNI_ERR("Out of mem, dropping pkts\n");
            /* Update statistics */
            kni->stats.rx_dropped++;
        }
        else {
            /* Align IP on 16B boundary */
            skb_reserve(skb, 2);
            memcpy(skb_put(skb, len), data_kva, len);
            skb->dev = dev;
            skb->protocol = eth_type_trans(skb, dev);
            skb->ip_summed = CHECKSUM_UNNECESSARY;
 
            /* 发送skb到协议栈 */
            /* Call netif interface */
            netif_receive_skb(skb);
            /* Call netif interface */
		    //netif_rx_ni(skb);//dpdk18.11
 
            /* Update statistics */
            kni->stats.rx_bytes += len;
            kni->stats.rx_packets++;
        }
    }
 
    /* 通知用户空间释放mbuf */
    /* Burst enqueue mbufs into free_q */
    ret = kni_fifo_put(kni->free_q, (void **)va, num);
    if (ret != num)
        /* Failing should not happen */
        KNI_ERR("Fail to enqueue entries into free_q\n");
}
```

sk_buf->mbuf（发包）

```
static int
kni_net_tx(struct sk_buff *skb, struct net_device *dev)
{
    int len = 0;
    unsigned ret;
    struct kni_dev *kni = netdev_priv(dev);
    struct rte_kni_mbuf *pkt_kva = NULL;
    struct rte_kni_mbuf *pkt_va = NULL;
 
    dev->trans_start = jiffies; /* save the timestamp */
 
    /* Check if the length of skb is less than mbuf size */
    if (skb->len > kni->mbuf_size)
        goto drop;
 
    /**
     * Check if it has at least one free entry in tx_q and
     * one entry in alloc_q.
     */
    if (kni_fifo_free_count(kni->tx_q) == 0 ||
            kni_fifo_count(kni->alloc_q) == 0) {
        /**
         * If no free entry in tx_q or no entry in alloc_q,
         * drops skb and goes out.
         */
        goto drop;
    }
 
    /* skb转mbuf */
    /* dequeue a mbuf from alloc_q */
    ret = kni_fifo_get(kni->alloc_q, (void **)&pkt_va, 1);
    if (likely(ret == 1)) {
        void *data_kva;
 
        pkt_kva = (void *)pkt_va - kni->mbuf_va + kni->mbuf_kva;
        data_kva = pkt_kva->data - kni->mbuf_va + kni->mbuf_kva;
 
        len = skb->len;
        memcpy(data_kva, skb->data, len);
        if (unlikely(len < ETH_ZLEN)) {
            memset(data_kva + len, 0, ETH_ZLEN - len);
            len = ETH_ZLEN;
        }
        pkt_kva->pkt_len = len;
        pkt_kva->data_len = len;
 
        /* enqueue mbuf into tx_q */
        ret = kni_fifo_put(kni->tx_q, (void **)&pkt_va, 1);
        if (unlikely(ret != 1)) {
            /* Failing should not happen */
            KNI_ERR("Fail to enqueue mbuf into tx_q\n");
            goto drop;
        }
    } else {
        /* Failing should not happen */
        KNI_ERR("Fail to dequeue mbuf from alloc_q\n");
        goto drop;
    }
 
    /* Free skb and update statistics */
    dev_kfree_skb(skb);
    kni->stats.tx_bytes += len;
    kni->stats.tx_packets++;
 
    return NETDEV_TX_OK;
 
drop:
    /* Free skb and update statistics */
    dev_kfree_skb(skb);
    kni->stats.tx_dropped++;
 
    return NETDEV_TX_OK;
}
```

## kni运行

### 编译

x86

```
make config T=x86_64-native-linuxapp-gcc EXTRA_CFLAGS='-g -Ofast -fPIC -ftls-model=local-dynamic'
make T=x86_64-native-linuxapp-gcc  CONFIG_RTE_KNI_KMOD=y CONFIG_RTE_EAL_IGB_UIO=y EXTRA_CFLAGS='-g -Ofast -fPIC -ftls-model=local-dynamic' install -j 8
make examples T=x86_64-native-linuxapp-gcc O=x86_64-native-linuxapp-gcc -j16
```

### 执行

```
kni [EAL options] -- -p PORTMASK --config="(port,lcore_rx,lcore_tx[,lcore_kthread,...])[,(port,lcore_rx,lcore_tx[,lcore_kthread,...])]" [-P] [-m]


-p PORTMASK:
十六进制接口掩码
--config="(port,lcore_rx,lcore_tx[,lcore_kthread,...])[,(port,lcore_rx,lcore_tx[,lcore_kthread,...])]":
指定对于每个物理网口，接收和发送DPDK线程绑定的核心，以及KNI内核线程绑定的核心
-P:
可选标志，设置的话意味着混杂模式，以便不区分以太网目的MAC地址，接收所有报文。不设置此选项，仅目的MAC地址等于接口MAC地址的报文被接收
-m:
可选标志。使能监控模式并更新以太网链路状态。此选项需要启动一个DPDK线程定期检查物理接口链路状态，同步相应的KNI虚拟网口状态。意味着当以太网链路down的时候，KNI虚拟接口将自动禁用，反之，自动启用。
# rmmod rte_kni
# insmod kmod/rte_kni.ko
# sudo ./kni -l 0-3 -n 4 -- -P -p 0x3 -m --config="(0,0,1),(1,2,3)"
# -l 0-3 使用0-3核
# -n 4   使用4个核
# -p 0x3 (11)使用两个网口，port 0和port 1
# --config="(0,0,1),(1,2,3)" 
# (0,0,1) port 0 使用0核rx，1核tx
# (1,2,3) port 1 使用2核rx，3核tx
./kni -l 0-1 -n 2 -- -p 0x1 -P --config="(0,0,1)"

ifconfig vEth0 121.168.1.12/24
```

### 定位追踪

在kni_net.c的kni_rx_normal中的382行添加如下代码：

```
pr_info("kni recv a packet\n");
```

使用dmesg观察输出。

```
RTE_LOG(INFO, APP, "kni send %lu packets\n",kni_stats[port_id].tx_packets);
```

## 测试

一端使用pktgen回放报文到kni口，分别使用大包和小包。大包使用GTPU报文或者其他TCP/IP报文，小包使用arp报文。

大包(包长1460)

```
./app/x86_64-native-linuxapp-gcc/pktgen -c 0xe0000 --socket-mem 1024 -n 2 -- -P -m [18:19].0 -s 0:gtpu.pcap -T --crc-strip
```

小包（）

```
./app/x86_64-native-linuxapp-gcc/pktgen -c 0xe0000 --socket-mem 1024 -n 2 -- -P -m [18:19].0 -s 0:arp_request.pcap -T --crc-strip
```

kni端的ip需要根据抓包报文进行修改。

10G网口大包大致可以跑到4Gbps线速，小包10G网口大致可以跑到50Mbps线速。

## reference

[dpdk中kni模块 - 笑侃码农 - 博客园](https://www.cnblogs.com/kb342/p/6033139.html)

[module_init解析_zhj失落之地的博客-CSDN博客](https://blog.csdn.net/u013216061/article/details/72511653)

[DPDK内核模块KNI - 灰信网（软件开发博客聚合）](https://www.freesion.com/article/148821991/)

[DPDK分析--KNI_whenloce的博客-CSDN博客_dpdk kni](https://blog.csdn.net/whenloce/article/details/94591357)

[misc_register、 register_chrdev 的区别总结](https://blog.csdn.net/nanhangfengshuai/article/details/50533230)

原文链接：https://blog.csdn.net/github_38294679/article/details/122250298
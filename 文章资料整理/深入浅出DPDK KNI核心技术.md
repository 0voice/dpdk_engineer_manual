# 深入浅出DPDK KNI核心技术

## **1.KNI**

**KNI全称：**Kernel NIC Interface，内核网卡接口，允许用户态程序访问linux控制平面。

在DPDK报文处理中，有些报文需要发送到内核协议栈进行处理，如GTP-C控制报文

如果报文数量较少，可以使用内核提供的TAP/TUN设备，但是鉴于这种设备使用的系统调用的方式，还涉及到copy_to_user()和copy_from_user()的开销，因此，提供了KNI接口用于改善用户态和内核态间报文的处理效率。



## **2.使用KNI的优势**

比 Linux TUN/TAP interfaces的操作快(通过取消系统调用copy_to_user()/copy_from_user())。

可使用Linux标准网络工具ethtool, ifconfig和tcpdump管理DPDK端口。

允许端口使用内核网络协议栈。



**kni的功能也是分为用户态KNI和内核态KNI两部分的**

**用户态的KNI代码在lib\librte_kni目录下**

**内核态的KNI代码在kernel/linux/kni目录下**

## **3.用户态KNI处理**

### **3.1 加载kni内核模块**

在加载kni模块时，可以设置它的内核线程模式

```text
insmod kmod/rte_kni.ko kthread_mode=single
insmod kmod/rte_kni.ko kthread_mode=multiple
```

**single模式（默认）**：只在内核侧创建一个内核线程，来接收所有kni设备上的数据包，一个线程 vs 所有kni设备

**multiple模式**：每个kni接口创建一个内核线程，用来接收数据包，一个线程 vs 一个kni设备

dpdk在加载kni模块时，默认是采用的single模式，同时还可以为此内核线程设置cpu亲和性

小伙伴们可能疑问，这里的single和multiple模式是什么意思

参考官网链接：[https://doc.dpdk.org/guides/prog_guide/kernel_nic_interface.html](https://link.zhihu.com/?target=https%3A//doc.dpdk.org/guides/prog_guide/kernel_nic_interface.html)



例子程序位于examples/kni/main.c文件

### **3.2 初始化流程**

```text
main(int argc, char** argv)
{
    /* eal初始化 */
    ret = rte_eal_init(argc, argv);

    /* 创建mbuf内存池*/
    pktmbuf_pool = rte_pktmbuf_pool_create("mbuf_pool", NB_MBUF,
        MEMPOOL_CACHE_SZ, 0, MBUF_DATA_SZ, rte_socket_id());

    /* 初始化KNI子系统*/
    init_kni();

    /* 初始化端口port，并调用kni_alloc */
    RTE_ETH_FOREACH_DEV(port) {
        init_port(port);//初始化端口
        kni_alloc(port);//kni申请资源
    }

    /* Launch per-lcore function on every lcore */
    // 每个lcore逻辑核心上运行一个函数main_loop,CALL_MASTER表示在master core上运行

    rte_eal_mp_remote_launch(main_loop, NULL, CALL_MASTER);

    /* 释放资源 */
    RTE_ETH_FOREACH_DEV(port) {
        kni_free_kni(port);
    }
    return 0;
}
```

下面重点分析下init_kni，kni_alloc，kni_free_kni，main_loop这四个函数。

### **3.3 init_kni函数**

**init_kni函数**中仅仅调用rte_kni_init。

```text
rte_kni_init(unsigned int max_kni_ifaces __rte_unused)
{
    /* Check FD and open */
    if (kni_fd < 0) {
        kni_fd = open("/dev/" KNI_DEVICE, O_RDWR);//打开/dev/kni设备
    }

    return 0;
}
```

DPDK在初始化阶段会调用rte_kni_init，打开kni设备。



### **3.4 kni_alloc函数**

**kni_alloc函数**主要调用了rte_kni_alloc函数



问题1：rte_kni_alloc对应KNI内核态的什么代码呢？

答：kni_ioctl_create()函数，后面会分析



### **3.5 kni_free_kni函数**

kni_free_kni主要是调用rte_kni_release函数

问题2：rte_kni_release对应KNI内核态的什么代码呢？

答：kni_ioctl_release()函数，后面会分析



### **3.6 主逻辑函数main_loop**

定位到main_loop逻辑处理函数

```text
static int
main_loop(__rte_unused void *arg)
{
    uint16_t i;
    int32_t f_stop;
    const unsigned lcore_id = rte_lcore_id();
    enum lcore_rxtx {
        LCORE_NONE,
        LCORE_RX,
        LCORE_TX,
        LCORE_MAX
    };
    enum lcore_rxtx flag = LCORE_NONE;

    //遍历设备列表，判断当前的lcore逻辑核心是负责rx还是tx
    RTE_ETH_FOREACH_DEV(i) {
        if (kni_port_params_array[i]->lcore_rx == (uint8_t)lcore_id) {
            flag = LCORE_RX;
            break;
        } else if (kni_port_params_array[i]->lcore_tx == lcore_id) {
            flag = LCORE_TX;
            break;
        }
    }

    //如果是接收数据，则循环调用kni_ingress，直到f_stop被设置为true跳出循环
    if (flag == LCORE_RX) {
        while (1) {
            kni_ingress(kni_port_params_array[i]);
        }
    } 
    //如果是发送数据，则循环调用kni_egress，直到f_stop被设置为true跳出循环
    else if (flag == LCORE_TX) {
        while (1) {
            kni_egress(kni_port_params_array[i]);
        }
    }

    return 0;
}
```



步骤如下：

获取配置，判断当前lcore是负责rx还是tx

如果lcore负责rx，则死循环调用接口kni_ingress进行报文的收取。

如果lcore负责tx，则死循环调用接口kni_egress进行报文的发送。

因为一个lcore只能负责一个死循环，所以最好给rx和tx都单独分配一个lcore去执行，不然容易出问题



### **3.7 kni_ingress函数**

疑惑1：数据如何从dpdk网口通过KNI网口发送给内核协议栈？

```text
/**
 * Interface to burst rx and enqueue mbufs into rx_q
 */
static void
kni_ingress(struct kni_port_params *p)
{
    struct rte_mbuf *pkts_burst[PKT_BURST_SZ];

    nb_kni = p->nb_kni;
    port_id = p->port_id;
    for (i = 0; i < nb_kni; i++) {
        /* Burst rx from eth */
        nb_rx = rte_eth_rx_burst(port_id, 0, pkts_burst, PKT_BURST_SZ);
        
        /* Burst tx to kni */
        num = rte_kni_tx_burst(p->kni[i], pkts_burst, nb_rx);
    }
}
```

kni_ingress函数是从dpdk网口接收数据，然后发送给kni接口。

从dpdk网口批量读取数据到mbuf数组pkts_burst中，然后将mbuf批量发送给kni接口。

**总结：通过KNI网口发送数据包给内核协议栈**



### **3.8 kni_egress函数**

```text
kni_egress(struct kni_port_params *p)
{
    struct rte_mbuf *pkts_burst[PKT_BURST_SZ];

    nb_kni = p->nb_kni;
    port_id = p->port_id;
    for (i = 0; i < nb_kni; i++) {
        /* Burst rx from kni */
        num = rte_kni_rx_burst(p->kni[i], pkts_burst, PKT_BURST_SZ);
        
        /* Burst tx to eth */
        nb_tx = rte_eth_tx_burst(port_id, 0, pkts_burst, (uint16_t)num);    
    }
}
```

从kni网口接收数据，存放到pkts_burst数组中，然后批量发送给dpdk网口。

kni_egress函数是从kni接口接收数据，然后发送给dpdk网口。



### **3.9用户态常用api总结**

rte_kni_init

rte_kni_close



rte_kni_alloc

rte_kni_release



kni_allocate_mbufs

kni_free_mbufs



rte_kni_tx_burst

rte_kni_rx_burst

后面我们会针对常用的api做深度的分析。



## **4.KNI用户态和内核态交互**

![img](https://pic4.zhimg.com/80/v2-0b2066b331faa8907efad69e84e754b3_720w.webp)

rx_q：从kni thread角度来说，是接收线程。

tx_q：从kni thread角度来说，是发送线程。

链接：[https://doc.dpdk.org/guides-1.8/prog_guide/kernel_nic_interface.html](https://link.zhihu.com/?target=https%3A//doc.dpdk.org/guides-1.8/prog_guide/kernel_nic_interface.html)

### **4.1 ingress部分**

图的上半部分 用例ingress：

1、在 DPDK RX 端，mbuf 由 PMD 在 RX 线程上下文中分配。

2、这个线程将mbuf入队rx_q FIFO。

3、KNI线程将为rx_q轮询所有KNI活动设备。如果一个mbuf出队列，它将被转换为sk_buff结构，并通过netif_rx()函数发送到网络堆栈。4、释放已出队的mbuf指针，并将mbuf指针归还到free_q FIFO队列。



### **4.2 egress部分**

图的下半部分 用例egress：

1、调用 kni_net_tx() 回调函数，从 Linux 网络堆栈接收数据包。

2、出队mbuf（无需等待缓存），用sk_buff数据来填充mbuf结构。

DPDK TX 线程将 mbuf 从tx_q队列出列，并将其发送到 PMD（通过 rte_eth_tx_burst()）。



## **5、内核态KNI处理**

### **5.1内核初始化**

#### **5.1.1模块初始化之注册misc设备**

```text
static int __init
kni_init(void)
{
    int rc;
    
    rc = register_pernet_subsys(&kni_net_ops);

    rc = misc_register(&kni_misc);

    /* Configure the lo mode according to the input parameter */
    kni_net_config_lo_mode(lo_mode);

    return 0;
}
```

模块初始化函数kni_init比较简单，核心函数有两个

**注册misc设备：misc_register**

**配置lo_mode：kni_net_config_lo_mode**

通过register_pernet_subsys或者register_pernet_gen_subsys，注册了kni_net_ops，保证每个namespace都会调用kni_init_net进行初始化。



注册为misc设备后，其工作机制由注册的misc device决定，即

```text
static const struct file_operations kni_fops = {
    .owner = THIS_MODULE,
    .open = kni_open,
    .release = kni_release,
    .unlocked_ioctl = (void *)kni_ioctl,
    .compat_ioctl = (void *)kni_compat_ioctl,
};

static struct miscdevice kni_misc = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = KNI_DEVICE,
    .fops = &kni_fops,
};
#define KNI_DEVICE "kni"
```



重点关注结构体变量kni_fops的kni_open，kni_release，kni_ioctl函数

### **5.2kni_open函数**

```text
static int
kni_open(struct inode *inode, struct file *file)
{
    struct net *net = current->nsproxy->net_ns;
    struct kni_net *knet = net_generic(net, kni_net_id);

    /* kni device can be opened by one user only per netns */
    if (test_and_set_bit(KNI_DEV_IN_USE_BIT_NUM, &knet->device_in_use))
        return -EBUSY;

    file->private_data = get_net(net);
    pr_debug("/dev/kni opened\n");

    return 0;
}
```

检查保证一个namespace一个用户只能打开一个kni设备。

打开后，将基于namespace的私有数据赋值给file->private_data。



### **5.3kni_ioctl函数**

如何使用kni设备呢？内核的kni模块，提供了ioctl的支持。

```text
static int
kni_ioctl(struct inode *inode, uint32_t ioctl_num, unsigned long ioctl_param)
{
    int ret = -EINVAL;
    struct net *net = current->nsproxy->net_ns;

    switch (_IOC_NR(ioctl_num)) {
    case _IOC_NR(RTE_KNI_IOCTL_CREATE):
        ret = kni_ioctl_create(net, ioctl_num, ioctl_param);
        break;
    case _IOC_NR(RTE_KNI_IOCTL_RELEASE):
        ret = kni_ioctl_release(net, ioctl_num, ioctl_param);
        break;
    }

    return ret;
}
```

有两个有效的case，RTE_KNI_IOCTL_CREATE和RTE_KNI_IOCTL_RELEASE，分别对应DPDK用户态的rte_kni_alloc和rte_kni_release，即申请kni interface和释放kni interface。



### **5.4rte_kni_alloc vs kni_ioctl_create 对比**

在rte_kni_alloc中，关键的代码是kni_reserve_mz申请连续的物理内存，并用其作为各个ring。



**kni_ioctl_create函数**

在kni_ioctl_create函数中会创建struct kni_dev结构体变量，并给它的成员赋值

```text
static int
kni_ioctl_create(struct net *net, uint32_t ioctl_num,
        unsigned long ioctl_param)
{
    struct kni_net *knet = net_generic(net, kni_net_id);
    int ret;
    struct rte_kni_device_info dev_info;
    struct net_device *net_dev = NULL;
    struct kni_dev *kni, *dev, *n;

    //申请netdev并赋值
    net_dev = alloc_netdev(sizeof(struct kni_dev), dev_info.name,kni_net_init);

    kni = netdev_priv(net_dev);

    //将ring的物理地址转为虚拟地址
    kni->tx_q = phys_to_virt(dev_info.tx_phys);
    kni->rx_q = phys_to_virt(dev_info.rx_phys);
    kni->alloc_q = phys_to_virt(dev_info.alloc_phys);
    kni->free_q = phys_to_virt(dev_info.free_phys);

    //注册netdev
    ret = register_netdev(net_dev);

    //启动内核接收线程
    ret = kni_run_thread(knet, kni, dev_info.force_bind);

    return 0;
}
```

- 通过phys_to_virt将ring的物理地址转成虚拟地址使用，这样就保证了KNI的用户态和内核态使用同一片物理地址，从而做到零拷贝。
- 注册netdev网络设备
- 启动内核接收线程



### **5.5kni_run_thread函数**

kni_run_thread是内核态要开启内核线程来接收所有kni设备上的数据

```text
static int
kni_run_thread(struct kni_net *knet, struct kni_dev *kni, uint8_t force_bind)
{
    /**
     * Create a new kernel thread for multiple mode, set its core affinity,
     * and finally wake it up.
     */
    if (multiple_kthread_on) {
        //线程函数为kni_thread_multiple
        kni->pthread = kthread_create(kni_thread_multiple,(void *)kni, "kni_%s", kni->name);
    } else {
        //加mutex锁
        mutex_lock(&knet->kni_kthread_lock);

        if (knet->kni_kthread == NULL) {
            //创建内核线程，线程函数为kni_thread_single
            knet->kni_kthread = kthread_create(kni_thread_single,
                (void *)knet, "kni_single");
        }
        
        //解mutex锁
        mutex_unlock(&knet->kni_kthread_lock);
    }

    return 0;
}
```

上述代码的思路：

如果KNI为多线程模式，每创建一个kni设备，就创建一个内核线程。

如果KNI为单线程模式，则检查是否已经启动了kni_thread。没有的话，创建唯一的kni内核thread kni_single，有的话，则什么都不做。

我们这里仅分析kni_thread_single线程函数。



**kni_thread_single线程函数**

```text
static int
kni_thread_single(void *data)
{
    struct kni_net *knet = data;
    int j;
    struct kni_dev *dev;

    while (!kthread_should_stop()) {
        down_read(&knet->kni_list_lock);
        //遍历所有kni设备
        for (j = 0; j < KNI_RX_LOOP_NUM; j++) {
            list_for_each_entry(dev, &knet->kni_list_head, list) {
                //接收动作
                kni_net_rx(dev);
                kni_net_poll_resp(dev);
            }
        }
        up_read(&knet->kni_list_lock);
    }

    return 0;
}
```

在持有读锁的情况下，遍历所有的kni设备，执行接收动作kni_net_rx。



**kni_net_rx函数如下**

```text
/* rx interface */
void
kni_net_rx(struct kni_dev *kni)
{
    /**
     * It doesn't need to check if it is NULL pointer,
     * as it has a default value
     */
    (*kni_net_rx_func)(kni);
}
```

kni_net_rx内部调用回调函数kni_net_rx_func



```text
 static kni_net_rx_t kni_net_rx_func = kni_net_rx_normal;
```

默认情况下kni_net_rx_func回调函数注册为kni_net_rx_normal



我们一起看下kni_net_rx_normal函数

```text
kni_net_rx_normal(struct kni_dev *kni)
{
    uint32_t ret;
    uint32_t len;
    uint32_t i, num_rx, num_fq;
    struct rte_kni_mbuf *kva;
    void *data_kva;
    struct sk_buff *skb;
    struct net_device *dev = kni->net_dev;

    /* Get the number of free entries in free_q */
    /*检查释放队列是否还有空位，没有的话，意味着读取后的数据无法增加到释放队列，故直接返回。*/
    num_fq = kni_fifo_free_count(kni->free_q);

    /* Burst dequeue from rx_q */
    /* 重点!!! 从kni->rx_q中弹出元素*/
    /* 重点!!! 从kni->rx_q中弹出元素*/
    num_rx = kni_fifo_get(kni->rx_q, kni->pa, num_rx);
    if(num_rx == 0)
        return;

    /* Transfer received packets to netif */
    // 循环处理收到的kni数据，将数据复制到申请的skb中。
    for (i = 0; i < num_rx; i++) {
        kva = pa2kva(kni->pa[i]);
        len = kva->pkt_len;
        data_kva = kva2data_kva(kva);
        kni->va[i] = pa2va(kni->pa[i], kva);

        /* 从内存申请skb */
        skb = dev_alloc_skb(len + 2);

        /* 如果只有一个mbuf segment段，则直接拷贝，如果是多个segment段，则分批拷贝*/
        if (kva->nb_segs == 1) {
            memcpy(skb_put(skb, len), data_kva, len);
        }

        /*设置skb相关参数*/
        skb->dev = dev;
        skb->protocol = eth_type_trans(skb, dev);
        skb->ip_summed = CHECKSUM_UNNECESSARY;

        //调用netif_rx_ni将skb传给内核协议栈处理
        netif_rx_ni(skb);
    }

    /* Burst enqueue mbufs into free_q */
    /*将使用后的mbuf指针加入到kni->free_q队列中，等待用户态KNI来进行释放*/
    ret = kni_fifo_put(kni->free_q, kni->va, num_rx);
}
```



流程如下

1.检查释放队列是否还有空位，没有的话，意味着读取后的数据无法增加到释放队列，故直接返回。

2.从kni->rx_q读取数据到kni->pa中。没有任何报文，则直接返回。

3.循环处理收到的kni数据，将数据复制到申请的skb中。

4.调用netif_rx_ni将skb传给内核协议栈处理

5.将读取的数据追加到free_q队列中，将使用后的kni->va投递到kni->free_q队列中，等待用户态KNI进行释放

这是DPDK app向KNI设备写入数据，也就是发给内核的情况。



而最前面的rte_kni_handle_request是什么作用呢？

rte_kni_handle_request从kni->req_q拿到request，然后根据修改mtu或者设置接口的命令做相应的操作，最后将response放回kni->resp_q。



### **5.6模块初始化之配置lo_mode**

```text
void
kni_net_config_lo_mode(char *lo_str)
{
    if (!strcmp(lo_str, "lo_mode_none"))
        pr_debug("loopback disabled");
    else if (!strcmp(lo_str, "lo_mode_fifo")) {
        pr_debug("loopback mode=lo_mode_fifo enabled");
        kni_net_rx_func = kni_net_rx_lo_fifo;
    } else if (!strcmp(lo_str, "lo_mode_fifo_skb")) {
        pr_debug("loopback mode=lo_mode_fifo_skb enabled");
        kni_net_rx_func = kni_net_rx_lo_fifo_skb;
    } else {
        pr_debug("Unknown loopback parameter, disabled");
    }
}

```

配置lo_mode，函数指针kni_net_rx_func指向不同的函数，默认为kni_net_rx_normal。

如果为lo_mode_fifo模式，函数指针kni_net_rx_func设置为kni_net_rx_lo_fifo；

如果为lo_mode_fifo_skb模式，函数指针kni_net_rx_func设置为kni_net_rx_lo_fifo_skb；



## **6.dpdk网口 -> KNI网口方向**

### **6.1 用户态KNI处理流程**

将pkts_burst中的mbuf报文批量发送给kni接口

```text
unsigned
rte_kni_tx_burst(struct rte_kni *kni, struct rte_mbuf **mbufs, unsigned num)
{
    void *phy_mbufs[num];
    unsigned int ret;
    unsigned int i;
    //记住这里传递的都是指向mbuf的指针

    //指向mubuf的地址是虚拟地址，将虚拟地址全部转为物理地址
    for (i = 0; i < num; i++)
        phy_mbufs[i] = va2pa(mbufs[i]);

    //将物理地址投递到kni->rx_q队列上
    ret = kni_fifo_put(kni->rx_q, phy_mbufs, num);

    /* Get mbufs from free_q and then free them */
    kni_free_mbufs(kni);

    return ret;
}
```



```text
static void
kni_free_mbufs(struct rte_kni *kni)
{
    int i, ret;
    struct rte_mbuf *pkts[MAX_MBUF_BURST_NUM];

    ret = kni_fifo_get(kni->free_q, (void **)pkts, MAX_MBUF_BURST_NUM);
    if (likely(ret > 0)) {
        for (i = 0; i < ret; i++)
            rte_pktmbuf_free(pkts[i]);
    }
}
```

准备rte_mbuf的指针数组pkts，然后调用kni_free_mbufs函数从kni->free_q队列上获取要释放的元素，保存到pkts中。

然后调用rte_pktmbuf_free函数进行释放内存资源

**用户态KNI做了两件事：** **1.将mbuf投递到kni->rx_q接收队列上**

**2.批量从kni->free_q队列上取元素，将内存归还给内存池**



### **6.2 内核态KNI处理流程**

**问题3：前文已将数据mbuf投递到kni->rx_q队列中，那么谁来读取kni->rx_q队列上的数据写入内核网络协议栈上呢？**

带着上述的疑问，我们来探寻下内核态的KNI处理流程



## **7.KNI网口方向 -> **dpdk网口

### **7.1从kni->tx_q队列读数据**

kni_egress函数中调用rte_kni_rx_burst从KNI网口中批量读取数据包，然后调用rte_eth_tx_burst批量发送给dpdk网口

```text
unsigned
rte_kni_rx_burst(struct rte_kni *kni, struct rte_mbuf **mbufs, unsigned num)
{
    unsigned ret = kni_fifo_get(kni->tx_q, (void **)mbufs, num);

    /* If buffers removed, allocate mbufs and then put them into alloc_q */
    if (ret)
        kni_allocate_mbufs(kni);

    return ret;
}
```



从kni->tx_q队列取元素存至mbufs指向的数组中，如果buffer已经删除，则调用kni_allocate_mbufs申请mbufs，然后添加到alloc_q队列

```text
static void
kni_allocate_mbufs(struct rte_kni *kni)
{
    int i, ret;
    struct rte_mbuf *pkts[MAX_MBUF_BURST_NUM];
    void *phys[MAX_MBUF_BURST_NUM];
    int allocq_free;

    //判断kni->alloc_q队列上还需要申请多少内存资源
    allocq_free = (kni->alloc_q->read - kni->alloc_q->write - 1) \
            & (MAX_MBUF_BURST_NUM - 1);
    for (i = 0; i < allocq_free; i++) {
        /*从内存池申请内存资源*/
        pkts[i] = rte_pktmbuf_alloc(kni->pktmbuf_pool);

        //申请到的内存是虚拟地址，将其转换为物理地址，内核才能进行操作
        phys[i] = va2pa(pkts[i]);
    }

    //将申请的内存投递到kni->alloc_q队列中
    ret = kni_fifo_put(kni->alloc_q, phys, i);

    /* Check if any mbufs not put into alloc_q, and then free them */
    if (ret >= 0 && ret < i && ret < MAX_MBUF_BURST_NUM) {
        int j;

        for (j = ret; j < i; j++)
            rte_pktmbuf_free(pkts[j]);
    }
}
```



上面rte_kni_rx_burst函数中kni_fifo_get(kni->tx_q, (void **)mbufs, num);

kni_fifo_get函数是从kni->tx_q队列取数据，那么谁往kni->tx_q队列投递数据呢？



### **7.2 向kni->tx_q队列写数据**

当内核向kni设备发送数据时，最终会调用到net_device_ops->ndo_start_xmit。

net_device_ops结构体的ndo_start_xmit回调函数注册为kni_net_tx(位于kernel/linux/kni/kni_net.c)

```text
static int
kni_net_tx(struct sk_buff *skb, struct net_device *dev)
{
    /* Check if the length of skb is less than mbuf size */
    //对skb报文做长度检查,不能超过mbuf的大小
    if (skb->len > kni->mbuf_size)
        goto drop;
    
    //检测tx_q队列中是否有空闲位置
    //检测alloc_q队列有否有剩余的mbuf
    if (kni_fifo_free_count(kni->tx_q) == 0 ||
            kni_fifo_count(kni->alloc_q) == 0) {
        /**
         * If no free entry in tx_q or no entry in alloc_q,
         * drops skb and goes out.
         */
        goto drop;
    }

    /* dequeue a mbuf from alloc_q */
    /* 从alloc_q队列取出一块内存*/
    ret = kni_fifo_get(kni->alloc_q, &pkt_pa, 1);
    if (likely(ret == 1)) {
        void *data_kva;

        //将pkt_pa转化为虚拟地址
        pkt_kva = pa2kva(pkt_pa);
        data_kva = kva2data_kva(pkt_kva);
        pkt_va = pa2va(pkt_pa, pkt_kva);

        /*skb -> mbuf 将skb的值赋值过去*/
        len = skb->len;
        memcpy(data_kva, skb->data, len);
        if (unlikely(len < ETH_ZLEN)) {
            memset(data_kva + len, 0, ETH_ZLEN - len);
            len = ETH_ZLEN;
        }
        pkt_kva->pkt_len = len;
        pkt_kva->data_len = len;

        /* 将pkt_va追加到发送队列 tx_q */
        ret = kni_fifo_put(kni->tx_q, &pkt_va, 1);
        if (unlikely(ret != 1)) {
            /* Failing should not happen */
            pr_err("Fail to enqueue mbuf into tx_q\n");
            goto drop;
        }
    } else {
        /* Failing should not happen */
        pr_err("Fail to dequeue mbuf from alloc_q\n");
        goto drop;
    }

    /* 释放skb并更新计数 */
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

 





原文链接：https://zhuanlan.zhihu.com/p/535078429    原文作者： 编程实战营
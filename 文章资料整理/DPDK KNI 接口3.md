# DPDK KNI 接口3 

![img](https://ask.qcloudimg.com/draft/3739951/4y8xd93i0d.png?imageView2/2/w/1620)

图1. kni结构图

## 1.从结构图中可以看到KNI需要内核模块的支持，即rte_kni.ko

当rte_kni模块加载时，创建/dev/kni设备节点（rte_kni模块创建kni杂项设备，文件系统节点/dev/kni需要手动或者通过udev机制创建），藉此节点，DPDK KNI应用可控制和与内核rte_kni模块交互。

在内核模块rte_kni加载时，可指定一些可选的参数以控制其行为：

```
# modinfo rte_kni.ko
lo_mode:        KNI loopback mode (default=lo_mode_none):
                lo_mode_none        Kernel loopback disabled
                lo_mode_fifo        Enable kernel loopback with fifo
                lo_mode_fifo_skb    Enable kernel loopback with fifo and skb buffer
 
kthread_mode:   Kernel thread mode (default=single):
                single    Single kernel thread mode enabled.
                multiple  Multiple kernel thread mode enabled.
 
carrier:        Default carrier state for KNI interface (default=off):
                off   Interfaces will be created with carrier state set to off.                on Interfaces will be created with carrier state set to on.
```

典型的情况是，在加载rte_kni模块时不指定任何参数，DPDK应用可由内核网络协议栈获取和向其发送报文。不指定任何参数，意味着仅创建一个内核线程处理所有的KNI虚拟设备在内核侧的报文接收，并且禁用回环模式，KNI接口的默认链路状态为关闭off

## 2.回环模式

以测试为目的，在加载rte_kni模块式可指定lo_mode参数:

\# insmod kmod/rte_kni.ko lo_mode=lo_mode_fifo
lo_mode_fifo回环模式将在内核空间中操作FIFO环队列，由函数kni_fifo_get(kni->rx_q,...)和kni_fifo_put(kni->tx_q,...)实现从rx_q接收队列读取报文，再写入发送队列tx_q来实现回环操作。


\# insmod kmod/rte_kni.ko lo_mode=lo_mode_fifo_skb

lo_mode_fifo_skb回环模式在以上lo_mode_fifo的基础之上，增加了sk_buff缓存的相关拷贝操作。具体包括将rx_q接收队列的数据拷贝到分配的接收skb缓存中。以及分配发送skb缓存，将之前由rx_q队列接收数据拷贝到发送skb缓存中，使用函数kni_net_tx(skb, dev)发送skb缓存数据。最终将数据报文拷贝到mbuf结构中，使用kni_fifo_put函数加入到tx_q发送队列。可见此回环测试模式，更加接近真实的使用场景

如果没有指定lo_mode参数，回环模式将禁用。

## 3.默认链路状态

内核模块rte_kni创建的KNI虚拟接口的链路状态，可通过模块加装时的carrier选项控制。

如果指定了carrier=off，当接口管理使能时，内核模块将接口的链路状态设置为关闭。DPDK应用可通过函数rte_kni_update_link设置KNI虚拟接口的链路状态。这对于需要KNI虚拟接口状态与对应的物理接口实际状态一致的应用是有用的。

如果指定了carrier=on，当接口管理使能时，内核模块将自动设置接口的链路状态为启用。这对于仅将KNI接口作为纯虚拟接口，而不对应任何物理硬件；或者并不想通过rte_kni_update_link函数显示设置接口链路状态的DPDK应用是有用的。对于物理口为连接任何链路而进行的回环模式测试也是有用的。

以下，设置默认的链路状态为启用:

\# insmod kmod/rte_kni.ko carrier=on

以下，设置默认的链路状态为关闭：

\# insmod kmod/rte_kni.ko carrier=off

如果carrier参数没有指定，KNI虚拟接口的默认链路状态为关闭。

在任何KNI虚拟接口创建之前，rte_kni内核模块必须加装到内核，并且经由rte_kni_init函数配置（获取/dev/kni设备节点文件句柄）

```
int rte_kni_init(unsigned int max_kni_ifaces __rte_unused)
{
    /* Check FD and open */
    if (kni_fd < 0) {
        kni_fd = open("/dev/" KNI_DEVICE, O_RDWR);
        if (kni_fd < 0) {
            RTE_LOG(ERR, KNI,
                "Can not open /dev/%s\n", KNI_DEVICE);
            return -1;
        }
}
```

模块初始化函数kni_init也非常简单。除了解析上面的参数配置外，比较重要的就是注册misc设备和配置lo_mode。

```
 1 static int __init
 2 kni_init(void)
 3 {
 4     int rc;
 5 
 6     if (kni_parse_kthread_mode() < 0) {
 7         pr_err("Invalid parameter for kthread_mode\n");
 8         return -EINVAL;
 9     }
10 
11     if (multiple_kthread_on == 0)
12         pr_debug("Single kernel thread for all KNI devices\n");
13     else
14         pr_debug("Multiple kernel thread mode enabled\n");
15 
16     if (kni_parse_carrier_state() < 0) {   //carrier可配置为off和on，默认为off
17         pr_err("Invalid parameter for carrier\n");
18         return -EINVAL;
19     }
20 
21     if (kni_dflt_carrier == 0)
22         pr_debug("Default carrier state set to off.\n");
23     else
24         pr_debug("Default carrier state set to on.\n");
25 
26 #ifdef HAVE_SIMPLIFIED_PERNET_OPERATIONS
27     rc = register_pernet_subsys(&kni_net_ops);
28 #else
29     rc = register_pernet_gen_subsys(&kni_net_id, &kni_net_ops);
30 #endif
31     if (rc)
32         return -EPERM;
33 
34     rc = misc_register(&kni_misc);
35     if (rc != 0) {
36         pr_err("Misc registration failed\n");
37         goto out;
38     }
39 
40     /* Configure the lo mode according to the input parameter */
41     kni_net_config_lo_mode(lo_mode);
42 
43     return 0;
44 
45 out:
46 #ifdef HAVE_SIMPLIFIED_PERNET_OPERATIONS
47     unregister_pernet_subsys(&kni_net_ops);
48 #else
49     unregister_pernet_gen_subsys(kni_net_id, &kni_net_ops);
50 #endif
51     return rc;
52 }
```

kni_net_config_lo_mode:



```
 1 void
 2 kni_net_config_lo_mode(char *lo_str)
 3 {
 4     if (!lo_str) {
 5         pr_debug("loopback disabled");
 6         return;
 7     }
 8 
 9     if (!strcmp(lo_str, "lo_mode_none"))
10         pr_debug("loopback disabled");
11     else if (!strcmp(lo_str, "lo_mode_fifo")) {
12         pr_debug("loopback mode=lo_mode_fifo enabled");
13         kni_net_rx_func = kni_net_rx_lo_fifo;
14     } else if (!strcmp(lo_str, "lo_mode_fifo_skb")) {
15         pr_debug("loopback mode=lo_mode_fifo_skb enabled");
16         kni_net_rx_func = kni_net_rx_lo_fifo_skb;
17     } else {
18         pr_debug("Unknown loopback parameter, disabled");
19     }
20 }
```

lo_mode可配置为lo_mode_none，lo_mode_fifo，和lo_mode_fifo_skb，默认为lo_mode_none。另外两个在实际产品中基本不会用到

配置lo_mode，函数指针kni_net_rx_func指向不同的函数，默认是kni_net_rx_func

通过register_pernet_subsys或者register_pernet_gen_subsys，注册了kni_net_ops，保证每个namespace都会调用kni_init_net进行初始化

注册为misc设备后，其工作机制由注册的miscdevice决定，即

```
 1 static const struct file_operations kni_fops = {
 2     .owner = THIS_MODULE,
 3     .open = kni_open,    //检查保证一个namespace只能打开kni一次，打开后将kni基于namespace的私有数据赋值给打开的文件file->private_data，以便后面使用
 4     .release = kni_release,
 5     .unlocked_ioctl = (void *)kni_ioctl,
 6     .compat_ioctl = (void *)kni_compat_ioctl,
 7 };
 8 
 9 static struct miscdevice kni_misc = {
10     .minor = MISC_DYNAMIC_MINOR,
11     .name = KNI_DEVICE,
12     .fops = &kni_fops,
13 };
```

## 4.如何使用kni设备呢?

```
 1 static int
 2 kni_ioctl(struct inode *inode, uint32_t ioctl_num, unsigned long ioctl_param)
 3 {
 4     int ret = -EINVAL;
 5     struct net *net = current->nsproxy->net_ns;
 6 
 7     pr_debug("IOCTL num=0x%0x param=0x%0lx\n", ioctl_num, ioctl_param);
 8 
 9     /*
10      * Switch according to the ioctl called
11      */
12     switch (_IOC_NR(ioctl_num)) {
13     case _IOC_NR(RTE_KNI_IOCTL_TEST):
14         /* For test only, not used */
15         break;
16     case _IOC_NR(RTE_KNI_IOCTL_CREATE):
17         ret = kni_ioctl_create(net, ioctl_num, ioctl_param);
18         break;
19     case _IOC_NR(RTE_KNI_IOCTL_RELEASE):
20         ret = kni_ioctl_release(net, ioctl_num, ioctl_param);
21         break;
22     default:
23         pr_debug("IOCTL default\n");
24         break;
25     }
26 
27     return ret;
28 }
```

RTE_KNI_IOCTL_CREATE和RTE_KNI_IOCTL_RELEASE，分别对应DPDK用户态的rte_kni_alloc和rte_kni_release，即申请kni interface和释放kni interface

rte_kni_alloc:

```
  1 struct rte_kni *
  2 rte_kni_alloc(struct rte_mempool *pktmbuf_pool,
  3           const struct rte_kni_conf *conf,
  4           struct rte_kni_ops *ops)
  5 {
  6     int ret;
  7     struct rte_kni_device_info dev_info;
  8     struct rte_kni *kni;
  9     struct rte_tailq_entry *te;
 10     struct rte_kni_list *kni_list;
 11 
 12     if (!pktmbuf_pool || !conf || !conf->name[0])
 13         return NULL;
 14 
 15     /* Check if KNI subsystem has been initialized */
 16     if (kni_fd < 0) {
 17         RTE_LOG(ERR, KNI, "KNI subsystem has not been initialized. Invoke rte_kni_init() first\n");
 18         return NULL;
 19     }
 20 
 21     rte_mcfg_tailq_write_lock();
 22 
 23     kni = __rte_kni_get(conf->name);
 24     if (kni != NULL) {
 25         RTE_LOG(ERR, KNI, "KNI already exists\n");
 26         goto unlock;
 27     }
 28 
 29     te = rte_zmalloc("KNI_TAILQ_ENTRY", sizeof(*te), 0);
 30     if (te == NULL) {
 31         RTE_LOG(ERR, KNI, "Failed to allocate tailq entry\n");
 32         goto unlock;
 33     }
 34 
 35     kni = rte_zmalloc("KNI", sizeof(struct rte_kni), RTE_CACHE_LINE_SIZE);
 36     if (kni == NULL) {
 37         RTE_LOG(ERR, KNI, "KNI memory allocation failed\n");
 38         goto kni_fail;
 39     }
 40 
 41     strlcpy(kni->name, conf->name, RTE_KNI_NAMESIZE);
 42 
 43     if (ops)
 44         memcpy(&kni->ops, ops, sizeof(struct rte_kni_ops));
 45     else
 46         kni->ops.port_id = UINT16_MAX;
 47 
 48     memset(&dev_info, 0, sizeof(dev_info));
 49     dev_info.core_id = conf->core_id;
 50     dev_info.force_bind = conf->force_bind;
 51     dev_info.group_id = conf->group_id;
 52     dev_info.mbuf_size = conf->mbuf_size;
 53     dev_info.mtu = conf->mtu;
 54     dev_info.min_mtu = conf->min_mtu;
 55     dev_info.max_mtu = conf->max_mtu;
 56 
 57     memcpy(dev_info.mac_addr, conf->mac_addr, RTE_ETHER_ADDR_LEN);
 58 
 59     strlcpy(dev_info.name, conf->name, RTE_KNI_NAMESIZE);
 60 
 61     ret = kni_reserve_mz(kni);//kni_reserve_mz申请连续的物理内存，并用其作为各个ring
 62     if (ret < 0)
 63         goto mz_fail;
 64 
 65     /* TX RING */
 66     kni->tx_q = kni->m_tx_q->addr;
 67     kni_fifo_init(kni->tx_q, KNI_FIFO_COUNT_MAX);
 68     dev_info.tx_phys = kni->m_tx_q->phys_addr;
 69 
 70     /* RX RING */
 71     kni->rx_q = kni->m_rx_q->addr;
 72     kni_fifo_init(kni->rx_q, KNI_FIFO_COUNT_MAX);
 73     dev_info.rx_phys = kni->m_rx_q->phys_addr;
 74 
 75     /* ALLOC RING */
 76     kni->alloc_q = kni->m_alloc_q->addr;
 77     kni_fifo_init(kni->alloc_q, KNI_FIFO_COUNT_MAX);
 78     dev_info.alloc_phys = kni->m_alloc_q->phys_addr;
 79 
 80     /* FREE RING */
 81     kni->free_q = kni->m_free_q->addr;
 82     kni_fifo_init(kni->free_q, KNI_FIFO_COUNT_MAX);
 83     dev_info.free_phys = kni->m_free_q->phys_addr;
 84 
 85     /* Request RING */
 86     kni->req_q = kni->m_req_q->addr;
 87     kni_fifo_init(kni->req_q, KNI_FIFO_COUNT_MAX);
 88     dev_info.req_phys = kni->m_req_q->phys_addr;
 89 
 90     /* Response RING */
 91     kni->resp_q = kni->m_resp_q->addr;
 92     kni_fifo_init(kni->resp_q, KNI_FIFO_COUNT_MAX);
 93     dev_info.resp_phys = kni->m_resp_q->phys_addr;
 94 
 95     /* Req/Resp sync mem area */
 96     kni->sync_addr = kni->m_sync_addr->addr;
 97     dev_info.sync_va = kni->m_sync_addr->addr;
 98     dev_info.sync_phys = kni->m_sync_addr->phys_addr;
 99 
100     kni->pktmbuf_pool = pktmbuf_pool;
101     kni->group_id = conf->group_id;
102     kni->mbuf_size = conf->mbuf_size;
103 
104     dev_info.iova_mode = (rte_eal_iova_mode() == RTE_IOVA_VA) ? 1 : 0;
105 
106     ret = ioctl(kni_fd, RTE_KNI_IOCTL_CREATE, &dev_info); //使用IOCTL创建内核对应的虚拟网口
107     if (ret < 0)
108         goto ioctl_fail;
109 
110     te->data = kni;
111 
112     kni_list = RTE_TAILQ_CAST(rte_kni_tailq.head, rte_kni_list);
113     TAILQ_INSERT_TAIL(kni_list, te, next);
114 
115     rte_mcfg_tailq_write_unlock();
116 
117     /* Allocate mbufs and then put them into alloc_q */
118     kni_allocate_mbufs(kni);  //分配队列内存资源
119 
120     return kni;
121 
122 ioctl_fail:
123     kni_release_mz(kni);
124 mz_fail:
125     rte_free(kni);
126 kni_fail:
127     rte_free(te);
128 unlock:
129     rte_mcfg_tailq_write_unlock();
130 
131     return NULL;
132 }
```

分配每个kni口对应的结构，并管理起来：

```
 1 /**
 2  * KNI context
 3  */
 4 struct rte_kni {
 5     char name[RTE_KNI_NAMESIZE];        /**< KNI interface name */
 6     uint16_t group_id;                  /**< Group ID of KNI devices */
 7     uint32_t slot_id;                   /**< KNI pool slot ID */
 8     struct rte_mempool *pktmbuf_pool;   /**< pkt mbuf mempool */
 9     unsigned int mbuf_size;                 /**< mbuf size */
10 
11     const struct rte_memzone *m_tx_q;   /**< TX queue memzone */
12     const struct rte_memzone *m_rx_q;   /**< RX queue memzone */
13     const struct rte_memzone *m_alloc_q;/**< Alloc queue memzone */
14     const struct rte_memzone *m_free_q; /**< Free queue memzone */
15 
16     struct rte_kni_fifo *tx_q;          /**< TX queue */
17     struct rte_kni_fifo *rx_q;          /**< RX queue */
18     struct rte_kni_fifo *alloc_q;       /**< Allocated mbufs queue */
19     struct rte_kni_fifo *free_q;        /**< To be freed mbufs queue */
20 
21     const struct rte_memzone *m_req_q;  /**< Request queue memzone */
22     const struct rte_memzone *m_resp_q; /**< Response queue memzone */
23     const struct rte_memzone *m_sync_addr;/**< Sync addr memzone */
24 
25     /* For request & response */
26     struct rte_kni_fifo *req_q;         /**< Request queue */
27     struct rte_kni_fifo *resp_q;        /**< Response queue */
28     void *sync_addr;                   /**< Req/Resp Mem address */
29 
30     struct rte_kni_ops ops;             /**< operations for request */
31 };
```

接着根据rte_kni_conf信息配置rte_kni_device_info：

```
 1 /*
 2  * Struct used to create a KNI device. Passed to the kernel in IOCTL call
 3  */
 4 
 5 struct rte_kni_device_info {
 6     char name[RTE_KNI_NAMESIZE];  /**< Network device name for KNI */
 7 
 8     phys_addr_t tx_phys;
 9     phys_addr_t rx_phys;
10     phys_addr_t alloc_phys;
11     phys_addr_t free_phys;
12 
13     /* Used by Ethtool */
14     phys_addr_t req_phys;
15     phys_addr_t resp_phys;
16     phys_addr_t sync_phys;
17     void * sync_va;
18 
19     /* mbuf mempool */
20     void * mbuf_va;
21     phys_addr_t mbuf_phys;
22 
23     uint16_t group_id;            /**< Group ID */
24     uint32_t core_id;             /**< core ID to bind for kernel thread */
25 
26     __extension__
27     uint8_t force_bind : 1;       /**< Flag for kernel thread binding */
28 
29     /* mbuf size */
30     unsigned mbuf_size;
31     unsigned int mtu;
32     unsigned int min_mtu;
33     unsigned int max_mtu;
34     uint8_t mac_addr[6];
35     uint8_t iova_mode;
36 };
```

kni的资源初始化完成后使用IOCTL创建内核对应的虚拟网口：

```scss
ioctl(kni_fd, RTE_KNI_IOCTL_CREATE, &dev_info);
```

驱动执行kni_ioctl_create：

```
  1 static int
  2 kni_ioctl_create(struct net *net, uint32_t ioctl_num,
  3         unsigned long ioctl_param)
  4 {
  5     struct kni_net *knet = net_generic(net, kni_net_id);
  6     int ret;
  7     struct rte_kni_device_info dev_info;
  8     struct net_device *net_dev = NULL;
  9     struct kni_dev *kni, *dev, *n;
 10 
 11     pr_info("Creating kni...\n");
 12     /* Check the buffer size, to avoid warning */
 13     if (_IOC_SIZE(ioctl_num) > sizeof(dev_info))
 14         return -EINVAL;
 15 
 16     /* Copy kni info from user space */
 17     if (copy_from_user(&dev_info, (void *)ioctl_param, sizeof(dev_info)))
 18         return -EFAULT;
 19 
 20     /* Check if name is zero-ended */
 21     if (strnlen(dev_info.name, sizeof(dev_info.name)) == sizeof(dev_info.name)) {
 22         pr_err("kni.name not zero-terminated");
 23         return -EINVAL;
 24     }
 25 
 26     /**
 27      * Check if the cpu core id is valid for binding.
 28      */
 29     if (dev_info.force_bind && !cpu_online(dev_info.core_id)) {
 30         pr_err("cpu %u is not online\n", dev_info.core_id);
 31         return -EINVAL;
 32     }
 33 
 34     /* Check if it has been created */
 35     down_read(&knet->kni_list_lock);
 36     list_for_each_entry_safe(dev, n, &knet->kni_list_head, list) {
 37         if (kni_check_param(dev, &dev_info) < 0) {
 38             up_read(&knet->kni_list_lock);
 39             return -EINVAL;
 40         }
 41     }
 42     up_read(&knet->kni_list_lock);
 43 
 44     net_dev = alloc_netdev(sizeof(struct kni_dev), dev_info.name,
 45 #ifdef NET_NAME_USER
 46                             NET_NAME_USER,
 47 #endif
 48                             kni_net_init);
 49     if (net_dev == NULL) {
 50         pr_err("error allocating device \"%s\"\n", dev_info.name);
 51         return -EBUSY;
 52     }
 53 
 54     dev_net_set(net_dev, net);
 55 
 56     kni = netdev_priv(net_dev);
 57 
 58     kni->net_dev = net_dev;
 59     kni->core_id = dev_info.core_id;
 60     strncpy(kni->name, dev_info.name, RTE_KNI_NAMESIZE);
 61 
 62     /* Translate user space info into kernel space info */
 63     if (dev_info.iova_mode) {
 64 #ifdef HAVE_IOVA_TO_KVA_MAPPING_SUPPORT
 65         kni->tx_q = iova_to_kva(current, dev_info.tx_phys);
 66         kni->rx_q = iova_to_kva(current, dev_info.rx_phys);
 67         kni->alloc_q = iova_to_kva(current, dev_info.alloc_phys);
 68         kni->free_q = iova_to_kva(current, dev_info.free_phys);
 69 
 70         kni->req_q = iova_to_kva(current, dev_info.req_phys);
 71         kni->resp_q = iova_to_kva(current, dev_info.resp_phys);
 72         kni->sync_va = dev_info.sync_va;
 73         kni->sync_kva = iova_to_kva(current, dev_info.sync_phys);
 74         kni->usr_tsk = current;
 75         kni->iova_mode = 1;
 76 #else
 77         pr_err("KNI module does not support IOVA to VA translation\n");
 78         return -EINVAL;
 79 #endif
 80     } else {
 81 
 82         kni->tx_q = phys_to_virt(dev_info.tx_phys);
 83         kni->rx_q = phys_to_virt(dev_info.rx_phys);
 84         kni->alloc_q = phys_to_virt(dev_info.alloc_phys);
 85         kni->free_q = phys_to_virt(dev_info.free_phys);
 86 
 87         kni->req_q = phys_to_virt(dev_info.req_phys);
 88         kni->resp_q = phys_to_virt(dev_info.resp_phys);
 89         kni->sync_va = dev_info.sync_va;
 90         kni->sync_kva = phys_to_virt(dev_info.sync_phys);
 91         kni->iova_mode = 0;
 92     }
 93 
 94     kni->mbuf_size = dev_info.mbuf_size;
 95 
 96     pr_debug("tx_phys:      0x%016llx, tx_q addr:      0x%p\n",
 97         (unsigned long long) dev_info.tx_phys, kni->tx_q);
 98     pr_debug("rx_phys:      0x%016llx, rx_q addr:      0x%p\n",
 99         (unsigned long long) dev_info.rx_phys, kni->rx_q);
100     pr_debug("alloc_phys:   0x%016llx, alloc_q addr:   0x%p\n",
101         (unsigned long long) dev_info.alloc_phys, kni->alloc_q);
102     pr_debug("free_phys:    0x%016llx, free_q addr:    0x%p\n",
103         (unsigned long long) dev_info.free_phys, kni->free_q);
104     pr_debug("req_phys:     0x%016llx, req_q addr:     0x%p\n",
105         (unsigned long long) dev_info.req_phys, kni->req_q);
106     pr_debug("resp_phys:    0x%016llx, resp_q addr:    0x%p\n",
107         (unsigned long long) dev_info.resp_phys, kni->resp_q);
108     pr_debug("mbuf_size:    %u\n", kni->mbuf_size);
109 
110     /* if user has provided a valid mac address */
111     if (is_valid_ether_addr(dev_info.mac_addr))
112         memcpy(net_dev->dev_addr, dev_info.mac_addr, ETH_ALEN);
113     else
114         /*
115          * Generate random mac address. eth_random_addr() is the
116          * newer version of generating mac address in kernel.
117          */
118         random_ether_addr(net_dev->dev_addr);
119 
120     if (dev_info.mtu)
121         net_dev->mtu = dev_info.mtu;
122 #ifdef HAVE_MAX_MTU_PARAM
123     net_dev->max_mtu = net_dev->mtu;
124 
125     if (dev_info.min_mtu)
126         net_dev->min_mtu = dev_info.min_mtu;
127 
128     if (dev_info.max_mtu)
129         net_dev->max_mtu = dev_info.max_mtu;
130 #endif
131 
132     ret = register_netdev(net_dev); //注册netdev
133     if (ret) {
134         pr_err("error %i registering device \"%s\"\n",
135                     ret, dev_info.name);
136         kni->net_dev = NULL;
137         kni_dev_remove(kni);
138         free_netdev(net_dev);
139         return -ENODEV;
140     }
141 
142     netif_carrier_off(net_dev);
143 
144     ret = kni_run_thread(knet, kni, dev_info.force_bind); //启动内核接收线
145     if (ret != 0)
146         return ret;
147 
148     down_write(&knet->kni_list_lock);
149     list_add(&kni->list, &knet->kni_list_head);
150     up_write(&knet->kni_list_lock);
151 
152     return 0;
153 }
```

通过phys_to_virt将ring的物理地址转成虚拟地址使用，这样就保证了KNI的用户态和内核态使用同一片物理地址，从而做到零拷贝

```
 1 static int
 2 kni_run_thread(struct kni_net *knet, struct kni_dev *kni, uint8_t force_bind)
 3 {
 4     /**
 5      * Create a new kernel thread for multiple mode, set its core affinity,
 6      * and finally wake it up.
 7      */
 8     if (multiple_kthread_on) {
 9         kni->pthread = kthread_create(kni_thread_multiple,
10             (void *)kni, "kni_%s", kni->name);
11         if (IS_ERR(kni->pthread)) {
12             kni_dev_remove(kni);
13             return -ECANCELED;
14         }
15 
16         if (force_bind)
17             kthread_bind(kni->pthread, kni->core_id);
18         wake_up_process(kni->pthread);
19     } else {
20         mutex_lock(&knet->kni_kthread_lock);
21 
22         if (knet->kni_kthread == NULL) {
23             knet->kni_kthread = kthread_create(kni_thread_single,
24                 (void *)knet, "kni_single");
25             if (IS_ERR(knet->kni_kthread)) {
26                 mutex_unlock(&knet->kni_kthread_lock);
27                 kni_dev_remove(kni);
28                 return -ECANCELED;
29             }
30 
31             if (force_bind)
32                 kthread_bind(knet->kni_kthread, kni->core_id);
33             wake_up_process(knet->kni_kthread);
34         }
35 
36         mutex_unlock(&knet->kni_kthread_lock);
37     }
38 
39     return 0;
40 }
```

如果KNI模块的参数指定了多线程模式，每创建一个kni设备，就创建一个内核线程。如果为单线程模式，则检查是否已经启动了kni_thread。没有的话，创建唯一的kni内核thread kni_single，有的话，则什么都不做。

 

## 5.用户态与内核态报文交互的方式: 

![img](https://img2018.cnblogs.com/i-beta/1414775/202002/1414775-20200213205632399-2137387644.png)

 

从ingress方向看，rte_eth_rx_burst时mbuf mempool中分配内存，通过rx_q发送给KNI，KNI线程将mbuf从rx_q中出队，将其转换成skb后，就将原mbuf通过free_q归还给用户rx thread，并由rx_thread释放，可以看出ingress方向的mbuf完全由用户态rx_thread自行管理。

从egress方向看，当KNI口发包时，从 mbuf cache中取得一个mbuf，将skb内容copy到mbuf中，进入tx_q队列，tx_thread将该mbuf出队并完成发送，因为发送后该mbuf会被释放。所以要重新alloc一个mbuf通过alloc_q归还给kernel。这部分mbuf是上面用户态填充的alloc_q。

这是DPDK app向KNI设备写入数据，也就是发给内核的情况。当内核从KNI设备发送数据时，按照内核的流程处理，最终会调用到net_device_ops->ndo_start_xmit。对于KNI驱动来说，即kni_net_tx。

```
 1 /*
 2  * Transmit a packet (called by the kernel)
 3  */
 4 static int
 5 kni_net_tx(struct sk_buff *skb, struct net_device *dev)
 6 {
 7     int len = 0;
 8     uint32_t ret;
 9     struct kni_dev *kni = netdev_priv(dev);
10     struct rte_kni_mbuf *pkt_kva = NULL;
11     void *pkt_pa = NULL;
12     void *pkt_va = NULL;
13 
14     /* save the timestamp */
15 #ifdef HAVE_TRANS_START_HELPER
16     netif_trans_update(dev);
17 #else
18     dev->trans_start = jiffies;
19 #endif
20 
21     /* Check if the length of skb is less than mbuf size */
22     if (skb->len > kni->mbuf_size)
23         goto drop;
24 
25     /**
26      * Check if it has at least one free entry in tx_q and
27      * one entry in alloc_q.
28      */
29     if (kni_fifo_free_count(kni->tx_q) == 0 ||
30             kni_fifo_count(kni->alloc_q) == 0) {
31         /**
32          * If no free entry in tx_q or no entry in alloc_q,
33          * drops skb and goes out.
34          */
35         goto drop;
36     }
37 
38     /* dequeue a mbuf from alloc_q */
39     ret = kni_fifo_get(kni->alloc_q, &pkt_pa, 1);
40     if (likely(ret == 1)) {
41         void *data_kva;
42 
43         pkt_kva = get_kva(kni, pkt_pa);
44         data_kva = get_data_kva(kni, pkt_kva);
45         pkt_va = pa2va(pkt_pa, pkt_kva);
46 
47         len = skb->len;
48         memcpy(data_kva, skb->data, len);
49         if (unlikely(len < ETH_ZLEN)) {
50             memset(data_kva + len, 0, ETH_ZLEN - len);
51             len = ETH_ZLEN;
52         }
53         pkt_kva->pkt_len = len;
54         pkt_kva->data_len = len;
55 
56         /* enqueue mbuf into tx_q */
57         ret = kni_fifo_put(kni->tx_q, &pkt_va, 1);
58         if (unlikely(ret != 1)) {
59             /* Failing should not happen */
60             pr_err("Fail to enqueue mbuf into tx_q\n");
61             goto drop;
62         }
63     } else {
64         /* Failing should not happen */
65         pr_err("Fail to dequeue mbuf from alloc_q\n");
66         goto drop;
67     }
68 
69     /* Free skb and update statistics */
70     dev_kfree_skb(skb);
71     dev->stats.tx_bytes += len;
72     dev->stats.tx_packets++;
73 
74     return NETDEV_TX_OK;
75 
76 drop:
77     /* Free skb and update statistics */
78     dev_kfree_skb(skb);
79     dev->stats.tx_dropped++;
80 
81     return NETDEV_TX_OK;
82 }
```

1.对skb报文长度做检查，不能超过mbuf的大小。然后检查发送队列tx_q是否还有空位，“内存队列”是否有剩余的mbuf

2.从alloc_q取出一个内存块，将其转换为虚拟地址，然后将skb的数据复制过去，最后将其追加到发送队列tx_q中

3.发送完成后，就直接释放skb并更新统计计数

 

DPDK提供了两个API rte_kni_rx_burst和rte_kni_tx_burst，用于从KNI接收报文和向KNI发送报文

```
 1 unsigned
 2 rte_kni_rx_burst(struct rte_kni *kni, struct rte_mbuf **mbufs, unsigned int num)
 3 {
 4     unsigned int ret = kni_fifo_get(kni->tx_q, (void **)mbufs, num);
 5 
 6     /* If buffers removed, allocate mbufs and then put them into alloc_q */
 7     if (ret)
 8         kni_allocate_mbufs(kni);
 9 
10     return ret;
11 }
12 
13 
14 unsigned
15 rte_kni_tx_burst(struct rte_kni *kni, struct rte_mbuf **mbufs, unsigned int num)
16 {
17     num = RTE_MIN(kni_fifo_free_count(kni->rx_q), num);
18     void *phy_mbufs[num];
19     unsigned int ret;
20     unsigned int i;
21 
22     for (i = 0; i < num; i++)
23         phy_mbufs[i] = va2pa_all(mbufs[i]);
24 
25     ret = kni_fifo_put(kni->rx_q, phy_mbufs, num);
26 
27     /* Get mbufs from free_q and then free them */
28     kni_free_mbufs(kni);
29 
30 
31 }
```

1.接收报文时，从kni->tx_q直接取走所有报文。前面内核用KNI发送报文时，填充的就是这个fifo。当取走了报文后，DPDK应用层的调用kni_allocate_mbufs，负责给tx_q填充空闲mbuf，供内核使用。

2.发送报文时，先将要发送给KNI的报文地址转换为物理地址，然后enqueue到kni->rx_q中（内核的KNI实现也是从这个fifo中读取报文），最后调用kni_free_mbufs释放掉内核处理完的mbuf报文





原文链接：https://www.cnblogs.com/mysky007/p/12305257.html 原文作者： 坚持，每天进步一点点
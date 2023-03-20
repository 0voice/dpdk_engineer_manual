# dpvs中fdir与sa_pool介绍

## 1.问题引入

dpvs是dpdk程序，特点就是每个核尽可能不与其他核交互，这就要求共享数据都有一份拷贝，或是数据私有。fnat模式中流表（session）保存连接信息，每个核独有。但这里有个问题，full-nat模式下，返程数据outbound packet也必须分配到与inbound packet收取的同一个lcore，否则在流表中找不到conn

dpvs解决方案

引入fdir机制

lcore上lip都是同步一致的，根据本地端口lport，来分配正确的核，此处用于fdir计算的lport根据启用的lcore数量做掩码（即dst_port_mask）

## 2.配置文件fdir

先看一下配置文件

```
<init> device dpdk0 {
		rx {
				queue_number 8
				descriptor_number 1024
				rss all
		}
 
		tx {
				queue_number 8
				descriptor_number 1024
		}
		
		fdir {
				mode perfect
				pballoc 64k
				status matched
		}
		! promisc_mode
		kni_name dpdk0.kni
}
```

可以看到，rx队列配置了rss，并且dpdk0网卡配置了fdir

## 3.默认fdir配置

在对网卡初始化时，会用到default_port_conf，这里有关于网卡fdir默认配置

```
static struct rte_eth_conf default_port_conf =
{
    .rxmode                     =
    {
        .mq_mode        = ETH_MQ_RX_RSS,
        .max_rx_pkt_len = ETHER_MAX_LEN,
        .split_hdr_size =                                  0,
        .offloads       = DEV_RX_OFFLOAD_IPV4_CKSUM,
    },
    .rx_adv_conf                =
    {
        .rss_conf               =
        {
            .rss_key = NULL,
            .rss_hf  = /*ETH_RSS_IP*/ ETH_RSS_TCP,
        },
    },
    .txmode                     =
    {
        .mq_mode                = ETH_MQ_TX_NONE,
    },
    .fdir_conf                  =
    {
        .mode    = RTE_FDIR_MODE_PERFECT,
        .pballoc = RTE_FDIR_PBALLOC_64K,
        .status  = RTE_FDIR_REPORT_STATUS /*_ALWAYS*/,
        .mask    =
        {
            .vlan_tci_mask =                                0x0,
            .ipv4_mask     =
            {
                .src_ip =                         0x00000000,
                .dst_ip =                         0xFFFFFFFF,
            },
            .ipv6_mask          =
            {
                .src_ip         =
                {
                    0, 0, 0, 0
                },
                .dst_ip         =
                { 0xFFFFFFFF, 0xFFFFFFFF, 0xFFFFFFFF, 0xFFFFFFFF },
            },
            .src_port_mask      =                             0x0000,
 
            /* to be changed according to slave lcore number in use */
            .dst_port_mask      =                             0x00F8,
 
            .mac_addr_byte_mask =                               0x00,
            .tunnel_type_mask   =                                  0,
            .tunnel_id_mask     =                                  0,
        },
        .drop_queue =                                127,
        .flex_conf  =
        {
            .nb_payloads  =                                  0,
            .nb_flexmasks =                                  0,
        },
    },
};
```

rx_adv_conf关于RSS配置，默认是ETH_RSS_TCP

最重要的是fdir_conf，可以看到mode，pballoc，status等。这里关注mask即可，fdir支持不同层的导流，ipv4_mask.src_ip掩码是0，ipv4_mask.dst_ip位全置1，所以fdir只看目的地ip不看源ip，src_port_mask是0，dst_port_mask非0，也就是说dpvs fdir只根据dst_ip,dst_port_mask计算，也就是对应<lip,lport>，由于lip只有一个，所以等同于只看lport，那么如何如何设置dst_port_mask掩码呢？

dst_port_mask设置参见下面netif_start_port流程

## 4.sa_pool与fdir初始化

每个lcore都有自己的sa_pool，用于管理本地分配的<lip,lport>，假如当前启用了64个lcore，一共有65535-1024可用端口，那么每个lcore在同一个lip上最多使用(65535-1024)/64个地址

相关结构体

```
enum
{
    SA_F_USED = 0x01,
};
 
/**
 * if really need to to save memory, we can;
 * 1. use hlist_head
 * 2. use uint8_t flag
 * 3. remove sa_entry.addr, and get IP from sa_pool->ifa
 * 4. to __packed__ sa_entry.
 * 5. alloc sa_entries[] for 65536/cpu_num only.
 * 6. create sa_entry_pool only if pool_hash hit.
 *    since when dest (like RS) num may small.
 */
 
/* socket address (sa) is <ip, port> pair. */
struct sa_entry
{
    struct list_head list;              /* node of sa_pool. */
    //标志,目前只有SA_F_USED,主要标记该端口是否被使用了(空闲/busy状态)
    uint32_t         flags;             /* SA_F_XXX */
    union inet_addr  addr;
    __be16           port;
};
 
struct sa_entry_pool
{
    //sa_entry hash表
    struct sa_entry  sa_entries[MAX_PORT];
    //used sa_entry链表头
    struct list_head used_enties;
    //free sa_entry链表头
    struct list_head free_enties;
 
    /* another way is use total_used/free_cnt in sa_pool,
     * so that we need not travels the hash to get stats.
     * we use cnt here, since we may need per-pool stats. */
     //统计计数
    uint16_t         used_cnt;
    uint16_t         free_cnt;
    uint32_t         miss_cnt;
};
 
/* no lock needed because inet_ifaddr.sa_pool
 * is per-lcore. */
struct sa_pool
{
    //sa_pool所属的ip层地址块
    struct inet_ifaddr *  ifa;          /* back-pointer */
    //low,high端口
    uint16_t              low;          /* min port */
    uint16_t              high;         /* max port */
    //引用计数
    rte_atomic32_t        refcnt;
 
    /* hashed pools by dest's <ip/port>. if no dest provided,
     * just use first pool. it's not need create/destroy pool
     * for each dest, that'll be too complicated. */
    struct sa_entry_pool *pool_hash;
    //hash表中桶个数
    uint8_t               pool_hash_sz;
    uint32_t              flags;        /* SA_POOL_F_XXX */
 
    /* fdir filter ID */
    uint32_t              filter_id[MAX_FDIR_PROTO];
};
 
struct sa_fdir
{
    /* the ports one lcore can use means
     * "(fdir.mask & port) == port_base" */
    //掩码
    uint16_t  mask;                     /* filter's port mask */
//所在lcore id,每个lcore上分配一个
    
    lcoreid_t lcore;
//在选取本地lcore可用port时时，lport 经过掩码后，得到的值如果等于 port_base 就会分配到这个核心
    __be16    port_base;
    uint16_t  soft_id;                  /* current unsed soft-id,
                                         * increase after use. */
};
```

程序初始化时调用sa_pool_init初始化全局fdir表

```
static struct sa_fdir sa_fdirs[DPVS_MAX_LCORE];
int sa_pool_init(void)
{
    int       shift;
    lcoreid_t cid;
    uint16_t  port_base;
 
    /* enabled lcore should not change after init */
    //获取已启用的lcore个数和掩码
    netif_get_slave_lcores(&sa_nlcore, &sa_lcore_mask);
 
    /* how many mask bits needed ? */
    //计算log2(sas_nlcore),向上取整
    for (shift = 0; (0x1 << shift) < sa_nlcore; shift++)
    {
        ;
    }
    if (shift >= 16)
    {
        return(EDPVS_INVAL); /* bad config */
    }
    port_base = 0;
    for (cid = 0; cid < DPVS_MAX_LCORE; cid++)
    {
        //如果cid>=64或者该lcore没有被启用，跳过
        if (cid >= 64 || !(sa_lcore_mask & (1L << cid)))
        {
            continue;
        }
        assert(rte_lcore_is_enabled(cid) && cid != rte_get_master_lcore());
        //sa_fdirs是per-lcore数据结构，设置mask掩码，主要是通过&操作代替取余操作
        sa_fdirs[cid].mask      = ~((~0x0) << shift);
        sa_fdirs[cid].lcore     = cid;
        //fdir计算时，lport 经过掩码后，得到的值如果等于 port_base 就会分配到这个核心
        sa_fdirs[cid].port_base = htons(port_base);
        sa_fdirs[cid].soft_id   = 0;
 
        port_base++;
    }
 
    return(EDPVS_OK);
}
```

在使用ipvsadm添加lip时，ifa_add_set调用sa_pool_create初始化sa_pool

```
int sa_pool_create(struct inet_ifaddr *ifa, uint16_t low, uint16_t high)
{
    int             err;
    struct sa_pool *ap;
    struct sa_fdir *fdir;
    uint32_t        filtids[MAX_FDIR_PROTO];
    lcoreid_t       cid = rte_lcore_id();
    //判断当前lcore是否在sa_lcore_mask中
    if (cid > 64 || !((sa_lcore_mask & (1UL << cid))))
    {
        if (cid == rte_get_master_lcore())
        {
            return(EDPVS_OK); /* no sapool on master */
        }
        return(EDPVS_INVAL);
    }
    //low,high端口默认值分别是1025,65535
    low  = low ? : DEF_MIN_PORT;
    high = high ? : DEF_MAX_PORT;
    //参数校验
    if (!ifa || low > high || low == 0 || high >= MAX_PORT)
    {
        RTE_LOG(ERR, SAPOOL, "%s: bad arguments\\n", __func__);
        return(EDPVS_INVAL);
    }
    //通过lcore id获取对应的sa_fdir对象
    fdir = &sa_fdirs[cid];
    //分配sa_pool对象
    ap = rte_zmalloc(NULL, sizeof(struct sa_pool), 0);
    if (unlikely(!ap))
    {
        return(EDPVS_NOMEM);
    }
    //设置sa_pool相关参数
    ap->ifa   = ifa;
    ap->low   = low;
    ap->high  = high;
    ap->flags = 0;
    rte_atomic32_set(&ap->refcnt, 1);
    //分配sa_pool中socket地址的哈希表，默认hash桶的数量为16
    err = sa_pool_alloc_hash(ap, sa_pool_hash_size, fdir);
    if (err != EDPVS_OK)
    {
        goto free_ap;
    }
    
    filtids[0] = fdir->soft_id++;
    filtids[1] = fdir->soft_id++;
    //增加fdir filter,用于fdir匹配
    err        = sa_add_filter(ifa->af, ifa->idev->dev, cid, &ifa->addr,
                               fdir->port_base, filtids); /* thread-safe ? */
    if (err != EDPVS_OK)
    {
        goto free_hash;
    }
 
    ap->filter_id[0] = filtids[0];
    ap->filter_id[1] = filtids[1];
 
    ifa->sa_pool = ap;
 
    /* inc ifa->refcnt to hold it */
    rte_atomic32_inc(&ifa->refcnt);
 
#ifdef CONFIG_DPVS_SAPOOL_DEBUG
    {
        char addr[64];
        RTE_LOG(INFO, SAPOOL, "[%02d] %s: sa pool created -- %s\\n", rte_lcore_id(),
                __func__, inet_ntop(ifa->af, &ifa->addr, addr, sizeof(addr)) ? : NULL);
    }
#endif
 
    return(EDPVS_OK);
 
free_hash:
    sa_pool_free_hash(ap);
free_ap:
    rte_free(ap);
    return(err);
}
```

sa_pool_alloc_hash

```
static int sa_pool_alloc_hash(struct sa_pool *ap, uint8_t hash_sz,
                              const struct sa_fdir *fdir)
{
    int hash;
    struct sa_entry_pool *pool;
    uint32_t port; /* should be u32 or 65535==0 */
    //分配sa_pool->pool_hash桶分配
    ap->pool_hash = rte_malloc(NULL, sizeof(struct sa_entry_pool) * hash_sz,
                               RTE_CACHE_LINE_SIZE);
    if (!ap->pool_hash)
    {
        return(EDPVS_NOMEM);
    }
    //设置哈希表桶大小为hash_sz
    ap->pool_hash_sz = hash_sz;
 
    /* the big loop takes about 17ms */
    for (hash = 0; hash < hash_sz; hash++)
    {
        pool = &ap->pool_hash[hash];
        //初始化used_entry和free_entry链表头
        INIT_LIST_HEAD(&pool->used_enties);
        INIT_LIST_HEAD(&pool->free_enties);
 
        pool->used_cnt = 0;
        pool->free_cnt = 0;
        //根据fdir->mask &&((uint16_t)port & fdir->mask) == ntohs(fdir->port_base) 为当前lcore分配地址
        for (port = ap->low; port <= ap->high; port++)
        {
            struct sa_entry *sa;
 
            if (fdir->mask &&
                ((uint16_t)port & fdir->mask) != ntohs(fdir->port_base))
            {
                continue;
            }
 
            sa       = &pool->sa_entries[(uint16_t)port];
            sa->addr = ap->ifa->addr;
            sa->port = htons((uint16_t)port);
            list_add_tail(&sa->list, &pool->free_enties);
            pool->free_cnt++;
        }
    }
 
    return(EDPVS_OK);
}
```

sa_add_filter

```
static inline int sa_add_filter(int af, struct netif_port *dev, lcoreid_t cid,
                                const union inet_addr *dip, __be16 dport,
                                uint32_t filter_id[MAX_FDIR_PROTO])
{
    return(__add_del_filter(af, dev, cid, dip, dport, filter_id, true));
}
__add_del_filter
```

主要对（目的ip，目的port），更确切讲是dst_port&fdir_conf.mask.dst_port_mask后的端口进行flow director设置

fdir_conf.mask.dst_port_mask的设置详见netif_start中调用netif_port_fdir_dstport_mask_set

```
static int __add_del_filter(int af, struct netif_port *dev, lcoreid_t cid,
                            const union inet_addr *dip, __be16 dport,
                            uint32_t filter_id[MAX_FDIR_PROTO], bool add)
{
    queueid_t          queue;
    int                err;
    enum rte_filter_op op, rop;
    //flow Director设置,action为接收数据
    struct rte_eth_fdir_filter filt[MAX_FDIR_PROTO] =
    {
        {
            .action.behavior      = RTE_ETH_FDIR_ACCEPT,
            .action.report_status = RTE_ETH_FDIR_REPORT_ID,
            .soft_id = filter_id[0],
        },
        {
            .action.behavior      = RTE_ETH_FDIR_ACCEPT,
            .action.report_status = RTE_ETH_FDIR_REPORT_ID,
            .soft_id = filter_id[1],
        },
    };
    
    if (af == AF_INET)
    {
        //IPv4 flow dirctor设置,只对(目的ip,目的端口）进行过滤，，具体为做过mask.dst_port_mask掩码后的dport
				//当dport为0，lcore_count=4,则0,4,8,12......的目的端口会进入该lcore
        filt[0].input.flow_type = RTE_ETH_FLOW_NONFRAG_IPV4_TCP;
        filt[0].input.flow.tcp4_flow.ip.dst_ip = dip->in.s_addr;
        filt[0].input.flow.tcp4_flow.dst_port  = dport;
        filt[1].input.flow_type = RTE_ETH_FLOW_NONFRAG_IPV4_UDP;
        filt[1].input.flow.udp4_flow.ip.dst_ip = dip->in.s_addr;
        filt[1].input.flow.udp4_flow.dst_port  = dport;
    }
    else if (af == AF_INET6)
    {
        filt[0].input.flow_type = RTE_ETH_FLOW_NONFRAG_IPV6_TCP;
        memcpy(filt[0].input.flow.ipv6_flow.dst_ip, &dip->in6, sizeof(struct in6_addr));
        filt[0].input.flow.tcp6_flow.dst_port = dport;
        filt[1].input.flow_type = RTE_ETH_FLOW_NONFRAG_IPV6_UDP;
        memcpy(filt[1].input.flow.ipv6_flow.dst_ip, &dip->in6, sizeof(struct in6_addr));
        filt[1].input.flow.udp6_flow.dst_port = dport;
    }
    else
    {
        return(EDPVS_NOTSUPP);
    }
    //判断设备是否flow director功能，主要对dpdk设备和bond设备，内部调用rte_eth_dev_filter_supported
    if (dev->netif_ops && dev->netif_ops->op_filter_supported)
    {
        if (dev->netif_ops->op_filter_supported(dev, RTE_ETH_FILTER_FDIR) < 0)
        {
            if (dev->nrxq <= 1)
            {
                return(EDPVS_OK);
            }
            RTE_LOG(ERR, SAPOOL, "%s: FDIR is not supported by device %s. Only"
                    " single rxq can be configured.\\n", __func__, dev->name);
            return(EDPVS_NOTSUPP);
        }
    }
    else
    {
        RTE_LOG(ERR, SAPOOL, "%s: FDIR support of device %s is not known.\\n",
                __func__, dev->name);
        return(EDPVS_INVAL);
    }
    //获取lcore上处理port的哪些接收队列
    err = netif_get_queue(dev, cid, &queue);
    if (err != EDPVS_OK)
    {
        return(err);
    }
    //设置flow Director的接收队列为queue，绑定网卡硬件队列
    filt[0].action.rx_queue = filt[1].action.rx_queue = queue;
    op = add ? RTE_ETH_FILTER_ADD : RTE_ETH_FILTER_DELETE;
    //将过滤条件更新到网卡
    netif_mask_fdir_filter(af, dev, &filt[0]);
    netif_mask_fdir_filter(af, dev, &filt[1]);
    //内部调用rte_eth_dev_filter_ctrl添加flow filter规则
    err = netif_fdir_filter_set(dev, op, &filt[0]);
    if (err != EDPVS_OK)
    {
        return(err);
    }
 
    err = netif_fdir_filter_set(dev, op, &filt[1]);
    if (err != EDPVS_OK)
    {
        rop = add ? RTE_ETH_FILTER_DELETE : RTE_ETH_FILTER_ADD;
        netif_fdir_filter_set(dev, rop, &filt[0]);
        return(err);
    }
 
#ifdef CONFIG_DPVS_SAPOOL_DEBUG
    {
        char ipaddr[64];
        RTE_LOG(DEBUG, SAPOOL, "FDIR: %s %s %s TCP/UDP "
                "ip %s port %d (0x%04x) mask 0x%04X queue %d lcore %2d filterID %d/%d\\n",
                add ? "add" : "del", dev->name,
                af == AF_INET ? "IPv4" : "IPv6",
                inet_ntop(af, dip, ipaddr, sizeof(ipaddr)) ? : "::",
                ntohs(dport), ntohs(dport), sa_fdirs[cid].mask, queue, cid,
                filter_id[0], filter_id[1]);
    }
#endif
 
    return(err);
}
```

netif_mask_fdir_filter

对目前设置做与操作，屏蔽之前设置的flow director信息

```
void netif_mask_fdir_filter(int af, const struct netif_port *port,
                            struct rte_eth_fdir_filter *filt)
{
    struct rte_eth_fdir_info         fdir_info;
    const struct rte_eth_fdir_masks *fmask;
    //首先获取fdir_filter中的input flow定义
    union rte_eth_fdir_flow *        flow = &filt->input.flow;
    /**
      union rte_eth_fdir_flow {
	struct rte_eth_l2_flow     l2_flow;
	struct rte_eth_udpv4_flow  udp4_flow;
	struct rte_eth_tcpv4_flow  tcp4_flow;
	struct rte_eth_sctpv4_flow sctp4_flow;
	struct rte_eth_ipv4_flow   ip4_flow;
	struct rte_eth_udpv6_flow  udp6_flow;
	struct rte_eth_tcpv6_flow  tcp6_flow;
	struct rte_eth_sctpv6_flow sctp6_flow;
	struct rte_eth_ipv6_flow   ipv6_flow;
	struct rte_eth_mac_vlan_flow mac_vlan_flow;
	struct rte_eth_tunnel_flow   tunnel_flow;
    };
      */
 
    /* There exists a defect here. If the netif_port 'port' is not PORT_TYPE_GENERAL,
     * mask fdir_filter of the port would fail. The correct way to accomplish the
     * function is to register this method for all device types. Considering the flow
     * is not changed after masking, we just skip netif_ports other than physical ones. */
    if (port->type != PORT_TYPE_GENERAL)
    {
        return;
    }
    //retrieve fdir information
    if (rte_eth_dev_filter_ctrl(port->id, RTE_ETH_FILTER_FDIR,
                                RTE_ETH_FILTER_INFO, &fdir_info) < 0)
    {
        RTE_LOG(DEBUG, NETIF, "%s: Fail to fetch fdir info of %s !\\n",
                __func__, port->name);
        return;
    }
    fmask = &fdir_info.mask;
 
    /* ipv4 flow */
    if (af == AF_INET)
    {
        flow->ip4_flow.src_ip    &= fmask->ipv4_mask.src_ip;
        flow->ip4_flow.dst_ip    &= fmask->ipv4_mask.dst_ip;
        flow->ip4_flow.tos       &= fmask->ipv4_mask.tos;
        flow->ip4_flow.ttl       &= fmask->ipv4_mask.ttl;
        flow->ip4_flow.proto     &= fmask->ipv4_mask.proto;
        flow->tcp4_flow.src_port &= fmask->src_port_mask;
        flow->tcp4_flow.dst_port &= fmask->dst_port_mask;
        return;
    }
 
    /* ipv6 flow */
    if (af == AF_INET6)
    {
        flow->ipv6_flow.src_ip[0]  &= fmask->ipv6_mask.src_ip[0];
        flow->ipv6_flow.src_ip[1]  &= fmask->ipv6_mask.src_ip[1];
        flow->ipv6_flow.src_ip[2]  &= fmask->ipv6_mask.src_ip[2];
        flow->ipv6_flow.src_ip[3]  &= fmask->ipv6_mask.src_ip[3];
        flow->ipv6_flow.dst_ip[0]  &= fmask->ipv6_mask.dst_ip[0];
        flow->ipv6_flow.dst_ip[1]  &= fmask->ipv6_mask.dst_ip[1];
        flow->ipv6_flow.dst_ip[2]  &= fmask->ipv6_mask.dst_ip[2];
        flow->ipv6_flow.dst_ip[3]  &= fmask->ipv6_mask.dst_ip[3];
        flow->ipv6_flow.tc         &= fmask->ipv6_mask.tc;
        flow->ipv6_flow.proto      &= fmask->ipv6_mask.proto;
        flow->ipv6_flow.hop_limits &= fmask->ipv6_mask.hop_limits;
        flow->tcp6_flow.src_port   &= fmask->src_port_mask;
        flow->tcp6_flow.dst_port   &= fmask->dst_port_mask;
        return;
    }
}
```

netif_port_fdir_dstport_mask_set

在netif_port_start启动网卡时调用

```
/*
 * fdir mask must be set according to configured slave lcore number
 * */
inline static int netif_port_fdir_dstport_mask_set(struct netif_port *port)
{
    uint8_t slave_nb;
    int     shift;
 
    netif_get_slave_lcores(&slave_nb, NULL);
    for (shift = 0; (0x1 << shift) < slave_nb; shift++)
    {
        ;
    }
    if (shift >= 16)
    {
        RTE_LOG(ERR, NETIF, "%s: %s's fdir dst_port_mask init failed\\n",
                __func__, port->name);
        return(EDPVS_NOTSUPP);
    }
#if RTE_VERSION >= 0x10040010
    port->dev_conf.fdir_conf.mask.dst_port_mask = htons(~((~0x0) << shift));
#else
    port->dev_conf.fdir_conf.mask.dst_port_mask = ~((~0x0) << shift);
#endif
 
    RTE_LOG(INFO, NETIF, "%s:dst_port_mask=%0x\\n", port->name,
            port->dev_conf.fdir_conf.mask.dst_port_mask);
    return(EDPVS_OK);
}
```

## 5.conn流表中使用fdir

上文fdir设置主要是用于fnat中outbound方向数据流量回到dpvs时，与inbound流量进入dpvs所在lcore保持一致，避免流表查找的锁和缓存失效等。

主要看下dpvs中如何从sa_pool中分配本地地址和本地端口，对于所有新建连接，都会调用dp_vs_conn_new注册流表

dp_vs_conn_new

```
只有 full-nat 才支持 local address/port
struct dp_vs_conn *dp_vs_conn_new(struct rte_mbuf *mbuf,
                                  const struct dp_vs_iphdr *iph,
                                  struct dp_vs_conn_param *param,
                                  struct dp_vs_dest *dest,
                                  uint32_t flags)
{
		//full-nat 特殊处理
    if (dest->fwdmode == DPVS_FWD_MODE_FNAT)
    {
        //绑定LB本地socket，fullnat做了双向nat
        if ((err = dp_vs_laddr_bind(new, dest->svc)) != EDPVS_OK)
        {
            goto unbind_dest;
        }
    }
}
```

dp_vs_laddr_bind

```
int dp_vs_laddr_bind(struct dp_vs_conn *conn, struct dp_vs_service *svc)
{
    struct dp_vs_laddr *laddr = NULL;
    int      i;
    uint16_t sport = 0;
    struct sockaddr_storage dsin, ssin;
    bool found = false;
 
    //一些常规校验
    if (!conn || !conn->dest || !svc)
    {
        return(EDPVS_INVAL);
    }
    //如果传输层协议不是TCP或者UDP也直接返回错误，因为在设置fdir时只关注了tcp和udp协议
    if (svc->proto != IPPROTO_TCP && svc->proto != IPPROTO_UDP)
    {
        return(EDPVS_NOTSUPP);
    }
    if (dp_vs_conn_is_template(conn))
    {
        return(EDPVS_OK);
    }
 
    /*
     * some time allocate lport fails for one laddr,
     * but there's also some resource on another laddr.
     */
    for (i = 0; i < dp_vs_laddr_max_trails && i < svc->num_laddrs; i++)
    {
        /* select a local IP from service */
        //首先选择一个laddr
        laddr = __get_laddr(svc);
        if (!laddr)
        {
            RTE_LOG(ERR, IPVS, "%s: no laddr available.\\n", __func__);
            return(EDPVS_RESOURCE);
        }
        //首先将dsin与ssin清零
        memset(&dsin, 0, sizeof(struct sockaddr_storage));
        memset(&ssin, 0, sizeof(struct sockaddr_storage));
 
        if (laddr->af == AF_INET)
        {
            //fnat要将目的信息修改为conn选择的RS的ip:port,同时将源ip和port修改为local ip相关,因为要保证outbound方向数据
            //同样返回值dpvs
            struct sockaddr_in *daddr, *saddr;
            daddr             = (struct sockaddr_in *)&dsin;
            daddr->sin_family = laddr->af;
            daddr->sin_addr   = conn->daddr.in;
            daddr->sin_port   = conn->dport;
            saddr             = (struct sockaddr_in *)&ssin;
            saddr->sin_family = laddr->af;
            saddr->sin_addr   = laddr->addr.in;
        }
        else
        {
            struct sockaddr_in6 *daddr, *saddr;
            daddr = (struct sockaddr_in6 *)&dsin;
            daddr->sin6_family = laddr->af;
            daddr->sin6_addr   = conn->daddr.in6;
            daddr->sin6_port   = conn->dport;
            saddr = (struct sockaddr_in6 *)&ssin;
            saddr->sin6_family = laddr->af;
            saddr->sin6_addr   = laddr->addr.in6;
        }
        //sa_fetch获取端口,来填充完整的dsin,ssin地址,选取port主要用于通过fdir后outbound方向数据返回当前lcore处理
        if (sa_fetch(laddr->af, laddr->iface, &dsin, &ssin) != EDPVS_OK)
        {
            char buf[64];
            if (inet_ntop(laddr->af, &laddr->addr, buf, sizeof(buf)) == NULL)
            {
                snprintf(buf, sizeof(buf), "::");
            }
 
#ifdef CONFIG_DPVS_IPVS_DEBUG
            RTE_LOG(DEBUG, IPVS, "%s: [%02d] no lport available on %s, "
                    "try next laddr.\\n", __func__, rte_lcore_id(), buf);
#endif
            put_laddr(laddr);
            continue;
        }
        //sport为选取出来的port
        sport = (laddr->af == AF_INET ? (((struct sockaddr_in *)&ssin)->sin_port)
                 : (((struct sockaddr_in6 *)&ssin)->sin6_port));
        found = true;
        break;
    }
    //如果选取laddr和lport失败,返回错误
    if (!found)
    {
#ifdef CONFIG_DPVS_IPVS_DEBUG
        RTE_LOG(ERR, IPVS, "%s: [%02d] no lip/lport available !!\\n",
                __func__, rte_lcore_id());
#endif
        return(EDPVS_RESOURCE);
    }
 
    rte_atomic32_inc(&laddr->conn_counts);
 
    /* overwrite related fields in out-tuplehash and conn */
    //设置conn的local address和local port
    conn->laddr = laddr->addr;
    conn->lport = sport;
    //设置outbound方向tuplehash_entry的连接地址信息
    tuplehash_out(conn).daddr = laddr->addr;
    tuplehash_out(conn).dport = sport;
    //关联选取的laddr
    conn->local = laddr;
    return(EDPVS_OK);
}
```

__get_laddr

```
static inline struct dp_vs_laddr *__get_laddr(struct dp_vs_service *svc)
{
    int step;
    struct dp_vs_laddr *laddr = NULL;
 
    /* if list not inited ? list_empty() returns true ! */
    assert(svc->laddr_list.next);
    //如果没有laddr,直接返回
    if (list_empty(&svc->laddr_list))
    {
        return(NULL);
    }
    //随记产生一个步数
    step = __laddr_step(svc);
    while (step-- > 0)
    {
        if (unlikely(!svc->laddr_curr))
        {
            svc->laddr_curr = svc->laddr_list.next;
        }
        else
        {
            svc->laddr_curr = svc->laddr_curr->next;
        }
        //循环链表,过滤链表头
        if (svc->laddr_curr == &svc->laddr_list)
        {
            svc->laddr_curr = svc->laddr_list.next;
        }
    }
    //获取struct dp_vs_laddr
    laddr = list_entry(svc->laddr_curr, struct dp_vs_laddr, list);
    rte_atomic32_inc(&laddr->refcnt);
 
    return(laddr);
}
```

__laddr_step

```
static inline int __laddr_step(struct dp_vs_service *svc)
{
    /* Why can't we always use the next laddr(rr scheduler) to setup new session?
     * Because realserver rr/wrr scheduler may get synchronous with the laddr rr
     * scheduler. If so, the local IP may stay invariant for a specified realserver,
     * which is a hurt for realserver concurrency performance. To avoid the problem,
     * we just choose 5% sessions to use the one after the next laddr randomly.
     * */
    if (strncmp(svc->scheduler->name, "rr", 2) == 0 ||
        strncmp(svc->scheduler->name, "wrr", 3) == 0)
    {
        return((random() % 100) < 5 ? 2 : 1);
    }
 
    return(1);
}
```

sa_fetch

包裹函数

```
int sa_fetch(int af, struct netif_port *dev,
             const struct sockaddr_storage *daddr,
             struct sockaddr_storage *saddr)
{
    if (unlikely(daddr && daddr->ss_family != af))
    {
        return(EDPVS_INVAL);
    }
    if (unlikely(saddr && saddr->ss_family != af))
    {
        return(EDPVS_INVAL);
    }
    if (AF_INET == af)
    {
        return(sa4_fetch(dev, (const struct sockaddr_in *)daddr,
                         (struct sockaddr_in *)saddr));
    }
    else if (AF_INET6 == af)
    {
        return(sa6_fetch(dev, (const struct sockaddr_in6 *)daddr,
                         (struct sockaddr_in6 *)saddr));
    }
    else
    {
        return(EDPVS_NOTSUPP);
    }
}
```

sa4_fetch

关注下ipv4 lport的选取

查找未使用的<saddr,sport>

```
/*
 * fetch unused <saddr, sport> pair by given hint.
 * given @ap equivalent to @dev+@saddr, and dport is useless.
 * with routing's help, the mapping looks like,
 *
 * +------+------------+-------+-------------------
 * |      |     ap     |       | Is possible to
 * |daddr | dev & saddr| sport | fetch addr pair?
 * +------+------------+-------+-------------------
 *    Y      Y     ?       Y       Possible
 *    Y      Y     Y       ?       Possible
 *    Y      Y     ?       ?       Possible
 *    Y      N     ?       Y       Possible
 *    Y      N     Y       ?       Possible
 *    Y      N     ?       ?       Possible
 *    N      Y     ?       Y       Possible
 *    N      Y     Y       ?       Possible
 *    N      Y     ?       ?       Possible
 *    N      N     ?       Y       Not Possible
 *    N      N     Y       ?       Possible
 *    N      N     ?       ?       Not Possible
 *
 * daddr is a hint to found dev/saddr (by route/netif module).
 * dev is also a hint, the saddr(ifa) is the key.
 * af is needed when both saddr and daddr are NULL.
 */
static int sa4_fetch(struct netif_port *dev,
                     const struct sockaddr_in *daddr,
                     struct sockaddr_in *saddr)
{
    struct inet_ifaddr *ifa;
    struct flow4        fl;
    struct route_entry *rt;
    int err;
 
    assert(saddr);
    //首先对参数进行校验
    if (saddr && saddr->sin_addr.s_addr != INADDR_ANY && saddr->sin_port != 0)
    {
        return(EDPVS_OK); /* everything is known, why call this function ? */
    }
 
    /* if source IP is assiged, we can find ifa->sa_pool
     * without @daddr and @dev. */
     //如果指定了sin_addr
    if (saddr->sin_addr.s_addr)
    {
        //在net_device上获取对应local ip对应的IP地址信息块inet_ifaddr,主要用于获取sa_pool
        ifa = inet_addr_ifa_get(AF_INET, dev, (union inet_addr *)&saddr->sin_addr);
        if (!ifa)
        {
            return(EDPVS_NOTEXIST);
        }
        //如果没有配置sa_pool,则返回出错
        if (!ifa->sa_pool)
        {
            RTE_LOG(WARNING, SAPOOL, "%s: fetch addr on IP without sapool.", __func__);
            inet_addr_ifa_put(ifa);
            return(EDPVS_INVAL);
        }
        //获取lport
        err = sa_pool_fetch(sa_pool_hash(ifa->sa_pool, (struct sockaddr_storage *)daddr),
                            (struct sockaddr_storage *)saddr);
        if (err == EDPVS_OK)
        {
            rte_atomic32_inc(&ifa->sa_pool->refcnt);
        }
        inet_addr_ifa_put(ifa);
        return(err);
    }
 
    /* try to find source ifa by @dev and @daddr */
    //如果未指定saddr,则首先根据目的地址查找出口路由
    memset(&fl, 0, sizeof(struct flow4));
    fl.fl4_oif          = dev;
    fl.fl4_daddr.s_addr = daddr ? daddr->sin_addr.s_addr : htonl(INADDR_ANY);
    fl.fl4_saddr.s_addr = saddr ? saddr->sin_addr.s_addr : htonl(INADDR_ANY);
    rt = route4_output(&fl);
    if (!rt)
    {
        return(EDPVS_NOROUTE);
    }
 
    /* select source address. */
    //选择一个源ip地址
    if (!rt->src.s_addr)
    {
        inet_addr_select(AF_INET, rt->port, (union inet_addr *)&rt->dest,
                         RT_SCOPE_UNIVERSE, (union inet_addr *)&rt->src);
    }
    //根据选取的源ip获取对应的ip信息控制块
    ifa = inet_addr_ifa_get(AF_INET, rt->port, (union inet_addr *)&rt->src);
    if (!ifa)
    {
        route4_put(rt);
        return(EDPVS_NOTEXIST);
    }
    route4_put(rt);
 
    if (!ifa->sa_pool)
    {
        RTE_LOG(WARNING, SAPOOL, "%s: fetch addr on IP without pool.",
                __func__);
        inet_addr_ifa_put(ifa);
        return(EDPVS_INVAL);
    }
 
    /* do fetch socket address */
    err = sa_pool_fetch(sa_pool_hash(ifa->sa_pool,
                                     (struct sockaddr_storage *)daddr),
                        (struct sockaddr_storage *)saddr);
    if (err == EDPVS_OK)
    {
        rte_atomic32_inc(&ifa->sa_pool->refcnt);
    }
 
    inet_addr_ifa_put(ifa);
    return(err);
}
```

sa_pool_hash

主要用于在sa_pool的不同sa_entry_pool哈希桶中查找

```
/* hash dest's <ip/port>. if no dest provided, just use first pool. */
static inline struct sa_entry_pool *
sa_pool_hash(const struct sa_pool *ap, const struct sockaddr_storage *ss)
{
    uint32_t hashkey;
 
    assert(ap && ap->pool_hash && ap->pool_hash_sz >= 1);
    if (!ss)
    {
        return(&ap->pool_hash[0]);
    }
 
    if (ss->ss_family == AF_INET)
    {
        uint16_t vect[2];
        const struct sockaddr_in *sin = (const struct sockaddr_in *)ss;
 
        vect[0] = ntohl(sin->sin_addr.s_addr) & 0xffff;
        vect[1] = ntohs(sin->sin_port);
        hashkey = (vect[0] + vect[1]) % ap->pool_hash_sz;
 
        return(&ap->pool_hash[hashkey]);
    }
    else if (ss->ss_family == AF_INET6)
    {
        uint32_t vect[5] = { 0 };
        const struct sockaddr_in6 *sin6 = (const struct sockaddr_in6 *)ss;
 
        vect[0] = sin6->sin6_port;
        memcpy(&vect[1], &sin6->sin6_addr, 16);
        hashkey = rte_jhash_32b(vect, 5, sin6->sin6_family) % ap->pool_hash_sz;
 
        return(&ap->pool_hash[hashkey]);
    }
    else
    {
        return(NULL);
    }
}
```

sa_pool_fetch

> 从free_enties中获取第一个free sa_entry节点，设置local ip和port

> 移动到used列表

> 更新统计计数

```
static inline int sa_pool_fetch(struct sa_entry_pool *pool,
                                struct sockaddr_storage *ss)
{
    assert(pool && ss);
 
    struct sa_entry *    ent;
    struct sockaddr_in * sin  = (struct sockaddr_in *)ss;
    struct sockaddr_in6 *sin6 = (struct sockaddr_in6 *)ss;
    //从free_entries链表中获取第一个可用sa_entry
    ent = list_first_entry_or_null(&pool->free_enties, struct sa_entry, list);
    if (!ent)
    {
#ifdef CONFIG_DPVS_SAPOOL_DEBUG
        RTE_LOG(DEBUG, SAPOOL, "%s: no entry (used/free %d/%d)\\n", __func__,
                pool->used_cnt, pool->free_cnt);
#endif
        pool->miss_cnt++;
        return(EDPVS_RESOURCE);
    }
    //设置local ip和port
    if (ss->ss_family == AF_INET)
    {
        sin->sin_family      = AF_INET;
        sin->sin_addr.s_addr = ent->addr.in.s_addr;
        sin->sin_port        = ent->port;
    }
    else if (ss->ss_family == AF_INET6)
    {
        sin6->sin6_family = AF_INET6;
        sin6->sin6_addr   = ent->addr.in6;
        sin6->sin6_port   = ent->port;
    }
    else
    {
        return(EDPVS_NOTSUPP);
    }
    //标记entry正在使用中
    ent->flags |= SA_F_USED;
    //将其移动到used列表中
    list_move_tail(&ent->list, &pool->used_enties);
    //同时更新使用统计信息
    pool->used_cnt++;
    pool->free_cnt--;
 
#ifdef CONFIG_DPVS_SAPOOL_DEBUG
    {
        char addr[64];
        RTE_LOG(DEBUG, SAPOOL, "%s: %s:%d fetched!\\n", __func__,
                inet_ntop(ss->ss_family, &ent->addr, addr, sizeof(addr)) ? : NULL,
                ntohs(ent->port));
    }
#endif
 
    return(EDPVS_OK);
}
```

原文链接：[https://blog.csdn.net/zjx345438858/article/details/108124950](https://blog.csdn.net/zjx345438858/article/details/108124950) 原文作者：ss_chris
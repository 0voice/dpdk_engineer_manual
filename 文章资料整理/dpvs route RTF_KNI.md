# dpvs route RTF_KNI

## 路由条目的意义

```
default via 10.10.16.254 dev enahisic2i0 proto static 
9.251.0.0/16 via 172.17.0.1 dev docker0 
10.10.16.0/24 dev enahisic2i0 proto kernel scope link src 10.10.16.47 
10.99.1.231 via 10.10.16.82 dev enahisic2i0 
10.110.79.116 via 10.10.16.82 dev enahisic2i0 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
```

## dpip命令

```
./dpip addr add 10.0.0.100/32 dev dpdk1
```

```
# route for WAN/LAN access
# add routes for other network or default route if needed.
./dpip route add 10.0.0.0/16 dev dpdk1
./dpip route add 192.168.100.0/24 dev dpdk0
```

##  **dpvs 路由操作**

路由配置工具是使用 dpip 来进行相关的配置

```
# 查看路由信息
$ dpip route show

# 添加路由
$ dpip route add 192.168.100.0/24 dev bond0

# 删除路由
$ dpip route del 192.168.100.0/24 dev bond0

$ dpip route flush

# 查看 dpip route 帮助信息
$ dpip route help
```

## **路由配置的实现过程**

dpip 通过 unix socket 与 dpvs 的 master 线程进行通信，master 根据消息类型执行路由模块注册的 sockopt 函数：

```
static struct dpvs_sockopts route_sockopts = {
    .version        = SOCKOPT_VERSION,
    .set_opt_min    = SOCKOPT_SET_ROUTE_ADD,
    .set_opt_max    = SOCKOPT_SET_ROUTE_FLUSH,
    .set            = route_sockopt_set,        // set
    .get_opt_min    = SOCKOPT_GET_ROUTE_SHOW,
    .get_opt_max    = SOCKOPT_GET_ROUTE_SHOW,
    .get            = route_sockopt_get,        // get
};
```

dpip 将用户输入的路由参数传递给 master，master 线程在当前 cpu 调用 `route_sockopt_set` 函数进行路由配置，设置完毕后通过 msg 将用户设置的路由信息通过多播的方式同步发送给各个 slave 线程，slave 线程在适当的时机执行路由处理函数 `route_msg_process` slave 线程也会按照传递的参数来设置相同的路由信息，所以 master 与各 slave 线程各自维护的路由信息完全一致。

```
可以知道，dpvs 是为了避免路由锁操作，提高系统整体性能，每个 cpu 都维护了份路由信息，这样各个 cpu 读写时无需加锁。
```

##  **路由的几种类型**

当我们执行 `dpip addr add xxx` 添加 IP 地址时，会同时添加相应的路由信息。除非添加 IP 时指定的掩码为 32，否则会同时添加一条 Local 路由和 Net 路由。

- Local 类型路由。local 类型路由的作用和 Linux 下的 local 路由表的功能基本一样，主要是记录本地的 IP 地址。我们知道进入的数据包，过了 prerouting 后是需要经过路由查找，如果确定是本地路由（本地 IP）就会进入 LocalIn 位置，否则丢弃或进入 Forward 代码位置了。Local 类型路由就是用来判定接收数据包是否是本机 IP，在 DPVS 调用的就是 `route4_input` 函数。
- Net 类型路由。数据包经过应用程序处理完毕后，要执行 output 时也需要根据目的 IP 来查找路由，确定从哪个网卡出去，下一跳等信息，然后执行后续操作。**在 DPVS 调用的就是 `route4_output` 函数。**

## **路由的相关数据结构字段**

local 类型路由采用是典型的哈希链表，每个 cpu 维护一张哈希表；而 net 路由却简单实用了普通的链表，猜想可能是 net 路由条目信息会比较少一些，性能影响不大。

```
struct route_lcore {
    struct list_head local_route_table[LOCAL_ROUTE_TAB_SIZE];
    struct list_head net_route_table;
};
#define this_route_lcore        (RTE_PER_LCORE(route_lcore))
#define this_local_route_table  (this_route_lcore.local_route_table)
#define this_net_route_table    (this_route_lcore.net_route_table)
```

路由条目的数据结构字段也相对简单如下：

```
struct route_entry {
    uint8_t netmask;
    short metric;
    uint32_t flag;
    unsigned long mtu;
    struct list_head list;
    struct in_addr dest;
    struct in_addr gw;
    struct in_addr src;
    struct netif_port *port;
    rte_atomic32_t refcnt;
};
```

DPVS 路由系统与邻居模块不同的时，路由信息完全是由管理员来配置，不存在动态学习的过程，理论上多核之间的路由信息同步设计上会稍简单了些。

## ipvsadm

 ipvsadm 是用来管理 LVS 集群配置的，运行在用户态空间，而 LVS 是运行在内核态的。ipvsadm 通过调用 `libipvs` 库，以 `raw socket` 或 `netlink` 的通信方式向 LVS 下发配置信息的，功能与 keepalived 对 LVS 的配置管理功能是完全一样的。ipvsadm 之与 LVS 的关系，就像 iptables 之与 netfilter 的关系。

```
static int route_add_lcore(struct in_addr* dest,uint8_t netmask, uint32_t flag,
              struct in_addr* gw, struct netif_port *port,
              struct in_addr* src, unsigned long mtu,short metric)
{

    if((flag & RTF_LOCALIN) || (flag & RTF_KNI))
        return route_local_add(dest, netmask, flag, gw,
                              port, src, mtu, metric);

    if((flag & RTF_FORWARD) || (flag & RTF_DEFAULT))
        return route_net_add(dest, netmask, flag, gw,
                             port, src, mtu, metric);


    return EDPVS_INVAL;
}
```







转载自： 作者[tycoon3](https://home.cnblogs.com/u/dream397/)   本文链接https://www.cnblogs.com/dream397/p/14764029.html
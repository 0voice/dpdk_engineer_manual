# iQiYi 高性能开源负载均衡器及应用

**[DPVS](https://github.com/iqiyi/dpvs)** 是 iQiYi 采用 DPDK 技术开发的的高性能四层负载均衡器。与 Linux 内核的 LVS （Linux Virtual Server）相比，DPVS 具有如下特点：

- 更高的性能：DPVS 的包处理速度，1 个工作线程可以达到 2.3 Mpps，6 个工作线程可以达到万兆网卡小包的转发线速（约 12Mpps)。这是主要因为 DPVS 绕过了内核复杂的协议栈，并采用轮询的方式收发数据包，避免了锁、内核中断、上下文切换、内核态和用户态数据拷贝产生的性能开销。
- 更完善的功能：从转发转发模式看，DPVS 支持 Direct Routing（DR）、NAT、Tunnel、Full-NAT、SNAT 五种转发模式，可以灵活适配各种网络应用场景；从协议支持上看，DPVS 支持 IPv4 和 IPv6 协议、且最新版本增加了 NAT64 的转发功能，实现了用户从 IPv6 网络访问 IPv4 服务。
- 更好的维护性：DPVS 是一个用户态程序，与内核功能相比，功能开发周期更短、调试更方便、问题修复更及时。

DPVS 自 2017 年 10 月开源以来，受到了业内的广泛关注。2018 年， DPVS 被列入 [DPDK 生态系统](https://www.dpdk.org/ecosystem/)。目前，DPVS 开源社区已经有关注度已突破一千，业内数十个开发者参与并贡献了代码，包括 iQiYi 在内的很多公司已经将 DPVS 应用到生产环境。

## DPVS 基本原理

DPVS 的总体架构如下图所示，下面对相关的几个点着重解释说明一下。

- Master/Worker 模型

  DPVS采用经典的 Master/Worker 模型。Master 处理控制平面，比如参数配置、统计获取等；Worker 实现核心负载均衡、调度、数据转发功能。另外，DPVS 使用多线程模型，每个线程绑定到一个 CPU 物理核心上，并且禁止这些 CPU 被调度。这些 CPU 只运行 DPVS 的 Master 或者某个 Worker，以此避免上下文切换，别的进程不会被调度到这些 CPU，Worker 也不会迁移到其他 CPU 造成缓存失效。

- 网卡队列 /CPU 绑定

  现代的网卡支持多个队列，队列可以和 CPU 绑定，让不同的 CPU 处理不同的网卡队列的流量，分摊工作量，实现并行处理和线性扩展。DPVS是由各个 Worker 使用 DPDK 的 API 处理不同的网卡队列，每个 Worker 处理某网卡的一个接收队列，一个发送队列，实现了处理能力随CPU核心、网卡队列数的增加而线性增长。

- 关键数据 per-cpu及无锁化

  内核性能问题的一大原因就是资源共享和锁。所以，被频繁访问的关键数据需要尽可能的实现无锁化，其中一个方法是将数据做到 per-cpu 化，每个 CPU 只处理自己本地的数据，不需要访问其他 CPU 的数据，这样就可以避免加锁。就 DPVS 而言，连接表，邻居表，路由表等，都是频繁修改或者频繁查找的数据，都做到了 per-cpu 化。

  在具体 per-cpu 的实现上，连接表和邻居表、路由表并不相同。对于连接表，高并发的情况下，不光是查找，还会被频繁地添加、删除。我们让每个 CPU 维护的是不相同的连接表，不同的网络数据流（TCP/UDP/ICMP）按照 N 元组被定向到不同的 CPU，在此特定 CPU 上创建、查找、转发、销毁。同一个数据流的包，只会出现在某个 CPU 上，不会落到其他的 CPU 上。这样就可以做到不同的 CPU 只维护自己本地的表，无需加锁。另一方面，对于邻居和路由表，这种系统“全局”的数据，每个 CPU 都是要用到它们的。如果不采用”全局表+锁保护“的方式，而要做成 per-cpu，也需要让每个 CPU 有同样的视图，也就是每个 CPU 需要维护同样的表。对于这两个表，采用了跨 CPU 无锁同步的方式，虽然在具体实现上有小的差别，本质上都是通过跨 CPU 通信（路由是直接传递信息，邻居是克隆数数据并传递分组给别的 CPU），将表的变化同步到每个 CPU。不论用了什么方法，关键数据做到了 per-cpu 之后没有了锁的需求，性能也就能提升了。

- 用户态轻量级协议栈

  四层负载均衡并不需要完整的协议栈，但还是需要基本的网络组件，以便完成和周围设备的交互（ARP/NS/NA）、确定分组走向 （Route）、回应 Ping 请求、健全性检查（分组完整性，Checksum校验）、以及 IP 地址管理等基本工作。使用 DPDK 提高了收发包性能，但也绕过了内核协议栈，DPVS 依赖的协议栈需要自己实现。

- 跨 CPU 无锁消息

  之前已经提到过这点了。首先，虽然采用了关键数据 per-cpu等优化，但跨 CPU 还是需要通信的，比如:

  - Master 获取各个 Worker 的各种统计信息
  - Master 将路由、黑名单等配置同步到各个 Worker
  - Master 将来自 KNI 的数据发送到 Worker（只有 Worker 能操作 DPDK 接口发送数据）

- 既然需要通信，就不能存在互相影响、相互等待的情况，因为那会影响性能。为此，我们使用了 DPDK 提供的无锁 rte_ring 库，从底层保证通信是无锁的，并且我们在此之上封装一层消息机制来支持一对一，一对多，同步或异步的消息。

![img](https://image.jiqizhixin.com/uploads/editor/61708319-54bf-412e-a3ad-63da5fc59de3/DPVS_%E6%80%BB%E4%BD%93%E6%9E%B6%E6%9E%84.png)

下图给出了 DPVS 详细的功能模块，主要包含如下五大部分：

- 网络设备层

  负责数据包收发和设备管理，支持 vlan、bonding、tunnel 等设备，支持 KNI 通信、流量控制。

- 轻量级协议栈层

  轻量级的 IPv4 和 IPv6 三层协议栈，包括邻居、路由、地址管理等功能。

- IPVS 转发层

  五种数据转发模式的连接管理、业务管理、调度算法、转发处理等。特别地，Full-NAT 转发模式下支持了 IPv6-to-IPv4（NAT64） 转发、 SYN flood 攻击防御、 TCP/UDP 的源地址获取（toa/uoa）等功能。

- 基础模块

  包含定时器、CPU 消息、进程通信接口、配置文件等基础功能模块。

- 控制面和工具

  用于配置和管理 DPVS 服务的工具，包括 ipvsadm、keepalived、dpip，也支持使用进行 quagga 集群化部署。

![image](https://user-images.githubusercontent.com/87457873/132093182-b291375a-b974-4116-9c9d-7d363c111a7c.png)

## DPVS 的功能和应用

DPVS 能够提供灵活多样的四层数据转发和负载均衡服务，下面我们通过实例介绍几个 DPVS 的常用功能和应用场景。

#### 1. 流量均衡

为高并发、大流量的业务提供流量均衡和访问入口地址是负载均衡器的基本功能。下图是 DPVS 采用 Full-NAT 转发方式组成的 IPv6 负载均衡网络：多台应用服务器部署在内部网络，并通过 DPVS 组成一个服务集群；在外部网络， DPVS 为该应用服务集群提供访问的外网入口 IP 地址 2001:db8::1（VIP)。用户对内部应用服务的请求将会通过 DPVS 转发到内部多台应用服务器中的其中一台上，解决单台应用服务器并发、流量等服务能力限制问题。

![img](https://image.jiqizhixin.com/uploads/editor/0b9ed5f8-b616-4427-8f27-a7ab58dedaba/%E6%B5%81%E9%87%8F%E5%9D%87%E8%A1%A1.jpg)

下面我们给出上述 IPv6 负载均衡网络对应的 DPVS 的网络和业务转发规则配置方法。

```
``` ./bin/dpip -6 addr add 2001:db8::1/64 dev eth1            # VIP
./bin/dpip -6 addr add 2001:db8:10::141/64 dev eth1        # local IP
 ./bin/ipvsadm -At [2001:db8::1]:80 -j enable            # TCP
./bin/ipvsadm -Pt [2001:db8::1]:80 -z 2001:db8:10::141 -F eth0 ./bin/ipvsadm -at [2001:db8::1]:80 -r [2001:db8:11::51]:80 -b ./bin/ipvsadm -at [2001:db8::1]:80 -r [2001:db8:11::52]:80 -b ./bin/ipvsadm -at [2001:db8::1]:80 -r [2001:db8:11::53]:80 -b 
./bin/ipvsadm -Au [2001:db8::1]:80                        # UDP
./bin/ipvsadm -Pu [2001:db8::1]:80 -z 2001:db8:10::141 -F eth0 ./bin/ipvsadm -au [2001:db8::1]:80 -r [2001:db8:11::51]:6000 -b ./bin/ipvsadm -au [2001:db8::1]:80 -r [2001:db8:11::52]:6000 -b ./bin/ipvsadm -au [2001:db8::1]:80 -r [2001:db8:11::53]:6000 -b
```

DPVS 转发规则配置结果如下。

```
Prot LocalAddress:Port Scheduler Flags  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  [2001:db8::1]:80 wlc synproxy  -> [2001:db8:11::51]:80         FullNat 1      0          0           -> [2001:db8:11::52]:80         FullNat 1      0          0           -> [2001:db8:11::53]:80         FullNat 1      0          0         UDP  [2001:db8::1]:80 wlc  -> [2001:db8:11::51]:6000       FullNat 1      0          0           -> [2001:db8:11::52]:6000       FullNat 1      0          0           -> [2001:db8:11::53]:6000       FullNat 1      0          0   
```

#### 2. NAT64

目前国内互联网服务正处于 IPv4 到 IPv6 的升级过渡阶段，运营商、客户端已经逐步支持了 IPv6 网络，但对于各种各样的内部复杂网络应用服务，从 IPv4 切到 IPv6 网络意味着繁重的内部基础网络的重建和应用服务的开发更新。通过 DPVS 的 Full-NAT 转发模式提供的 NAT64 转发功能，可以在不需要改变内部 IPv4 基础网络的情况下，为内部的应用服务提供 IPv6 的网络接入能力。

下图给出了这种情况下的网络结构，可以看到，用户（Client）网络是 IPv6 网络，应用服务（RealServer）的网络是 IPv4 网络，DPVS 使用 IPv6 的 VIP（Virtual IP）和 IPv4 的 Local IP，将 用户的 IPv6 请求转发到内部的 IPv4 网络，并将内部 IPv4 应用服务器的响应数据返回给 IPv6 用户。

![img](https://image.jiqizhixin.com/uploads/editor/1c3278e0-3812-4fc0-9433-e8d8cc4e9631/NAT64.jpg)

DPVS 的网络和业务转发规则配置方法如下。

```
./bin/dpip -6 addr add 2001:db8::1/64 dev eth1    # VIP
./bin/dpip addr add 192.168.88.141/24 dev eth0    # local IP
./bin/dpip addr add 192.168.88.142/24 dev eth0    # local IP
 ./bin/ipvsadm -At [2001:db8::1]:80 -j enable    # TCP
./bin/ipvsadm -Pt [2001:db8::1]:80 -z 192.168.88.141 -F eth0 ./bin/ipvsadm -Pt [2001:db8::1]:80 -z 192.168.88.142 -F eth0 ./bin/ipvsadm -at [2001:db8::1]:80 -r 192.168.12.51:80 -b ./bin/ipvsadm -at [2001:db8::1]:80 -r 192.168.12.53:80 -b ./bin/ipvsadm -at [2001:db8::1]:80 -r 192.168.12.54:80 -b 
./bin/ipvsadm -Au [2001:db8::1]:80        # UDP
./bin/ipvsadm -Pu [2001:db8::1]:80 -z 192.168.88.141 -F eth0 ./bin/ipvsadm -Pu [2001:db8::1]:80 -z 192.168.88.142 -F eth0 ./bin/ipvsadm -au [2001:db8::1]:80 -r 192.168.12.51:6000 -b ./bin/ipvsadm -au [2001:db8::1]:80 -r 192.168.12.53:6000 -b ./bin/ipvsadm -au [2001:db8::1]:80 -r 192.168.12.54:6000 -b
```

DPVS 转发规则配置结果如下。

```
Prot LocalAddress:Port Scheduler Flags  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn TCP  [2001:db8::1]:80 wlc synproxy  -> 192.168.12.51:80             FullNat 1      0          0           -> 192.168.12.53:80             FullNat 1      0          0           -> 192.168.12.54:80             FullNat 1      0          0         UDP  [2001:db8::1]:80 wlc  -> 192.168.12.51:6000           FullNat 1      0          0           -> 192.168.12.53:6000           FullNat 1      0          0           -> 192.168.12.54:6000           FullNat 1      0          0 
```

#### 3. SNAT 访问外网

大部分企业内网的用户，由于资源、安全等诸多因素的限制，无法直接访问 Internet 外部网络。DPVS SNAT 转发模式提供了一种高性能、安全可控的外网访问方式。如下图，内网用户没有外网网卡和地址，但想要访问 Internet 上地址为 59.37.97.124 的网络资源。我们可以通过路由、内网隧道等途径，让该用户对外网的访问请求经过 DPVS SNAT 服务器；SNAT 服务器上配置相应的转发规则，将用户的私有内网源地址 102.168.88.59 转换为 DPVS 的外网地址 101.227.17.140，然后请求 Internet 上的资源。Internet 的资源响应后，响应数据首先发到 DPVS SNAT 服务器，SNAT 服务器把目标地址 101.227.17.140 更换回用户的私有地址 192.168.88.59 后，把响应数据传给用户。

![img](https://image.jiqizhixin.com/uploads/editor/aaadabfe-373a-4561-8ca7-dcf11165085c/SNAT.jpg)

DPVS 的网络和 SNAT 数据转发规则配置方法如下。

```
./bin/dpip addr add 192.168.88.1/24 dev dpdk0               # VIP
./bin/dpip addr add 101.227.17.140/25 dev bond1             # WLAN IP
./bin/dpip route add default via 101.227.17.254 dev bond1   # default Gateway

# TCP Rule
./bin/ipvsadm -A -H proto=tcp,src-range=192.168.88.1-192.168.88.253,oif=bond1 -s rr ./bin/ipvsadm -a -H proto=tcp,src-range=192.168.88.1-192.168.88.253,oif=bond1 -r 101.227.17.140:0 -J
# UDP Rule
./bin/ipvsadm -A -H proto=udp,src-range=192.168.88.1-192.168.88.253,oif=bond1 -s rr ./bin/ipvsadm -a -H proto=udp,src-range=192.168.88.1-192.168.88.253,oif=bond1 -r 101.227.17.140:0 -J
# ICMP Rule
./bin/ipvsadm -A -H proto=icmp,src-range=192.168.88.1-192.168.88.253,oif=bond1 -s rr ./bin/ipvsadm -a -H proto=icmp,src-range=192.168.88.1-192.168.88.253,oif=bond1 -r 101.227.17.140:0 -J
```

DPVS 转发规则配置结果如下。

```
Prot LocalAddress:Port Scheduler Flags  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn MATCH icmp,from=192.168.88.1-192.168.88.253:0-0,oif=bond1 rr  -> 101.227.17.140:0             SNAT    1      0          0         MATCH tcp,from=192.168.88.1-192.168.88.253:0-0,oif=bond1 rr  -> 101.227.17.140:0             SNAT    1      0          0         MATCH udp,from=192.168.88.1-192.168.88.253:0-0,oif=bond1 rr  -> 101.227.17.140:0             SNAT    1      0          0 
```

## 

## DVPS的性能和高可用性

自 2018年底，DPVS开始发布 v1.7 版本。该版本的核心功能是支持了 IPv6-to-IPv6包的四层转发（v1.7.0） 和IPv6-to-IPv4 包的四层转发  （v1.7.2）。下图给出了该版本三种协议转发的性能数据对比（由于测试环境的限制，目前我们只测试了 3 个 Worker 以下的性能数据），可以看到纯 IPv6转发的性能与纯 IPv4 转发性能相当，IPv6-to- IPv4转发时由于在三层协议转换时有数据拷贝开销，性能相对差一些。 

![img](https://image.jiqizhixin.com/uploads/editor/425e3b99-0919-48f3-99c2-610d5deb9c93/1551924901800.png)DPVS作为四层负载均衡器，在 iQiYi生产环境中运营已有 2年多的时间。目前线上有上千条业务转发规则，每天承接着来自公司内部、外部共近 5T的服务带宽和 1Billion并发连接量。下图是我们一种常用的 DPVS集群化部署方式：多台 DPVS服务器通过等价多路径路由构成一个服务集群，每台 DPVS服务器配置多台 nginx作为其后端服务器，nginx  以反向代理的方式为最终的应用服务器提供七层负载均衡和高可用性服务。这是一种高可用、高伸缩性的集群化方式，任何一台 APPServer或中间转发节点的故障不会影响整体服务，四层负载均衡器 DPVS、七层反向代理服务器 nginx和应用服务器都能在不影响现有服务的条件下很方便地完成扩容。 

![img](https://image.jiqizhixin.com/uploads/editor/a61de65b-1e52-41d7-9c29-ec2542840dc1/1551924920186.png)

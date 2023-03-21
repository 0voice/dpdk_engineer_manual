# DPDK——环境准备与编写第一个简易的NAT程序

## 1.前言

正如摘要所述，博主我早期是用这么一种奇怪的方式实现的SDN，至于性能嘛，emmmmm想必你应该也知道了。近期正好在进行数据平面开发套件的调研，喊香喊了半年的DPDK一定也就成为了第一个初探的对象。本篇文章，就来一起准备一下DPDK的开发环境，并且来写一个虽然还不能用的简单的程序~

由于DPDK的加速原理说白了就是绕内核协议栈、共享内存不复制、用户程序轮询不中断，涉及了比较大的知识面，因此在这里我建议阅读本文的朋友们，如果你感兴趣并且打算玩一玩这东西，可以先去看看参考文献中的1[1]，这本书是截止到目前为止，我个人认为的唯一一本从理论到代码均比较完善的书了，花半天翻一下大致了解一下工作原理和代码逻辑即可~

这篇文章中，博主我使用的环境为**虚拟机**，分配了4C10G的内存，其中4G以4张1GB大页的形式预留给了DPDK。在这个虚拟机中，开发使用的CLion with Projector，以方便远程开发与简易调试。如果你也想使用Projector开发，可以看我早时候的一篇文章~



## 2.注意

本篇文章的代码、拓扑仅为测试开发使用，部分操作也会比较简单粗暴，如有仿照部署的兄弟，还请多多谅解，可能会有些问题。



## 3.拓扑

本文所使用的拓扑如下。为一个非常简易的拓扑，三台虚拟机即可，上联的路由器也只是带外管理使用。这个测试环境在广东珠海，通过我的雪糕云SD-WAN与成都端互通，因此我也就可以直接访问我的测试环境啦~如果你也需要低成本的异地组网解决方案或者需要定制开发，欢迎在博客的关于页面里找到我的联系方式并咨询~



![图片](https://mmbiz.qpic.cn/mmbiz_svg/rqvn1hjHytcf6giaLHCxpTZjLDkbF800SwZbBPKO0XEL8VWrIzE42afRh7QDic4cZvicUG6VGtdeqAy5YURtQrUsyx2M4YTZDib6/640?wx_fmt=svg&wxfrom=5&wx_lazy=1&wx_co=1)



在这个拓扑中，由于DPDK使用UIO驱动接管了两张网卡，因此就不会在系统中显示了，我就直接使用总线编号代替了接口名来绘图~



## 4.环境安装

环境安装部分，主要为安装新版本DPDK库到操作系统中。此处我并不是非常建议使用预编译的二进制文件，因为实际上到生产环境部分参数要进行调整，使用源代码编译方便修改参数。



### 4.1DPDK测试机操作

▼ 当然，一开始一定是得先把用到的依赖装齐了。由于Rocky Linux 8.4的源中带的CMake版本过低[2]，要直接使用源代码编译安装最新版本的CMake，因此会安装很多额外的依赖。

```c++
yum install python3 python3-devel libibverbs-devel openssl-devel gcc vim nano -y
yum group install "Development Tools" -y
python3 -m pip install meson
python3 -m pip install ninja

```

**请兄弟萌务必检查SELinux是否关闭了！**如果开着SELinux，没有添加policy的程序会因为不允许执行相关的系统调用而无法正常运行。

▼ 装完依赖了，紧接着要给DPDK预留大页内存[2]。在系统启动时就预留，而不是在系统启动后才预留，这样一来就可以防止内存分配出现碎片的问题。如果你使用虚拟机的情况下，请确保内存的分配为完整预留，以避免内存分配出现碎片。但是，由于博主我目前还不是特别清楚KVM对于虚拟机的大页内存管理是否有抽象，因此暂时无法断言虚拟机中的大页和物理机中的大页性能差别，这个还需后边查证后再补充。

```c++
# cat /etc/default/grub
...
GRUB_CMDLINE_LINUX="resume=UUID=1233a4c9-aba7-486d-838d-b6d0f1626f26 default_hugepagesz=1G hugepagesz=1G hugepages=4 intel_iommu=off"
...

```

由于此处我使用UIO的驱动而不是VFIO，因此我将IOMMU关闭了。注意，AMD平台与Intel平台关闭IOMMU的方式不一样，详细请见参考文献[2]。

在网络上看到有*VFIO比UIO更加安全*的说法，这个晚点还要再查证一下原理再来解释。

更新了grub配置之后，记得不要忘记重新生成对应的grub启动文件~需要重新启动才能生效，重启后，就可以通过命令`grep HugePages_ /proc/meminfo`看到是否生效。

▼ 由于我图省事，因此就直接把大页内存手动挂载了一下。如果后续还要使用，可以直接添加到fstab中去。

```c++
mkdir /mnt/hugepage
mount -t hugetlbfs pagesize=4GB /mnt/hugepage

```

▼ 紧接着，我们就要开始编译CMake了。在源码编译之前，记得卸载系统存在的cmake~

```c++
wget https://github.com/Kitware/CMake/releases/download/v3.20.5/cmake-3.20.5.tar.gz
tar xvf cmake-3.20.5.tar.gz 
cd cmake-3.20.5/
./bootstrap 
gmake -j4
gmake install

```

▼ 再接着，下载DPDK并且进行编译[3]。

```c++
wget http://fast.dpdk.org/rel/dpdk-20.11.2.tar.xz
tar xvf dpdk-20.11.2.tar.xz 
cd dpdk-stable-20.11.2/
meson build
ninja -C build
cd build && ninja install

```

▼ 由于源码编译安装的DPDK安装pkg-config的文件时路径有些特殊，可能会导致pkg-config无法正常找到依赖，如果出现这个问题的话，需要手工修复一下。可以通过`pkg-config --list-all`看一下libdpdk是否存在，如果不存在的话，就需要手工拷贝一下对应的文件。或者你也可以使用符号链接

```c++
cp /usr/local/lib64/pkgconfig/*/usr/lib64/pkgconfig/

```

修复完成后，使用命令`pkg-config --cflags libdpdk`可以检查一下，如果显示的参数正确，那么就可以继续后续的事情辣~

▼ 最后，加载对应的驱动并且将网卡绑定即可~查看对应网卡的总线编号可以使用`lspci`

```c++
modprobe uio_pci_generic
dpdk-devbind.py --bind=uio_pci_generic 0000:00:13.0
dpdk-devbind.py --bind=uio_pci_generic 0000:00:14.0
dpdk-devbind.py --status

```

完成环境准备后，就可以开始开发辣~



### 4.2打流测试机操作

打流测试机即为刚刚拓扑图中的C1与C2。由于我使用l2fwd例程进行测试，因此需要对两台打流测试机进行一些操作，大概分别如下两个代码块。

代码块中的部分操作实际上是我写完了NAT动作之后才加上的，此处因为正好提到，因此就提前写上了。

```c++
ip addr add 1.1.1.1/30 dev ens19
ip neigh del 1.1.1.2 dev ens19 && ip neigh add 1.1.1.2 lladdr 11:11:11:11:11:11 dev ens19
ip link set dev ens19 address 02:00:00:00:00:00

```

```c++
ip addr add 1.1.1.2/30 dev ens19ip neigh del 1.1.1.1 dev ens19 && ip neigh add 1.1.1.1 lladdr 22:22:22:22:22:22 dev ens19ip link set dev ens19 address 02:00:00:00:00:01
ip route add 8.8.8.8 via 1.1.1.1 dev ens19

```



## 4.代码改动

注意，由于本文只是初探DPDK，因此就不搞太高端的玩法了，博主我就稍微改了一下官方l2fwd的例程，做了一个NAT的测试程序。这个NAT测试程序在官方原本的l2fwd基础之上，增加了IP地址`8.8.8.8`与`1.1.1.1`的映射。是的，只会改地址（含计算校验和），没有其他内容了，也没有记录Session，因此真的完完全全是个“网络地址转换”，也不能正常使用的那种，但是测试看看效果还是可以的。

```c++
static void
l2fwd_mac_updating(struct rte_mbuf *m, unsigned dest_portid)
{
    struct rte_ether_hdr *eth;
    void *tmp;

    eth = rte_pktmbuf_mtod(m, struct rte_ether_hdr *);

    /* 02:00:00:00:00:xx */
    tmp = &eth->d_addr.addr_bytes[0];
    *((uint64_t *)tmp) = 0x000000000002 + ((uint64_t)dest_portid << 40);

    /* match addr */
    rte_ether_addr_copy(&l2fwd_ports_eth_addr[dest_portid], &eth->s_addr);

    /* update src or dest address */
    struct rte_ipv4_hdr *ipv4;
    ipv4 = rte_pktmbuf_mtod_offset(m, struct rte_ipv4_hdr *, sizeof(struct rte_ether_hdr));
    uint8_t match[4] = {0x1, 0x1, 0x1, 0x1};
    uint8_t want[4] = {0x8, 0x8, 0x8, 0x8};
    if(ipv4->src_addr == *((uint32_t *)match)) {
        ipv4->src_addr = *((uint32_t *)want);
    } else {
        ipv4->dst_addr = *((uint32_t *)want);
    }
    ipv4->hdr_checksum = 0;
    ipv4->hdr_checksum = rte_ipv4_cksum(ipv4);
}
```



在这个函数里，我只增加了`update src or dest address`下边那几行内容。基本思路就是，得到相应的mbuf[4]的位置（在DPDK中所有的包实际上都仅仅是一个内存中的偏移量，这也正是零复制的精髓），计算IPv4头部偏移量，然后比对地址进行修改就好了~最后，使用DPDK库中的函数重新计算一下校验和即可~



## 5.测试结果

由于本文仅仅是对DPDK的一个预热，因此我就不再做性能测试的数据了~感兴趣的朋友们可以自行使用虚拟机的CPU性能计数器分析~

图片字太小不方便看的话，可以图片右键新窗口打开就好啦~

在这个测试过程中，命令直接参考官方的[5][6]即可，不要忘记使用大页就好了。

▼ 博主我在测试时，忘记了使用大页，不过不会有太大影响。可以看到对MAC地址的修改和对IP地址的修改已经生效了。这张图是DPDK运行的机器上的，由于使用了轮询，因此CPU会一直处于拉满的状态。

![图片](https://mmbiz.qpic.cn/mmbiz_png/7Ln62BuRwMib40CkG2PNJN9IbqJABlY5Bk7GBBfRtSKurlWKXj2wqDNHKSbgq3YOGtHeCfsuQeaejgZWNqGX0Rw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

▼ 下图是C1设备上一直发起ping，使得数据包通过DPDK的机器到达另外一个VM（C2）。可以看到抓包回程的MAC地址已经被修改，同时目标IP也已经被修改。

![图片](https://mmbiz.qpic.cn/mmbiz_png/7Ln62BuRwMib40CkG2PNJN9IbqJABlY5BFTSVDpuC2mI1iaCb3u5myiaRAGib9mu8P6Eux11zJfp3AKT55uhMyEMicg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

▼ 下图是C2设备上的抓包，可以看到到达的ICMP包源MAC地址已经被修改，同时源IP也被我刚刚的代码逻辑修改为了8.8.8.8，校验和也正常，回程就会正常回复到8.8.8.8。

![图片](https://mmbiz.qpic.cn/mmbiz_png/7Ln62BuRwMib40CkG2PNJN9IbqJABlY5BzAX02Qp84pJJ6lnWOV8Y1Et2iboibkjDhVbzIHfQCfXqaibfg6XBO51QA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



## 6.最后

这时候，你可能会发现一个问题。为什么代码里判断目标IP为`1.1.1.1`的修改源IP为`8.8.8.8`，回程进入else中，修改了目标IP为`8.8.8.8`，可是目标IP本来就是8.8.8.8啊？确定是该这样做的吗？

是的，确实是这样，因为我不小心写错了（逃

▼ 于是我把代码改了改，改成了下边这样。

```c++
static void
l2fwd_mac_updating(struct rte_mbuf *m, unsigned dest_portid)
{
    struct rte_ether_hdr *eth;
    void *tmp;

    eth = rte_pktmbuf_mtod(m, struct rte_ether_hdr *);

    /* 02:00:00:00:00:xx */
    tmp = &eth->d_addr.addr_bytes[0];
    *((uint64_t *)tmp) = 0x000000000002 + ((uint64_t)dest_portid << 40);

    /* origin addr */
    rte_ether_addr_copy(&l2fwd_ports_eth_addr[dest_portid], &eth->s_addr);

    /* update origin and dest address */
    struct rte_ipv4_hdr *ipv4;
    ipv4 = rte_pktmbuf_mtod_offset(m, struct rte_ipv4_hdr *, sizeof(struct rte_ether_hdr));
    uint8_t origin[4] = {0x1, 0x1, 0x1, 0x1};
    uint8_t want[4] = {0x8, 0x8, 0x8, 0x8};
    if(ipv4->src_addr == *((uint32_t *)origin)) {
        ipv4->src_addr = *((uint32_t *)want);
    }
    if(ipv4->dst_addr == *((uint32_t *)want)) {
        ipv4->dst_addr = *((uint32_t *)origin);
    }
    ipv4->hdr_checksum = 0;
    ipv4->hdr_checksum = rte_ipv4_cksum(ipv4);
}
```



▼ 进而再使用`./l2fwd-static -l 2-3 -n 2 --huge-dir /mnt/hugepage -- -q 2 -p 0x3`启动即可，即可看到回程已经正常~

```c++
[root@T-DPDK-C1 ~]# ping 1.1.1.2 
PING 1.1.1.2 (1.1.1.2) 56(84) bytes of data.
64 bytes from 1.1.1.2: icmp_seq=1 ttl=64 time=0.454 ms
64 bytes from 1.1.1.2: icmp_seq=2 ttl=64 time=0.534 ms
64 bytes from 1.1.1.2: icmp_seq=3 ttl=64 time=0.500 ms
64 bytes from 1.1.1.2: icmp_seq=4 ttl=64 time=0.472 ms
64 bytes from 1.1.1.2: icmp_seq=5 ttl=64 time=0.382 ms
64 bytes from 1.1.1.2: icmp_seq=6 ttl=64 time=0.345 ms

```

```c++
[root@T-DPDK-C2 ~]# tcpdump -i ens19 icmp -n -vv -e
dropped privs to tcpdump
tcpdump: listening on ens19, link-type EN10MB (Ethernet), capture size 262144 bytes
17:38:44.551530 ba:f1:dd:39:19:4e > 02:00:00:00:00:01, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 45076, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 1.1.1.2: ICMP echo request, id 1049, seq 13, length 64
17:38:44.551591 02:00:00:00:00:01 > 02:00:ff:ff:00:00, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 31794, offset 0, flags [none], proto ICMP (1), length 84)
    1.1.1.2 > 8.8.8.8: ICMP echo reply, id 1049, seq 13, length 64
17:38:45.575475 ba:f1:dd:39:19:4e > 02:00:00:00:00:01, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 45937, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 1.1.1.2: ICMP echo request, id 1049, seq 14, length 64
17:38:45.575526 02:00:00:00:00:01 > 02:00:ff:ff:00:00, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 32101, offset 0, flags [none], proto ICMP (1), length 84)
    1.1.1.2 > 8.8.8.8: ICMP echo reply, id 1049, seq 14, length 64
17:38:46.599476 ba:f1:dd:39:19:4e > 02:00:00:00:00:01, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 46186, offset 0, flags [DF], proto ICMP (1), length 84)
    8.8.8.8 > 1.1.1.2: ICMP echo request, id 1049, seq 15, length 64
17:38:46.599538 02:00:00:00:00:01 > 02:00:ff:ff:00:00, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 32203, offset 0, flags [none], proto ICMP (1), length 84)
    1.1.1.2 > 8.8.8.8: ICMP echo reply, id 1049, seq 15, length 64
```

原文链接：https://mp.weixin.qq.com/s/EejMy5N_XQxBkK04NE-_rQ

原文作者：雪糕


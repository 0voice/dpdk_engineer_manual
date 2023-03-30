# Pktgen-DPDK 网络性能测试

在未使用该工具之前，一直使用的是 iperf 在 10G 网卡场景下进行 64 字节小包性能测试。若要将 64 字节小包流量发到限速，要么一直增加 iperf 客户端，或者在一个高配 iperf 客户端虚拟机中不断的增加 iperf 进程。即使是这样，将发包达到网卡上限，却依然无法利用好 6wind 的性能。所以考虑使用 DPDK-pktgen 发包工具。该工具基于 DPDK 快速报文处里框架开发，以内核模块的形式存在于系统。

## 1.环境部署

### 1.1 安装DPDK

DPDK 可以将用户态的数据不经过内核直接转发到网卡，同样网卡数据也不经过内核直接映射到用户态内存实现加速目的。

使用 pktgen-dpdk 工具，需要先安装 DPDK 环境。下面内容以 18.05 版本的 DPDK 为例进行简要介绍。获取安装包地址请移步：[DPDK Download](https://link.zhihu.com/?target=http%3A//core.dpdk.org/download/)

1. 添加环境变量。这里把环境变量添加到 `/root/.bashrc`，使环境变量永久生效。

```bash
export RTE_SDK=<DPDKInstallDir>
export RTE_TARGET=x86_64-native-linuxapp-gcc
```

2. 安装所需依赖包。需要注意 kernel 相关安装包版本必须要和操作系统内核版本一致，否则会出现安装失败。

```bash
# yum install gcc-c++ gcc glib-devel numactl-devel numactl-libs 
# yum install kernel-headers kernel-devel
```

3. 使用脚本安装 DPDK 环境。具体每一步这里不详细描述，可参考其他资料。

```bash
[root@vm1 ~]# cd dpdk-18.05
[root@vm1 dpdk-18.05]# ./usertools/dpdk-setup.sh

# 需执行的步骤如下
[15] x86_64-native-linuxapp-gcc                     # 下载环境
[18] Insert IGB UIO module                          # 加载igb_uio驱动
[21] Setup hugepage mappings for non-NUMA systems   # 配置大页内存
[24] Bind Ethernet/Crypto device to IGB UIO module  # 绑定要使用DPDK的网卡

# 查看配置信息
[23] Display current Ethernet/Crypto device settings
[29] List hugepage info from /proc/meminfo
```

### 1.2 安装 pktgen-dpdk

下面简单介绍该工具的安装步骤。获取安装包请移步：[pktgen-dpdk](https://link.zhihu.com/?target=http%3A//git.dpdk.org/apps/pktgen-dpdk)

1. 环境编译。在编译结束之后，目录`app/build/pktgen`就是编译出来的程序。

```bash
[root@vm1 ~]# cd pktgen-dpdk
[root@vm1 pktgen-dpdk]# make
```

2. 在安装过程中若出现如下错误，需要给 lua 源码打 patch。 错误信息如下：

```text
No rule to make target `/root/pktgen-dpdk-pktgen-3.5.1/app/../lib/lua/src/x86_64-native-linuxapp-gcc/lib/librte_lua.a'
```

解决办法：

```bash
[root@vm1 lua]# pwd
/root/pktgen-dpdk/lib/lua
[root@wangyuwei-1 lua]# patch -p0 < lua-5.3.4.patch   安装补丁
```

## 2.测试过程

学会使用一个工具最好的方式就是通过 help 查看帮助文档。这里不对使用方式进行详细说明，将只介绍在之后测试中我使用到的参数。

### 2.1 网卡之间互打流量

以该测试为例，对使用 pktgen-dpdk 工具先产生感性认识，比较容易理解该工具的主要功能和基本的使用方式。然后再通过编写 lua 脚本构造数据包，实现自己真正需要测试的内容和获取的信息结果。

**环境说明：**

使用 SRIOV 配置多个 VF 网卡，将其中的三个 VF 网卡配置给虚拟机，一个 VF 绑定可访问的 ip 地址，将另外两个 VF 绑定为使用 igb_uio 驱动。

另外，使用 pktgen-dpdk 管理的网卡内核是看不到的，所以无法使用 ip command 对其进行查看或者操作。事实上，作为 traffic gen 也没有必要配置接口 ip 地址。

1. 测试脚本

```bash
# ./app/build/pktgen -l 2-10 -n 4 --proc-type auto --socket-mem 1024 -- -P -m "[3-4:5-6].0,[7-8:9-10].1" -f themes/black-yellow.theme
```

2. 执行结果

![img](https://pic2.zhimg.com/80/v2-fa66074c22696182be310ec4db57eccd_720w.webp)

以上结果符合预期，Port 0 进行发包，Port 1 进行收包。这两个网卡属于同一块物理网卡的 VF ，所以万兆网卡带宽进行了均分。

### 2.2 虚拟机与 NAT 网关互打流量

**环境说明：**

1. 在物理机 Host01 上，使用 PCI passthrough 将两个物理网卡透传给虚拟机 DPDKGEN vm1 。进入虚拟机，将两个物理网卡配置成 DPDK 驱动。
2. 在物理机 Host02 上，使用 PCI passthrough 将两个物理网卡透传给虚拟机 6wind，进行如下配置。

```text
# 1. 配置ip地址
6wind:
    ens3: 10.0.0.1
    ens7: 20.0.0.1

# 2. 配置arp，将发包机指定的目的ip地址与dpdkgen p1 mac地址进行绑定
root@router:~# arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
20.0.0.2                 ether   90:e2:ba:f7:58:59   CM                    eth1
```

**测试拓扑图：**

![img](https://pic2.zhimg.com/80/v2-2c27dd2b23027bc808846ab8b5519815_720w.webp)

1. 测试脚本

```bash
# ./app/build/pktgen -l 2-10 -n 4 --proc-type auto --socket-mem 1024 -- -P -m "[3-4:5-6].0,[7-8,9-10].1"

Pktgen:/> set 0 src ip 10.0.0.2/24              # 虚拟一个与6wind ens3 同网段的ip地址
Pktgen:/> set 0 src mac 90:e2:ba:f7:58:58       # dpdkgen p0 mac 地址
Pktgen:/> set 0 dst ip 20.0.0.2                 # 虚拟一个与6wind ens7 同网段的ip地址，注意做arp配置
Pktgen:/> set 0 dst mac 90:e2:ba:f7:53:10       # 6wind ens3 mac 地址
Pktgen:/> set 0 proto udp                       # 配置64字节udp数据包
Pktgen:/> set 0 size 64
Pktgen:/> start 0
```

2. 执行结果

6wind cpu 利用率：

```bash
root@router:~# fp-cpu-usage 
Fast path CPU usage:
cpu: %busy     cycles   cycles/packet   cycles/ic pkt
  1:   <1%     156790            8537               0
  2:  100%  458889164             209               0
  3:   <1%      75228            5761               0
average cycles/packets received from NIC: 209 (458989044/2186254)
```

dpdkgen结果如下：

![img](https://pic2.zhimg.com/80/v2-eb3e338cc3c41b2220a44a181f7c4465_720w.webp)

从以上结果可以看到，在使用 6wind 一个 core 情况下，64 字节线速发包能力为 9914Mbps ，包转发性能为 14Mpps，接近物理网卡极限。





原文链接：https://zhuanlan.zhihu.com/p/154066913  原文作者：追风亦追梦
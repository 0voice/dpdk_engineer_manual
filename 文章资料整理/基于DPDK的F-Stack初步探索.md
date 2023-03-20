# 基于DPDK的F-Stack初步探索

## 1.DPDK开发背景

近年来，随着网络技术的不断创新和市场的发展，越来越多的网络设备基础架构开始向基于通用处理器平台的架构方面融合，期望用更低的成本和更短的产品开发周期来提供多样的网络单元和丰富的功能，如应用处理、控制处理、包处理、信号处理等。

为了适应这一新的产业趋势，英特尔公司联合第三方软件开发公司及时推出了基于Intel®x86架构的DPDK(Data Plane Development Kit，数据面开发套件)，实现了高效灵活的包处理解决方案。

经过多年时间，DPDK已经发展成支持多种高性能网卡和多通用处理器平台的开源软件工具包，并已成为通用处理器平台上影响力最大的数据面解决方案。



## 2.F-Stack 开发背景

随着网卡性能的飞速发展，10GE 网卡已经大规模普及，25GE/40GE/100GE 网卡也在逐步推广，linux 内核在网络数据包处理上的瓶颈也越发明显。

在传统的内核协议栈中，网卡通过硬件中断通知协议栈有新的数据包到达，内核的网卡驱动程序负责处理这个硬件中断，将数据包从网卡队列拷贝到内核开辟的缓冲区中（DMA），然后数据包经过一系列的协议处理流程，最后送到用户程序指定的缓冲区中。

在这个过程中中断处理、内存拷贝、系统调用（锁、软中断、上下文切换）等严重影响了网络数据包的处理能力。操作系统对应用程序和数据包处理的调度可能跨 CPU 调度，局部性失效进一步影响网络性能。



![图片](https://mmbiz.qpic.cn/mmbiz_png/ibu0S1GTZ5TzPfgjkQic4YYbQviaR3XK2nA3PYSwANPyZp9qibONfA90a6xacsVM01GubsmmcNQjTliaMJFedEI6Gzw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



而互联网的快速发展亟需高性能的网络处理能力，kernel bypass 方案也越来越被人所接受，市场上也出现了多种类似技术，如 DPDK、NETMAP、PF_RING 等。

其核心思想就是内核只用来处理控制流，所有数据流相关操作都在用户态进行处理，从而规避内核的包拷贝、线程调度、系统调用、中断等性能瓶颈，并辅以各种性能调优手段，从而达到更高的性能。

其中 DPDK 因为更彻底的脱离内核调度以及活跃的社区支持从而得到了更广泛的使用。



![图片](https://mmbiz.qpic.cn/mmbiz_png/ibu0S1GTZ5TzPfgjkQic4YYbQviaR3XK2nA1sK06kOh1KjRX6zYMmibicicdcOW5zztTErRF59ZxbJgFHvuMRYElkq6w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



## 3.F-Stack 开发框架

F-Stack 是一款兼顾高性能、易用性和通用性的网络开发框架，传统上 DPDK 大多用于 SDN、NFV、DNS 等简单的应用场景下，对于复杂的 TCP 协议栈上的七层应用很少，市面上已出现了部分用户态协议栈，如 mTCP、Mirage、lwIP、NUSE 等，也有用户态的编程框架，如 SeaStar 等，但统一的特点是应用程序接入门槛较高，不易于使用。

F-Stack 使用纯 C 实现，充当胶水粘合了 DPDK、FreeBSD 用户态协议栈、Posix API、微线程框架和上层应用（Nginx、Redis），使绝大部分的网络应用可以通过直接修改配置或替换系统的网络接口即可接入 F-Stack，从而获得更高的网络性能。



## 4.F-Stack 架构

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibu0S1GTZ5TzPfgjkQic4YYbQviaR3XK2nAq4eAkbpfR1I5YvrR25pOiaBWuodZ0jTGpBs1jNtuIq6p4LXiaGoSicxmg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



F-Stack 总体架构如上图所示，具有以下特点:

· 使用多进程无共享架构。

· 各进程绑定独立的网卡队列和 CPU，请求通过设置网卡 RSS 散落到各进程进行处理。

· 各进程拥有独立的协议栈、PCB 表等资源。

· 每个 NUMA 节点使用独立的内存池。

· 进程间通信通过无锁环形队列（rte_ring）进行。

· 使用 DPDK 作为网络 I/O 模块，将数据包从网卡直接接收到用户态。

移植 FreeBSD Release 11.0.1 协议栈到用户态并与 DPDK 对接。



## 5.F-Stack环境构建

以下演示在CentOS7上的构建过程

### 5.1克隆F-Stack

git clone https://github.com/F-Stack/f-stack.git /data/f-stack

### 5.2安装libnuma-dev

yum install numactl-devel

### 5.3编译dpdk

cd dpdk/usertools

./dpdk-setup.sh # dpdk提供的快速构建脚本

说明：选择当前系统环境对应的目标项(例如x86_64-native-linuxapp-gcc)后即开始dpdk的构建，如果当前系统没有安装对应的内核开发包，需要进行安装

### 5.4设置大页

对于单节点系统

echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

对于支持NUMA架构系统

echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages

echo 1024 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages

### 5.5挂载大页系统

mkdir /mnt/huge

mount -t hugetlbfs nodev /mnt/huge

说明：为提高性能，dpdk使用大页内存，如果不进行以上设置，dpdk的环境抽象层(EAL)会初始化失败

### 5.6关闭ASLR，多进程环境下使用有必要关闭

echo 0 > /proc/sys/kernel/randomize_va_space

### 5.7网卡硬件卸载

加载uio模块

modprobe uio

加载dpdk提供的内核驱动模块

insmod /data/f-stack/dpdk/x86_64-native-linuxapp-gcc/kmod/igb_uio.ko

insmod /data/f-stack/dpdk/x86_64-native-linuxapp-gcc/kmod/rte_kni.ko carrier=on

查看当前网卡设备状态

python dpdk-devbind.py --status

卸载网卡，如当前网卡名为eth0

ifconfig eth0 down

绑定dpdk提供的驱动模块

python dpdk-devbind.py --bind=igb_uio eth0

### 5.8安装DPDK

cd ../x86_64-native-linuxapp-gcc

make install

### 5.9编译安装F-Stack

export FF_PATH=/data/f-stack

export FF_DPDK=/data/f-stack/dpdk/x86_64-native-linuxapp-gcc

cd ../../lib/

make

make install

其他



Nginx构建及运行

说明：在运行之前需要如F-Stack构建说明中的设置大页内存，加载驱动等流程

cd app/nginx-1.16.1

bash ./configure --prefix=/usr/local/nginx_fstack --with-ff_module

make

make install

cd ../..

/usr/local/nginx_fstack/sbin/nginx



原文链接：https://mp.weixin.qq.com/s/D0tSKe8neAOi6SKEjIA0zA

原文作者：小小牧码人
# DPDK网络：利用libpcap捕获

## 1.任务描述

本文针对libpcap在捕获DPDK包的原理及用法方面展开分析，描述了libpcap的基本使用方式，以及结合DPDK的使用方法。
libpcap是网络抓包的常用工具，它是跨平台的用户层包捕获接口，libpcap提供了低层网络监控的一套灵活框架。
DPDK（Data Plane Development Kit）是一套高速网络包处理的库和驱动的集合框架，用户可以在用户空间来处理网络包，加速网络的的转发和处理。

## 2.libpcap捕获DPDK原理

正如OSI模型，网络包的接收和发送要经过各网络层的处理，传统网络包的收发要经过内核网络协议栈，再由用户层接口发送给应用程序。
下图中左侧是传统的抓包流程，libpcap通过BPF来抓取流经内核网络协议栈的包，下图中右侧是抓取dpdk包的流程，libpcap会捕获经过DPDK中MBUF结构中的网络包，然后经由用户层调用来发送给用户。

![dpdk-libpcap](https://img-blog.csdnimg.cn/ac353b77fb6d467aa2bdc06ecc67864f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAY3h6ODI1,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

## 3.编译

### 3.1由源代码编译DPDK

配置dpdk

```c++
meson <options> build
```

build并在系统中安装DPDK

```c++
cd build
ninja
ninja install
ldconfig

```

把网卡bind到DPDK上

```c++
usertools/dpdk-devbind.py -h

```

因为DPDK要用到大页内存，所以还需要设置系统hugepage

```c++
usertools/dpdk-hugepages.py -h

```

### 3.2编译libpcap

下载libpcap源代码并编译

```c++
./configure
make -s all && make -s testprogs && make install

```

## 4.测试方法

这里用TRex来发包来验证libpcap可以捕获到利用dpdk接收的网络包，这里可以用TRex的STL模式来发送包。在配置阶段要在发包的机器上设置目的mac地址，要与DUT（测试机）的mac地址相一致。
可以参考TRex的示例脚本来设置和发送网络包：

```c++
base_pkt =  Ether(dst="00:0c:29:89:ea:c1")/IP(src="16.0.0.1",dst="48.0.0.1")/UDP(dport=12,sport=1025)

```

更多的细节可以参考[TRex doc](https://trex-tgn.cisco.com/trex/doc/index.html)。

## 5.libpcap常用函数

(函数解析待更新)

```c++
// dpdk libpcap
static int pcap_dpdk_dispatch(pcap_t *p, int max_cnt, pcap_handler cb, u_char *cb_arg)
int pcap_dpdk_findalldevs(pcap_if_list_t *devlistp, char *ebuf)
pcap_t * pcap_dpdk_create(const char *device, char *ebuf, int *is_ours)

// normal libpcap
pcap_t * pcap_create(const char *device, char *errbuf)
int pcap_findalldevs(pcap_if_t **alldevsp, char *errbuf)
void pcap_freealldevs(pcap_if_t *alldevs)
int pcap_lookupnet(const char *device, bpf_u_int32 *netp, bpf_u_int32 *maskp, char *errbuf)
int pcap_activate(pcap_t *p)
int pcap_dispatch(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
int pcap_loop(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
int pcap_compile()
pcap_t * pcap_open_offline(const char *fname, char *errbuf)
int pcap_setfilter(pcap_t *p, struct bpf_program *fp)
int pcap_datalink(pcap_t *p)

```

## 6.总结

本文对libpcap进行了简要介绍，本对比了libpcap与dpdk-libpcap的区别，并实验验证了dpdk-libpcap的使用，后续会对libpcap的常用[API函数](https://so.csdn.net/so/search?q=API函数&spm=1001.2101.3001.7020)进行解析。







原文链接：https://blog.csdn.net/weixin_37859866/article/details/120539752

原文作者:cxz825
# DPDK之网卡imissed问题分析

## **1.发现**

在开发DPDK应用的时候，我们可以通过rte_eth_stats_get函数获取网卡统计信息中的imissed计数来判断网卡是否出现丢包。

![img](https://pic3.zhimg.com/80/v2-6262917c0316f3e5ba9bbf3d2688b066_720w.webp)

![img](https://pic4.zhimg.com/80/v2-c66faec562d8dcb84748ad0912c40bb3_720w.webp)

## **2.分析**

![img](https://pic1.zhimg.com/80/v2-d7a6b1fe716b23541a58901d7f3fd688_720w.webp)

一个网络帧从网卡接收到被应用处理，中间主要需要经历两个阶段，我们分别从这两个阶段进行分析。

- 阶段一：网卡通过其DMA硬件将收到的报文写入到收包队列中，如果入队道路拥塞将会导致报文无法入队（**入队**）
- 阶段二：应用从收包队列中读取报文，如果出队慢将导致队列溢出（**出队**）

## **3.入队**

![img](https://pic4.zhimg.com/80/v2-08264ff5dd736b1adee79ee9cfb13cf3_720w.webp)

入队问题主要集中在**PCIe异常“降速”**方面。因为报文从网卡到系统是经过PCIe总线来传输的，PCIe总线的吞吐将直接影响入队的速率。

**现象**

![img](https://pic4.zhimg.com/80/v2-a277805c1315f5017a6c91aef681bbff_720w.webp)

若按网口能力满带宽接入流量，网口出现imissed丢包情况，可通过lspci -vv命令查看网口能力与实际使用是否一致。

- 情况1：网口能力是传输速率**5GT/s**，总线宽带**x8**（LnkCap），实际使用的是传输速率**5GT/s**，总线宽带**x4**（LnkSta）。吞吐能力从4GB/s下降到2GB/s。

![img](https://pic4.zhimg.com/80/v2-4346d5f06a69cc405c36a7c13c31743f_720w.webp)

- 情况2：网口能力是传输速率**8GT/s**，总线宽带**x8**（LnkCap），实际使用的是传输速率**5GT/s**，总线宽带**x8**（LnkSta）。吞吐能力从7.877GB/s下降到4GB/s。

![img](https://pic1.zhimg.com/80/v2-82e077c2d81cfed8aa4d285c1550c320_720w.webp)

**解决**

一般是服务器与网卡兼容性问题，可以更换网卡或者更换服务器。如果有条件，可以找服务器厂商从bios等方面进行详细定位解决兼容性问题。



## **4.出队**

![img](https://pic4.zhimg.com/80/v2-2e35814062d0f99258df854b243d4ac3_720w.webp)

出队问题主要集中在性能不高、设计不优和CPU错误降频等方面。



### **4.1.性能不高**

**现象**

![img](https://pic1.zhimg.com/80/v2-7ce8106a0c4b4bd11b54650361a62a24_720w.webp)

出队工作核心使用率100%，不能及时处理队列报文，导致队列报文溢出，**持续imissed++**，此时出队平均速率是小于入队平均速率。

**解决**

linux系统下可以使用perf性能分析工具，做热点函数分析，perf安装命令yum install perf。perf常用的热点函数定位命令如下：

- 进程级：perf top -p <pid>
- 线程级：perf top -t <tid>

![img](https://pic2.zhimg.com/80/v2-de7b0e9ba352b6000dcc993e139a5775_720w.webp)

线程tid可以通过pidstat -t -p <pid>获取。

![img](https://pic1.zhimg.com/80/v2-5bca7948fc6b2d8d968a285f90266430_720w.webp)

对于热点函数，可以选中函数进入内部分析热点代码。

![img](https://pic2.zhimg.com/80/v2-f455d8cd2f232faa9dfe33421f5a8c4d_720w.webp)



### **4.2.设计不优**

**现象**

**![动图封面](https://pic4.zhimg.com/v2-cc11bd52cfe644216683e109c3fb6057_b.jpg)**



出队工作核心使用率没有达到100%，但依然偶尔会丢包，**断续imissed++**。一般这是因为**出队速率抖动引发溢出**，即出队最低速率小于入队平均速率，而收包队列的规格（几K）不足以缓存拥塞的报文。



**解决**

偶尔丢包有时不容易从热点函数中发现，需要分析代码流程进行调优。比如某个报文会触发密集cpu计算，需要更长处理时间，如收到fin报文流结束，对流内容进行复杂处理。针对这种情况，我们可以从以下两个方面进行调优：

- 异步处理

将复杂处理由同步处理(Run-to-completion)改为异步处理(Pipeline)，降低抖动（一般异步还会有更好的cache局部性）。

![动图封面](https://pic2.zhimg.com/v2-766212a510c2dc16926ea2130a0ca09d_b.jpg)



- 加软队列

增加一级容量更大的软队列，缓存抖动（一般业务有按规则分发的需要也需要构建一级软队列，软队列会或多或少增加报文处理的延时）。

![img](https://pic3.zhimg.com/80/v2-24dd7eb6579720e3767f499aa5ddf6d2_720w.webp)

### **4.3.CPU错误降频**

如果CPU被降频，将直接影响出队性能。

![img](https://pic1.zhimg.com/80/v2-25cd1e80028aae6994a279fb29487564_720w.webp)

**现象**

查看系统日志，发现出现CPU被错误降频。

![img](https://pic2.zhimg.com/80/v2-f2635ec0f1c74813400b480e67d862ed_720w.webp)

**分析**

![img](https://pic4.zhimg.com/80/v2-cc8c002fcb2442157d856d5407b92dc3_720w.webp)

**解决**

![img](https://pic3.zhimg.com/80/v2-4d0ad3dd74a0ae2effcec10470b7ba3e_720w.webp)

相关阅读

[DPDK之网卡收包流程](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s%3F__biz%3DMzU3MTM5MzQ2OA%3D%3D%26mid%3D2247484033%26idx%3D1%26sn%3D78cf2f06e4303d51424e3e625fe29fb4%26scene%3D21%23wechat_redirect)



![img](https://pic1.zhimg.com/80/v2-74c30ccd93a40af96c1618eaf4f3257c_720w.webp)





转载自： 作者[l7dpi](https://www.zhihu.com/people/qie-li-37-47)      本文链接https://zhuanlan.zhihu.com/p/73393629
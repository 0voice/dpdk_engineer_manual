# DPDK KNI 示例程序

Kernel NIC Interface (KNI)是DPDK提供的控制平面解决方案，运行DPDK用户层应用与Linux内核网络协议栈交换数据。DPDK用户层应用通过使用IOCTL系统调用在Linux内核中创建KNI虚拟设备实现此功能。此IOCTL调用提供接口和DPDK的物理地址空间信息，并且地址空间被KNI内核模块重映射到内核地址空间中，KNI内核模块还将相关信息保存在虚拟设备上下文中。DPDK为每个设备创建FIFO队列用于数据报文到内核模块的收发。
KNI内核模块时标准的网络驱动，当接收到IOCTL调用，访问DPDK的FIFO队列以从DPDK用户空间应用接收和发送数据报文。FIFO队列中包含有指向DPDK中数据报文的指针。此举：

- 提供更快速的到内核网络栈的接口，消除系统调用
- 方便DPDK使用标准的Linux用户空间网络工具（tcpdump, ftp, 等）
- 消除针对报文的copy_to_user和copy_from_user操作。

KNI示例程序简单演示了在DPDK和Linux内核间创建数据报文通路。通过为每个DPDK接口创建一个或者多个内核网络设备实现。示例程序允许标准Linux工具（ethtool, ifconfig, tcpdump）允许与DPDK接口，并且实现DPDK应用与Linux内核间报文交互。

KNI示例程序要求内核加载rte_kni模块。

## 1.概述

KNI示例程序kni为每个物理网卡分配一个或多个KNI虚拟接口。对于每个物理网口，kni启用两个DPDK用户线程；一个负责从网卡读取数据、写入到相应KNI虚拟接口；另一个负责从KNI虚拟接口读取数据、无修改的写入物理网口。

推荐为每个物理网口配置一个KNI虚拟接口。基于性能测试，示例程序可为每个物理网口配置多个KNI虚拟接口，或者将来一同工作用于VMDq支持。
KNI示例的报文流程如下图：

![img](https://pic3.zhimg.com/80/v2-829ed2b15970bc5da646eb231edaa5be_720w.webp)

可通过命令行选项-m使能链路监控功能，需要另外启动一个pthread线程检查每个物理网口的链路状态，并且更新相应的KNI虚拟接口的链路状态。意味着当以太网链路down的时候，KNI虚拟接口将自动禁用，反之，自动启用。
如果链路监控使能的话，rte_kni内核模块加载时链路默认状态为禁止off。以保证KNI虚拟接口仅在相应的物理网卡链路起来up之后才会使能。
如果链路监控未使能，rte_kni内核模块将默认链路状态设置为开启on。这就KNI虚拟接口的链路状态设置为开启on，而不要考虑实际的对应物理网卡的链路状态。对于处于loopback模式而物理网口又没有任何连接的测试比较有用。

## 2.编译应用

编译示例程序可参考DPDK的：Compiling the Sample Applications
此示例程序位于examples/kni子目录下。
注意：示例程序仅在linux系统运行。
运行KNI示例

kni示例应用的命令行选项如下：
kni [EAL options] -- -p PORTMASK --config="(port,lcore_rx,lcore_tx[,lcore_kthread,...])[,(port,lcore_rx,lcore_tx[,lcore_kthread,...])]" [-P] [-m]
-p PORTMASK:
十六进制接口掩码
--config="(port,lcore_rx,lcore_tx[,lcore_kthread,...])[,(port,lcore_rx,lcore_tx[,lcore_kthread,...])]":
指定对于每个物理网口，接收和发送DPDK线程绑定的核心，以及KNI内核线程绑定的核心。

-P:
可选标志，设置的话意味着混杂模式，以便不区分以太网目的MAC地址，接收所有报文。不设置此选项，仅目的MAC地址等于接口MAC地址的报文被接收。
-m:
可选标志。使能监控模式并更新以太网链路状态。此选项需要启动一个DPDK线程定期检查物理接口链路状态，同步相应的KNI虚拟网口状态。意味着当以太网链路down的时候，KNI虚拟接口将自动禁用，反之，自动启用。
参考DPDK Getting Started Guide中运行应用所需的通用信息以及EAL选项。
EAL选项核心掩码-c或者-l核心列表参数必须包含有以上的lcore_rx和lcore_tx参数中指定的核心，但是，不需要包含lcore_thread参数指定的核心，因为其实rte_kni模块中用来绑定内核线程的核心（不能超出CPU核心数量）。

参数--config必须为-p选项中指定的每个物理接口设置值：(port,lcore_rx,lcore_tx,[lcore_kthread,...])。
对于每个物理接口，可选参数lcore_kthread可指定为一个或者多个0值。如果不为lcore_kthread指定核心ID，仅为物理接口创建一个KNI虚拟接口，并且KNI内核线程没有核心的亲核性。

如果为lcore_thread参数指定一个或者多个核心ID，将为每个核心ID创建一个KNI虚拟接口，绑定到同一物理接口。如果rte_kni内核模块以多线程模式加载，将为每个KNI虚拟接口创建一个内核线程，绑定到指定的核心。反之，如果rte_kni以当线程模式加载，仅创建一个内核线程服务于所有KNI虚拟接口。此内核线程绑定在lcore_thread指定的首个核心ID上。

## 3.配置示例

以下命令首先以多线程模式加载rte_kni内核模块。其次，kni应用指定两个接口（-p 0x3）启动；根据--config参数可知，接口0（0,4,6,8）使用核心4运行接收任务，核心6运行发送任务，并且创建一个KNI虚拟接口vEth0_0，启动一个内核处理线程绑定在核心8上。类似的接口1（0,5,7,9）使用核心5运行接收任务，核心7运行发送任务，并且创建一个KNI虚拟接口vEth1_0，启动一个内核处理线程绑定在核心9上。
\# rmmod rte_kni # insmod kmod/rte_kni.ko kthread_mode=multiple # ./build/kni -l 4-7 -n 4 -- -P -p 0x3 -m --config="(0,4,6,8),(1,5,7,9)"
以下的示例相同，只是为每个物理接口指定了额外的lcore_thread核心。据此，kni将总共创建4个KNI虚拟接口：vEth0_0/vEth0_1绑定到物理接口0；vEth1_0/vEth1_1绑定到物理接口1。
每个KNI虚拟接口的内核线程绑定关系如下：
vEth0_0 - bound to lcore 8. vEth0_1 - bound to lcore 10. vEth1_0 - bound to lcore 9. vEth1_1 - bound to lcore 11

## 4.配置命令如下：

\# rmmod rte_kni # insmod kmod/rte_kni.ko kthread_mode=multiple # ./build/kni -l 4-7 -n 4 -- -P -p 0x3 -m --config="(0,4,6,8,10),(1,5,7,9,11)"
以下示例可供测试DPDK kni测试程序与rte_kni内核模块间的接口。此例中，内核模块rte_kni以单线程模式加载，并且使能回环模式，默认的链路状态设置为开启on，以便相应的物理接口无连接也可使用KNI虚拟接口。为物理接口0创建KNI虚拟接口vEth0_0，为物理接口1创建vEth1_0。既然rte_kni工作在单线程模式，唯一的内核线程工作在核心8上（首个lcore_thread核心）。
既然物理接口没有被使用，可不指定-m选项不使能链路检测。
\# rmmod rte_kni # insmod kmod/rte_kni.ko lo_mode=lo_mode_fifo carrier=on # ./build/kni -l 4-7 -n 4 -- -P -p 0x3 --config="(0,4,6,8),(1,5,7,9)"
KNI接口操作

一旦kni应用启动，用户可使用通用的Linux命令去管理KNI虚拟接口，就像其是另外的Linux网络接口一样。

使能KNI虚拟接口并设置IP地址：
***# ifconfig vEth0_0 192.168.0.1\***

显示KNI虚拟接口配置和统计信息：
***# ifconfig vEth0_0\***

用户可查看和复位kni应用中的报文统计信息，通过USR1和USR2两个信号实现：
// 显示统计信息
***# kill -SIGUSR1 `pidof kni`\***
// 清零统计信息
***# kill -SIGUSR2 `pidof kni`\***

显示网络流量：
***# tcpdump -i vEth0_0\***

通用Linux命令还可更改KNI虚拟接口所对应的物理网口的MAC地址以及MTU值。尽管如此，如果多个KNI虚拟接口绑定在同一个物理网口上，这些命令仅在第一个KNI虚拟接口上工作。
修改MAC地址

***# ifconfig vEth0_0 hw ether 0C:01:02:03:04:08\***
修改MTU值：
***# ifconfig vEth0_0 mtu 1450\***

如果DPDK编译时开启了选项CONFIG_RTE_KNI_KMOD_ETHTOOL=y，并且系统使用的是Intel网卡，用户可在KNI虚拟接口上使用ethtool工具，就像操作普通Linux网口一样。
显示网卡寄存器：
***# ethtool -d vEth0_0\***
当kni示例程序关闭后，所有的KNI虚拟接口将从内核中删除。

代码阐述
以下章节提供一些代码讲解。

### **4.1初始化**

mbuf pool、driver和queues的建立与L2 Forwarding示例程序相似（真实和虚拟环境下）。另外，根据命令行参数为每个配置的物理网口创建一个或者多个KNI虚拟网口。
函数kni_alloc代码负责为指定的物理网口分配KNI虚拟接口。
其它特定于本示例的初始化流程包括每个物理网口的接收、发送任务以及内核线程的关联。

一个核心由物理网口读取报文，写入关联的一个或多个KNI虚拟网口；
另一个核心从一个或者多个KNI虚拟网口读取报文，写入物理网口；
绑定内核线程的核心依次运行处理

以上由kni_port_params_array[]数组实现，其以物理网口ID为索引，参见函数parse_config。

### **4.2报文转发**

初始化完成之后，每个核心上运行main_loop代码。首先将本地核心的lcore_id与用户提供的lcore_rx和lcore_tx比较，来决定是读取还是写入数据到内核KNI虚拟接口。

当情况是从物理网口读取，写入内核KNI虚拟接口时（kni_ingress函数），报文接收与L2 Forwarding示例相同（参见其Receive, Process and Transmit Packets）。报文的发送由函数rte_kni_tx_burst()实现，将mbufs发送到内核KNI虚拟接口。当内核成功拷贝mbufs之后，KNI库代码自动释放mbufs。

另外的情况，从内核KNI虚拟网口读取，写入物理网口（kni_egress函数），函数rte_kni_rx_burst()由内核KNI接口读取mbufs。报文发送与L2 Forwording示例程序相同（参见Receive, Process and Transmit Packets）。

END

原文链接：[https://blog.csdn.net/sinat_20184565 ](https://link.zhihu.com/?target=https%3A//blog.csdn.net/sinat_20184565/article/details/92700223%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522166685730516781432947358%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D%26request_id%3D166685730516781432947358%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-12-92700223-null-null.142%5Ev62%5Epc_search_tree%2C201%5Ev3%5Eadd_ask%2C213%5Ev1%5Et3_esquery_v2%26utm_term%3Dkni%26spm%3D1018.2226.3001.4187)原文作者：redwingz
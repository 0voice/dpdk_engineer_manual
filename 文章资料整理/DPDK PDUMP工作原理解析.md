# DPDK PDUMP工作原理解析

## **1 .DPDK capture 原理**

### **1.1 pdump库的使用**

在DPDK的16.07版本中，添加了Packet capture特性，通过pdump库可以非常方便的进行报文的捕获。

- 在程序初始化过程中调用rte_pdump_init，启动dump_thread进行消息的监听；
- *在程序退出前调用rte_pdump_uninit进行资源的释放；*
- 启动pdump程序，发送抓包命令，进行抓包。

### **1.2 KNI方式**

kNI，全称 Kernel NIC Interface，，下面是DPDK官方手册对它的介绍：

```
The Kernel NIC Interface (KNI) is a DPDK control plane solution that allows userspace applications to exchange packets with the kernel networking stack.
```

这里对data plane和control plane进行简单说明：

data plane专注于报文的转发；control plane专注于协议处理，如ospf计算。如果采用DPDK架构，那么就会遇到如下问题：

- 不论是协议报文还是数据报文通过port接收后，都到了data plane，data plane如何将协议报文交给control plane呢？
- control plane（往往不是DPDK程序）与网络设备进行协议交互时，报文又如何投递给port发送出去呢？

KNI就是control plane与data plane间的桥梁：加载rte_kni.ko驱动后，Linux内核响应DPDK程序中发送的IOCTL消息创建虚拟接口并转换FIFO地址用于后续的报文交互。

在DPDK提供pdump特性前，对报文抓取主要就是采用KNI方式：

- DPDK程序创建虚拟接口；
- 将收到的报文发送给虚拟接口；
- 启用类似tcpdump的工具抓取虚拟接口上的报文。

### **1.3 优劣比较**

- pdump库使用方便，性能高；不过它在16.07才支持，低版本不支持，且16.07版本中还不支持filter；
- KNI方式，需要创建thread，性能应该略低于pdump（KNI内部fifo处理有一定开销），开发工作较多，但可以自行实现过滤功能，相比pdump的全copy在某些场景会更适用，且使用tcpdump抓取在运维上更方便；
- pdump方式，拷贝由port的驱动完成，dump到文件则由pdump程序完成；KNI方式，拷贝在用户态空间完成，发送到内核态后，dump到文件则有tcpdump完成；在DPDK框架中进行报文捕获和dump，仅推荐用于旁路抓包。

此篇文章主要讲解一下pdump。

## **2 .librte_pdump库**

Server 端：

- rte_pdump_init()：初始化 PDUMP 抓包框架，并创建线程和 Server Socket。Socket 在线程里监听 Client Start/Stop 抓包的请求。
- rte_pdump_uninit()：清理退出 PDUMP 抓包框架，并关闭线程和 Server Socket。

Client 端：

- rte_pdump_enable()：在一个端口队列开启抓包。
- rte_pdump_enable_by_deviceid()：在一个设备 ID（vdev 名称或者 PCI 地址）和队列抓包。

每调用一次这两个 API，PDUMP 库就会创建一个独立的 Client Socket，并发送 pdump enable 的请求到 Server。Server 监听到这个请求就会通过给定的端口或者设备 ID 以及队列的组合，在 Ethernet RX/TX 注册回调函数，之后 Server 就会镜像数据包到一个新的内存池，并将让它们在 Client 传入的 rte_ring 队列上入队。Server 会发送响应给 Client 关于请求处理的状态。在收到 Server 的回应后，Client 的 Socket 就关闭了。

- rte_pdump_disable()：在一个端口队列停止抓包。
- rte_pdump_disable_by_deviceid()：停止在一个设备 ID（vdev 名称或者 PCI 地址）和队列抓包。

每调用一次这两个API，PDUMP 库就会创建一个独立的 Client Socket，并发送 pdump disable 的请求到 Server。Server 监听到这个请求就会通过给定的端口或者设备 ID 以及队列的组合,在 Ethernet RX/TX 删除回调函数，之后 Server 就会镜像数据包到一个新的内存池，并将让它们在 Client 传入的 rte_ring 队列上入队。Server 会发送请求回应给 Client 关于请求处理的状态。在收到 Server 的回应后，Client 的 Socket 就关闭了。

- rte_pdump_set_socket_dir()：设置 Server 或 Client 的 Socket 文件的路径。注意，这个接口是非线程安全的，并在 DPDK 18.05 版本移除。

### 2.1 运行原理

在使用dpdk-pdump时，dpdk-pdump会作为Secondary（从）进程，并依附于 Primary（主）进程，即 DPDK App，例如：testpmd、l2fwd、l3fwd。Primary 进程作为 Server 端，需要 include rte_pdump 库进行开发，初始化 PDUMP 抓包框架。Secondary 进程作为 Client 端，同样需要 include rte_pdump 库进行开发，通过标准接口向 Primary 进程发送 Start/Stop 抓包的请求，然后 Primary 进程会拷贝一份数据包到 Ring 中，Secondary 进程再从 Ring 中读取出来，并发送到 PCAP PMD 设备。可以保存为 pcap 文件，或发送至 Linux Console 等外部接口输出。

DPDK官方提供了Secondary（从）进程的app，源码位于：dpdk-stable-17.11.10/app/pdump

**注意**：因为 dpdk-pdump 抓包存在报文Copy的过程，所以对性能会有影响，建议仅在 DEBUG 时使用。

### **2.2 工作流程**

![图片](https://mmbiz.qpic.cn/mmbiz_png/mfV5Fqo96bBbC65lcpw455qRaSgrCnRT09J5ibpQ3LgSVBMjQFoQmq7x82rFWBkrBFjuKa0T82NYRfZysibk15Ww/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

1. A采用rx-worker-tx的模型进行报文的处理，其中调用rte_pdump_init会启动dump_thread，即图中红色的message线程；
2. pdump采用secondary模式启动，与A共享mmap映射的内存空间；
3. pdump启动过程中会创建mbuf_pool和ring，用于后续接收A中报文的拷贝；
4. pdump会通过rte_eth_dev_attach方式创建vdev，且采用eth_pcap驱动进行初始化，留意init中的open_tx_pcap；
5. pdump向A发送开启抓包的消息（UDP方式），消息内容为前面创建的mbuf_pool、ring以及抓包的port和对应的queue；
6. A中的dump_thread收到消息后，获取相应信息，在port上注册call_back函数；
7. 对于开启抓包的port，在rx_burst/tx_burst时会先调用call_back，这里对应pdump_rx/pdump_tx，它会由mbuf_pool中分配mbuf进行报文的复制，同时enqueue到ring中；（mbuf_pool和ring在步骤3中创建，在步骤5中传递给A）
8. pdump进行ring的dequeue操作获取拷贝报文；
9. 拷贝报文通过rte_eth_tx_burst发送给vdev；
10. vdev通过eth_pcap的tx_pkt_burst发送报文，即调用eth_pcap_tx_dumper完成报文的pcap存储（pcap_dump）。

## 3 .源码分析

### **3.1 使用方法**

以 testpmd 为例，使用 dpdk-pdump 进行抓包。

启动 testpmd 作为 Primary 进程

```
$ ./testpmd -l 1,2,3 --socket-mem 1024 -n 4 --log-level=8 -- -i
```

启动 dpdk-pdump 作为 Secondary 进程

```
$ ./build/app/dpdk-pdump -- --pdump 'port=0,queue=*,rx-dev=/tmp/capture.pcap'
```

查看 pcap 文件

```
$ tcpdump -nr /tmp/capture.pcap
```

dpdk-pdump 指令

```
./build/app/dpdk-pdump -- --pdump '(port=<port id> | device_id=<pci id or vdev name>), (queue=<queue_id>), (rx-dev=<iface or pcap file> | tx-dev=<iface or pcap file>), [ring-size=<ring size>], [mbuf-size=<mbuf data size>], [total-num-mbufs=<number of mbufs>]' [--server-socket-path=<server socket dir>] [--client-socket-path=<client socket dir>]
```

- --pdump：是必选的，多个 --pdump 可以用于捕获不同的接口和队列
- --server-socket-path：是可选的，指定 Server socket 的目录。root 用户就默认为 /var/run/.dpdk/，非 root 用户默认为 ~/.dpdk/。
- --client-socket-path：是可选的，指定 Client Socket 的目录。root 用户就默认为 /var/run/.dpdk/，非 root 用户默认为 ~/.dpdk/。

--pdump 的子参数：

- port：需要被抓包的以太网 Port id。
- device_id：需要被抓包的以太网 PCI 设备的 id。
- queue：需要被抓包的以太网 PCI 设备的 Queue id，`*` 表示所有队列。
- rx-dev：入口方向被抓取的报文，参数应该是一个 pcap 文件名或者 Linux 接口。
- tx-dev：出口方向被抓取的报文，参数应该是一个 pcap 文件名或者 Linux 接口。

NOTE：如果两个方向都分别要，tx-dev 与 rx-dev 应该被同时指定两个不同的文件或者接口；如果两个方向都同时要，tx-dev 与 rx-dev 应该指定相同的文件或接口。

- ring-size：指定用于存储数据包的 Ring 的大小，默认是16384。用于主程序向抓包程序入队用的。
- mbuf-sizze：MBuf Data 的 大小，用于 mempool 的创建,默认是2176。用于入队列的mbuf用的。用于主程序向抓包程序传数据用的。
- total-num-mbufs：指创建 mbug 的个数，默认值为 65535。
- --server-socket-path 和 --client-socket-path 用于指定 Server 和 Client 之间进行通信的 Socket 路径。默认的，DPDK App 的 Socket 路径为 /var/run/dpdk/rte/，但是在多 DPDK App 进行的环境中，可能会为每个 DPDK App 指定不同的路径，从而隔离开配置空间。此时就需要使用 --server-socket-path 来指定特定 DPDK App 的配置路径了。

### **3.2 server端**

在使用dpdk-pdump之前，需要设置配置参数，在CONFIG中的common_base中分别设置：

```
CONFIG_RTE_LIBRTE_PMD_PCAP=yCONFIG_RTE_LIBRTE_PDUMP=y
```

初始化：

```
#ifdef RTE_LIBRTE_PDUMP/* initialize packet capture framework */rte_pdump_init(NULL);    //pdump server端初始化#endif
```

去初始化：

```
#ifdef RTE_LIBRTE_PDUMP/* uninitialize packet capture framework */rte_pdump_uninit();    //pdump server端去初始化#endif
```

初始化流程分析：

```
rte_pdump_init    rte_pdump_set_socket_dir        /* 设置socket directory */    pdump_create_server_socket      /* 创建server socket */    pthread_create                  /* pdump_thread_main */    rte_thread_setname              /* 设置线程名 */
```

核心代码在pdump_thread_main中：

```
pdump_thread_main    recvfrom                            /* 接收配置消息 */    set_pdump_rxtx_cbs        rte_eth_dev_get_port_by_name    /* 获取port */        pdump_register_rx_callbacks     /* 注册抓包函数 */            rte_eth_add_tx_callback     /* 注册 pdump_tx 函数 */    sendto                              /* 给pdump进程回复应答消息 */
```

pdump_tx函数注册给 struct rte_eth_rxtx_callback，会在发包函数时调用，代码如下:

```
static inline uint16_trte_eth_tx_burst(uint16_t port_id, uint16_t queue_id,struct rte_mbuf **tx_pkts, uint16_t nb_pkts){struct rte_eth_dev *dev = &rte_eth_devices[port_id];                ......        #ifdef RTE_ETHDEV_RXTX_CALLBACKSstruct rte_eth_rxtx_callback *cb = dev->pre_tx_burst_cbs[queue_id];        if (unlikely(cb != NULL)) {do {                nb_pkts = cb->fn.tx(port_id, queue_id, tx_pkts, nb_pkts,                cb->param);             /* 调用注册的pdump_tx函数 */                cb = cb->next;        } while (cb != NULL);        }#endif       return (*dev->tx_pkt_burst)(dev->data->tx_queues[queue_id], tx_pkts, nb_pkts);}
```

pdump_tx中的实现逻辑如下：

```
pdump_tx    pdump_copy        pdump_pktmbuf_copy          /* copy 报文到 dup_bufs，这里使用的 mempool 是pdump进程中创建的 */        rte_ring_enqueue_burst      /* 报文入队列，同样ring也是pdump中创建的 */
```

rte_ring_enqueue_burst 是rte_ring库提供的接口，会在后续文章分析它的实现。

server端的实现我们就分析到这里，只是一个简略的流程分析，细节的还需要大家去仔细研究代码，接下来我们分析DPDK官方提供的抓包工具pdump实现。

### 3.3 pdump

pdump整体流程如下：

```
main    signal                  /* 注册捕获信号函数 signal_handler */    rte_eal_init            /* dpdk环境初始化 */    launch_args_parse       /* 命令行参数解析 */    create_mp_ring_vdev     /* 创建rte_mempool、rte_ring、vdev */    enable_pdump            /* 使能pdump */    dump_packets            /* 抓包 */    cleanup_pdump_resources /* 清除资源 */    print_pdump_stats       /* 打印统计信息 */
```

接下来分析下每个函数的实现，简单分析下这个小工具的整体架构：

收到信号SIGINT（按下ctrl+c，停止抓包）调用注册的信号处理函数，在此函数的实现中，我们把变量quit_signal赋值为1，后续根据这个变量的值退出读取报文的循环，清空资源，pdump进程退出，代码如下：

```
static voidsignal_handler(int sig_num){if (sig_num == SIGINT) {        ......        quit_signal = 1;    }}
```

rte_ela_init函数，dpdk环境的初始化函数，没有什么好说的（但其实能说一大堆，请听后续分析），这里指出一点，需要把其指定为secondary进程方式：

```
char mp_flag[] = "--proc-type=secondary";
```

launch_args_parse命令行参数的解析函数，此解析过程使用了c库的函数getopt_long及dpdk提供的函数rte_kvargs_parse用来解析key=value的形式，解析的配置数据都存储在如下数组中：

```
static struct pdump_tuples pdump_t[APP_ARG_TCPDUMP_MAX_TUPLES];
```

create_mp_ring_vdev创建mempool、ring、vdev的函数，根据命令行参数的配置，创建相关资源。

enable_pdump使能抓包，创建socket，通过socket发送请求消息到主进程，通知主进程注册抓包函数。

dump_packets循环读取队列，报文出队列，写入Ethernet device，代码流程如下：

```
dump_packets    pdump_rxtx        rte_ring_dequeue_burst      /* 从队列中读取报文 */        rte_eth_tx_burst            /* 报文发送到 Ethernet device */
```

cleanup_pdump_resources清空资源函数，处理流程如下：

```
cleanup_pdump_resources(void)    disable_pdump       /* socket 方式通知主进程抓包停止 */    free_ring_data    cleanup_rings        rte_ring_free   /* 释放ring */
```

print_pdump_stats打印统计信息函数。

## **4 .总结**

以上就是DPDK官方提供的rte_pdump库以及提供的抓包工具pdump的简单的原理分析，对于实现细节还有许多值得分析的点，我们后续继续讲解，通过分析抓包方式的原理，可以对于我们使用dpdk框架做数据包转发的进程提供一种做抓包的方法：

- 开辟一段共享内存mempool及ring；
- 主进程将报文clone后写入ring；
- 从进程读取队列，完成报文的dump。

此种方法只涉及一次报文的copy，没有下内核的开销，可以做一定的参考。



原文链接：https://mp.weixin.qq.com/s/4UF9o9dSPhZqVjHzpYRfuQ

原文作者：小k


# dpdk pdump 原理浅析

## 1.dpdk-pdump 程序支持的功能

支持动态使能与禁能抓包功能
动态分配与释放抓包需要使用的数据结构
支持抓取 dpdk primary 进程特定 port 的某个收、发包队列上的报文
支持多核同时抓包
支持 dump 统计数据
预留了 ebpf 规则字段

## 2.dpdk pdump 库内部数据结构

![在这里插入图片描述](https://img-blog.csdnimg.cn/0ac98fa170194b4bbdd237c3b897ed1b.jpeg#pic_center)

pdump程序根据用户配置参数构建pdump_request结构，然后通过 dpdk 多进程间通信机制将请求发送到 primary进程，primary 进程接收到这个请求后解析请求并调用pdump库提供的使能接口注册抓包回调，然后构造一条pdump_response返回消息发给 pdump程序。

pdump_rxtx_cbs只在 libpdump 库代码内部使用，用于保存 pdump 程序注册的所有包过滤请求规则，在关闭 pdump 的时候会使用此结构来释放相关的数据。

## 3.功能实现原理

### 3.1动态使能与禁能抓包功能的实现

原理图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/ba9f05d2686a43a4902072b5f6612c11.png)

此功能使用 dpdk 多进程间通信框架 mp_channel 实现，其特征如下：

某个 secondary 进程可以向 primary 进程发送消息
primary 进程可以向所有的 secondary 进程发送广播消息
消息基于本地套接字，每个 dpdk 进程都会创建并监控一个自己的本地套接字
mp_channel 是一个相对通用的消息处理框架，支持注册不同消息的 action 回调
dpdk primary 进程初始化时调用 rte_mp_action_register注册一个以 mp_pdump 字符串标识的 pdump 消息处理 action 回调——　pdump_server函数，当 mp 处理线程接收到 pdump 请求消息时调用 pdump_server 来完成 pdump 功能的使能与关闭。

pdump_server 的主要逻辑如下：

检查消息格式
执行 set_pdump_rxtx_cbs函数使能、关闭抓包
根据执行结果构造一条回复消息发往 pdump 程序
使能抓包的语义：

抓取收到的包时，为某个接口的某个收包队列注册一个 pdump_rx 收包回调函数，当 primary 进程调用到 rte_eth_rx_burst 接口从注册了 pdump_rx 回调的队列收到包后就会调用 pdump_rx 回调来拷贝报文并入 ring
抓取发出的包时，为某个接口的某个发包队列注册一个 pdump_tx 发包回调函数，当 primary 进程调用到 rte_eth_tx_burst 接口向注册了 pdump_tx 回调的队列发包前，pdump_tx 回调被调用以完成报文拷贝并入 ring
每一个使能的队列都会分配一个 pdump_rxtx_cbs 结构，在关闭抓包的时候会使用到
关闭抓包的语义：

根据配置找到对应的 pdump_rxtx_cbs 结构，根据此结构中保存的回调函数句柄移除某个接口的某个收、发队列的 pdump_rx、pdump_tx 收发包回调函数。

### 3.2pdump 库抓取报文依赖的数据结构

mempool
ring
pdump 库通过注册 dpdk 收发包回调函数实现抓包，其语义为在配置的 mempool 中申请 mbuf，然后完成报文 copy，最后再将 mbuf 地址投递到配置的 ring 中。

dpdk primary 进程启动时并不会创建用于 pdump 抓包的 mempool、ring，这两个结构是在 pdump 进程中创建与销毁的，dpdk primary 进程会根据收到的 pdump_request 中的 mp 与 ring 字段配置抓包回调的参数，抓包回调中直接使用这两个数据结构地址。

这里有个问题：为什么 pdump 进程创建的 mempool 与 ring 的虚拟地址可以在 dpdk primary 进程中直接使用？进程的虚拟地址不是相互独立的吗？
![在这里插入图片描述](https://img-blog.csdnimg.cn/f724d16090db41f881bcde5978345bf7.png)

上图为 dpdk primary 进程与 secondary 进程共享资源描述图，中间的 hugepage dpdk memory 是两个进程共享的大页内存，其映射的基地址在 priamry 进程中与 secondary 进程中是相同的，这样在大页内存上分配的内存其虚拟地址可以在 primary 进程与 secondary 进程中直接使用，底层的共享过程隐藏在 rte_eal_init 函数调用中。

### 3.3pdump 抓取 primary 进程从某个 port 的某个 queue 上收到的报文的关键流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/fd5bb54865974599bc08ecddc936c2c8.png)

上图右上角为 pdump 进程，它从 ring 中拿到 mbuf 后并不会直接调用 libpcap 接口保存报文，而是使用 dpdk pcap 虚拟网口，通过调用 rte_eth_tx_burst 向 pcap 虚拟网口发包来完成报文的保存过程。

这里存在一个问题：当 primary 进程运行后，接口的数目已经确定，pdump 要动态添加、删除 vdev port 该如何实现？

这一看似复杂的功能是通过调用如下两个接口实现的：

rte_eal_hotplug_add
rte_eal_hotplug_remove
当 pdump 进程调用rte_eal_hotplug_add接口时，此接口向 primary 进程发送一个接口热插拔消息，primary 进程调用 vdev bus 提供的 plug 方法完成新接口的创建并根据创建结果回复消息。当 vdev 成功创建后，pdump 进程通过调用 rte_eth_dev_get_port_by_name函数找到此 vdev 接口的 port_id 来使用。

同理，当 pdump 进程退出时，它会调用 rte_eal_hotplug_remove接口继续向 primary 进程发送接口热插拔消息，primary 进程调用 vdev bus 提供的 unplug 方法完成接口的销毁。

### 3.4如何处理 primary 进程退出的情况

当 pdump 进程运行时，如果 primary 进程终止，pdump 进程需要检测到这一状态并退出。检测逻辑创建一个线程周期性设置一个定时事件回调函数，定时调用 rte_eal_primary_proc_alive接口来检测 primary 进程是否存活。

rte_eal_primary_proc_alive函数的实现有些小技巧，其代码如下：

```c++
 145int                                                                                                                                                                                      
 146rte_eal_primary_proc_alive(const char *config_file_path)                                                                                                                                 
 147{                                                                                                                                                                                        
 148        int config_fd;                                                                                                                                                                   
 149                                                                                                                                                                                         
 150        if (config_file_path)                                                                                                                                                            
 151                config_fd = open(config_file_path, O_RDONLY);                                                                                                                            
 152        else {                                                                                                                                                                           
 153                const char *path;                                                                                                                                                        
 154                                                                                                                                                                                         
 155                path = eal_runtime_config_path();                                                                                                                                        
 156                config_fd = open(path, O_RDONLY);                                                                                                                                        
 157        }                                                                                                                                                                                
 158        if (config_fd < 0)                                                                                                                                                               
 159                return 0;                                                                                                                                                                
 160                                                                                                                                                                                         
 161        int ret = lockf(config_fd, F_TEST, 0);                                                                                                                                           
 162        close(config_fd);                                                                                                                                                                
 163                                                                                                                                                                                         
 164        return !!ret;                                                                                                                                                                    
 165}   

```

上述代码核心逻辑为调用 lockf 来检查 rte_config 的文件锁是否被其它进程占有，可以想到 primary 进程是唯一占有 rte_config 文件锁的用户，当没有人占用就表示 primary 进程未在运行。

### 3.5pdump 中加入 ebpf 过滤规则

dpdk ebpf 过滤规则可以支持使用高级语言开发 ebpf 指令来实现过滤报文的功能，pdump 起初只支持全量抓包，当它与 dpdk ebpf 框架结合后能够实现抓取特定规则的报文。

pdump_request 中新增的 prm 结构字段将 pdump 与 dpdk ebpf 框架结合起来，主要的修改为在 pdump_copy 拷贝报文前调用 rte_bpf_exec_burst函数，使用 ebpf 虚拟机执行 ebpf 规则过滤报文，然后只 copy 命中 ebpf 规则的报文即可。

目前 pdump 库已经支持 ebpf 规则过滤报文，可以调用rte_pdump_enable_bpf来在 pdump 抓包的基础上注入 ebpf 规则。此接口会将 ebpf 指令码翻译为本地机器码，但是目前并没有执行 jit 翻译的机器码，而是执行了 ebpf 虚拟机。

需要注意的是 pdump 目前并不支持注册 ebpf 过滤规则。

pdump 支持抓取的报文类型与报文大小
pdump 支持抓取 single-segment 类型与 multi-segment 类型 mbuf 报文，single-segment 类型支持抓取的报文大小由 pdump 程序启动时配置的 mempool 中每个 mbuf 的 dataroom size 决定，dataroom size 缺省大小为 2048。

pdump 支持多核抓包以及同时运行多个实例
pdump 程序支持通过配置一个 “–multi” 参数来在多个 cpu 逻辑核上同时运行不同的抓包任务，同时 pdump 程序作为一个 secondary 进程，本身也支持多进程启动（不同进程的抓包目标不能相同）。多核 + 多进程的设计能够很好的利用多核优势，增强抓包能力。

pdump 支持的两个版本
pdump 当前支持 V1 与 V2 两个版本，主要区别在于 copy 报文的过程。V2 版本使用 pcappng 接口将报文 copy 为 pcappng 格式，V1 版本使用 dpdk mbuf copy 接口 copy 报文为原始报文格式。

pdump 数据统计功能
pdump 定义了 rte_pdump_stats 结构来进行数据统计，此结构定义如下

```c++
234struct rte_pdump_stats {
235        uint64_t accepted; /**< Number of packets accepted by filter. */
236        uint64_t filtered; /**< Number of packets rejected by filter. */
237        uint64_t nombuf;   /**< Number of mbuf allocation failures. */
238        uint64_t ringfull; /**< Number of missed packets due to ring full. */
239  
240        uint64_t reserved[4]; /**< Reserved and pad to cache line */
241};  

```

dump 在大页上为所有收发包接口的收发包队列都创建一个rte_pdump_stats结构，此统计在 primary 进程中被修改，在 secondary 进程中被读取用于显示，在大页上分配支持 seondary 进程中够获取此数据。

同时，此数据的增加使用 gcc 提供的原子函数完成，确保了数据的一致性。

## 4.总结

pdump 看似简单，实则集成了 dpdk 内部众多子模块的功能，主要依赖的功能如下：

dpdk 大页内存共享
dpdk mp_channel 多进程通信机制
dpdk bus 接口热插拔技术
pcap 驱动发包存储报文技术
dpdk mempool 与 ring
上述功能都是 dpdk 的基石，通过整合将这些基础的模块连接到一起实现更为复杂的功能。这一功能又可以成为新的基石，被其它的对象整合，这就是软件设计的魔力，值得学习与借鉴。







原文链接：https://blog.csdn.net/Longyu_wlz/article/details/126217242?spm

原文作者：longyu_wlz
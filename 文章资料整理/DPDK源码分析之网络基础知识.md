# DPDK源码分析之网络基础知识

## 1.字节对齐 __attribute__ ((aligned (1)));

- 在设计不同CPU下的通信协议时，或者编写硬件驱动程序时寄存器的结构这两个地方都需要按一字节对齐。即使看起来本来就自然对齐的也要使其对齐，以免不同的编译器生成的代码不一样.
- 如果跨越了4字节边界存储，那么cpu要读取两次，这样效率就低了

## 2.内存池设计原理

![img](https://pic3.zhimg.com/80/v2-34c24f6814ef8a639b34108f824e8ea6_720w.webp)

好处：

- 比malloc/free进行内存申请/释放的方式快
- 不会产生或很少产生堆碎片
- 可避免内存泄漏

![img](https://pic1.zhimg.com/80/v2-acaf33e3eb88ac745f61c8ddc184c2e0_720w.webp)

![img](https://pic4.zhimg.com/80/v2-b9aefb85d10cba6d852617026b2fa1ab_720w.webp)

伙伴系统：

![img](https://pic3.zhimg.com/80/v2-152e6bba94f8e38e5ff8c001cf34a4ee_720w.webp)

## 3.包在系统协议栈的流图，操作系统做了那些事情

![img](https://pic2.zhimg.com/80/v2-48244713d9737cb7f80449bf794bc05d_720w.webp)

- 开始收包之前，Linux要做许多的准备工作，创建ksoftirqd线程，协议栈注册，linux要实现许多协议，比如arp，icmp，ip，udp，tcp，每一个协议都会将自己的处理函数注册一下；网卡驱动初始化，每个驱动都有一个初始化函数，内核会让驱动也初始化一下。在这个初始化过程中，把自己的DMA准备好，把NAPI的poll函数地址告诉内核；启动网卡，分配RX，TX队列，注册中断对应的处理函数
- 网卡通过DMA将packet写入内核的[rx_ring环形队列](https://www.zhihu.com/search?q=rx_ring环形队列&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType":"answer","sourceId":1327054886}" \t "https://www.zhihu.com/question/405704520/answer/_blank)缓冲区，并触发硬中断（如果没有DMA,CPU就会负责赋值网卡数据到内存中，这个过程非常耗时）
- CPU在收到中断之后，调用网卡ISR也就是所谓的中断handler,分配sk_buf并入input_pkt_queue（如果队列已满则丢弃）,发出一个软中断NET_RX_SOFTIRQ，软中断被调度
- ksoftirqd线程开始调用驱动的poll函数收包，然后将sk_buf从input_pkt_queue传入process_queue，根据协议类型调用网络层协议的handler，ip_rcv执行包头检查，ip_router_input()进行路由，决定本机/转发/丢弃，tcp_v4_rcv执行包头检查，tcp_v4_lookup查询对应的socket和connection，如果正常，tcp_prequeue将skb放进socket接收队列，socket随即唤醒所在的进程
- 唤醒的进程调用socket recv系统调用，如果是TCP则调用tcp_recvmsg从sk_buffer拷贝数据

## 4.Dpdk调优有哪些

- 控制层和数据层分离。将数据包处理、内存管理、处理器调度等任务转移到用户空间去完成，而内核仅仅负责部分控制指令的处理。这样就不存在上述所说的系统中断、上下文切换、系统调用、系统调度等等问题。
- 使用多核编程技术代替多线程技术，并设置 CPU 的亲和性，将线程和 CPU 核进行一比一绑定，减少彼此之间调度切换。
- 针对 NUMA (Non Uniform Memory Access Architecture)系统，尽量使 CPU 核使用所在 NUMA 节点的内存，避免跨内存访问。
- 使用大页内存代替普通的内存，减少 cache-miss。
- 采用无锁技术解决资源竞争问题。

## 5.Tcp建链的异常状态维护

- 第一次握手消息丢失，触发超时重传机制，包括重传次数和重传周期
- 第二次握手消息丢失：（SYN+ACK），客户端和服务端都会重传
- 第三次握手消息丢失：（ACK），服务会重传SYN+ACK报文段，直到收到ACK响应或者达到最大重传次数

![img](https://pic4.zhimg.com/80/v2-ddcaaa649de203b310a4159cc741c747_720w.webp)

![img](https://pic1.zhimg.com/80/v2-f6e17234205fe179ecd0aed4224c521c_720w.webp)

- 第一次挥手消息丢失：（FIN），客户端会开启重传流程，达到最大次数后客户端将停止重传，直接进入 CLOSE 状态。
- 第二次挥手消息丢失：（ACK），客户端会重传FIN报文，直到收到ACK报文，或达到FIN的最大重传次数。
- 第三次挥手消息丢失：（FIN），TCP状态将从 CLOSE_WAIT 状态转入 LAST_ACK 状态，如果在超时时间内未收到客户端的第四次挥手 ACK报文，则重传FIN报文
- 第四次挥手消息丢失：（ACK）,ACK报文不会重传，服务端在超时未收到ACK后会重传FIN，直至成功收到ACK会达到最大重传次数

## 6.TIME_WAIT存在的两个理由

- 可靠地实现TCP全双工连接的终止；
- 允许老的重复分节（数据报）在网络中消逝。

## 7.B+树

在B +树中, 记录（数据）只能存储在叶节点上, 而内部节点只能存储键值。

B +树的叶节点以单链接列表的形式链接在一起, 以使搜索查询更高效创建



![img](https://pic3.zhimg.com/80/v2-60e753809802bcda667e5c0b14f45af2_720w.webp)

## 8.为啥线程的地址空间能一样

线程使用的底层函数和进程一样，都是clone。从内核里看进程和线程是一样的，都有各自不同的PCB，但是PCB中指向内存资源的三级页表是相同的。进程可以蜕变成线程。线程可看做寄存器和栈的集合。

三级映射：进程PCB --> 页目录(可看成数组，首地址位于PCB中) --> 页表 --> 物理页面 --> 内存单元

两个线程具有各自独立的PCB，但共享同一个页目录，也就共享同一个页表和物理页面。

## 9.Reference

[(7条消息) 图解 Linux 网络包接收过程_Peter的专栏-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/melody157398/article/details/110251647%3Futm_medium%3Ddistribute.pc_relevant.none-task-blog-2~default~baidujs_title~default-12.queryctrv2%26spm%3D1001.2101.3001.4242.7%26utm_relevant_index%3D15)

[(7条消息) linux内存管理笔记(二十二）----伙伴系统原理_奇小葩-CSDN博客_伙伴系统原理](https://link.zhihu.com/?target=https%3A//blog.csdn.net/u012489236/article/details/106676060/%3Futm_medium%3Ddistribute.pc_relevant.none-task-blog-2~default~baidujs_title~default-1.pc_relevant_default%26spm%3D1001.2101.3001.4242.2%26utm_relevant_index%3D4)





原文链接：https://zhuanlan.zhihu.com/p/460993490  原文作者：**于顾而言**
# SPDK代码结构浅析

最近这三周时间一直因为工作的需要在研究SPDK移植到RISCV平台上，在编译通过的时候，也顺带把SPDK的代码粗粗过了一遍，顺便做了一点笔记。

SPDK (Storage Performance Development Kit)其实就是在用户空间，采用轮询的方式无锁的NVMe的驱动，提供从用户空间应用程序到SSD的零拷贝。

整体的代码结构框图，大家可以从网上搜搜，有很多的文章讲这部分，笔者这里从代码细节的角度来逐个分析它的一些功能。

1. SPDK系统的建立，通过如下函数的调用，SPDK主要做了如下几个步骤：

- 为每个logical-core建立一个对应的reactor的线程，其线程的入口函数为eal_thread_loop()
- 同时将reactor_run()注册到每cpu的变量lcore*_*config[].f中
- 当前线程分别通过pipe发送消息给其他logic-core上运行thread，此时由于其他logic-core上的线程会等待各自pipe上的消息，当前core上发送的pipe消息一旦被对端的线程所受到，则分别注册reactor run函数就可以分别运行在各自的线程中了

![img](https://pic1.zhimg.com/80/v2-087e2062baaceaf228e8b7cae1ed0768_720w.webp)

\2. 以下函数调用栈是整个SPDK架构的关键所在。

- 在reactor的run函数中是个死循环
- 它分别运行在各自的logic-core的线程上
- 每个线程上的reactor可以互相通过ring-buffer进行通信，收到消息的一方，如果监测到有回调函数，则会处理相应的回调函数
- reactor->threads链表上也链接了多个thread的实例，他是一个个结构体
- 每个thread实例也管理着多个poller实例的链表，如：active_pollers, paused_pollers, timed_pollers等链表。分别表示一次性的poller，暂停的poller以及定时poller。每个poller也可能有回调函数，通过poller->fn()位置进行回调函数的处理。

![img](https://pic3.zhimg.com/v2-80bdb5af3016214773636c7551fd6b96_r.jpg)

整个reactor中链表的拓扑结构如下所示：

![img](https://pic3.zhimg.com/v2-992b1c825b0d41a4c720c34a6d64c682_r.jpg)

\3. 对于SPDK中最重要的I/O数据的交互，就是通过每个poller的回调函数层层调用实现的。

先构造一个poller，参看如下的调用stack，截图中可以看到，这个过程注册了多个poller，但我们目前只关心nvme_ctrlr_*create()函数所注册的那个poller函数：bdev_nvme_poll_adminq().*

![img](https://pic3.zhimg.com/80/v2-8ca6f356c92535e95255b00f48e3235e_720w.webp)

此处是进行注册的收发消息的API，可以看到cuse_ns_*ioctl()就是真正的收发消息的API。消息其实是通过spdk_ring*_enqueue()放入了ring-buffer中的。在后续的poller挂在的函数中对ring-buffer中的消息进行处理。

![img](https://pic4.zhimg.com/80/v2-70a1b86b887c0642109b22fec5fd54ff_720w.webp)

下面是NVMe协议通过RDMA的I/O传送的调用栈，从一个poller->fn的回调开始，可以看到在poller的处理中，通过从消息队列中取出request的消息，再从收到的消息所包含的回调函数io->fn中，才真正实现通过RDMA的写操作。

![img](https://pic4.zhimg.com/v2-ed1d905708f361cff86472e6103e8d93_r.jpg)

这是上面的io->fn()调用：

写操作的处理

![img](https://pic1.zhimg.com/80/v2-ae175d15efaa8a2ff66b9fb35856b648_720w.webp)

读操作的处理

![img](https://pic4.zhimg.com/80/v2-134dee5b14ea6068ef676c7dd3982adf_720w.webp)

可以通过以上调用栈，看到有三种类型的接口：pcie/rdma/tcp

下面是这三种类型接口的调用栈：

PCIe

![img](https://pic2.zhimg.com/80/v2-26def3b9b3622e7dddb75e358f498229_720w.webp)

RDMA

![img](https://pic1.zhimg.com/v2-557b57b12c91a0ddb6ba51f1e27c4d20_r.jpg)

TCP

![img](https://pic3.zhimg.com/v2-3418a773aa15b7652238bd2ba185dc42_r.jpg)

未完待续

作者：[ZhigangWu](https://www.zhihu.com/people/wu_zhigang)  原文地址：https://zhuanlan.zhihu.com/p/423779832
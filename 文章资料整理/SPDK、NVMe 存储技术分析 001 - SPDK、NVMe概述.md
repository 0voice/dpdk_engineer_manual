# [SPDK/NVMe 存储技术分析\]001 - SPDK/NVMe概述

## **1. NVMe概述**

- NVMe是一个针对基于PCIe的固态硬盘的高性能的、可扩展的主机控制器接口。
- NVMe的显著特征是提供多个队列来处理I/O命令。单个NVMe设备支持多达64K个I/O 队列，每个I/O队列可以管理多达64K个命令。
- 当主机发出一个I/O命令的时候，主机系统将命令放置到提交队列(SQ)，然后使用门铃寄存器(DB)通知NVMe设备。
- 当NVMe设备处理完I/O命令之后，设备将处理结果写入到完成队列(CQ)，并引发一个中断通知主机系统。
- NVMe使用MSI/MSI-X和中断聚合来提高中断处理的性能。

## **2. SPDK概述**

The Storage Performance Development Kit (SPDK) provides a set of tools and libraries for writing high performance, scalable, user-mode storage applications. It achieves high performance by moving all of the necessary drivers into userspace and operating in a polled mode instead of relying on interrupts, which avoids kernel context switches and eliminates interrupt handling overhead.

SPDK(存储性能开发套件)为编写高性能的、可扩展的、用户态存储应用提供了一套工具和库函数。SPDK之所以能实现高性能，是因为所有必要的驱动被挪到了用户空间运行，使用轮询模式代替了中断模式，从而避免了内核上下文切换和消除了中断处理开销。

The bedrock of SPDK is a user space, polled-mode, asynchronous, lockless [NVMe](http://www.nvmexpress.org/) driver. This provides zero-copy, highly parallel access directly to an SSD from a user space application. The driver is written as a C library with a single public header. Similarly, SPDK provides a user space driver for the I/OAT [DMA](https://en.wikipedia.org/wiki/Direct_memory_access) engine present on many Intel Xeon-based platforms with all of the same properties as the NVMe driver.

SPDK的基石是一个运行在用户空间的、采用轮询模式的、异步的、无锁的NVMe驱动。用户空间应用程序可直接访问SSD盘，而且是零拷贝、高度并行地访问SSD盘。该驱动程序实现为一个C函数库，该函数库携带一个单一的公共头文件。类似地，SPDK为许多基于Intel至强平台的I/OAT DMA引擎提供了一个用户空间驱动程序，NVMe驱动所具备的所有属性，该驱动程序都具备。

SPDK also provides [NVMe-oF](http://www.nvmexpress.org/nvm-express-over-fabrics-specification-released) and [iSCSI](https://en.wikipedia.org/wiki/ISCSI) servers built on top of these user space drivers that are capable of serving disks over the network. The standard Linux kernel iSCSI and NVMe-oF initiator can be used (or the Windows iSCSI initiator even) to connect clients to the servers. These servers can be up to an order of magnitude more CPU efficient than other implementations.

SPDK还提供了NVMe-oF和基于这些用户空间驱动程序构建的iSCSI服务器, 从而有能力提供网络磁盘服务。客户端可以使用标准的Linux内核iSCSI和NVMe-oF initiator(或者甚至使用Windows的iSCSI initiator)来连接服务器。跟其他实现比起来，这些服务器在CPU利用效率方面可以达到数量级的提升。

SPDK is an [open source, BSD licensed](https://opensource.org/licenses/BSD-3-Clause) set of C libraries and executables hosted on [GitHub](https://github.com/spdk/spdk). All new development is done on the [master branch](https://github.com/spdk/spdk/tree/master) and [stable releases](https://github.com/spdk/spdk/releases) are created quarterly. Contributors and users are welcome to [submit patches](http://www.spdk.io/development/), [file issues](https://github.com/spdk/spdk/issues), and ask questions on our [mailing list](https://lists.01.org/mailman/listinfo/spdk).

SPDK是一个开源的、BSD授权的集C库和可执行文件一体的开发套件，其源代码通过GitHub托管。所有新的开发都放到master分支上，每个季度发布一个稳定版本。欢迎代码贡献者和用户提交补丁、报告问题，并通过邮件列表提问。

## **3. SPDK/NVMe驱动概述**

The NVMe driver is a C library that may be linked directly into an application that provides direct, zero-copy data transfer to and from [NVMe SSDs](http://nvmexpress.org/). It is entirely passive, meaning that it spawns no threads and only performs actions in response to function calls from the application itself. The library controls NVMe devices by directly mapping the [PCI BAR](https://en.wikipedia.org/wiki/PCI_configuration_space) into the local process and performing [MMIO](https://en.wikipedia.org/wiki/Memory-mapped_I/O). I/O is submitted asynchronously via queue pairs and the general flow isn't entirely dissimilar from Linux's [libaio](http://man7.org/linux/man-pages/man2/io_submit.2.html).

NVMe驱动是一个C函数库，可直接链接到应用程序从而在应用与NVMe固态硬盘之间提供直接的、零拷贝的数据传输。这是完全被动的，意味着不会开启线程，只是执行来自应用程序本身的函数调用。这套库函数直接控制NVMe设备，通过将PCI BAR寄存器直接映射到本地进程中然后执行基于内存映射的I/O(MMIO)。I/O是通过队列对(QP)进行异步提交，其一般的执行流程跟Linux的libaio相比起来，并非完全不同。

进一步的详细信息，请阅读[这里](http://www.spdk.io/doc/nvme.html)。

## **4. 其他NVMe驱动实现**

- Linux内核NVMe驱动 ： 去官网[www.kernel.org](https://www.kernel.org/)或者[镜像](https://mirror.tuna.tsinghua.edu.cn/)下载内核源代码，然后阅读include/linux/nvme.h, drivers/nvme
- **[NVMeDirect](https://github.com/nvmedirect/nvmedirect)** : 这是韩国人发起的一个开源项目，跟SPDK/NVMe驱动类似，但是严重依赖于Linux内核NVMe驱动的实现。 可以说，NVMeDirect是一个站在巨人肩膀上的用户态I/O框架。





转载自： 作者[vlhn](https://home.cnblogs.com/u/vlhn/)   本文地址https://www.cnblogs.com/vlhn/p/7727141.html
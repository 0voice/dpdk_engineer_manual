# DPDK源码分析之rte_eal_init

## 1.EAL是什么

环境抽象层(EAL)负责获得对底层资源(如硬件和内存空间)的访问。对于应用程序和其他库来说，使用这个通用接口可以不用关系具体操作系统的环境细节。rte_eal_init初始化例程负责决定如何分配操作系统的这些资源(即内存空间、设备、定时器、控制台等等)。

EAL提供的典型服务是：

- DPDK加载和启动：DPDK及其提供的应用程序是被链接为单个应用程序，因此必须由某种方式加载。
- 核心绑定/分配过程：EAL提供用于将执行单元分配给特定核心的机制以及创建其执行实例。
- 系统内存预留：EAL便于预留不同的存储区，例如，用于设备交互的物理内存区域。
- 跟踪和调试功能：日志，打印栈信息，类似于printf函数功能等。
- 实用功能：标准c中未提供的旋转锁和原子锁功能。
- CPU功能标识：确定运行时的一些CPU特性，以确定该CPU是否支持DPDK功能
- 中断处理：提供注册/取消注册的一系列接口，用于出现中断源时调用
- 报警功能：提供注册/取消注册的一系列接口，在特定时间或事件时调用

## 2.EAL涉及的专业名词

### 2.1__atomic_compare_exchange_n

Built-in Function:*bool***__atomic_compare_exchange_n***(type\*ptr,type\*expected,typedesired, bool weak, int success_memorder, int failure_memorder)*

此内置功能是C++11用于多线程中对共享变量的原子比较和交换操作。

This compares the contents of`*ptr`with the contents of`*expected`. If equal, the operation is a*read-modify-write*operation that writesdesiredinto`*ptr`. If they are not equal, the operation is a*read*and the current contents of`*ptr`are written into`*expected`.

If desiredis written into`*ptr`then true is returned，Otherwise, false is returned.

### 2.2mmap映射

mmap是一种内存映射文件的方法，即将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用read,write等系统调用函数。相反，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件共享。如下图所示：

![img](https://pic3.zhimg.com/80/v2-3044e1b48f96655587e0b22ff3cf665e_720w.webp)

总结来说，常规文件操作为了提高读写效率和保护磁盘，使用了页缓存机制。这样造成读文件时需要先将文件页从磁盘拷贝到页缓存中，由于页缓存处在内核空间，不能被用户进程直接寻址，所以还需要将页缓存中数据页再次拷贝到内存对应的用户空间中。这样，通过了两次数据拷贝过程，才能完成进程对文件内容的获取任务。写操作也是一样，待写入的buffer在内核空间不能直接访问，必须要先拷贝至内核空间对应的主存，再写回磁盘中（延迟写回），也是需要两次数据拷贝。

而使用mmap操作文件中，创建新的虚拟内存区域和建立文件磁盘地址和虚拟内存区域映射这两步，没有任何文件拷贝操作。而之后访问数据时发现内存中并无数据而发起的缺页异常过程，可以通过已经建立好的映射关系，只使用一次数据拷贝，就从磁盘中将数据传入内存的用户空间中，供进程使用。

**总而言之，常规文件操作需要从磁盘到页缓存再到用户主存的两次数据拷贝。而mmap操控文件，只需要从磁盘到用户主存的一次数据拷贝过程。**

### 2.3runtime

应用程序运行过程中的环境参数

### 2.4uio device

UIO出现的原因是，对于标准的硬件设备如PCI设备，USB设备等。它们被不同的内核子系统支持。这些标准的设备的驱动编写较为容易而且容易维护。很容易加入主内核源码树，但是有一些设备如I/O卡，现场总线接口或者定制的FPGA。通常这些非标准设备的驱动被实现为字符驱动。这些驱动使用了很多内核内部函数和宏。而这些内部函数和宏是变化的。这样驱动的编写者必须编写一个完全的内核驱动，而且一直维护这些代码。像这种设备如果把驱动放入Linux内核,不但增大了内核的负担而且还很少使用，所以UIO框架应运而生。

一个设备驱动的主要任务有两个：1. 存取设备的内存 2. 处理设备产生的中断

对于第一个任务，UIO 核心实现了mmap()可以处理物理内存(physical memory)，虚拟内存(virtual memory)。第二个任务，如果用户空间要等待一个设备中断，它只需要简单的阻塞在对 /dev/uioX的read()操作上。当设备产生中断时，read()操作立即返回。UIO 也实现了poll()系统调用，你可以使用select()来等待中断的发生。其次，对设备的控制还可以通过/sys/class/uio下的各个文件的读写来完成。你注册的uio设备将会出现在该目录下。假如你的uio设备是uio0那么映射的设备内存文件出现/sys/class/uio/uio0/maps/mapX，对该文件的读写就是对设备内存的读写。

![img](https://pic4.zhimg.com/80/v2-59254b5f5de0bab581db41cfc2542143_720w.webp)

内核操作：1. - 使能设备 2. - 申请资源 3. - 读取并记录配置信息 4. - 注册uio设备// uio_register_device()

用户态操作：5. 判断是否产生了硬件中断 6. 响应硬件中断

### 2.5Sysfs

Sysfs是Linux 2.6所提供的一种[虚拟文件系统](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E8%99%9B%E6%93%AC%E6%AA%94%E6%A1%88%E7%B3%BB%E7%B5%B1)。这个[文件系统](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E6%AA%94%E6%A1%88%E7%B3%BB%E7%B5%B1)不仅可以把[设备](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E8%A3%85%E7%BD%AE)（devices）和[驱动程序](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E9%A9%85%E5%8B%95%E7%A8%8B%E5%BC%8F)（drivers）的信息从内核输出到[用户空间](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E7%94%A8%E6%88%B7%E7%A9%BA%E9%97%B4)，也可以用来对设备和驱动程序做设置。

### **2.6epoll**

在高并发场景，随`epoll`使用了内核文件级别的回调机制O(1)。它有两种触发机制，水平触发(level-triggeredsocke)，即t接收缓冲区不为空 有数据可读，读事件一直触发，和边沿触发(edge-triggered)，即socket的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时触发读事件，边沿触发仅触发一次，水平触发会一直触发。

具体原理如下：

调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个 **红黑树** 用于存储以后epoll_ctl传来的socket外，还会再建立一个[list链表](https://www.zhihu.com/search?q=list链表&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A93609693})，用于存储准备就绪的事件.

当**epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效**。而且，通常情况下即使我们要监控百万计的句柄，大多一次也只返回很少量的准备就绪句柄而已，所以，epoll_wait仅需要从内核态copy少量的句柄到用户态而已.

当我们执行epoll_ctl时，除了把socket放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个句柄的中断到了，就把它放到准备就绪list链表里。所以，当一个socket上有数据到了，**内核在把网卡上的数据copy到内核中后就来把socket插入到准备就绪链表里了**。

![img](https://pic2.zhimg.com/80/v2-5d0cf309bc60427979de229666a07c71_720w.webp)

### 2.7**创建pipe**

**pipe用于实现进程间的通信，工作原理如下：**1）父进程创建管道，得到两个⽂件描述符指向管道的两端2）父进程fork出子进程，⼦进程也有两个⽂件描述符指向同⼀管道。3）父进程关闭fd[0],子进程关闭fd[1]，即⽗进程关闭管道读端,⼦进程关闭管道写端（因为管道只支持单向通信，父进程关闭fd[1]子进程关闭fd[0]也可以）。

⽗进程可以往管道⾥写,⼦进程可以从管道⾥读,管道是⽤环形队列实现的,数据从写端流⼊从读端流出,这样就实现了进程间通信。

**管道的特点如下：**1.管道只允许具有血缘关系的进程间通信，如父子进程间的通信。 2.管道只允许单向通信。 3.管道内部保证同步机制，从而保证访问数据的一致性。 4.面向字节流 5.管道随进程，进程在管道在，进程消失管道对应的端口也关闭，两个进程都消失管道也消失。

### 2.8**bus设备**

总线(bus)是linux发展过程中抽象出来的一种设备模型，为了统一管理所有的设备，内核中每个设备都会被挂载在总线上，这个bus可以是对应硬件的bus(i2c bus、spi bus)、可以是虚拟bus(platform bus)。

驱动程序对应driver、需要的具体型号的硬件资源对应device，将其挂在bus上。将driver注册到bus上，当用户需要使用AT24C01时，以AT24C01的参数构建一个对应device，注册到bus中，bus的match函数匹配上之后，调用probe函数，即可完成AT24C01的初始化，完成在用户空间的文件接口注册。

以此类推，添加所有的AT24CXX设备都可以以这样的形式实现，只需要构建一份device，而不用为每个设备重写一份驱动，提高了复用性，节省了内存空间。

linux将设备挂在总线上，对应设备的注册和匹配流程由总线进行管理，在linux内核中，每一个bus，都由struct bus_type来描述：

```c
struct bus_type {
    const char	*name;
    const char	*dev_name;
    struct device *dev_root;
    ...
    int (*match)(struct device *dev, struct device_driver *drv);
    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
    int (*probe)(struct device *dev);
    int (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);
    struct subsys_private *p;
    ...
};
```

> name : 该bus的名字，这个名字是这个bus在sysfs文件系统中的体现，对应/sys/bus/$name.
> dev_name : 这个dev_name并不对应bus的名称，而是对应bus所包含的struct device的名字，即对应dev_root。
> dev_root：bus对应的device结构，每个设备都需要对应一个相应的struct device.
> match:bus的device链表和driver链表进行匹配的实际执行回调函数，每当有device或者driver添加到bus中时，调用match函数，为device(driver)寻找匹配的driver(device)。
> uevent:bus时间回调函数，当属于这个bus的设备发生添加、删除、修改等行为时，都将出发uvent事件。
> probe:当device和driver经由match匹配成功时，将会调用总线的probe函数实现具体driver的初始化。事实上每个driver也会提供相应的probe函数，先调用总线的probe函数，在总线probe函数中调用driver的probe函数。
> remove:移除挂载在设备上的driver，bus上的driver部分也会提供remove函数，在执行移除时，先调用driver的remove，然后再调用bus的remove以清除资源。

- **--iova-mode=<pa|va>**

DPDK是一个用户态应用框架，使用DPDK的软件可以像其他软件一样使用常规虚拟地址。但除此之外，DPDK还提供了用户态PMD和一组API，以实现完全以用户态执行IO操作

作为PA的IOVA模式下，分配到整个DPDK存储区的IOVA地址都是实际的物理地址，而虚拟内存的分配与物理内存的分配相匹配。该模式的一大优点就是它很简单：它适用于所有硬件（也就是说，不需要IOMMU），并且它适用于内核空间（将真实物理地址转换为内核空间地址的开销是微不足道的）。

作为VA的IOVA模式不需遵循底层物理内存的分布。而是重新分配物理内存，与虚拟内存的分配匹配，即使底层物理内存可能不存在，内存看上去还是IOVA连续的。由于重新映射，IOVA空间片段化的问题就变得无关紧要。不管物理内存被分段得多么严重，它总能被重新映射为IOVA-连续的大块内存。

![img](https://pic4.zhimg.com/80/v2-c28b7159daf5cb1edba81b2a55ee338f_720w.webp)



![img](https://pic1.zhimg.com/80/v2-f32bfcb85ce5d6d64e24bf5b99be538c_720w.webp)

### **2.9soket_id**

socket_id存放着物理cpu号

### **3.0TSC frequency**

Linux中有3种timer：

1、Real Time Clock(RTC)

2、Programmalbe Interval Timer(PIT)

3、Time Stamp Counter.(TSC)

其中RTC是位于CMOS中的，其频率范围是2HZ--8192HZ.

PIT主要由8254时钟芯片实现的

TSC的主体是位于CPU里面的一个64位的TSC寄存器。每个CPU时钟周期其值加一。

### 3.1**numa socket**

> socket是一个物理上的概念，指的是主板上的cpu插槽。node是一个逻辑上的概念，对应于socket。
> numa架构下，访问本地内存的速度要快于访问远端内存的速度，访问速度与node的距离有关系。

![img](https://pic2.zhimg.com/80/v2-9a26f738eef9854495a62751cc21c9f9_720w.webp)

> node 0到本地内存的距离为10，到node 1的内存距离为20；node1到本地内存的距离为10，到node 0的内存距离为20。

## 3.EAL源码框图

![img](https://pic2.zhimg.com/80/v2-3a46302e6c1ca08a58816db92e2b12cd_720w.webp)

## 4.Reference

[3. Environment Abstraction Layer — Data Plane Development Kit 21.11.0 documentation (dpdk.org)](https://link.zhihu.com/?target=http%3A//doc.dpdk.org/guides/prog_guide/env_abstraction_layer.html%23eal-in-a-linux-userland-execution-environment)

[DPDK初始化 - 坚持，每天进步一点点 - 博客园 (cnblogs.com)](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/mysky007/p/11044542.html)

[(21条消息) 深度分析mmap：是什么 为什么 怎么用 性能总结_精诚所至-CSDN博客_mmap](https://link.zhihu.com/?target=https%3A//blog.csdn.net/qq_33611327/article/details/81738195%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522163825685316780274181811%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D%26request_id%3D163825685316780274181811%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-1-81738195.pc_search_result_cache%26utm_term%3Dmmap%26spm%3D1018.2226.3001.4187)

[Using the GNU Compiler Collection (GCC): __atomic Builtins](https://link.zhihu.com/?target=https%3A//gcc.gnu.org/onlinedocs/gcc-7.3.0/gcc/_005f_005fatomic-Builtins.html%23g_t_005f_005fatomic-Builtins%EF%BC%89)

[Linux 设备驱动之 UIO 机制（基本概念）](https://link.zhihu.com/?target=https%3A//blog.csdn.net/xy010902100449/article/details/46917623%3Fspm%3D1001.2101.3001.6650.2%26utm_medium%3Ddistribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-2.no_search_link%26depth_1-utm_source%3Ddistribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-2.no_search_link)

[(21条消息) Linux 设备驱动之 UIO 机制（测试 UIO 机制）_ZP1015-CSDN博客_linux uio](https://link.zhihu.com/?target=https%3A//blog.csdn.net/xy010902100449/article/details/46917663%3Futm_medium%3Ddistribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0.no_search_link%26spm%3D1001.2101.3001.4242.1)

[深入理解 Epoll - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/93609693)

[(21条消息) 进程间的通信方式——pipe（管道）_sky_Mata的博客-CSDN博客_管道通信](https://link.zhihu.com/?target=https%3A//blog.csdn.net/skyroben/article/details/71513385)

[linux设备驱动程序--bus - 牧野星辰 - 博客园 (cnblogs.com)](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/downey-blog/p/10507703.html)

[(21条消息) DPDK内存篇（二）: 深入学习 IOVA_老马农的博客-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/majianting/article/details/102938013%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522163826564016780274185374%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D%26request_id%3D163826564016780274185374%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-1-102938013.pc_search_result_cache%26utm_term%3Diova%26spm%3D1018.2226.3001.4187)

[(23条消息) numa下socket node cpu thread的关系_冰糖银耳盅的博客-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/envy13/article/details/80241886)

[(23条消息) DPDK18.11.11内存初始化流程总结_aixueai的博客-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/aixueai/article/details/114821076%3Fops_request_misc%3D%26request_id%3D%26biz_id%3D102%26utm_term%3Dinternal_config%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-1-114821076.first_rank_v2_pc_rank_v29%26spm%3D1018.2226.3001.4187)

[(23条消息) intel dpdk rte_eal_cpu_init() 函数介绍_编码人生——朝阳_tony-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/linzhaolover/article/details/9263141%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522163854744016780271562770%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D%26request_id%3D163854744016780271562770%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~baidu_landing_v2~default-1-9263141.first_rank_v2_pc_rank_v29%26utm_term%3Drte_eal_cpu_init%26spm%3D1018.2226.3001.4187)

[(23条消息) DPDK跟踪库：trace library_RToax-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/Rong_Toa/article/details/108716237%3Fops_request_misc%3D%26request_id%3D%26biz_id%3D102%26utm_term%3D%E8%B7%9F%E8%B8%AA%E5%BA%93%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-1-108716237.first_rank_v2_pc_rank_v29%26spm%3D1018.2226.3001.4187)

[DPDK中断机制简析 - MerlinJ - 博客园 (cnblogs.com)](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/MerlinJ/p/4104039.html)

[https://blog.csdn.net/xiajun07061225/article/details/9250579](https://link.zhihu.com/?target=https%3A//blog.csdn.net/xiajun07061225/article/details/9250579)

[DPDK的进程间通信机制 – 孙希栋的博客 (sunxidong.com)](https://link.zhihu.com/?target=https%3A//www.sunxidong.com/567.html)

[(24条消息) DPDK学习记录9 - 内存初始化2之rte_eal_memzone_init_jeawayfox的博客-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/jeawayfox/article/details/105827788%3Fops_request_misc%3D%257B%2522request%255Fid%2522%253A%2522163869795416780366569848%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D%26request_id%3D163869795416780366569848%26biz_id%3D0%26utm_medium%3Ddistribute.pc_search_result.none-task-blog-2~all~baidu_landing_v2~default-1-105827788.first_rank_v2_pc_rank_v29%26utm_term%3Drte_eal_memzone_init%26spm%3D1018.2226.3001.4187)

 原文链接：https://zhuanlan.zhihu.com/p/439119807   原文作者：于顾而言

# 
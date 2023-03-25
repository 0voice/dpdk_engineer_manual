# DPDK原理探索: igb_uio

我们知道，DPDK的核心特性在于绕开内核对网卡设备进行操作，而要实现这一点，DPDK采用的是Linux内核提供的UIO框架，并且对其进行封装，最终暴露给用户的东西就是——`igb_uio`。

## 1.什么是`igb_uio`？

`igb_uio`本质上是一个驱动`(driver)`。它是以一个内核模块的形式被加入到内核中的。我们使用的`insmod`命令其实就是一个插入内核模块`(kernel module)`的工具。

## 2.`igb_uio`的初始化：

![img](https://pic1.zhimg.com/80/v2-b35907cd06d862e91b06272b0cd5d6f8_720w.webp)

模块初始化

可以看到，`igb_uio`的初始化是通过`igbuio_pci_init_module`来实现的，而这个函数又是怎么被调用的呢？事实上，在`igb_uio`内核模块被插入时，就会触发上图中`module_init`所设置的初始化回调函数。

![img](https://pic3.zhimg.com/80/v2-f5ee05b5873fcbfd9335814b07b3684a_720w.webp)

igb_uio初始化回调函数

在`igbuio_pci_init_module`函数中，我们关心的是`pci_register_driver`函数，该函数将`igbuio_pci_driver`向内核中进行注册。同时，在`igbuio_pci_driver`结构体中，还有一个`id_table`的字段，它是一个`struct pci_device_id`数组，表示当前驱动所管理的所有设备。在`pci_register_driver`函数中，会调用`probe`回调函数去探测所有设备。但特殊的是，`igb_uio`在初始化时是没有管理任何设备的，即`id_table`数组为空。

奇怪了，显然`igb_uio`的驱动是需要管理网卡设备的，但是我们这里又没有任何驱动可以被探测到，那我们的网卡设备是什么时候被加入到`igb_uio`驱动的管理中的呢？

答案就是DPDK提供的`usertools/dpdk-devbind.py`脚本，这个脚本可以将某个网卡设备绑定为`igb_uio`驱动下的网卡设备，而绑定之后，就可以触发`probe`回调函数探测这个新的网卡设备，探测成功后，我们的新网卡就进入了`igb_uio`的设备数组中。

## 3.`igb_uio`的设备探测过程：

在`igbuio_pci_probe`函数中， 完成的主要工作有五个：

1. 使能当前设备，即通过`pci_enable_device(dev)`函数`enable`当前设备
2. `igbuio_setup_bars`，该函数的功能是将当前设备的所有`PCI BAR`的全部信息读取到`struct uio_info`结构体中，后续注册`UIO`设备时需要使用
3. 设置`DMA`模式
4. 初始化`UIO`设备的`File Operations`，即`open`、`release`和`irqcontrol`等
5. 通过`uio_register_device`函数将当前设备注册为`UIO`设备。

## 4. `PCI BAR`设备信息的读取：

事实上，一个`PCI`设备通常有6个`PCI BAR`，这些`PCI BAR`在内核中被称为`resource`。而每一个`PCI BAR`都有各自的类型和物理地址。其中，类型有两种，一种是`Memory`类型的资源，另一种是`I/O Port`。

![img](https://pic3.zhimg.com/80/v2-ac13a591dd1a31dfe693e50f2a4adbea_720w.webp)

PCI BAR的数据读取

其中，对于IORESOURCE_MEM类型，通过`igbuio_pci_setup_iomem`函数进行读取，而对于IORESOURCE_IO类型，通过`igbuio_pci_setup_ioport`来进行读取。

那么，到底是什么`PCI BAR`呢？`BAR`的全称是`Base Address Register`，它是一个存储基地址的寄存器。在PCI设备内部的内存布局是这样的：

![img](https://pic4.zhimg.com/80/v2-87015db4d3dd217433c902ff862f782f_720w.webp)

PCI内存布局

而每个`BAR`的数据分布如下：

![img](https://pic4.zhimg.com/80/v2-88b4a03599759cf21ea40af3f720c3eb_720w.webp)

每一个PCI BAR的数据分布

### 4.1 igbuio_pci_setup_iomem`的过程：

在`igbuio_pci_setup_iomem`函数中，通过`pci_resource_start`内核函数可以获取当前`PCI BAR`的物理地址起点，而通过`pci_resource_len`可以获取当前`PCI BAR`的物理地址长度。

![img](https://pic3.zhimg.com/80/v2-b9d61212e6c2572600ee9ff363d87fae_720w.webp)

IORESOURCE_MEM类型的PCI BAR

在上图中，最后方框标注的部分，就是将我们所读取到的`PCI BAR`的数据存入`struct uio_info`结构体中，以供后面注册`UIO`设备时使用。

值得一提的是，`internal_addr`其实在DPDK中是没有用到的，它表示当前`PCI BAR`的物理地址映射到内核中的虚拟地址。

### 4.2`igbuio_pci_setup_ioport`的过程：

`igbuio_pci_setup_ioport`函数的过程与`igbuio_pci_setup_iomem`大致相同。

![img](https://pic2.zhimg.com/80/v2-7a74abaf52b681274e1ba796d2b65dc5_720w.webp)

IORESOURCE_IO

### 4.3为什么有`I/O Memory`和`I/O Port`两种类型？

对于一个设备来说，除了自带的内存之外，还可能有寄存器等存储手段。显然，设备自带的内存就属于`I/O Memory`，而寄存器等其他的存储手段属于`I/O Port`类型。

![img](https://pic2.zhimg.com/80/v2-e255f978552ad0faae055611b13f9855_720w.webp)

I/O Space 和 Memory Space

### 4.4`igb_uio`如何绕开内核？

事实上，所有`UIO`设备在`sysfs`文件系统下都有一系列的文件接口，而用户程序就可以通过这些文件读取所需的设备信息，例如我们关心的`PCI BAR`等。得到了`PCI BAR`里存储的设备物理地址，就可以建立内存映射，进而与设备进行通信。

但是在DPDK的PMD中，并未使用`PCI BAR`的信息建立内存映射，而是直接使用`mmap`，直接映射`/dev/uioX`文件从而与某个`UIO`设备建立联系。

而不同的`PCI BAR`对应的内存区域在`mmap`时如何区分呢？ 简单地说，可以通过计算偏移量来区分，第n个`PCI BAR`对应的内存偏移量为`n * pagesz`，即每个`PCI BAR`对应的内存区域都在不同的页内。

## 5.参考资料：

1. 5.1[DPDK-IGB_UIO](https://link.zhihu.com/?target=https%3A//www.codetd.com/article/11212174)
2. [Userspace I/O HOWTO](https://link.zhihu.com/?target=https%3A//www.kernel.org/doc/html/v4.14/driver-api/uio-howto.html)
3. [How to Write Linux PCI Driver](https://link.zhihu.com/?target=https%3A//01.org/linuxgraphics/gfx-docs/drm/PCI/pci.html)
4. [PCI - OSDev Wiki](https://link.zhihu.com/?target=https%3A//wiki.osdev.org/PCI)
5. **Linux Driver Development, Edition 3, by Jonathan Corbet, Alessandro Rubini, and Greg Kroah-Hartman**





原文链接：https://zhuanlan.zhihu.com/p/483868843 原文作者：Gin
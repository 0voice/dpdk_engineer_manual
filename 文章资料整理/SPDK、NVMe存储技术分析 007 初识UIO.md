# [SPDK/NVMe存储技术分析\]007 初识UIO

注： 要进一步搞清楚SSD盘对应的PCI的BAR寄存器的映射，有必要先了解一下UIO(Userspace I/O)。 

UIO(Userspace I/O)是运行在用户空间的I/O技术。在Linux系统中，一般的设备驱动都是运行在内核空间，而在用户空间使用应用程序调用即可。而UIO则是将设备驱动的很少一部分运行在内核空间，而在用户空间实现驱动的绝大多数功能。那么，在内核空间UIO要做的事情就变得很简单，分为两种：

1. 分配和记录设备需要的资源和注册UIO设备
2. 实现必须在内核空间实现的中断处理函数

为了对UIO有一个直观的认识，先上个图：

![img](https://images2017.cnblogs.com/blog/1264595/201710/1264595-20171031155110496-1865063913.png)

了解了UIO 驱动在Linux系统中的位置后，让我们对参考资料(Linux User Space Device Drivers)的部分内容做一个中英文对照翻译以加深对UIO的理解。

## **1. Device Driver Architectures | 设备驱动架构**

- Linux device drivers are typically designed as kernel drivers running in kernel space
  典型的Linux设备驱动都是被设计为运行在内核空间的内核驱动
- User space I/O is another alternative device driver architecture that has been supported by the Linux kernel since 2.6.24
  从Linux内核版本2.6.24开始，就支持另一种可作为内核设备驱动的替代方案的设备驱动架构，也就是用户空间I/O
- People in the Linux kernel community may not always agree on the need to have user space I/O
  在Linux内核社区的人们不总是赞成使用用户空间I/O
- Industrial I/O cards have been taking advantage of user space I/O for quite some time
  在工业中使用的I/O卡利用用户空间I/O的优点已经有一阵子了
- For some types of devices, creating a Linux kernel driver may be overkill
  对某些类型的设备来说，创建对应的Linux内核驱动很可能代价太高
- Soft IP for FPGAs can have unique requirements that don't always fit the mold
  FPGA的软IP有独特的需求，将驱动放在内核实现并不总是适合的

## **2. Legacy User Space Driver Methods (/dev/mem) | 传统的用户态驱动实现方法(/dev/mem)**

- A character driver referred to as /dev/mem exists in the kernel that will map device memory into user space
- With this driver user space applications can access device memory
- Memory access can be disabled in the kernel configuration as this is a big security hole (CONFIG_STRICT_DEVMEM)
- Must be root user
- A great tool for prototyping or maybe testing new hardware, but is not considered to be an acceptable production solution for a user space device driver
- Since it can map any address into user space a buggy user space driver could crash the kernel

## **3. Introduction to UIO | UIO概述**

- The Linux kernel provides a framework for doing user space drivers called UIO
- The framework is a character mode kernel driver (in drivers/uio) which runs as a layer under a user space driver
- UIO helps to offload some of the work to develop a driver
- The "U" in UIO is not for universal
  - \- Devices well handled by kernel frameworks should ideally stay in the kernel (if you ask many kernel developers)
  - \- Networking is one area where semiconductor vendors are doing user space I/O to get improved performance
- UIO handles simple device drivers really well
  - \- Simple driver: Device access and interrupt processing with no need to access kernel frameworks

## **4. Kernel Space Driver Characteristics | 内核空间驱动的特点**

### 4.1 Advantages | 优点

- Runs in kernel space in the highest privilege mode to allow access to interrupts and hardware resources
- There are a lot of kernel services such that kernel space drivers can be designed for complex devices
- The kernel provides an API to user space which allows multiple applications to access a kernel space driver simultaneously
  - \- Larger and more scalable software systems can be architected
- Many drivers tend to be kernel space
  - \- Asking questions in the open source community is going to be easier
  - \- Pushing drivers to the open source community is likely easier

### 4.2 Disadvantages | 缺点

- System call overhead to access drivers
  - \- A switch from user space to kernel space (and back) is required
  - \- Overhead can be non-deterministic having impact on real time applications
- Challenging learning curve for developers
  - \- The kernel API is different from the application level API such that it takes time to become productive
- Bugs can be fatal causing a kernel crash
- Challenging to debug
  - \- Kernel code is highly optimized and there are different debug tools
- Frequent kernel API changes
  - \- Kernel drivers built for one kernel version may not build for another

## **5. User Space Device Driver Characteristics | 用户空间驱动的特点**

### 5.1 Advantages | 优点

- Less challenging to debug as debug tools are more readily available and common to normal application development
- User space services such as floating point are available
- Device access is very efficient as there is no system call required
- The application API of Linux is very stable
- The driver can be written in any language, not just "C"

### 5.2 Disadvantages | 缺点

- No access to the kernel frameworks and services
  - \- Contiguous memory allocation, direct cache control, and DMA are not available
  - \- May have to duplicate kernel code or use a kernel driver to supplement
- Interrupt handling cannot be done in user space
  - \- It must be handled by a kernel driver which notifies user space causing some delay
- There is no predefined API to allow applications to access the device driver
  - \- Concurrency must also be considered if multiple applications access a driver

## **6. UIO Framework Features | UIO框架的特性**

- There are two distinct UIO device drivers provided by Linux in drivers/uio
- UIO Driver (drivers/uio.c)
  - \- For more advanced users as a minimal kernel space driver is required to setup the UIO framework
  - \- This is the most universal and likely to handle all situations since the kernel space driver can be very custom
  - \- The majority of work can be accomplished in the user space driver
- UIO Platform Device Driver (drivers/uio_pdev_irqgen.c)
  - This driver augments the UIO driver such that no kernel space driver is required
    - It provides the required kernel space driver for uio
  - It works with device tree making it easy to use
    - The device tree node for the device needs to use "generic uio" in it's compatible
  - Best starting point since no kernel space code is needed

## **7. UIO Driver Kernel Configuration | 支持UIO驱动所需要的内核配置**

- UIO drivers must be configured in the Linux kernel

```
CONFIG_UIO=y
CONFIG_UIO_PDRV_GENIRQ=y
```

## **8. UIO Platform Device Driver Details | UIO平台服务驱动详解**

- The user provides only a user space driver
- The UIO platform device driver configures from the device tree and registers a UIO device
- The user space driver has direct access to the hardware
- The user space driver gets notified of an interrupt by reading the UIO device file descriptor

![img](https://images2017.cnblogs.com/blog/1264595/201711/1264595-20171101153817451-779993243.png)

## **9. Kernel UIO API - Sys Filesystem**

- The UIO driver in the kernel creates file attributes in the sys filesystem describing the UIO device
- /sys/class/uio is the root directory for all the file attributes
- A separate numbered directory structure is created under /sys/class/uio for each UIO device
  - \- First UIO device: /sys/class/uio/uio0
  - \- /sys/class/uio/uio0/name contains the name of the device which correlates to the name in the uio_info structure
  - \- /sys/class/uio/uio0/maps is a directory that has all the memory ranges for the device
  - \- Each numbered map directory has attributes to describe the device memory including the address, name, offset and size
    -  /sys/class/uio/uio0/maps/map0

## **10. User Space Driver Flow | 用户态驱动工作流程**

- 01 - The kernel space UIO device driver(s) must be loaded before the user space driver is started (if using modules)
- 02 - The user space application is started and the UIO device file is opened (/dev/uioX where X is 0, 1, 2 ...)
  - \- From user space, the UIO device is a device node in the file system just like any other device
- 03 - The device memory address information is found from the relevant sysfs directory, only the size is needed
- 04 - The device memory is mapped into the process address space by calling the mmap() function of the UIO driver
- 05 - The application accesses the device hardware to control the device
- 06 - The device memory is unmapped by calling munmap()
- 07 - The UIO device file is closed

## **11. User Space Driver Example | 用户态驱动示例**

```
#define UIO_SIZE "/sys/class/uio/uio0/maps/map0/size"

int main(int argc, char **argv)
{
        int             uio_fd;
        unsigned int    uio_size;
        FILE            *size_fp;
        void            *base_address;

        /*
         * 1. Open the UIO device so that it is ready to use
         */
        uio_fd = open("/dev/uio0", O_RDWR);

        /*
         * 2. Get the size of the memory region from the size sysfs file
         *    attribute
         */
        size_fp = fopen(UIO_SIZE, O_RDONLY);
        fscanf(size_fp, "0x%08X", &uio_size);

        /*
         * 3. Map the device registers into the process address space so they
         *    are directly accessible
         */
        base_address = mmap(NULL, uio_size,
                           PROT_READ|PROT_WRITE,
                           MAP_SHARED, uio_fd, 0);

        // Access to the hardware can now occur ...

        /*
         * 4. Unmap the device registers to finish
         */
        munmap(base_address, uio_size);

        ...
}
```

## 12. Mapping Device Memory Details | 设备内存映射详解

- The character device driver framework of Linux provides the ability to map device memory into a user space process address space
- A character driver may implement the mmap() function which a user space application can call
- The mmap() function creates a new mapping in the virtual address space of the calling process
  - \- A virtual address, corresponding to the physical address specified is returned
  - \- It can also be used to map a file into a memory space such that the contents of the file are accessed by memory reads and writes
- Whenever the user space program reads or writes in the virtual address range it is accessing the device
- This provides improved performance as no system calls are required



## **13. Mapping Device Memory Flow | 设备内存映射流程**

![img](https://images2018.cnblogs.com/blog/1264595/201711/1264595-20171127091108206-685658568.png)

## **14. User Space Application Interrupt Processing | 用户空间应用程序中断处理**

- Interrupts are never handled directly in user space
- The interrupt can be handled by the UIO kernel driver which then relays it on to user space via the UIO device file descriptor
- The user space driver that wants to be notified when interrupts occur calls select() or read() on the UIO device file descriptor
  -  \- The read can be done as blocking or non-blocking mode
- read() returns the number of events (interrupts)
- A thread could be used to handle interrupts
- Alternatively a user provided kernel driver can handle the interrupt and then communicate data to the user space driver through other mechanisms like shared memory
  - \- This may be necessary for devices which have very fast interrupts

## 15. **User Space Application Interrupt Processing Example | 用户空间应用程序中断处理示例**

```
int pending = 0;
int reenable = 1;

/*
 * 1. The UIO device is opened as previously described
 */
int uio_fd = open("/dev/uio0", O_RDWR);

/*
 * 2. Read the UIO device file descriptor to wait for an interrupt,
 *    the read blocks by default, a non blocking read can also be used
 *
 *    NOTE: The pending variable contains the number of interrupts that have
 *          occurred if multiple
 */
read(uio_fd, (void *)&pending, sizeof(int));


//
// add device specific processing like acking the interrupt in the device here
//


/*
 * 3. Re-enable the interrupt at the interrupt controller level
 */
write(uio_fd, (void *)&reenable, sizeof(int));
```

**Part II: Advanced UIO With Both User Space Application and Kernel Space Driver**

## **16. UIO Driver Details | UIO驱动详解**

![img](https://images2018.cnblogs.com/blog/1264595/201711/1264595-20171127103429581-1922470363.png)

- The user provides a kernel driver and a user space driver
- The kernel space driver is a platform driver configuring from the device tree and registering a UIO device
- The kernel space driver can also provide an interrupt handler in kernel space
- The user space driver has direct access to the hardware

## **17. Kernel UIO API - Basics | 内核UIO API基础**

- The API is small and simple to use API小且易用

```
struct uio_info
-- name      : device name
-- version   : device driver version
-- irq       : interrupt number
-- irq_flags : flags for request_irq()
-- handler   : driver irq handler (optional)
-- mem[]     : memory regions that can be mapped to user space
   o    addr : memory address
   o memtype : type of memory region (physical, logical, virtual)
```

## **18. Kernel UIO API - Registration | 内核UIO API - 注册**

- The function

   

  uio_register_device()

   

  connects the driver to the UIO framework

  - Requires a struct uio_info as an input
  - Typically called from the probe() function of a platform device driver
  - Creates device file /dev/uio# (#starting from 0) and all associated sysfs file attributes

- The function uio_unregister_device() disconnects the driver from the UIO framework

  - Typically called from the cleanup function of a platform device driver
  - Deletes the device file /dev/uio#

## **19. Kernel Space Driver Example | 内核空间驱动示例**

```
probe()
{
    /*
     * 1. Platform device driver initialization in the driver probe() function
     */
    dev = devm_kzalloc(&pdev->dev, (sizeof(struct uio_timer_dev)), GFP_KERNEL);
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    dev->regs = devm_ioremap_resource(&pdev->dev, res);
    irq = platform_get_irq(pdev, 0);

    /*
     * 2. Add basic UIO structure initialization
     */
    dev->uio_info.name = "uio_timer";
    dev->uio_info.version = 1;
    dev->uio_info.priv = dev;

    /*
     * 3. Add the memory region initialization for the UIO
     */
    dev->uio_info.mem[0].name = "registers";
    dev->uio_info.mem[0].addr = res->start;
    dev->uio_info.mem[0].size = resource_size(res);
    dev->uio_info.mem[0].memtype = UIO_MEM_PHYS;

    /*
     * 4. Add the interrupt initialization for the UIO
     */
    dev->uio_info.irq = irq;
    dev->uio_info.handler = uio_irq_handler;

    /*
     * 5. Register the UIO device with the kernel framework
     */
    uio_register_device(&pdev->dev, &dev->info);
}
```

## **20. UIO Framework Details | UIO框架详解**

- UIO Driver
  - \- The device tree node for the device can use whatever you want in the compatible property as it only has to match what is used in the kernel space driver as with any platform device driver
- UIO Platform Device Driver
  - \- The device tree node for the device needs to use "generic - uio" in it's compatible property

 

**参考资料**

- [Linux User Space Device Drivers](http://blog.idv-tech.com/wp-content/uploads/2014/09/drivers-session3-uio-4public.pdf)
- [UIO: user-space drivers](https://lwn.net/Articles/232575/)
- [Howto: Accessing PCI devices from userspace](https://github.com/rumpkernel/wiki/wiki/Howto:-Accessing-PCI-devices-from-userspace)
- [Kernel space: the UIO interface for device drivers](https://www.networkworld.com/article/2299025/software/kernel-space--the-uio-interface-for-device-drivers.html)
- Paper: [Userspace I/O drivers in a realtime context](https://www.osadl.org/fileadmin/dam/rtlws/12/Koch.pdf)







**转载自：**  **作者** [vlhn](https://home.cnblogs.com/u/vlhn/)    **本文链接**https://www.cnblogs.com/vlhn/p/7761869.html
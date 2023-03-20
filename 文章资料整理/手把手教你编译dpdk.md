# 手把手教你编译dpdk

## **1.源码准备**



下载dpdk-20.11.tar.xz源码包

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIFHm7ES3I6VGJFOd3QfIP8RR54npKAp6LSvj7ft3AMAOrfO2HgcFLdiaibtkMyCGmYLDiaCRy4ANcyA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

采用如下命令命令进行解压

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIFHm7ES3I6VGJFOd3QfIP8iawFQrtstcutn8xGUFmISsPp1CVFeZU2z8pgAuIQphbkWJbz3xmzulQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

解压后用ls命令再确认一下

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIFHm7ES3I6VGJFOd3QfIP8qD674ZCRIYibQnpOKTHLMREPJnMV1IXmibAJZvtjDDMF3QhNe9Vydgqw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

发现多了一个dpdk-20.11文件夹，我们进去瞅瞅他的目录结构，发现该有的都有啦。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIFHm7ES3I6VGJFOd3QfIP8MicwL02XDokvjaqlhEoUMwve65uxxT2QkpZPJYHmuBee3NnPV24lT1g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

稍微展开介绍一下几个重要目录：

- lib：DPDK库的源代码
- drivers: DPDK 查询模式驱动源码
- app: DPDK应用源码（自动化测试）
- examples: DPDK应用示例程序源码
- config,buildtools: 框架相关的脚本和配置



## **2.编译安装DPDK**

**
**

1. 进入dpdk-20.11目录执行meson -Dexamples=helloworld  build命令并回车，如下图红色框所示。如对meson命令不熟悉可以瞅瞅[编译工具meson+ninja简介（dpdk编译工具）](http://mp.weixin.qq.com/s?__biz=MzU4Mzc0NTcwNw==&mid=2247490528&idx=1&sn=eb463ee54107c3d43adfc028aa200c56&chksm=fda53224cad2bb32493a7d39e93d9b75155d7d23495a911047b3b7aa2c3d379ae360eda08754&scene=21#wechat_redirect)

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7YODXvia4mbPKfiaQleKPmnFmPpnk2AtGNWEgibTaS11dgEsXylZQexhVQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   稍等片刻出现下图所示内容，我们用ls查看发现多出来一个buid文件夹，没错，接下来的故事在build文件夹内展开。

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7tVWC3QS8TeI1TNsQ4m5WrWmPvOmgIqXY9w40CvMyia2KMCfOJ8NqRQg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

2. 进入build文件夹ls命令查看一下上一个命令生成的内容，接着依次执行ninja-build命令、ninja-build install命令，如对ninja相关命令比较陌生，可以瞅瞅[编译工具meson+ninja简介（dpdk编译工具）](http://mp.weixin.qq.com/s?__biz=MzU4Mzc0NTcwNw==&mid=2247490528&idx=1&sn=eb463ee54107c3d43adfc028aa200c56&chksm=fda53224cad2bb32493a7d39e93d9b75155d7d23495a911047b3b7aa2c3d379ae360eda08754&scene=21#wechat_redirect)。执行以上命令后效果如下图。

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7RYcapqiciay4dm2wVwkz1nz5tQY792oQJghnxSSLOtblAdEd5MkcMntw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

3. ninja install命令会拷贝编译后的目标文件到系统范围内可执行的地方，最后执行以下ldconfig命令使ld.so更新它的cache，以便包含刚才生成的目标文件。执行完成以上三步，恭喜你，dpdk就算安装完成了。

   

## **3.测试验证**

 **
**

前面执行meson -Dexamples=helloworld  build这条命令时不知道大家是否 对-Dexamples=helloworld的用意已理解，这个参数就是告诉meson编译DPDK的同时编译helloworld例程。所以我们可以通过运行这个例程来检验dpdk是否能正常工作。



在运行DPDK应用前咱一般还需要干几件事儿：

1. 设置Hugepages

   Hugepages是DPDK用于提升性能的重要手段。通过使用Hugepages，可以降低内存页数，减少TLB页表数量，增加TLB hit率。在Linux上设置Hugepages有两种方式:

   **修改Kernel cmdline(推荐)**

   **修改sysfs节点**

   ###### a、修改Kernel cmdline(推荐)

   ###### 通过修改kernel command line可以在kernel初始化时传入Hugepages相关参数并进行配置。具体的操作步骤如下：

   ###### **修改grub文件**

   修改`/etc/default/grub`文件，在`GRUB_CMDLINE_LINUX`中加入如下配置:

   default_hugepagesz=1G hugepagesz=1G hugepages=4

   ```
   
   ```

   该配置表示默认的hugepages的大小为1G，设置的hugepages大小为1G，hugepages的页数为4页，即以4个1G页面的形式保留4G的hugepages内存

   修改后的grub文件示例如下：

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw78UVUrRGusZLkZ5TemDWTBYkXfLq6z6iba0kef4dgICOvVFcTJoiby7zg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   **编译grub配置文件**

   可以通过命令`grub2-mkconfig -o /boot/grub2/grub.cfg`

   `![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw783L7M0AoIqE1aic54xodNfVdiaGm5zNPtvib7vPBhsXgRGlG54IaiclaKA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)`

   **重启**
   通过`reboot`命令重启，随后可以通过`cat /proc/cmdline`查看kernel的command line是否包含之前的配置。

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7McvO3x3483HIwTEqExY6XpfET1SE6e2icVATFg6zOsg7EAfZDfsJRQg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
   也可以通过`cat /proc/meminfo | grep Huge`命令查看是否设置成功，若设置成功可以看到如下配置:

   `![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7utwxDCws1D6VZ8QgkP4wcXynRaSrCtdWKWGUj9ia0xaldrerKDRUpZQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)`

   DPDK官方建议，64位的应用应配置**1GB** hugepages。

   这种配置方式的优点是可以在系统开机时即配置预留好hugepages，避免系统运行起来后产生内存碎片；另外，对于较大的例如1G pages，是不支持在系统运行起来后配置的，只能通过kernel cmdline的方式进行配置centos7.7，大页为1G 可动态调

   注：对于双插槽的NUMA系统（dual-socket NUMA system），预留的hugepages会被均分至两个socket，可以通过`lscpu`查看CPU信息：

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw763bHQR9nZHLszg5S92ia6zCibbg85OaJMrE9phI0ULsugbsicFa06eImw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   ```
   
   ```

   可见测试主机使用的是双socket的NUMA系统。

   ###### b、修改sysfs节点

   对于2 MB页面，还可以选择在系统启动后进行分配。这是通过修改 `/sys/devices/`中的`nr_hugepages`节点来实现的。对于单节点系统，若需要1024个页面，可使用如下命令：

   ```
   
   ```

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7kuQnqs7txNSibWXb3hjSiaLPtRZndhhibR8ht2x4ZWJ8VbDNYEa9OG35A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   在NUMA机器上，页面的需要明确分配在不同的node上（若只有一个node只需要分配一次）：

   ```
   
   ```

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7soicZWjrRCUXdNlOBU1x05Mm1geVmHprxmbrV6AyMticg0ZZRXlO0YQw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   这种配置方式的优点是配置简单，无需编译、重启，但是无法配置1GB这样的大hugepages。

   ###### **DPDK使用Hugepages**

   预留好Hugepages之后，想要让DPDK使用预留的Hugepages，需要进行下述操作:

   ```
   
   ```

   可以将这个挂载点添加到`/etc/fstab`中，这样可以永久生效，即重启后也仍然可以生效:

   ```
   
   ```

   对于1GB pages，在`/etc/fstab`中必须指定page size作为mount选项。添加上面这样一行内容至`/etc/fstab`后并重启，可以通过`df -a`看到挂载成功：

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7wSo2Zz87S3xY0XHskmibKaX33ySEKXZ5JXwu49aT8KkQ4F4fd6F8Grw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

2. 加载内核驱动

   DPDK的驱动可以选择UIO也可以选择VFIO,VFIO安全性性能都比UIO好，不过要求肯定也比UIO高，所以一般的建议是有条件尽量用VFIO，用不了再用UIO。

   

   ### 3.1**UIO**

   UIO（Userspace I/O）是运行在用户空间的I/O技术。Linux系统中一般的驱动设备都是运行在内核空间，而在用户空间用应用程序调用即可。
   而UIO则是将驱动的很少一部分运行在内核空间，而在用户空间实现驱动的绝大多数功能。
   在许多情况下，Linux内核中包含的标准uio_pci_generic模块可以提供uio功能。可以使用以下命令加载此模块：

   ```
   sudo modprobe uio_pci_generic
   ```

   除了Linux内核中包含的标准uio_pci_generic模块，DPDK也提供了一个可替代的igb_uio模块，可以在`kmod`路径中找到。可以通过以下方法加载igb_uio模块。

   ```
   sudo modprobe uio
   sudo insmod kmod/igb_uio.ko
   ```

   如果用于DPDK的设备绑定为uio_pci_generic内核模块，需要确保IOMMU已禁用或passthrough。以intel x86_64系统为例，可以在的`GRUB_CMDLINE_LINUX`中添加`intel_iommu = off`或`intel_iommu = on iommu = pt`。

   

   ### 3.2**VFIO（推荐）**

   VFIO与UIO相比，它更加强大和安全，依赖于IOMMU。要使用VFIO，需要:

   Linix kernel version>=3.6.0

   内核和BIOS必须支持并配置为使用IO virtualization（例如Intel®VT-d）在确认硬件配置支持的情况下，要使用VFIO驱动绑定到NIC必须先使能iommu，否则会导致绑定失败。具体的现象就是查看或修改sysfs节点`/sys/bus/pci/drivers/vfio-pci/bind`出现io错误，以及dmesg中出现：

   ```
   vfio-pci: probe of 0000:05:00.0 failed with error -22
   ```

   使能iommu的方法也是修改kernel的command line将`iommu=pt intel_iommu=on`传入，具体步骤：

   a、修改grub文件
   修改`/etc/default/grub`文件，在`GRUB_CMDLINE_LINUX`中加入如下配置:iommu=pt intel_iommu=on

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7H7iaeJIFAqibxNjW1R3BKZjdXxJg5zQZ1dLAJ8peeoCkOE8snB4Ebw9A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   ```
   
   ```

   b、编译grub配置文件
   可以通过命令`grub2-mkconfig -o /boot/grub2/grub.cfg`

   c、重启
   通过`reboot`命令重启，随后可以通过`cat /proc/cmdline`查看kernel的command line是否包含之前的配置。

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw72lcCK4N6Dcp4Safc3U8bVUlHmOzYibMz1cziaNU491OstBhQEJFLJcKA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   d、iommu配置成功后，dmesg中会有iommu配置group的log，可以通过`dmesg | grep iommu`查看：

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw76siciaBwFleyicDbw3VMZDqeAeibYZnANn0uoAzxOSvRdJgA6mOV2n7tow/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   并没有出现类似如下iommu的device加入组的相关信息。

   `[0.594500] iommu: Adding device 0000:05:00.0 to group 18`

   ```
   [0.594512] iommu: Adding device 0000:06:00.0 to group 19
   ```

   感觉不太妙，不知vfio能否用的起来。
   随后需要调用`modprobe`来加载VFIO的驱动:

   sudo modprobe vfio-pci

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7jRLoAgIMvZUAwb9qoJNtpFc4GfiaVjLiay9jTXRo22AedaCnhUibF8Djg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   lsmod | grep vfio一下，可以看到vfio驱动已经挂载。

   为了保险起见，我们把uio的也挂载上吧，如下：

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7ErRV8okiba2PywYOVkvG9gW3icM6iccf8K9xPFmvBrnTuQYhAknkzBKug/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   现在uio和vfio的驱动都已成功挂载。

   

3. 应用程序使用的端口应绑定到相应的内核驱动程序（可选）

   上面的UIO和VFIO驱动可以加载一项，也可以全都加载。但是在驱动绑定NIC的时候，只能选择一种驱动绑定到NIC，UIO和VFIO该如何选可以回看[UIO,IOMMU,VFIO傻傻分不清楚](http://mp.weixin.qq.com/s?__biz=MzU4Mzc0NTcwNw==&mid=2247490374&idx=1&sn=55e632a3896b2932fcdfac5beb0c8f53&chksm=fda53282cad2bb94977825a2c60f3eab7793a60baa0d74c06e58ab6b832300698f67b0ed52dc&scene=21#wechat_redirect)，我们先尝试采用VFIO驱动。可以调用dpdk路径下的`usertools/dpdk-devbind.py`实用脚本来进行VFIO驱动与NIC绑定，需要注意的是使用这个脚本进行绑定（bind）动作时是需要root权限的。可以调用脚本传入`--status`查看当前的网络端口的状态：

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7HppG2FScasnkAN3brDDRolXwfJJbpfTtGwLvsZuibdzJwOPNRvU22Wg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   可以看到，当前NIC的状态都是`Network devices using kernel driver。`随后可以调用脚本传入`--bind`将网卡`0b:00.0`，也就是`ens192`绑定到VFIO驱动：

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7y1V0FPQam3GYrCQfiab6lPIGpH4FddXgZPKsja1QVozF2yNa3TTTfJg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   结果报错了，看来咱今天演示用的机器是支持不了vfio了，那咱们就只能上uio了，具体如下：

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7bNFAN0yvGlnR1p3bKp14iaKicyxxg0wOTrwvaLU5xKqz827YdS88Ziaog/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   再次调用脚本传入`--status`，可以从上图看到设备`0b:00.0`已经配置为`drv=uio_pci_generic了。`

   若想要恢复为kernel默认的vmxnet3驱动，则可以继续调用脚本：

   python usertools/dpdk-devbind.py --bind=vmxnet3 0b:00.0

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7oAqG9Tk6icBEsibP0xRYVT6nEsYKDDM3d4wRbYYG9JqjFFPXcYGNl97w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

   不过为了今后的实验，咱还是把0b:00.0设置为uio_pci_generic吧。接下来咱们用前面编译好的helloworld例程看看效果怎么样，具体如下：

   输入红色框命令并回车，接下来例程期望的效果真的出现了，惊不惊喜，意不意外~

   ![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVIvUCAR3sREW64VZPmfkBw7ibDFEkyoIHCjqoaM1hUGZ5oLOu7dGXNECz9xY1C6POba9Ix5VibmUJDA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

原文链接：https://mp.weixin.qq.com/s/kVUFW3yngpFsY5n-bBhnmA
原文作者：扫地僧666


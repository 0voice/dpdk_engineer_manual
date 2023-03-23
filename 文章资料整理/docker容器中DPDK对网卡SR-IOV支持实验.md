# docker容器中DPDK对网卡SR-IOV支持实验

主要是验证下容器运行DPDK，如何对SR-IOV进行支持

### **1.** **VF创建方法**

在未开启SR-IOV时，通过DPDK提供的./dpdk-devbind.py脚本可知，当前系统一共有一块82599网卡，拥有2个网口，PCI的地址是18:00.0和18:00.1,如下图所示

![img](https://pic4.zhimg.com/80/v2-1013bcee35d0626bd19c4bd644638b5b_720w.webp)

启用命令

Rmmod ixgbe

modprobe ixgbe max_vfs=2 开启2个VF

启用命令

ip link set enp24s0f0 vf 0 mac aa:bb:cc:11:22:30

ip link set enp24s0f0 vf 1 mac aa:bb:cc:11:22:31

设定两个VF0和VF1与enp24s0f0，此时就可以看到两个VF网卡的存在

![img](https://pic3.zhimg.com/80/v2-d228b74cee0414ed85faf801818400c6_720w.webp)



### **2. Docker使用DPDK遇到的主要问题**

1）Docker没有自己的文件系统，谈不上插入uio模块

2）Docker中没有自己的大页内存

3）Docker中运行dpdk应用程序，如果使用直通模式，要想办法

解决办法：

在主机中完成DPDK的初始化工作，并把相关的资源map挂载到容器内部



Docker使用-v命令，将可以将主机上的资源挂在到容器内部

所有的网卡在linux系统中，实际上是一个文件

大页内存在linux系统中，实际上也是一个文件

而用户态驱动只要能读取到对应的PCI地址，获取网卡寄存器访问的能力，即可以完成SR-IOV的功能。

当容器需要通过DPDK访问vf网卡时，需要将这些文件（设备，内存，主机的igb_uio等），都挂在到容器的内部。

使用方法如下

启动时添加

```text
--privileged （增加权限）
-v /sys/bus/pci/devices:/sys/bus/pci/devices
 （把网卡挂到容器里面，容器内部可以读到sr-iov的VF网卡的PCI地址）
-v /sys/kernel/mm/hugepages:/sys/kernel/mm/hugepages
（大页内存挂载到容器内部） 
-v /sys/devices/system/node:/sys/devices/system/node
（numa相关的节点信息挂到容器内部） 
-v /dev:/dev 
（主机上的igb_uio挂到容器内部）
```

启动容器后可以在/sys/bus/pci/devices下读到对应的pci地址

![img](https://pic4.zhimg.com/80/v2-38ab0e17fc36d1b7b3b770e0ff27866f_720w.webp)

以dpdk自带的basicfwd测试程序测试，启动方式为

./build/basicfwd -w 18:10.2 -- -p 0x1

-w后18:10:2是虚拟的VF的PCI地址,可以看到程序正常启动，并读到了MAC地址

### **3. 测试与验证**

测试的整体的结构如下图所示：

![img](https://pic2.zhimg.com/80/v2-d5f32ae456b97b1e55bcb4483b7690e1_720w.webp)

发送两种MAC的数据包，结果显示2个VF可以256B数据包达到9.91Gbps的转发速率，接近线速。混合发送时，两个VF加起来差不多也是接近网卡线速（82599 10G），可以看到所有发送出去的数据包都转发了回来。

### 4. **结论与部分问题**

本质上，DPDK大部分的东西都还是放到了主机上完成初始化，容器内的程序只是调用这些初始化好的资源，因此这部分可以只放一个DPDK应用程序即可

如果容器内启动时报

eth_ixgbevf_dev_init(): VF Initialization Failure:

说明主机的PF网卡没能启动

## **5. 相关文献**

1： [https://forum.huawei.com/enterprise/zh/thread-161073-1-1.html](https://link.zhihu.com/?target=https%3A//forum.huawei.com/enterprise/zh/thread-161073-1-1.html) 华为服务器如何开启支持SR-IOV支持

2：82599 datasheet

3：dpdk官方testpmd关于vf状态测试

[http://doc.dpdk.org/guides/tes](https://link.zhihu.com/?target=http%3A//doc.dpdk.org/guides/testpmd_app_ug/testpmd_funcs.html%23show-vf-stats)





原文链接：https://zhuanlan.zhihu.com/p/508583116  原文作者：张张


# DPDK KNI介绍

## 1.DPDK KNI

KNI = Kernel NIC Interface 可以通过KNI与内核交互数据包。一些不便于用户态的处理的包可以通过KNI交于内核处理。



## 2.DPDK KNI 优势

比 linux TUN/TAP接口更快，减少拷贝
让DPDK接口可以使用标准linux网络工具如ifconfig, ethtool, tcpdump等
让接口接入内核网络协议栈

ifconfig/ethtool等管理工具通过IOCTL sockfd管理KNI接口；
普通用户态APP可以通过sendto/recv 访问KNI接口；
DPDK应用通过共享内存队列与KNI接口交互数据，通过IOCTL访问/dev/kni设备来注册KNI接口。![image-20221217204543229](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221217204543229.png)

## 3.KNI 内核模块

rte_kni模块提供了DPDK应用的内核接口。
rte_kni模块加载进内核以后，会创建/dev/kni设备，DPDK KNI API使用/dev/kni与内核通信。

### 3.1rte_kni模块参数说明

```c++
lo_mode： KNI loopback mode (default=lo_mode_none)
	lo_mode_none Kernel loopback disabled
	lo_mode_fifo Enable kernel loopback with fifo
	lo_mode_fifo_skb loopback with fifo and skb(charp)

kthread_mode: Kernel thread mode(default=single)
	single 内核单线程
	multiple 内核多线程(charp)

carrier: KNI接口的carrier默认状态(defailt=off)
	off  carrier state off
	on   carrier state on

```

rte_kni加载不带任何参数是最典型的使用方式，没有参数，单线程，[loopback](https://so.csdn.net/so/search?q=loopback&spm=1001.2101.3001.7020)关闭，carrier状态off。

### 3.2loopback模式

```c++
# insmod kmod/rte_kni.ko lo_mode=lo_mode_fifo
lo_mode_fifo 在内核中回环入队和出队操作

# insmod kmod/rte_kni.ko lo_mode=lo_mode_fifo_skb
lo_mode_fifo_skb 在内核中回环入队和出队操作和sk buffer的拷贝

```

### 3.3内核线程模式

```c++
# insmod kmod/rte_kni.ko kthread_mode=single

```

该模式创建一个内核线程为所有的KNI接口接收数据。
默认情况下，这个线程没有绑定核心，但是可以创建KNI接口时，通过struct rte_kni_conf中的core_id和force_bind字段为其绑定核心。

### 3.4carrier状态

carrier状态可以通过加载rte_kni参数指定，也可以DPDK APP通过rte_kni_update()中设置。
carrier=off时， KNI接口反映出真是物理网卡的链路状态
carrier=on时， KNI接口作为纯虚拟接口，可以和loopback模式一起使用

```c++
# insmod kmod/rte_kni.ko carrier=on
# insmod kmod/rte_kni.ko carrier=off

```

## 4.KNI创建和删除

首先rte_kni模块已经加载进内核，rte_kni_init()函数初始化配置。
DPDK应用通过rte_kni_alloc()函数动态创建KNI接口、
struct rte_kni_conf结构包含了接口名字，mtu大小，MAC地址，CPU亲和性字段。
struct rte_kni_ops结构包含了一组函数指针，用来是处理rte_kni模块发来的请求。
当KNI接口被外部命令或者外部应用操作的时候，这些操作会让rte_kni模块发出通知，DPDK应用通过rte_kni_ops中的函数处理这些通知，采取相应的操作。



原文链接：https://blog.csdn.net/jacicson1987/article/details/125909677
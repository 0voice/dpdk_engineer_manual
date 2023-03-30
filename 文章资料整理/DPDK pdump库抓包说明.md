# DPDK pdump库抓包说明 

## 1．环境与编译

pdump库是在16.07版本引入的，提供了一个抓包调试功能。在`$(RTE_SDK)/app`目录下就有一个`dpdk-pdump`的工具。配置这个这个工具可以用于抓取指定接口、队列的数据包。

### 1.1 库及依赖

Pdump依赖于libpcap库及`libpcap-dev`等相关库，要预先安装。

### 1.2 编译选项

Pdump依赖于基于libpcap的PMD驱动，需要开启两个设置，来生成运行pdump工具

1. `CONFIG_RTE_LIBRTE_PMD_PCAP=y ($(RTE_SDK)/config/common_base文件)`
2. `CONFIG_RTE_LIBRTE_PDUMP=y ($(RTE_SDK)/config/common_base文件)`

### 1.3 编译`dpdk-pdump`

这里以17.02版本为例说明。按照官方文档，在`$(RTE_SDK)`目录下，

1. 设置编译的目录 `export RTE_SDK=XXX` ,XXX就是dpdk的源码包的目录
2. 设置编译后的安装目录（主要就是拷贝生成的库，头文件等） `export DESTDIR=XXX`，这个安装目录可以自行设置。推荐自己建一个目录，好找就行，生成的pdump工具就在这个目录里。
3. 执行编译安装 `make install T=x86_64-native-linuxapp-gcc`；这里的T是指编译的TARGET，根据机器和编译器选择组合，具体可参考官方文档——`《Getting started Guide For Linux》`

## 2. 测试与使用

在编译安装后，就会在安装目录的bin目录中发现`dpdk-pdump`可执行文件。可以拷贝出来后续运行。

### 2.1 运行原理

`dpdk-pdump`使用时，作为secondary进程依附于primary进程。primary进程中启动server端，初始化pdump抓包框架任务；`dpdk-pdump`进程是作为client端向primary进程发送开始/停止抓包请求，然后primary进程拷贝一份数据包到ring中，secondary进程从ring中读取出来，并保存为pcap文件。因此，可以看出在primary进程中需要初始化pdump server。

### 2.2 简单示例

1. 在示例中，使用l3fd来当primary进程，但是需要做些小修改，在`rte_eal_init()`后，初始化pdump框架，添加如下代码：

```c
#ifdef RTE_LIBRTE_PDUMP
	/* initialize packet capture framework */
	rte_pdump_init(NULL);
#endif
然后编译l3fd生成l3fd可执行文件，就以官方例子参数运行。
```

1. 运行`dpdk-pdump`，作为secondary进程，依附于前面启动的l3fd。执行如下参数命令: `./dpdk-pdump -- --pdump 'port=0,queue=*,rx-dev=/tmp/rx.pcap'`
   这里需要注意port的取值，一定是DPDK绑定的网卡，如绑定了3张网卡，那port取值范围就是0-2,对应于每个网卡。自然，也可以使用PCI号来传参，指明抓哪个网卡。`Dev=/tmp/rx.pcap`就指明了最后抓的包存放的路径。

更详细的`dpdp-pdump`运行参数可以根据情况设置，格式如下：

```c
usage: %s [EAL options] -- --pdump "
			"'(port=<port id> | device_id=<pci id or vdev name>),"
			"(queue=<queue_id>),"
			"(rx-dev=<iface or pcap file> |"
			" tx-dev=<iface or pcap file>,"
			"[ring-size=<ring size>default:16384],"
			"[mbuf-size=<mbuf data size>default:2176],"
			"[total-num-mbufs=<number of mbufs>default:65535]'\n"
			"[--server-socket-path=<server socket dir>"
				"default:/var/run/.dpdk/ (or) ~/.dpdk/]\n"
			"[--client-socket-path=<client socket dir>"
				"default:/var/run/.dpdk/ (or) ~/.dpdk/]\n"
注意：参数中注意单引号那个字符，还有指定server-socket-path的路径时的”\n”字符。
 
```







转载自：   作者 [AISEED](https://home.cnblogs.com/u/yhp-smarthome/)    本文链接 https://www.cnblogs.com/yhp-smarthome/p/7102557.html
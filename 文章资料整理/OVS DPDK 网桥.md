# OVS DPDK 网桥

DPDK数据路径需要特殊配置的网桥才能使用DPDK支持的physical <phy>端口和virtual <vhost-user>端口。

## 1.入门示例

此示例演示如何使用为DPDP数据路径添加网桥：

```text
 $ ovs-vsctl add-br br0 -- set bridge br0 datapath_type=netdev
```

这假设Open vSwitch是带有DPDK支持选项编译的。请参阅OVS文档/intro/install/dpdk了解更多信息。

```text
扩展及自定义统计信息
```

DPDK扩展统计API接口允许PMDs驱动公开一组一致的统计数据信息。扩展的统计信息仅实现在DPDK物理端口和vHost端口。自定义统计是一组动态计数器，具体取决于PMD驱动程序。这些统计数据用于DPDK物理端口，包含来自XSTATS的所有丢弃、错误和管理计数器。有关所有XSTATS计数器的列表可参见[https://wiki.opnfv.org/display/fastpath/Collectd+Metrics+and+Events].

## 2.注意

vHost端口仅支持基于RX数据包大小的计数器。TX数据包大小计数器不可用。

要启用统计信息，必须启用OpenFlow 1.4对OVS的支持。要配置一个名为br0的网桥, 支持OpenFlow 1.4版本，运行如下命令：

```text
$ ovs-vsctl set bridge br0 datapath_type=netdev \
 protocols=OpenFlow10,OpenFlow11,OpenFlow12,OpenFlow13,OpenFlow14
```

配置完成后，检查网桥表中的OVSDB 协议列以确保OpenFlow 1.4支持已启用：

```text
$ ovsdb-client dump Bridge protocols
```

你还可以在查询端口统计信息时，显式指定-O OpenFlow14选项：

```text
$ ovs-ofctl -O OpenFlow14 dump-ports br0
```

EMC插入概率

默认情况下，每100个流中有1个插入到精确匹配缓存（EMC）中。可以通过设置emc-insert-inv-prob选项改变此插入概率:

```text
$ ovs-vsctl --no-wait set Open_vSwitch . other_config:emc-insert-inv-prob=N
```

where:

N

正整数表示插入概率的逆，即在每N个流中产生一次EMC插入。

如果N设置为1，则将对每个流执行插入。如果设置为0，将不执行任何插入操作，并且，EMC功能实际上被禁用。

由于默认的N值为100，最初将产生较高的megaflow命中量，可通过PMD统计观察到：

```text
$ ovs-appctl dpif-netdev/pmd-stats-show
```

对于具有许多并行流的特定状况，建议设置N等于0，以获得更高的转发性能。

还可以使用以下方法在每个端口上启用/禁用EMC：

```text
$ ovs-vsctl set interface <iface> other_config:emc-enable={true,false}
```

## 3.注意

这对于在不同的端口上预期有不同数量流量的情况很有用。例如，如果其中一个虚拟机使用附加头部封装流量，并且它将接收大量数据流，但只有少量流从这个虚拟机中出来。在这种情况下，使用EMC要快得多而不是对来自虚拟机的流量应用分类器，但是最好对流向虚拟机的流量禁用EMC。

更多信息参见OVS文档/intro/install/dpdk 中关于EMC的内容.

## 4.SMC 缓存 (实验性质)

SMC缓存或签名匹配缓存是继EMC缓存之后的一个新的缓存级别。SMC和EMC的区别在于SMC只存储流的签名，因此它的内存效率要高得多。使用相同的内存空间，EMC可以存储8K流，而SMC可以存储1M流。当流量大于EMC的大小时，通常关闭EMC并打开SMC是有益的。当前SMC默认情况下是关闭的，并且还是一个实验功能。

开启SMC:

```text
$ ovs-vsctl --no-wait set Open_vSwitch . other_config:smc-enable=true
```

原文链接：[https://blog.csdn.net/sinat_20184565/article/details/93422498](https://link.zhihu.com/?target=https%3A//blog.csdn.net/sinat_20184565/article/details/93422498) 原文作者：redwingz
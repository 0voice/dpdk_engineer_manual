# NUMA体系结构详解

由于OpenStack Kilo增加很多针对NUMA体系结构的增强功能，所以又重新温习了下NUMA相关的知识，简单做个笔记。

## 1.NUMA的几个概念（Node，socket，core，thread）

对于socket，core和thread会有不少文章介绍，这里简单说一下，具体参见下图：

![img](https://img-blog.csdn.net/20150512112954867?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdXN0Y19keWxhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

一句话总结：socket就是主板上的CPU插槽; Core就是socket里独立的一组程序执行的硬件单元，比如寄存器，计算单元等; Thread：就是超线程hyperthread的概念，逻辑的执行单元，独立的执行上下文，但是共享core内的寄存器和计算单元。

NUMA体系结构中多了Node的概念，这个概念其实是用来解决core的分组的问题，具体参见下图来理解（图中的OS CPU可以理解thread，那么core就没有在图中画出），从图中可以看出每个Socket里有两个node，共有4个socket，每个socket 2个node，每个node中有8个thread，总共4（Socket）× 2（Node）× 8 （4core × 2 Thread） = 64个thread。

另外每个node有自己的内部CPU，总线和内存，同时还可以访问其他node内的内存，NUMA的最大的优势就是可以方便的增加CPU的数量，因为Node内有自己内部总线，所以增加CPU数量可以通过增加Node的数目来实现，如果单纯的增加CPU的数量，会对总线造成很大的压力，所以UMA结构不可能支持很多的核。

​                 《此图出自：NUMA Best Practices for Dell PowerEdge 12th Generation Servers》

![img](https://img-blog.csdnimg.cn/2022010620252810580.png)

根据上面提到的，由于每个node内部有自己的CPU总线和内存，所以如果一个虚拟机的vCPU跨不同的Node的话，就会导致一个node中的CPU去访问另外一个node中的内存的情况，这就导致内存访问延迟的增加。在有些特殊场景下，比如NFV环境中，对性能有比较高的要求，就非常需要同一个虚拟机的vCPU尽量被分配到同一个Node中的pCPU上，所以在OpenStack的Kilo版本中增加了基于NUMA感知的虚拟机调度的特性。（OpenStack Kilo中NFV相关的功能具体参见：《[OpenStack Kilo新特性解读和分析（1）](http://blog.csdn.net/ustc_dylan/article/details/45672649)》）

## **2.如何查看机器的NUMA拓扑结构**

比较常用的命令就是lscpu，具体输出如下：

```c++
dylan@hp3000:~$ lscpu 
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                48                                       //共有48个逻辑CPU（threads）
On-line CPU(s) list:   0-47
Thread(s) per core:    2                               //每个core有2个threads
Core(s) per socket:    6                                //每个socket有6个cores
Socket(s):             4                                      //共有4个sockets
NUMA node(s):          4                               //共有4个NUMA nodes
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 45
Stepping:              7
CPU MHz:               1200.000
BogoMIPS:              4790.83
Virtualization:        VT-x
L1d cache:             32K                           //L1 data cache 32k
L1i cache:             32K                            //L1 instruction cache 32k  （牛x机器表现，冯诺依曼+哈弗体系结构）
L2 cache:              256K
L3 cache:              15360K
NUMA node0 CPU(s):     0-5,24-29      
NUMA node1 CPU(s):     6-11,30-35
NUMA node2 CPU(s):     12-17,36-41
NUMA node3 CPU(s):     18-23,42-47
```

从上图输出，可以看出当前机器有4个sockets，每个sockets包含1个numa node，每个numa node中有6个cores，每个cores包含2个thread，所以总的threads数量=4（sockets）×1（node）×6（cores）×2（threads）=48.

另外，也可以通过下面的脚本来打印出当前机器的socket，core和thread的数量。

```c++
#!/bin/bash
 
# Simple print cpu topology
# Author: kodango
 
function get_nr_processor()
{
    grep '^processor' /proc/cpuinfo | wc -l
}
 
function get_nr_socket()
{
    grep 'physical id' /proc/cpuinfo | awk -F: '{
            print $2 | "sort -un"}' | wc -l
}
 
function get_nr_siblings()
{
    grep 'siblings' /proc/cpuinfo | awk -F: '{
            print $2 | "sort -un"}'
}
 
function get_nr_cores_of_socket()
{
    grep 'cpu cores' /proc/cpuinfo | awk -F: '{
            print $2 | "sort -un"}'
}
 
echo '===== CPU Topology Table ====='
echo
 
echo '+--------------+---------+-----------+'
echo '| Processor ID | Core ID | Socket ID |'
echo '+--------------+---------+-----------+'
 
while read line; do
    if [ -z "$line" ]; then
        printf '| %-12s | %-7s | %-9s |\n' $p_id $c_id $s_id
        echo '+--------------+---------+-----------+'
        continue
    fi
 
    if echo "$line" | grep -q "^processor"; then
        p_id=`echo "$line" | awk -F: '{print $2}' | tr -d ' '` 
    fi
 
    if echo "$line" | grep -q "^core id"; then
        c_id=`echo "$line" | awk -F: '{print $2}' | tr -d ' '` 
    fi
 
    if echo "$line" | grep -q "^physical id"; then
        s_id=`echo "$line" | awk -F: '{print $2}' | tr -d ' '` 
    fi
done < /proc/cpuinfo
 
echo
 
awk -F: '{ 
    if ($1 ~ /processor/) {
        gsub(/ /,"",$2);
        p_id=$2;
    } else if ($1 ~ /physical id/){
        gsub(/ /,"",$2);
        s_id=$2;
        arr[s_id]=arr[s_id] " " p_id
    }
} 
END{
    for (i in arr) 
        printf "Socket %s:%s\n", i, arr[i];
}' /proc/cpuinfo
 
echo
echo '===== CPU Info Summary ====='
echo
 
nr_processor=`get_nr_processor`
echo "Logical processors: $nr_processor"
 
nr_socket=`get_nr_socket`
echo "Physical socket: $nr_socket"
 
nr_siblings=`get_nr_siblings`
echo "Siblings in one socket: $nr_siblings"
 
nr_cores=`get_nr_cores_of_socket`
echo "Cores in one socket: $nr_cores"
 
let nr_cores*=nr_socket
echo "Cores in total: $nr_cores"
 
if [ "$nr_cores" = "$nr_processor" ]; then
    echo "Hyper-Threading: off"
else
    echo "Hyper-Threading: on"
fi
 
echo
echo '===== END ====='
```

————————————————————
email: ustc.dylan@gmail.com
微博：@Marshal-Liu

原创：[刘军卫](https://blog.csdn.net/ustc_dylan)  本文链接：https://blog.csdn.net/ustc_dylan/article/details/45667227
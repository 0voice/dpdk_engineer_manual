# DPDK基础：DDos攻击实验

## 1.实验概述&实验目的

分布式拒绝服务攻击(英文意思是Distributed Denial of Service，简称DDoS)是指处于不同位置的多个攻击者同时向一个或数个目标发动攻击，或者一个攻击者控制了位于不同位置的多台机器并利用这些机器对受害者同时实施攻击。由于攻击的发出点是分布在不同地方的，这类攻击称为分布式拒绝服务攻击，其中的攻击者可以有多个。

本实验利用10Gbps网卡前端的服务器来模拟僵尸网络所产生的流量。利用Trex进行发包来尝试逼近带宽的理论值，以达到模拟DDos攻击时服务器没有多余流量来为非攻击者使用的情况。

根据以太网帧的结构分析，帧的大小介于64bit和1518bit，本实验采用这两个极限值作为网络包的大小来进行流量监测。dpdk是绕过linux内核的网络发包环境，实验将采用控制变量法，对不同大小的报文用不同的核数进行流量监测，分析不同核数时的cpu利用率以及吞吐量（实验结果用图示表示，自变量为核数和包大小）。

带*为非必要操作

## 2.物理实验环境

服务器：DELL EMC 至强服务器

工作站：DELL 工作站

网卡：INTEL X710 10Gbps

## 3.软件环境配备

系统 Ubuntu Server 18.04 LTS，用户名：seu，密码：seu

### 3.1连接外网

服务器共有四个网口 eno1～eno4，将网线插在左数第一个网口对应 eno1，并配置 interfaces 文件

```c++
$ vi /etc/network/interfaces

auto eno1
iface eno1 inet dhcp

```

在`/etc/resolv.conf`中加入DNS配置

```c++
nameserver 223.5.5.5
nameserver 223.6.6.6

```

要先手动设置 IP 再使用 dhclient 命令启用 DHCP，**每次重启服务器后如需连接外网，需输入这两行命令**

```c++
ifconfig eno1 192.168.0.102 netmask 255.255.255.0
dhclient eno1

```

执行完成就可以连通外网了

### 3.2安装依赖

使用 `apt-get` 命令安装 dpdk 依赖

```c++
sudo apt update
sudo apt install -y dpdk dpdk-dev dpdk-doc

```

安装其他依赖，可以直接在本地保存为 `install.sh`，服务器开启 netcat 监听 `nc -l 8080 < install.sh`

然后本地使用 `nc $SERVER_IP 8080 > install.sh` 传到服务器上 `sh install.sh`执行

```c++
sudo apt-get update

sudo apt-get upgrade
 
sudo apt-get install -y cmake gcc g++ git automake llvm llvm-dev llvm-runtime libtool bison flex build-essential vim

# Install pkg-config here, as it is required for p4lang/PI
# installation to succeed.
sudo apt-get install -y pkg-config

sudo apt-get install -y wget curl zip unzip rar unrar unar

sudo apt-get install -y libgc-dev libfl-dev libgmp-dev libevent-dev libssl-dev libjudy-dev libpcap-dev tcpdump

sudo apt-get install -y libboost-dev libboost-iostreams-dev libboost-graph-dev libboost-test-dev libboost-program-options-dev libboost-system-dev libboost-filesystem-dev libboost-thread-dev

sudo apt-get install -y libreadline6 libreadline6-dev
# 这里如果提示废弃就执行下面的安装
sudo apt-get install -y libreadline-dev 

# Deps needed to build PI:
sudo apt-get install -y libjudy-dev libreadline-dev valgrind libtool-bin libboost-dev libboost-system-dev libboost-thread-dev

# Things needed for `cd tutorials/exercises/basic ; make run` to work:
sudo apt-get install -y libgflags-dev net-tools

sudo apt-get install -y doxygen graphviz texlive-full

sudo apt-get install -y bridge-utils tcpreplay
sudo apt-get install -y zlib1g-dev pciutils kmod strace  ## needed by cisco trex

```

### 3.3设置网口

```c++
服务器网卡的ip为：192.168.200.2 和192.168.201.2
工作站ip为：		192.168.0.101

```

服务器平台下：

`ifconfig -a`查看网口信息，列出了 X710 网卡的两个网口信息，默认驱动是 Kernel Driver i40e，先设置 IP

```c++
# ddos.py
import os
from trex_stl_lib.api import *
​
c = STLClient(server = "localhost")
​
try:
​
    ports = [0,1]
    c.connect()
    c.reset(ports)
​
    c.push_remote(pcap_filename = '/home/seu/Downloads/PCAP-01-12_0750-0818/SAT-01-12-2018_0750',
                  ports = [0,1],
                  ipg_usec = 0.1,
                  count = 100)
​
    c.wait_on_traffic()
​
​
    stats = c.get_stats()
    for port in ports:
        opackets = stats[port]['opackets']
        print("{0} packets were Tx on port {1}\n".format(opackets, port))
​
except STLError as e:
    print(e)
    sys.exit(1)
​
finally:
    c.disconnect()#cd /etc/v2.75
sudo ifconfig enp59s0f0 192.168.200.2
sudo ifconfig enp59s0f1 192.168.201.2

```

在工作站的主机上 ping 一下服务器保证双向联通

ping 192.168.200.2

ping 192.168.201.2

### 3.4安装TRex（）

将装有 TRex 压缩包的U盘插在服务器上，`fdisk -l`查看是哪个设备，并挂载设备

```c++
mount /dev/sdb1 /mnt
cd /mnt

```

将压缩包拷贝到 /home/seu 目录下使用 `tar -xzvf trex.tar`解压，版本是 v2.75

## 4.进行端口绑定

```c++
cd v2.75
sudo ./dpdk_setup_ports.py -i	

```

## 5.进行发包测试

服务器下执行：

```c++
sudo ./t-rex-64 -f avl/sfr_delay_10_1g.yaml -m 5 -l 1000

#delay_10_1g代表1Gbps
#参数m是发包重放次数，此处是5倍，为5Gpbs
#l 是网络抖动检测

```

显示的结果如下图：

```c++
-Per port stats table 

      ports |               0 |               1 
 -----------------------------------------------------------------------------------------

   opackets |        27882688 |        35434728 
     obytes |      8217921859 |     28586813888 
   ipackets |         5007937 |             709 
     ibytes |      4647534484 |           79608 
    ierrors |               0 |               0 
    oerrors |               0 |               0 
      Tx Bw |       1.18 Gbps |       3.81 Gbps 

-Global stats enabled 
 Cpu Utilization : 31.6  %  31.6 Gb/core 
 Platform_factor : 1.0  
 Total-Tx        :       5.00 Gbps  
 Total-Rx        :       0.00  bps  
 Total-PPS       :       1.07 Mpps  
 Total-CPS       :      20.51 Kcps  

 Expected-PPS    :       1.08 Mpps  
 Expected-CPS    :      20.61 Kcps  
 Expected-BPS    :       5.02 Gbps  

 Active-flows    :    21328  Clients :      511   Socket-util : 0.0775 %    
 Open-flows      :  1342791  Servers :     5621   Socket :    24927 Socket/Clients :  48.8 
 drop-rate       :       5.00 Gbps   
 current time    : 66.5 sec  
 test duration   : 3533.5 sec  

-Latency stats enabled 
 Cpu Utilization : 0.1 %  
 if|   tx_ok , rx_ok  , rx check ,error,       latency (usec) ,    Jitter          max window 

   |         ,        ,          ,     ,   average   ,   max  ,    (usec)                     
 ---------------------------------------------------------------------------------------------------------------- 

 0 |    65254,   12858,         0,    0,          7  ,      23,       1      |  10  10  15  10  12  11  12  23  10  10  15  18  12 
 1 |    65254,     136,         0,    0,          3  ,       0,       0      |  0  0  0  0  0  0  0  0  0  0  0  0  0 


```

## 6.进行发包实验

变更-c的参数，观察cpu利用率以及吞吐率并绘制不同包大小、不同核数时cpu利用率以及吞吐率。

```c++
sudo ./t-rex-64 -f cap2/imix_1518.yaml -m 823451 -l 1000 -c 2

#参数c为服务器应用的核数
#cap2/imix_1518.yaml 为包大小1518B的配置文件

```

其他包大小的配置文件名如下：

```c++
cap2/imix_64.yaml

cap2/imix_594.yaml

```

以下为输出的结果：

```c++
-Per port stats table 

      ports |               0 |               1 
 -----------------------------------------------------------------------------------------

   opackets |        34232892 |           42080 
     obytes |     51904428378 |         2777280 
   ipackets |               0 |               0 
     ibytes |               0 |               0 
    ierrors |               0 |               0 
    oerrors |               0 |               0 
      Tx Bw |       9.89 Gbps |     527.55 Kbps 

-Global stats enabled 
 Cpu Utilization : 100.0  %  9.9 Gb/core 
 Platform_factor : 1.0  
 Total-Tx        :       9.89 Gbps  
 Total-Rx        :       0.00  bps  
 Total-PPS       :     816.02 Kpps  
 Total-CPS       :       0.00  cps  

 Expected-PPS    :       6.59 Gpps  
 Expected-CPS    :       6.59 Gcps  
 Expected-BPS    :      80.00 Tbps  

 Active-flows    :     1600  Clients :      254   Socket-util : 0.0100 %    
 Open-flows      :     1600  Servers :    65534   Socket :     1600 Socket/Clients :  6.3 
 Total_queue_full : 72264932         
 drop-rate       :       9.89 Gbps   
 current time    : 43.3 sec  
 test duration   : 3556.7 sec  

-Latency stats enabled 
 Cpu Utilization : 0.1 %  
 if|   tx_ok , rx_ok  , rx check ,error,       latency (usec) ,    Jitter          max window 

   |         ,        ,          ,     ,   average   ,   max  ,    (usec)                     
 ---------------------------------------------------------------------------------------------------------------- 

 0 |    42080,       0,         0,    0,          0  ,       0,       0      |  0  0  0  0  0  0  0  0  0  0  0  0  0 
 1 |    42081,       0,         0,    0,          0  ,       0,       0      |  0  0  0  0  0  0  0  0  0  0  0  0  0 
 

```







原文链接:https://blog.csdn.net/wu_tongtong/article/details/118669278

原文作者：Coco_T_
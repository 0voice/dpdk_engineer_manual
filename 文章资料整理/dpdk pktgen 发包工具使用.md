# dpdk pktgen 发包工具使用

## 1.编译方法 [dpdk pktgen发包工具编译](https://zhuanlan.zhihu.com/p/457628487)

接下来要做的是修改配置文件。在/pktgen-dpdk/cfg里

\# 备份
cp default.cfg backup
vim default.cfg

这是官方给我们的示例 default.cfg

description = 'A Pktgen default simple configuration'

\# Setup configuration
setup = {
'exec': (
'sudo',
'-E'
),

'devices': (
'81:00.0 81:00.1 81:00.2 81:00.3',
'85:00.0 85:00.1 85:00.2 85:00.3'
),

'opts': (
'-b igb_uio'
)
}

\# Run command and options
run = {
'exec': (
'sudo',
'-E'
),

\# Application name and use app_path to help locate the app
'app_name': 'pktgen',

\# using (sdk) or (target) for specific variables
\# add (app_name) of the application
\# Each path is tested for the application
'app_path': (
'./app/%(target)s/%(app_name)s',
'%(sdk)s/%(target)s/app/%(app_name)s',
),

'dpdk': (
'-l 14,15-22',
'-n 4',
'--proc-type auto',
'--log-level 7',
'--socket-mem 2048,2048',
'--file-prefix pg'
),

'blacklist': (
\#'-b 81:00.0 -b 81:00.1 -b 81:00.2 -b 81:00.3',
\#'-b 85:00.0 -b 85:00.1 -b 85:00.2 -b 85:00.3',
'-b 81:00.0 -b 81:00.1',
'-b 85:00.0 -b 85:00.1',
'-b 83:00.0'
),

'app': (
'-T',
'-P',
'--crc-strip',
'-m [15:16].0',
'-m [17:18].1',
'-m [19:20].2',
'-m [21:22].3'
),

'misc': (
'-f', 'themes/black-yellow.theme'
)
}

需要修改的地方有三处：

1. 网卡设备的PCI号，可以用dpdk的usertools查看。
2. DPDK EAL 的 command line arguments。改成与你系统对应的参数。参照DPDK的文档。
3. pktgen 的 command line arguments，参照pktgen的文档。

贴上我的版本作为参考：

\# 只贴有修改的部分
\# ……
'devices': (
'02:01.0 02:02.0' # 只绑定了两个网卡到DPDK做实验
),
\# ……

'dpdk': (
'-l 0-3',
'-n 4',
'--proc-type auto',
'--log-level 7',
\#'--socket-mem 2048,2048',
'--socket-mem 2048', # 只有一个socket
'--file-prefix pg',
),

\# .......

'app': (
'-T',
'-P',
'--crc-strip',
'-m [1].0', # 查看官方手册了解 -m 用法，用于提供lcore到port的映射
'-m [2].1'
\#'-m [19:20].2',
\#'-m [21:22].3'
),

修改完后即可执行。

cd pktgen-dpdk
./tools/run.py -s default
./tools/run.py default

启动会出现错误

```text
                                                                       Port memory used = 780295 KB
Src MAC e8:61:1f:13:02:cd
 <Promiscuous mode Enabled> <Promiscuous mode Enabled>
                                                                      Total memory used = 1560590 KB


=== Display processing on lcore 1
  RX/TX processing lcore:   2 rx:  1 tx:  1
For RX found 1 port(s) for lcore 2
For TX found 1 port(s) for lcore 2
  RX/TX processing lcore:   3 rx:  1 tx:  1
For RX found 1 port(s) for lcore 3
For TX found 1 port(s) for lcore 3
PANIC in pktgen_main_rxtx_loop():
*** port 1 on socket ID 0 has different socket ID on lcore 3 socket ID 1
7: [/lib64/libc.so.6(clone+0x6d) [0x7f5819094b0d]]
6: [/lib64/libpthread.so.0(+0x7ea5) [0x7f581936bea5]]
5: [./app/x86_64-native-linuxapp-gcc/pktgen(eal_thread_loop+0x28b) [0x7adb4a]]
4: [./app/x86_64-native-linuxapp-gcc/pktgen(pktgen_launch_one_lcore+0x84) [0x4b9121]]
3: [./app/x86_64-native-linuxapp-gcc/pktgen() [0x4b87c1]]
2: [./app/x86_64-native-linuxapp-gcc/pktgen(__rte_panic+0xd9) [0x7b4f28]]
1: [./app/x86_64-native-linuxapp-gcc/pktgen(rte_dump_stack+0x27) [0x7b4dd4]]
[root@localhost pktgen-dpdk-pktgen-19.12.0]# 
```

表示 cpu 的socket 不在一个槽，需要设置到一个槽

```text
        'cores': '1,2,4',
        'nrank': '4',
        'proc': 'auto',
        'log': '7',
        'prefix': 'pg',

        'opts': (
                '-v',
                '-T',
                '-P',
                '-j',
                ),
        'map': (
                '[2].0',
                '[4].1',
                ),
```

进入交互命令

./tools/run.py -s default

./tools/run.py default

```text
运行 start 0开始第一个网口，stop 0停止第一个网口，第二个网口类似；

运行 start all开始所有网口，stop all停止所有网口；

运行page help可以看到可用命令，主要有page stats, page xstats, page rate, quit
```

- 运行pktgen 把两个网口互联

set 0 dst ip 192.168.74.132
set 0 dst mac 00:0c:29:45:e2:b9
set 0 count 100
start 0

图中port1和port2已经有明显区别，收包数相差100个包

设置参数

```text
clear 0 stats
reset 0
set 0 size 500
enable screen
enable 0 range
disable 0 vlan
set 0 size 64
set 0 rate 100
set 0 burst 64
set 0 type ipv4
set 0 proto udp
set 0 dst ip 192.168.0.1/24
set 0 src ip 172.0.0.1/16
set 0 sport 12325
set 0 dport 12325
set 0 dst mac 20:04:0f:34:aa:3d
set 0 src mac f8:f2:1e:1a:d6:00
range 0 proto udp
range 0 src port 10000 10000 60000 1
range 0 dst port 10000 10000 60000 1
set 0 src ip 172.0.0.1/16
range 0 src ip start 172.0.0.1
range 0 src ip min 172.0.0.1
range 0 src ip max 172.0.255.254
range 0 src ip inc 0.0.0.1
set 0 dst ip 192.168.0.1
range 0 dst ip start 192.168.0.1
range 0 dst ip min 192.168.0.1
range 0 dst ip max 192.168.0.1
range 0 dst ip inc 0.0.0.0
disable 0 process
disable 0 bonding
disable 0 mac_from_arp
start 0 arp request
range 0 dst mac start 20:04:0f:34:aa:3d
range 0 dst mac min 20:04:0f:34:aa:3d
range 0 dst mac max 20:04:0f:34:aa:3d
range 0 src mac start f8:f2:1e:1a:d6:00
range 0 src mac min f8:f2:1e:1a:d6:00
range 0 src mac max f8:f2:1e:1a:d6:00
```

用pkggen再发1000个包

set 0 count 10000000

```text
** Version: DPDK 19.11.10, Command Line Interface without timers
Pktgen:/> theme default white white off
Pktgen:/> theme top.spinner cyan none bold


/ Ports 0-1 of 2   <Main Page>  Copyright (c) <2010-2019>, Intel Corporation
  Flags:Port        : P------Single      :0 P------Single      :1
Link State          :          <UP-1000-FD>          <UP-1000-FD>      ---Total Rate---
Pkts/s Max/Rx       :                   0/0             1502485/0             1502485/0
       Max/Tx       :             1502486/0                   0/0             1502486/0
MBits/s Rx/Tx       :                   0/0                   0/0                   0/0
Broadcast           :                     0                     0
Multicast           :                     0                     0
Sizes 64            :                     0              10000000
      65-127        :                     0                     0
      128-255       :                     0                     0
      256-511       :                     0                     0
      512-1023      :                     0                     0
      1024-1518     :                     0                     0
Runts/Jumbos        :                   0/0                   0/0
ARP/ICMP Pkts       :                   0/0                   0/0
Errors Rx/Tx        :                   0/0                   0/0
Total Rx Pkts       :                     0              10000000
      Tx Pkts       :              10000000                     0
      Rx MBs        :                     0                  6720
      Tx MBs        :                  6720                     0
                    :
Pattern Type        :               abcd...               abcd...
Tx Count/% Rate     :        10000000 /100%         Forever /100%
Pkt Size/Tx Burst   :             64 /   64             64 /   64
TTL/Port Src/Dest   :         4/ 1234/ 5678         4/ 1234/ 5678
Pkt Type:VLAN ID    :       IPv4 / TCP:0001       IPv4 / TCP:0001
802.1p CoS/DSCP/IPP :             0/  0/  0             0/  0/  0
VxLAN Flg/Grp/vid   :      0000/    0/    0      0000/    0/    0
IP  Destination     :           192.168.1.1           192.168.0.1
    Source          :        192.168.0.1/24        192.168.1.1/24
MAC Destination     :     e8:61:1f:13:02:cd     e8:61:1f:13:02:cc
    Source          :     e8:61:1f:13:02:cc     e8:61:1f:13:02:cd
PCI Vendor/Addr     :     8086:1521/04:00.0     8086:1521/04:00.1

-- Pktgen 19.12.0 (DPDK 19.11.10)  Powered by DPDK  (pid:22927) ---------------
```

1g 网卡 64字节 Max/Tx : 1502486 每秒 150w = 733M/秒

1g 网卡 1024字节 Max/Tx : 122185 每秒 12w = 1033M/秒

 原文链接;https://zhuanlan.zhihu.com/p/457654071 原文作者：江义波
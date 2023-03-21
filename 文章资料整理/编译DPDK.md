# 编译DPDK

## 1.准备工作

DPDK是C语言写的，编译需要gcc，DPDK自带很多工具脚本，会调用到其他命令行工具 numactl numactl-devel pciutils net-tools。

```bash
# 准备依赖包
[root@backendcloud-centos7 ~]# yum install -y numactl numactl-devel pciutils net-tools gcc
# 查看网卡信息
[root@backendcloud-centos7 ~]# dmesg |grep -i eth
[    1.231028] e1000 0000:02:01.0 eth0: (PCI:66MHz:32-bit) 00:0c:29:db:15:9e
[    1.231030] e1000 0000:02:01.0 eth0: Intel(R) PRO/1000 Network Connection
[    1.646402] e1000 0000:02:02.0 eth1: (PCI:66MHz:32-bit) 00:0c:29:db:15:a8
[    1.646406] e1000 0000:02:02.0 eth1: Intel(R) PRO/1000 Network Connection
[root@backendcloud-centos7 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:db:15:9e brd ff:ff:ff:ff:ff:ff
    inet 192.168.126.135/24 brd 192.168.126.255 scope global noprefixroute dynamic ens33
       valid_lft 1383sec preferred_lft 1383sec
    inet6 fe80::4165:dab8:c225:4e76/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:db:15:a8 brd ff:ff:ff:ff:ff:ff
    inet 192.168.126.138/24 brd 192.168.126.255 scope global noprefixroute dynamic ens34
       valid_lft 1346sec preferred_lft 1346sec
    inet6 fe80::60f4:e3f5:874f:5ed6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:2e:84:02:a7 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
# 下载DPDK安装包
[root@backendcloud-centos7 ~]# wget   http://fast.dpdk.org/rel/dpdk-18.05.1.tar.gz
[root@backendcloud-centos7 ~]# tar -zxvf  dpdk-18.05.1.tar.gz

```

## 2.DPDK编译和DPDK环境配置

```bash
[root@backendcloud-centos7 ~]# cd  dpdk-stable-18.05.1/usertools
[root@backendcloud-centos7 usertools]# ./dpdk-setup.sh
------------------------------------------------------------------------------
 RTE_SDK exported as /root/dpdk-stable-18.05.1
------------------------------------------------------------------------------
----------------------------------------------------------
 Step 1: Select the DPDK environment to build
----------------------------------------------------------
[1] arm64-armv8a-linuxapp-clang
[2] arm64-armv8a-linuxapp-gcc
[3] arm64-dpaa2-linuxapp-gcc
[4] arm64-dpaa-linuxapp-gcc
[5] arm64-stingray-linuxapp-gcc
[6] arm64-thunderx-linuxapp-gcc
[7] arm64-xgene1-linuxapp-gcc
[8] arm-armv7a-linuxapp-gcc
[9] i686-native-linuxapp-gcc
[10] i686-native-linuxapp-icc
[11] ppc_64-power8-linuxapp-gcc
[12] x86_64-native-bsdapp-clang
[13] x86_64-native-bsdapp-gcc
[14] x86_64-native-linuxapp-clang
[15] x86_64-native-linuxapp-gcc
[16] x86_64-native-linuxapp-icc
[17] x86_x32-native-linuxapp-gcc

----------------------------------------------------------
 Step 2: Setup linuxapp environment
----------------------------------------------------------
[18] Insert IGB UIO module
[19] Insert VFIO module
[20] Insert KNI module
[21] Setup hugepage mappings for non-NUMA systems
[22] Setup hugepage mappings for NUMA systems
[23] Display current Ethernet/Crypto device settings
[24] Bind Ethernet/Crypto device to IGB UIO module
[25] Bind Ethernet/Crypto device to VFIO module
[26] Setup VFIO permissions

----------------------------------------------------------
 Step 3: Run test application for linuxapp environment
----------------------------------------------------------
[27] Run test application ($RTE_TARGET/app/test)
[28] Run testpmd application in interactive mode ($RTE_TARGET/app/testpmd)

----------------------------------------------------------
 Step 4: Other tools
----------------------------------------------------------
[29] List hugepage info from /proc/meminfo

----------------------------------------------------------
 Step 5: Uninstall and system cleanup
----------------------------------------------------------
[30] Unbind devices from IGB UIO or VFIO driver
[31] Remove IGB UIO module
[32] Remove VFIO module
[33] Remove KNI module
[34] Remove hugepage mappings

[35] Exit Script

Option: 

```

3.编译DPDK `[15] x86_64-native-linuxapp-gcc`

报错：

```bash
make: *** /lib/modules/3.10.0-1160.el7.x86_64/build: No such file or directory.  Stop.

```

需要安装内核开发包，yum install -y kernel-devel

并重新做下软链接

```bash
[root@backendcloud-centos7 ~]# cd /lib/modules/3.10.0-1160.el7.x86_64
[root@backendcloud-centos7 3.10.0-1160.el7.x86_64]# ll
total 3300
lrwxrwxrwx.  1 root root     39 Sep 20 16:18 build -> /usr/src/kernels/3.10.0-1160.el7.x86_64
drwxr-xr-x.  2 root root      6 Oct 20  2020 extra
drwxr-xr-x. 12 root root    128 Sep 20 16:18 kernel
-rw-r--r--.  1 root root 860326 Sep 20 16:20 modules.alias
-rw-r--r--.  1 root root 819744 Sep 20 16:20 modules.alias.bin
-rw-r--r--.  1 root root   1333 Oct 20  2020 modules.block
-rw-r--r--.  1 root root   7391 Oct 20  2020 modules.builtin
-rw-r--r--.  1 root root   9440 Sep 20 16:20 modules.builtin.bin
-rw-r--r--.  1 root root 273209 Sep 20 16:20 modules.dep
-rw-r--r--.  1 root root 382108 Sep 20 16:20 modules.dep.bin
-rw-r--r--.  1 root root    361 Sep 20 16:20 modules.devname
-rw-r--r--.  1 root root    140 Oct 20  2020 modules.drm
-rw-r--r--.  1 root root     69 Oct 20  2020 modules.modesetting
-rw-r--r--.  1 root root   1810 Oct 20  2020 modules.networking
-rw-r--r--.  1 root root  97935 Oct 20  2020 modules.order
-rw-r--r--.  1 root root    569 Sep 20 16:20 modules.softdep
-rw-r--r--.  1 root root 397513 Sep 20 16:20 modules.symbols
-rw-r--r--.  1 root root 486211 Sep 20 16:20 modules.symbols.bin
lrwxrwxrwx.  1 root root      5 Sep 20 16:18 source -> build
drwxr-xr-x.  2 root root      6 Oct 20  2020 updates
drwxr-xr-x.  2 root root     95 Sep 20 16:18 vdso
drwxr-xr-x.  2 root root      6 Oct 20  2020 weak-updates
# 若软链接没有因找不到路径标红，就不需要做下面的步骤
[root@backendcloud-centos7 3.10.0-1160.el7.x86_64]# rm build
rm: remove symbolic link ‘build’? y
[root@backendcloud-centos7 3.10.0-1160.el7.x86_64]# ln -s /usr/src/kernels/3.10.0-1160.76.1.el7.x86_64 build
[root@backendcloud-centos7 3.10.0-1160.el7.x86_64]# ll
total 3300
lrwxrwxrwx.  1 root root     44 Sep 20 17:06 build -> /usr/src/kernels/3.10.0-1160.76.1.el7.x86_64
drwxr-xr-x.  2 root root      6 Oct 20  2020 extra
drwxr-xr-x. 12 root root    128 Sep 20 16:18 kernel
-rw-r--r--.  1 root root 860326 Sep 20 16:20 modules.alias
-rw-r--r--.  1 root root 819744 Sep 20 16:20 modules.alias.bin
-rw-r--r--.  1 root root   1333 Oct 20  2020 modules.block
-rw-r--r--.  1 root root   7391 Oct 20  2020 modules.builtin
-rw-r--r--.  1 root root   9440 Sep 20 16:20 modules.builtin.bin
-rw-r--r--.  1 root root 273209 Sep 20 16:20 modules.dep
-rw-r--r--.  1 root root 382108 Sep 20 16:20 modules.dep.bin
-rw-r--r--.  1 root root    361 Sep 20 16:20 modules.devname
-rw-r--r--.  1 root root    140 Oct 20  2020 modules.drm
-rw-r--r--.  1 root root     69 Oct 20  2020 modules.modesetting
-rw-r--r--.  1 root root   1810 Oct 20  2020 modules.networking
-rw-r--r--.  1 root root  97935 Oct 20  2020 modules.order
-rw-r--r--.  1 root root    569 Sep 20 16:20 modules.softdep
-rw-r--r--.  1 root root 397513 Sep 20 16:20 modules.symbols
-rw-r--r--.  1 root root 486211 Sep 20 16:20 modules.symbols.bin
lrwxrwxrwx.  1 root root      5 Sep 20 16:18 source -> build
drwxr-xr-x.  2 root root      6 Oct 20  2020 updates
drwxr-xr-x.  2 root root     95 Sep 20 16:18 vdso
drwxr-xr-x.  2 root root      6 Oct 20  2020 weak-updates

```

link不标红后再次编译DPDK `[15] x86_64-native-linuxapp-gcc`

```bash
Option: 15

Configuration done using x86_64-native-linuxapp-gcc
== Build lib
== Build lib/librte_compat
== Build lib/librte_eal
== Build lib/librte_eal/common
== Build lib/librte_eal/linuxapp
== Build lib/librte_eal/linuxapp/eal
== Build lib/librte_pci
== Build lib/librte_ring
== Build lib/librte_mempool
== Build lib/librte_mbuf
...
== Build app/test-eventdev
  CC evt_main.o
  CC evt_options.o
  CC evt_test.o
  CC parser.o
  CC test_order_common.o
  CC test_order_queue.o
  CC test_order_atq.o
  CC test_perf_common.o
  CC test_perf_queue.o
  CC test_perf_atq.o
  CC test_pipeline_common.o
  CC test_pipeline_queue.o
  CC test_pipeline_atq.o
  LD dpdk-test-eventdev
  INSTALL-APP dpdk-test-eventdev
  INSTALL-MAP dpdk-test-eventdev.map
Build complete [x86_64-native-linuxapp-gcc]
Installation cannot run with T defined and DESTDIR undefined
------------------------------------------------------------------------------
 RTE_TARGET exported as x86_64-native-linuxapp-gcc
------------------------------------------------------------------------------

```

编译完，说编译成功，但是“Installation cannot run with T defined and DESTDIR undefined”，提示你没有指定安装路径,这里只需要编译,本来也不需要安装,所以忽略,不影响使用。

### 2.1.添加igb_uio内核驱动 `[18] Insert IGB UIO module`

```bash
Option: 18

Unloading any existing DPDK UIO module
Loading uio module
Loading DPDK UIO module

```

### 2.2根据NUMA环境配置大页内存 `[21] Setup hugepage mappings for non-NUMA systems`

检查numa

```bash
[root@backendcloud-centos7 3.10.0-1160.el7.x86_64]# numactl -H
available: 1 nodes (0)
node 0 cpus: 0 1
node 0 size: 7963 MB
node 0 free: 7128 MB
node distances:
node   0 
  0:  10 
```

根据实际情况选择`[21] Setup hugepage mappings for non-NUMA systems`或者`[22] Setup hugepage mappings for NUMA systems`，本篇环境是21

```bash
Option: 21

Removing currently reserved hugepages
Unmounting /mnt/huge and removing directory

  Input the number of 2048kB hugepages
  Example: to have 128MB of hugepages available in a 2MB huge page system,
  enter '64' to reserve 64 * 2MB pages
Number of pages: 64
Reserving hugepages
Creating /mnt/huge and mounting as hugetlbfs
```

检查大页内存：

```bash
[root@backendcloud-centos7 3.10.0-1160.el7.x86_64]# cat /proc/meminfo 
...
HugePages_Total:      64
HugePages_Free:       64
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      149312 kB
DirectMap2M:     5093376 kB
DirectMap1G:     5242880 kB
```

### 2.3显示系统中可用的硬件网卡 `[23] Display current Ethernet/Crypto device settings`

```bash
Option: 23


Network devices using DPDK-compatible driver
============================================
<none>

Network devices using kernel driver
===================================
0000:02:01.0 '82545EM Gigabit Ethernet Controller (Copper) 100f' if=ens33 drv=e1000 unused=igb_uio *Active*
0000:02:02.0 '82545EM Gigabit Ethernet Controller (Copper) 100f' if=ens34 drv=e1000 unused=igb_uio *Active*
...
```

### 2.4绑定igb网卡 `[24] Bind Ethernet/Crypto device to IGB UIO`

先要停用打算要绑定的网卡 ifconfig ens34 down

```bash
Option: 24


Network devices using DPDK-compatible driver
============================================
<none>

Network devices using kernel driver
===================================
0000:02:01.0 '82545EM Gigabit Ethernet Controller (Copper) 100f' if=ens33 drv=e1000 unused=igb_uio *Active*
0000:02:02.0 '82545EM Gigabit Ethernet Controller (Copper) 100f' if=ens34 drv=e1000 unused=igb_uio 
...

Enter PCI address of device to bind to IGB UIO driver: 0000:02:02.0
OK

Option: 23


Network devices using DPDK-compatible driver
============================================
0000:02:02.0 '82545EM Gigabit Ethernet Controller (Copper) 100f' drv=igb_uio unused=e1000

Network devices using kernel driver
===================================
0000:02:01.0 '82545EM Gigabit Ethernet Controller (Copper) 100f' if=ens33 drv=e1000 unused=igb_uio *Active*
...

```

### 2.5解绑DPDK网卡 `[30] Unbind devices from IGB UIO or VFIO driver`

```bash
Option: 30


Network devices using DPDK-compatible driver
============================================
0000:02:02.0 '82545EM Gigabit Ethernet Controller (Copper) 100f' drv=igb_uio unused=e1000

Network devices using kernel driver
===================================
0000:02:01.0 '82545EM Gigabit Ethernet Controller (Copper) 100f' if=ens33 drv=e1000 unused=igb_uio *Active*
...
Enter PCI address of device to unbind: 0000:02:02.0

Enter name of kernel driver to bind the device to: igb_uio
0000:02:02.0 already bound to driver igb_uio, skipping

OK
```

重启node生效。

## 3.DPDK简单测试

```bash
[root@backendcloud-centos7 ~]# cd dpdk-stable-18.05.1/examples/helloworld/
[root@backendcloud-centos7 helloworld]# pwd
/root/dpdk-stable-18.05.1/examples/helloworld
[root@backendcloud-centos7 helloworld]# export  RTE_SDK=/root/dpdk-stable-18.05.1
[root@backendcloud-centos7 helloworld]# export  RTE_TARGET=x86_64-native-linuxapp-gcc
[root@backendcloud-centos7 helloworld]# make
  CC main.o
  LD helloworld
  INSTALL-APP helloworld
  INSTALL-MAP helloworld.map
[root@backendcloud-centos7 helloworld]# ./build/helloworld 
EAL: Detected 2 lcore(s)
EAL: Detected 1 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: No free hugepages reported in hugepages-1048576kB
EAL: Probing VFIO support...
EAL: PCI device 0000:02:01.0 on NUMA socket -1
EAL:   Invalid NUMA socket, default to 0
EAL:   probe driver: 8086:100f net_e1000_em
EAL: PCI device 0000:02:02.0 on NUMA socket -1
EAL:   Invalid NUMA socket, default to 0
EAL:   probe driver: 8086:100f net_e1000_em
EAL: Error reading from file descriptor 14: Input/output error
...
hello from core 1
hello from core 0
```

报了很多重复的行：

```bash
EAL: Error reading from file descriptor 14: Input/output error
```

原因：INTX toggle check is not work with VMware E1000 Ethernet.
INTX is badly emulated in VMWare; the disable logic doesn’t work.
I thought the DPDK API detected when link state interrupt would not work.
But of course the application needs to check that before enabling link state.

可以通过下面的方式规避：

将 dpdk-stable-18.05.1/kernel/linux/igb_uio/igb_uio.c 的260行：

```c arduino
if (pci_intx_mask_supported(udev->pdev)) {
```

替换成

```c arduino
if (pci_intx_mask_supported(udev->pdev)||true) {
language-c arduino复制代码
[root@backendcloud-centos7 helloworld]# make
Makefile:43: *** "Please define RTE_SDK environment variable".  Stop.
[root@backendcloud-centos7 helloworld]# export  RTE_TARGET=x86_64-native-linuxapp-gcc
[root@backendcloud-centos7 helloworld]# make
[root@backendcloud-centos7 helloworld]# ./build/helloworld 
EAL: Detected 2 lcore(s)
EAL: Detected 1 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: No free hugepages reported in hugepages-1048576kB
EAL: Probing VFIO support...
EAL: PCI device 0000:02:01.0 on NUMA socket -1
EAL:   Invalid NUMA socket, default to 0
EAL:   probe driver: 8086:100f net_e1000_em
EAL: PCI device 0000:02:02.0 on NUMA socket -1
EAL:   Invalid NUMA socket, default to 0
EAL:   probe driver: 8086:100f net_e1000_em
hello from core 1
hello from core 0
```





原文链接：https://mp.weixin.qq.com/s/jsk4qgPUAvsX5WEhjTRKoQ

原文作者：后端云


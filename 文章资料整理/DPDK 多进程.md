# DPDK 多进程

DPDK库里是支持多进程和多线程，本文主要总结多进程的相关的操作。

DPDK多进程使用的关键启动参数：

- --proc-type：指定一个dpdk进程是主进程还是副进程（参数值就用上面的primary或是secondary，或者是auto）
- --file-prefix：允许非合作的进程拥有不同的内存区域。主副进程默认文件路径/var/run/.rte_config，同一个处理组的主副进程使用相同的参数，
  如果想运行多个主进程，这个参数就必须指定！
- --socket-mem：设置从hugepages分配多大的存储空间。默认会用掉所有的hugepages，所以建议指定这个参数，不管是单cpu还是在NUMA中。
  eg：单socket**，--socket-mem=512；在numa中，--socket-mem=512,512；多个socket间用‘,’号隔开；**
- -w ： 后面跟网卡的PCI号，指定使用网卡。设置了这参数，DPDK只会使用这个参数对应的网卡，不会初始化其他的。

在Multi-process Sample Application中介绍了4种使用场景：

Basic Multi-process Example，DPDK进程间通过ring，内存池，队列，进行信息交互。
Symmetric Multi-process Example，主进程初始化所有资源，副进程直接获取资源进行数据包处理，副进程除了不初始化资源，数据包处理和主进程是一样的。每个进程获取每个端口的一个RX， TX队列。
Client-Server Multi-process Example，主进程初始化资源和接收所有收到的数据包并轮询分发给副进程处理。
Master-slave Multi-process Example，这个模式主要是介绍各进程之间存在依赖关系，主进程和副进程，副进程和副进程

```
[root@localhost simple_mp]#   ./build/simple_mp -l 0-1   --proc-type=primary
EAL: Detected 128 lcore(s)
EAL: Detected 4 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'PA'
EAL: Probing VFIO support...
EAL: VFIO support initialized
EAL: PCI device 0000:05:00.0 on NUMA socket 0
EAL:   probe driver: 19e5:200 net_hinic
net_hinic: Initializing pf hinic-0000:05:00.0 in primary process
net_hinic: Device 0000:05:00.0 hwif attribute:
net_hinic: func_idx:0, p2p_idx:0, pciintf_idx:0, vf_in_pf:0, ppf_idx:0, global_vf_id:15, func_type:2
net_hinic: num_aeqs:4, num_ceqs:4, num_irqs:32, dma_attr:2
net_hinic: API CMD poll status timeout
net_hinic: chain type: 0x7
net_hinic: chain hw cpld error: 0x1
net_hinic: chain hw check error: 0x0
net_hinic: chain hw current fsm: 0x0
net_hinic: chain hw current ci: 0x0
net_hinic: Chain hw current pi: 0x1
net_hinic: Send msg to mgmt failed
net_hinic: Failed to get board info, err: -110, status: 0x0, out size: 0x0
net_hinic: Check card workmode failed, dev_name: 0000:05:00.0
net_hinic: Create nic device failed, dev_name: 0000:05:00.0
net_hinic: Initialize 0000:05:00.0 in primary failed
EAL: Requested device 0000:05:00.0 cannot be used
EAL: PCI device 0000:06:00.0 on NUMA socket 0
EAL:   probe driver: 19e5:200 net_hinic
EAL: PCI device 0000:7d:00.0 on NUMA socket 0
EAL:   probe driver: 19e5:a222 net_hns3
EAL: PCI device 0000:7d:00.1 on NUMA socket 0
EAL:   probe driver: 19e5:a221 net_hns3
EAL: PCI device 0000:7d:00.2 on NUMA socket 0
EAL:   probe driver: 19e5:a222 net_hns3
EAL: PCI device 0000:7d:00.3 on NUMA socket 0
EAL:   probe driver: 19e5:a221 net_hns3
APP: Finished Process Init.
Starting core 1

simple_mp > primary send hello1
Command not found

simple_mp > send hello by primary
Bad arguments

simple_mp > 
simple_mp > send hello_by_primary

simple_mp > core 1: Received 'hello_by_sencondary'


simple_mp > 
```

```
 ./build/simple_mp -l 0-1  --proc-type=secondary
```

 

![img](https://img2020.cnblogs.com/blog/665372/202008/665372-20200828152949672-433866407.png)

 

 

1. 1. ```
      [root@localhost lib]# ps -elf | grep simple_mp
      0 S root       8457 124128 13  80   0 - 8389378 wait_w 03:24 pts/1  00:00:16 ./build/simple_mp -l 0-1 --proc-type=primary
      0 S root       8471   7504  6  80   0 - 8389498 wait_w 03:24 pts/2  00:00:06 ./build/simple_mp -l 0-1 --proc-type=secondary
      0 S root       8564  57486  0  80   0 -  1729 pipe_w 03:26 pts/0    00:00:00 grep --color=auto simple_mp
      ```

      ```
      [root@localhost lib]# ps -mo pid,tid,%cpu,psr -p 8471
         PID    TID %CPU PSR
        8471      -  6.0   -
           -   8471  0.0   0
           -   8472  0.0  33
           -   8473  0.0  10
           -   8474  6.0   1
      [root@localhost lib]# ps -T  -p 8471
         PID   SPID TTY          TIME CMD
        8471   8471 pts/2    00:00:00 simple_mp
        8471   8472 pts/2    00:00:00 eal-intr-thread
        8471   8473 pts/2    00:00:00 rte_mp_handle
        8471   8474 pts/2    00:00:10 lcore-slave-1
      [root@localhost lib]# ps -mo pid,tid,%cpu,psr -p 8457
         PID    TID %CPU PSR
        8457      - 10.7   -
           -   8457  5.1   0
           -   8458  0.0   9
           -   8459  0.0   9
           -   8460  5.6   1
      [root@localhost lib]# ps -T  -p 8457
         PID   SPID TTY          TIME CMD
        8457   8457 pts/1    00:00:10 simple_mp
        8457   8458 pts/1    00:00:00 eal-intr-thread
        8457   8459 pts/1    00:00:00 rte_mp_handle
        8457   8460 pts/1    00:00:11 lcore-slave-1
      [root@localhost lib]# 
      ```

      [![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

      ![img](https://img2020.cnblogs.com/blog/665372/202008/665372-20200828153223866-1523426605.png)

       **两个mp_socket， mp_socket_8471_167e85391023是seconary进程的**

 

```
[root@localhost dpdk-stable-17.11.2]# killall simple_mp
[root@localhost dpdk-stable-17.11.2]# ps -elf | grep simple_mp
0 S root       9154  36716  0  80   0 -  1729 pipe_w 03:38 pts/3    00:00:00 grep --color=auto simple_mp
[root@localhost dpdk-stable-17.11.2]#
```

```
kill之后还存在哦[root@localhost lib]# ls /var/run/dpdk/rte/ -al
total 15680
drwx------. 2 root root   1100 Aug 28 03:24 .
drwx------. 3 root root     60 Aug 26 03:45 ..
-rw-------. 1 root root  18816 Aug 28 03:24 config
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-0-0
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-0-0_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-0-1
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-0-1_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-0-2
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-0-2_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-0-3
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-0-3_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-1-0
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-1-0_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-1-1
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-1-1_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-1-2
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-1-2_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-1-3
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-1-3_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-2-0
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-2-0_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-2-1
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-2-1_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-2-2
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-2-2_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-2-3
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-2-3_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-3-0
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-3-0_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-3-1
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-3-1_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-3-2
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-3-2_8471
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-3-3
-rw-------. 1 root root 458752 Aug 28 03:24 fbarray_memseg-2048k-3-3_8471
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-0-0
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-0-0_8471
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-0-1
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-0-1_8471
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-1-0
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-1-0_8471
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-1-1
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-1-1_8471
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-2-0
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-2-0_8471
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-2-1
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-2-1_8471
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-3-0
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-3-0_8471
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-3-1
-rw-------. 1 root root  65536 Aug 28 03:24 fbarray_memseg-524288k-3-1_8471
-rw-------. 1 root root 196608 Aug 28 03:24 fbarray_memzone
-rw-------. 1 root root  16576 Aug 28 03:24 hugepage_info
srwxr-xr-x. 1 root root      0 Aug 28 03:24 mp_socket
srwxr-xr-x. 1 root root      0 Aug 28 03:24 mp_socket_8471_167e85391023
[root@localhost lib]#
```

```
[root@localhost simple_mp]#   ./build/simple_mp -l 126-127   --proc-type=primary
EAL: Detected 128 lcore(s)
EAL: Detected 4 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'PA'
EAL: Probing VFIO support...
EAL: VFIO support initialized
EAL: PCI device 0000:05:00.0 on NUMA socket 0
EAL:   probe driver: 19e5:200 net_hinic
net_hinic: Initializing pf hinic-0000:05:00.0 in primary process
net_hinic: Device 0000:05:00.0 hwif attribute:
net_hinic: func_idx:0, p2p_idx:0, pciintf_idx:0, vf_in_pf:0, ppf_idx:0, global_vf_id:15, func_type:2
net_hinic: num_aeqs:4, num_ceqs:4, num_irqs:32, dma_attr:2
net_hinic: API CMD poll status timeout
net_hinic: chain type: 0x7
net_hinic: chain hw cpld error: 0x1
net_hinic: chain hw check error: 0x0
net_hinic: chain hw current fsm: 0x0
net_hinic: chain hw current ci: 0x0
net_hinic: Chain hw current pi: 0x1
net_hinic: Send msg to mgmt failed
net_hinic: Failed to get board info, err: -110, status: 0x0, out size: 0x0
net_hinic: Check card workmode failed, dev_name: 0000:05:00.0
net_hinic: Create nic device failed, dev_name: 0000:05:00.0
net_hinic: Initialize 0000:05:00.0 in primary failed
EAL: Requested device 0000:05:00.0 cannot be used
EAL: PCI device 0000:06:00.0 on NUMA socket 0
EAL:   probe driver: 19e5:200 net_hinic
EAL: PCI device 0000:7d:00.0 on NUMA socket 0
EAL:   probe driver: 19e5:a222 net_hns3
EAL: PCI device 0000:7d:00.1 on NUMA socket 0
EAL:   probe driver: 19e5:a221 net_hns3
EAL: PCI device 0000:7d:00.2 on NUMA socket 0
EAL:   probe driver: 19e5:a222 net_hns3
EAL: PCI device 0000:7d:00.3 on NUMA socket 0
EAL:   probe driver: 19e5:a221 net_hns3
APP: Finished Process Init.
Starting core 127

simple_mp > 
```

```
[root@localhost simple_mp]#   ./build/simple_mp -l 120-121  --proc-type=secondary
EAL: Detected 128 lcore(s)
EAL: Detected 4 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket_9543_169b3a72effc
EAL: Selected IOVA mode 'PA'
EAL: Probing VFIO support...
EAL: VFIO support initialized
EAL: PCI device 0000:05:00.0 on NUMA socket 0
EAL:   probe driver: 19e5:200 net_hinic
EAL: Cannot find resource for device
EAL: PCI device 0000:06:00.0 on NUMA socket 0
EAL:   probe driver: 19e5:200 net_hinic
EAL: PCI device 0000:7d:00.0 on NUMA socket 0
EAL:   probe driver: 19e5:a222 net_hns3
EAL: PCI device 0000:7d:00.1 on NUMA socket 0
EAL:   probe driver: 19e5:a221 net_hns3
EAL: PCI device 0000:7d:00.2 on NUMA socket 0
EAL:   probe driver: 19e5:a222 net_hns3
EAL: PCI device 0000:7d:00.3 on NUMA socket 0
EAL:   probe driver: 19e5:a221 net_hns3
APP: Finished Process Init.
Starting core 121

simple_mp > 
```

```
[root@localhost lib]# ls /var/run/dpdk/rte/ -al
total 15680
drwx------. 2 root root   1100 Aug 28 03:45 .
drwx------. 3 root root     60 Aug 26 03:45 ..
-rw-------. 1 root root  18816 Aug 28 03:45 config
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-0
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-0_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-1
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-1_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-2
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-2_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-3
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-3_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-0
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-0_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-1
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-1_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-2
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-2_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-3
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-3_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-0
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-0_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-1
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-1_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-2
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-2_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-3
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-3_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-0
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-0_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-1
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-1_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-2
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-2_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-3
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-3_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-0-0
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-0-0_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-0-1
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-0-1_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-1-0
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-1-0_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-1-1
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-1-1_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-2-0
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-2-0_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-2-1
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-2-1_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-3-0
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-3-0_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-3-1
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-3-1_9543
-rw-------. 1 root root 196608 Aug 28 03:45 fbarray_memzone
-rw-------. 1 root root  16576 Aug 28 03:45 hugepage_info
srwxr-xr-x. 1 root root      0 Aug 28 03:45 mp_socket
srwxr-xr-x. 1 root root      0 Aug 28 03:45 mp_socket_9543_169b3a72effc
[root@localhost lib]# 
```

```
[root@localhost simple_mp]#   ./build/simple_mp -l 126-127   --proc-type=primary
EAL: Detected 128 lcore(s)
EAL: Detected 4 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'PA'
EAL: Probing VFIO support...
EAL: VFIO support initialized
EAL: PCI device 0000:05:00.0 on NUMA socket 0
EAL:   probe driver: 19e5:200 net_hinic
```

```
simple_mp > Terminated
[root@localhost simple_mp]#   ./build/simple_mp -l 120-121  --proc-type=secondary
EAL: Detected 128 lcore(s)
```

```
[root@localhost lib]# ps -mo pid,tid,%cpu,psr -p 9530
   PID    TID %CPU PSR
  9530      -  6.9   -
     -   9530  0.2 126
     -   9531  0.0   9
     -   9532  0.0  10
     -   9533  6.6 127
[root@localhost lib]# ps -T  -p 9530
   PID   SPID TTY          TIME CMD
  9530   9530 pts/1    00:00:10 simple_mp
  9530   9531 pts/1    00:00:00 eal-intr-thread
  9530   9532 pts/1    00:00:00 rte_mp_handle
  9530   9533 pts/1    00:04:11 lcore-slave-127
[root@localhost lib]# ps -mo pid,tid,%cpu,psr -p 9543
   PID    TID %CPU PSR
  9543      -  6.6   -
     -   9543  0.0 120
     -   9544  0.0  10
     -   9545  0.0  11
     -   9546  6.6 121
[root@localhost lib]# ps -T  -p 9543
   PID   SPID TTY          TIME CMD
  9543   9543 pts/2    00:00:00 simple_mp
  9543   9544 pts/2    00:00:00 eal-intr-thread
  9543   9545 pts/2    00:00:00 rte_mp_handle
  9543   9546 pts/2    00:04:08 lcore-slave-121
[root@localhost lib]#
```

```
[root@localhost dpdk-stable-17.11.2]# ls /var/run/dpdk/rte/ -al
total 15680
drwx------. 2 root root   1100 Aug 28 03:45 .
drwx------. 3 root root     60 Aug 26 03:45 ..
-rw-------. 1 root root  18816 Aug 28 03:45 config
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-0
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-0_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-1
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-1_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-2
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-2_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-3
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-0-3_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-0
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-0_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-1
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-1_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-2
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-2_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-3
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-1-3_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-0
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-0_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-1
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-1_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-2
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-2_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-3
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-2-3_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-0
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-0_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-1
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-1_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-2
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-2_9543
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-3
-rw-------. 1 root root 458752 Aug 28 03:45 fbarray_memseg-2048k-3-3_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-0-0
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-0-0_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-0-1
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-0-1_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-1-0
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-1-0_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-1-1
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-1-1_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-2-0
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-2-0_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-2-1
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-2-1_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-3-0
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-3-0_9543
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-3-1
-rw-------. 1 root root  65536 Aug 28 03:45 fbarray_memseg-524288k-3-1_9543
-rw-------. 1 root root 196608 Aug 28 03:45 fbarray_memzone
-rw-------. 1 root root  16576 Aug 28 03:45 hugepage_info
srwxr-xr-x. 1 root root      0 Aug 28 03:45 mp_socket
srwxr-xr-x. 1 root root      0 Aug 28 03:45 mp_socket_9543_169b3a72effc
[root@localhost dpdk-stable-17.11.2]# ls
```

**退出** 

![img](https://img2020.cnblogs.com/blog/665372/202008/665372-20200828165503771-1478109892.png)



原文链接：https://www.cnblogs.com/dream397/category/1696181.html  原文作者：tycoon3
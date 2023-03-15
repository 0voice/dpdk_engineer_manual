# DPDK BPF

## DPDK BPF

DPDK 自版本 18.05 已集成了 librte_bpf, 主要利用rte_eth_rx_burst/rte_eth_tx_burst 回调函数机制, 执行eBPF字节码. 当前支持以下特性:

> base eBPF ISA (except tail-pointer)
> JIT (x86_64 and arm64 only)
> eBPF code verifier
> user-defined helper functions (64-bit only)
> RX/TX filter (加载 eBPF grog 作为 DPDK RX/TX 回调函数处理数据包, 单独跟每个RX/TX绑定)
> rte_mbuf access (64-bit only)

不支持的功能特性:

> cBPF
> eBPF MAP
> tail-pointer calls
> external function calls for 32-bit platforms

## DPDK BPF 执行流程

![img](https://pic1.zhimg.com/80/v2-934f2c993f0a9f1d4f31f920273a0934_720w.webp)

### Fedora

```text
sudo dnf install -y git gcc ncurses-devel elfutils-libelf-devel bc \
  openssl-devel libcap-devel clang llvm graphviz bison flex glibc-static
1
```

###  generate bpf prog

```text
examples/bpf/t1.c 提供了一个处理原始数据报文的例子, 检测到匹配IP地址与UDP目的端口5000则丢弃:

/* SPDX-License-Identifier: BSD-3-Clause
 * Copyright(c) 2018 Intel Corporation
 */

/*
 * eBPF program sample.
 * Accepts pointer to first segment packet data as an input parameter.
 * analog of tcpdump -s 1 -d 'dst 1.2.3.4 && udp && dst port 5000'
 * (000) ldh      [12]
 * (001) jeq      #0x800           jt 2    jf 12
 * (002) ld       [30]
 * (003) jeq      #0x1020304       jt 4    jf 12
 * (004) ldb      [23]
 * (005) jeq      #0x11            jt 6    jf 12
 * (006) ldh      [20]
 * (007) jset     #0x1fff          jt 12   jf 8
 * (008) ldxb     4*([14]&0xf)
 * (009) ldh      [x + 16]
 * (010) jeq      #0x1388          jt 11   jf 12
 * (011) ret      #1
 * (012) ret      #0
 *
 * To compile on x86:
 * clang -O2 -U __GNUC__ -target bpf -c t1.c
 *
 * To compile on ARM:
 * clang -O2 -I/usr/include/aarch64-linux-gnu/ -target bpf -c t1.c
 */

#include <stdint.h>
#include <net/ethernet.h>
#include <netinet/ip.h>
#include <netinet/udp.h>
#include <arpa/inet.h>

uint64_t
entry(void *pkt)
{
	struct ether_header *ether_header = (void *)pkt;

	if (ether_header->ether_type != htons(0x0800))
		return 0;

	struct iphdr *iphdr = (void *)(ether_header + 1);
	if (iphdr->protocol != 17 || (iphdr->frag_off & 0x1ffff) != 0 ||
			iphdr->daddr != htonl(0x1020304))
		return 0;

	int hlen = iphdr->ihl * 4;
	struct udphdr *udphdr = (void *)iphdr + hlen;

	if (udphdr->dest != htons(5000))
		return 0;

	return 1;
}
```

编译bpf字节码:

```text
# clang -O2 -U __GNUC__ -I${RTE_SDK}/${RTE_TARGET}/include -target bpf  -Wno-int-to-void-pointer-cast -c t1.c
# llvm-objdump  --arch=bpf -S t1.o
t1.o:	file format elf64-bpf
Disassembly of section .text:
0000000000000000 <entry>:
       0:	b7 00 00 00 00 00 00 00	r0 = 0
       1:	69 12 0c 00 00 00 00 00	r2 = *(u16 *)(r1 + 12)
       2:	55 02 0f 00 08 00 00 00	if r2 != 8 goto +15 <LBB0_6>
       3:	71 12 17 00 00 00 00 00	r2 = *(u8 *)(r1 + 23)
       4:	55 02 0d 00 11 00 00 00	if r2 != 17 goto +13 <LBB0_6>
       5:	69 12 14 00 00 00 00 00	r2 = *(u16 *)(r1 + 20)
       6:	55 02 0b 00 00 00 00 00	if r2 != 0 goto +11 <LBB0_6>
       7:	61 12 1e 00 00 00 00 00	r2 = *(u32 *)(r1 + 30)
       8:	55 02 09 00 01 02 03 04	if r2 != 67305985 goto +9 <LBB0_6>
       9:	07 01 00 00 0e 00 00 00	r1 += 14
      10:	71 12 00 00 00 00 00 00	r2 = *(u8 *)(r1 + 0)
      11:	67 02 00 00 02 00 00 00	r2 <<= 2
      12:	57 02 00 00 3c 00 00 00	r2 &= 60
      13:	0f 21 00 00 00 00 00 00	r1 += r2
      14:	69 11 02 00 00 00 00 00	r1 = *(u16 *)(r1 + 2)
      15:	b7 00 00 00 01 00 00 00	r0 = 1
      16:	15 01 01 00 13 88 00 00	if r1 == 34835 goto +1 <LBB0_6>
      17:	b7 00 00 00 00 00 00 00	r0 = 0

0000000000000090 <LBB0_6>:
      18:	95 00 00 00 00 00 00 00	exit
```

load/unload bpf prog

testpmd 提供了一组bpf命令用于验证bpf功能:

```text
testpmd> bpf-load rx|tx <portid> <queueid> <load-flags> <filename>
testpmd> bpf-unload rx|tx <portid> <queueid>
```

### bpf with rte_mbuf*

bpf入参为 rte_mbuf *

```text
bpf-load rx 0 0 M <path>/t3.o
...
bpf-load rx 0 n M <path>/t3.o
```

#### **bpf with raw packet**

bpf入参为原始报文数据

```text
bpf-load rx 0 0 J <path>/t4.o
...
bpf-load rx 0 n J <path>/t4.o
```

#### bpf with vm

bpf入参为原始报文数据, 使用 bpf vm 执行字节码:

```text
bpf-load rx 0 0 - <path>/t5.o
...
bpf-load rx 0 n - <path>/t5.o
```

#### unload bpf

```text
bpf-unload rx 0 0
...
bpf-unload rx 0 n
```

### Performance

硬件

```text
CPU: Intel(R) Xeon(R) Platinum 9242 CPU @ 2.30GHz
Mellanox Technologies MT2892 Family [ConnectX-6 Dx]
```

#### dpdk 21.05 testpmd:

```text
!/bin/sh

#EAL_ARGS+=" --log-level="lib.eal":8 --log-level=pmd:8 --log-level="pmd.net.mlx5":3 "
NR_Q=18

APP=./dpdk-testpmd-21.05
$APP -l 24-47 --socket-mem=4096,4096 -n 4  -w '54:00.1,dv_flow_en=1,mprq_en=1,rxqs_min_mprq=1,rx_vec_en=1' ${EAL_ARGS}  -- \
	-i  --rxq=${NR_Q} --txq=${NR_Q} --nb-cores=23 --forward-mode icmpecho --no-numa --enable-rx-cksum --auto-start --rxd=2048 --txd=2048 --burst=64
```

#### bpf prog

bpf 丢弃UDP目的端口为5000所有数据报文: t1.c 简化版, 移除了IP地址判断:

```text
#include <stdint.h>
#include <net/ethernet.h>
#include <netinet/ip.h>
#include <netinet/udp.h>
#include <arpa/inet.h>

uint64_t
entry(void *pkt)
{
	struct ether_header *ether_header = (void *)pkt;

	if (ether_header->ether_type != htons(0x0800))
		return 1;

	struct iphdr *iphdr = (void *)(ether_header + 1);
	if (iphdr->protocol != 17)
		return 1;

	int hlen = iphdr->ihl * 4;
	struct udphdr *udphdr = (void *)iphdr + hlen;
	if (udphdr->dest != htons(5000))
		return 0;

	return 0;
}
```

#### 编译:

```text
clang -O2 -U __GNUC__ -I${RTE_SDK}/${RTE_TARGET}/include -target bpf  -Wno-int-to-void-pointer-cast -c t4.c
```

### Load:

```text
bpf-load rx 0 0 J <path>/t4.o
bpf-load rx 0 1 J <path>/t4.o
bpf-load rx 0 2 J <path>/t4.o
bpf-load rx 0 3 J <path>/t4.o
bpf-load rx 0 4 J <path>/t4.o
bpf-load rx 0 5 J <path>/t4.o
bpf-load rx 0 6 J <path>/t4.o
bpf-load rx 0 7 J <path>/t4.o
bpf-load rx 0 8 J <path>/t4.o
bpf-load rx 0 9 J <path>/t4.o
bpf-load rx 0 10 J <path>/t4.o
bpf-load rx 0 11 J <path>/t4.o
bpf-load rx 0 12 J <path>/t4.o
bpf-load rx 0 13 J <path>/t4.o
bpf-load rx 0 14 J <path>/t4.o
bpf-load rx 0 15 J <path>/t4.o
bpf-load rx 0 16 J <path>/t4.o
bpf-load rx 0 17 J <path>/t4.o
```

### result

在当前测试硬件环境下, icmpecho 为 RX-DROP处理模式, 执行bpf字节码只做简单丢弃, 这种方式对性能几乎无影响, 可考虑用于插件处理数据包:

```cpp
testpmd> show port stats all

  ######################## NIC statistics for port 0  ########################
  RX-packets: 81360790320 RX-missed: 8141       RX-bytes:  4881647419320
  RX-errors: 0
  RX-nombuf:  0         
  TX-packets: 4          TX-errors: 0          TX-bytes:  360

  Throughput (since last show)
  Rx-pps:    149155140          Rx-bps:  71594467312
  Tx-pps:            0          Tx-bps:            0
  ############################################################################
 
bpf-unload rx 0 0
bpf-unload rx 0 1
bpf-unload rx 0 2
bpf-unload rx 0 3
bpf-unload rx 0 4
bpf-unload rx 0 5
bpf-unload rx 0 6
bpf-unload rx 0 7
bpf-unload rx 0 8
bpf-unload rx 0 9
bpf-unload rx 0 10
bpf-unload rx 0 11
bpf-unload rx 0 12
bpf-unload rx 0 13
bpf-unload rx 0 14
bpf-unload rx 0 15
bpf-unload rx 0 16
bpf-unload rx 0 17
testpmd> show port stats all

  ######################## NIC statistics for port 0  ########################
  RX-packets: 60151600493 RX-missed: 8141       RX-bytes:  3609096029700
  RX-errors: 0
  RX-nombuf:  0         
  TX-packets: 4          TX-errors: 0          TX-bytes:  360

  Throughput (since last show)
  Rx-pps:    149159900          Rx-bps:  71596752112
  Tx-pps:            0          Tx-bps:            0
  ############################################################################
testpmd> 
```

## Reference

> eBPF spec
> DPDK- Berkeley Packet Filter Library
> Awesome eBPF
> Cilium - BPF and XDP Reference Guide



原文链接：[DPhttps://blog.csdn.net/force_eagle/article/details/117365557](https://link.zhihu.com/?target=https%3A//blog.csdn.net/force_eagle/article/details/117365557)
# DPDK基础：bpftrace的使用

## 1.简介

bpftrace 是 Linux 高级追踪工具和语言。该工具基于 eBPF 和 BBC 实现了通过探针机制采集内核和程序运行的信息，然后用图表等方式将信息展示出来，帮助开发者找到隐藏较深的 Bug、安全问题和性能瓶颈。

bpftrace 使用 LLVM 作为后端，将脚本编译为 BPF 字节码，并利用 BCC 与 Linux BPF 系统交互，以及现有的 Linux 跟踪功能：内核动态跟踪（kprobes）、用户级动态跟踪（uprobes）、和跟踪点。bpftrace 语言的灵感来自 awk 和 C，以及 DTrace 和 SystemTap 等前置探测器。



![img](https://pic4.zhimg.com/80/v2-056842f4896d1ed9ee535d51e4c117ab_720w.webp)



项目地址是：

[https://github.com/iovisor/bpftrace](https://link.zhihu.com/?target=https%3A//github.com/iovisor/bpftrace)

## 2.下载安装

官方建议运行 Linux 4.9 内核或更高版本。某些工具可能适用于较旧的内核，但不再测试这些旧内核。

### 2.1Ubuntu

```text
# 适用于 Ubuntu 19.04 及更高版本
sudo apt-get install -y bpftrace

# 在 Ubuntu 16.04 及更高版本上，bpftrace 也可用作 snap 包
sudo snap install --devmode bpftrace
sudo snap connect bpftrace:system-trace
```

### 2.2Fedora

```text
# 对于 Fedora 28（及更高版本），bpftrace 已包含在官方仓库中。只需使用 dnf 安装软件包
sudo dnf install -y bpftrace
```

### 2.4Gentoo

```text
# 在 Gentoo 上，bpftrace 包含在官方仓库中。可以通过 emerge 安装
sudo emerge -av bpftrace
sudo emerge -av bpftrace
```

### 2.5其他

```text
# 其他如 Debian、openSUSE、CentOS 可以分别在以下链接中进行下载安装
https://tracker.debian.org/pkg/bpftrace
https://software.opensuse.org/package/bpftrace
https://github.com/fbs/el7-bpf-specs/blob/master/README.md#repository
```

### 2.6Docker

```text
$ docker run -v $(pwd):/output quay.io/iovisor/bpftrace:master-vanilla_llvm_clang_glibc2.23 \
  /bin/bash -c "cp /usr/bin/bpftrace /output"
$ ./bpftrace -V
v0.9.4
```

## 3.工具

bpftrace 包含各种工具，这些工具也可作为 bpftrace 语言编程的示例。简单介绍几个工具。

### 3.1bashreadline.bt

在系统范围内打印输入的 bash 命令：

```text
# ./bashreadline.bt
Attaching 2 probes...
Tracing bash commands... Hit Ctrl-C to end.
TIME      PID    COMMAND
06:40:06  5526   df -h
06:40:09  5526   ls -l
06:40:18  5526   echo hello bpftrace
06:40:42  5526   echooo this is a failed command, but we can see it anyway
^C
```

### 3.2biolatency.bt

I/O 延迟直方图：

```text
# ./biolatency.bt
Attaching 3 probes...
Tracing block device I/O... Hit Ctrl-C to end.
^C

@usecs:
[256, 512)             2 |                                                    |
[512, 1K)             10 |@                                                   |
[1K, 2K)             426 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[2K, 4K)             230 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@                        |
[4K, 8K)               9 |@                                                   |
[8K, 16K)            128 |@@@@@@@@@@@@@@@                                     |
[16K, 32K)            68 |@@@@@@@@                                            |
[32K, 64K)             0 |                                                    |
[64K, 128K)            0 |                                                    |
[128K, 256K)          10 |@                                                   |
```

### 3.3bitesize.bt

将磁盘 I/O 大小显示为直方图:

```text
# ./bitesize.bt
Attaching 3 probes...
Tracing block device I/O... Hit Ctrl-C to end.
^C
I/O size (bytes) histograms by process name:

@[cleanup]:
[4K, 8K)               2 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

@[postdrop]:
[4K, 8K)               2 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

@[jps]:
[4K, 8K)               1 |@@@@@@@@@@@@@@@@@@@@@@@@@@                          |
[8K, 16K)              0 |                                                    |
[16K, 32K)             0 |                                                    |
[32K, 64K)             2 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

@[kworker/2:1H]:
[0]                    3 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[1]                    0 |                                                    |
[2, 4)                 0 |                                                    |
[4, 8)                 0 |                                                    |
[8, 16)                0 |                                                    |
[16, 32)               0 |                                                    |
[32, 64)               0 |                                                    |
[64, 128)              0 |                                                    |
[128, 256)             0 |                                                    |
[256, 512)             0 |                                                    |
[512, 1K)              0 |                                                    |
[1K, 2K)               0 |                                                    |
[2K, 4K)               0 |                                                    |
[4K, 8K)               0 |                                                    |
[8K, 16K)              0 |                                                    |
[16K, 32K)             0 |                                                    |
[32K, 64K)             0 |                                                    |
[64K, 128K)            1 |@@@@@@@@@@@@@@@@@                                   |

@[jbd2/nvme0n1-8]:
[4K, 8K)               3 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)              0 |                                                    |
[16K, 32K)             0 |                                                    |
[32K, 64K)             2 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                  |
[64K, 128K)            1 |@@@@@@@@@@@@@@@@@                                   |

@[dd]:
[16K, 32K)           921 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
```

### 3.4cpuwalk.bt

查看哪些 CPU 正在执行进程：

```text
# ./cpuwalk.bt
Attaching 2 probes...
Sampling CPU at 99hz... Hit Ctrl-C to end.
^C

@cpu:
[0, 1)               130 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@    |
[1, 2)               137 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  |
[2, 3)                99 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                |
[3, 4)                99 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                |
[4, 5)                82 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                      |
[5, 6)                34 |@@@@@@@@@@@@                                        |
[6, 7)                67 |@@@@@@@@@@@@@@@@@@@@@@@@                            |
[7, 8)                41 |@@@@@@@@@@@@@@@                                     |
[8, 9)                97 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                |
[9, 10)              140 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[10, 11)             105 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@             |
[11, 12)              77 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@                        |
[12, 13)              39 |@@@@@@@@@@@@@@                                      |
[13, 14)              58 |@@@@@@@@@@@@@@@@@@@@@                               |
[14, 15)              64 |@@@@@@@@@@@@@@@@@@@@@@@                             |
[15, 16)              57 |@@@@@@@@@@@@@@@@@@@@@                               |
[16, 17)              99 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                |
[17, 18)              56 |@@@@@@@@@@@@@@@@@@@@                                |
[18, 19)              44 |@@@@@@@@@@@@@@@@                                    |
[19, 20)              80 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                       |
[20, 21)              64 |@@@@@@@@@@@@@@@@@@@@@@@                             |
[21, 22)              59 |@@@@@@@@@@@@@@@@@@@@@                               |
[22, 23)              88 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                    |
[23, 24)              84 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                     |
[24, 25)              29 |@@@@@@@@@@                                          |
[25, 26)              48 |@@@@@@@@@@@@@@@@@                                   |
[26, 27)              62 |@@@@@@@@@@@@@@@@@@@@@@@                             |
[27, 28)              66 |@@@@@@@@@@@@@@@@@@@@@@@@                            |
[28, 29)              57 |@@@@@@@@@@@@@@@@@@@@@                               |
[29, 30)              59 |@@@@@@@@@@@@@@@@@@@@@                               |
[30, 31)              56 |@@@@@@@@@@@@@@@@@@@@                                |
[31, 32)              23 |@@@@@@@@                                            |
[32, 33)              90 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                   |
[33, 34)              62 |@@@@@@@@@@@@@@@@@@@@@@@                             |
[34, 35)              39 |@@@@@@@@@@@@@@                                      |
[35, 36)              68 |@@@@@@@@@@@@@@@@@@@@@@@@@                           |
```

### 3.5tcpconnect.bt

跟踪 TCP 活动连接（connect()）:

```text
# ./tcpconnect.bt
TIME     PID      COMM             SADDR          SPORT  DADDR          DPORT
00:36:45 1798396  agent            127.0.0.1      5001   10.229.20.82   56114
00:36:45 1798396  curl             127.0.0.1      10255  10.229.20.82   56606
00:36:45 3949059  nginx            127.0.0.1      8000   127.0.0.1      37780
```

`开源前哨` 日常分享热门、有趣和实用的开源项目。参与维护 10万+ Star 的开源技术资源库，包括：Python、Java、C/C++、Go、JS、CSS、Node.js、PHP、.NET 等。





原文链接：https://zhuanlan.zhihu.com/p/457335441

原文作者：开源前哨
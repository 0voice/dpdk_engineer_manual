# SPDK块设备bdev简介

## 1.**介绍**

SPDK Bdev架构

![img](https://pic1.zhimg.com/80/v2-13b6036a1b4f47c4c46b3ed2952df3d8_720w.webp)

SPDK块设备层（通常简称为*bdev*）是一个C库，旨在等同于操作系统块存储层，该层通常位于传统内核存储堆栈中设备驱动程序的正上方。具体来说，此库提供以下功能：

- 一种可插拔模块API，用于实现与不同类型的块存储设备连接的块设备。
- NVMe，malloc（ramdisk），Linux AIO，virtio-scsi，Ceph RBD，Pmem和Vhost-SCSI Initiator等驱动程序模块。
- 用于枚举和声明SPDK块设备，然后在这些设备上执行操作（读取，写入，取消映射等）的应用程序API。
- 堆栈块设备以创建复杂I / O管道的工具，包括逻辑卷管理（lvol）和分区支持（GPT）。
- 通过JSON-RPC配置块设备。
- 请求排队，超时和重置处理。
- 多个无锁队列，用于将I / O发送到块设备。

Bdev模块创建抽象层，为所有设备提供通用API。用户可以使用可用的bdev模块或使用下面任何类型的设备创建自己的模块（有关详细信息，请参阅[编写自定义块设备模块](https://link.zhihu.com/?target=https%3A//spdk.io/doc/bdev_module.html)）。SPDK还提供了vbdev模块，可在现有的bdev上创建块设备。例如[逻辑卷](https://link.zhihu.com/?target=https%3A//spdk.io/doc/bdev.html%23bdev_ug_logical_volumes)或[SPDK GPT分区表](https://link.zhihu.com/?target=https%3A//spdk.io/doc/bdev.html%23bdev_ug_gpt)

## 2.**先决条件**

假设已经在平台上构建标准SPDK分发版。块设备层是一个C库，其中包含一个名为[bdev.h的](https://link.zhihu.com/?target=https%3A//spdk.io/doc/bdev_8h.html)公共头文件。以下章节中描述的所有SPDK配置都是使用JSON-RPC命令完成的。SPDK提供了一个基于python的命令行工具，用于发送位于的RPC命令`scripts/rpc.py`。用户可以通过使用`-h`或`--help`标记运行此脚本来列出可用命令。此外，用户可以通过运行直接从SPDK应用程序检索当前支持的RPC命令集`scripts/rpc.py get_rpc_methods`。通过添加`-h`flag作为命令参数，可以显示每个命令的详细帮助。

## 3.**通用RPC**

### 3.1**get_bdevs**

可以使用`get_bdevs`RPC命令获取当前可用的块设备列表，包括有关它们的详细信息。用户可以添加可选参数`name`以获取该名称bdev指定的详细信息。

response示例

```text
{
"num_blocks": 32768,
"assigned_rate_limits": {
"rw_ios_per_sec": 10000,
"rw_mbytes_per_sec": 20
},
"supported_io_types": {
"reset": true,
"nvme_admin": false,
"unmap": true,
"read": true,
"write_zeroes": true,
"write": true,
"flush": true,
"nvme_io": false
},
"driver_specific": {},
"claimed": false,
"block_size": 4096,
"product_name": "Malloc disk",
"name": "Malloc0"
}
```

### 3.2**set_bdev_qos_limit**

用户可以使用`set_bdev_qos_limit`RPC命令在现有bdev上启用，调整和禁用速率限制。支持两种类型的速率限制：IOPS和带宽。可以随时为指定的bdev启用，调整和禁用速率限制。bdev名称是此RPC命令的必需参数，`rw_ios_per_sec`并且`rw_mbytes_per_sec`必须至少指定其中一个。启用两个速率限制后，第一个满足限制将生效。可以指定值0以禁用相应的速率限制。用户可以使用`-h`或运行此命令以`--help`获取更多信息。

### 3.3**delete_bdev**

要删除以前创建的bdev用户，可以使用`delete_bdev`RPC命令。可以随时删除Bdev，这将由任何上层完全处理。作为参数，用户应提供bdev名称。此RPC命令应仅用于调试目的。要删除特定的bdev，请使用特定于其bdev模块的delete命令。

### 3.4**Ceph RBD**

SPDK RBD bdev驱动程序提供对Ceph RADOS块设备（RBD）的SPDK块层访问。通过librbd和librados库访问Ceph RBD设备，以访问Ceph导出的RADOS块设备。要创建Ceph，`construct_rbd_bdev`应该使用bdev RPC命令。

示例命令

```
rpc.py construct_rbd_bdev rbd foo 512
```

此命令将创建一个bdev，表示来自名为'rbd'的池中的'foo'图像。

要删除块设备表示，请使用delete_rbd_bdev命令。

```
rpc.py delete_rbd_bdev Rbd0
```

### **3.5Crypto Virtual Bdev模块**

加密虚拟bdev模块可以配置为为任何底层bdev提供静态数据加密。该模块依赖DPDK CryptoDev框架来提供所有加密功能。该框架仅支持许多不同的软件加密模块以及英特尔QAT板的硬件辅助支持。该框架还提供对密码，散列，身份验证和AEAD功能的支持。此时SPDK虚拟bdev模块仅支持密码，如下所示：

- AESN-NI多缓冲加密轮询模式驱动程序：RTE_CRYPTO_CIPHER_AES128_CBC
- 英特尔（R）QuickAssist（QAT）加密轮询模式驱动程序：RTE_CRYPTO_CIPHER_AES128_CBC（注意：QAT功能正常，但在硬件与SPDK CI系统完全集成之前，标记为实验性。）

为了支持使用bdev块偏移量（LBA）作为初始化向量（IV），加密模块将所有I / O分解为大小等于底层bdev的块大小的加密操作。例如，具有512B块大小的bdev的4K I / O将导致8次加密操作。

对于读取，提供给加密模块的缓冲区将用作未加密数据的目标缓冲区。但是，对于写入，临时暂存缓冲区用作加密的目标缓冲区，然后作为写缓冲区传递给底层bdev。这样做是为了避免加密原始源缓冲区中的数据，这可能在某些用例中引起问题。

示例命令

```
rpc.py construct_crypto_bdev -b NVMe1n1 -c CryNvmeA -d crypto_aesni_mb -k 0123456789123456
```

此命令将在NVMe bdev“NVMe1n1”之上创建一个名为“CryNvmeA”的加密vbdev，并将使用DPDK软件驱动程序“crypto_aesni_mb”和密钥“0123456789123456”。

要删除vbdev，请使用delete_crypto_bdev命令。

```
rpc.py delete_crypto_bdev CryNvmeA
```

## **4.GPT（GUID分区表）**

GPT虚拟bdev驱动程序默认启用，不需要任何配置。它将自动检测任何连接的bdev上的[SPDK GPT分区表](https://link.zhihu.com/?target=https%3A//spdk.io/doc/bdev.html%23bdev_ug_gpt)，并可能创建多个虚拟bdev。

## 5.**SPDK GPT分区表**

SPDK分区类型GUID是`7c5222bd-8f5d-4087-9c00-bf9843c7b58c`。现有的SPDK bdev可以通过NBD公开为Linux块设备，然后使用标准分区工具进行分区。分区后，需要删除bdev并再次附加，以便GPT bdev模块查看任何更改。必须先加载NBD内核模块。要创建NBD，bdev用户应该使用`start_nbd_disk`RPC命令。

示例命令

```
rpc.py start_nbd_disk Malloc0 /dev/nbd0
```

这将`Malloc0`在`/dev/nbd0`块设备下公开SPDK bdev 。

要删除NBD设备，用户应使用`stop_nbd_disk`RPC命令。

示例命令

```
rpc.py stop_nbd_disk /dev/nbd0
```

要显示完整或指定的nbd设备列表，用户应使用`get_nbd_disks`RPC命令。

示例命令

```
rpc.py stop_nbd_disk -n /dev/nbd0
```

## **6.使用NBD创建GPT分区表**

＃通过JSON-RPC将bdev Nvme0n1公开为内核块设备/ dev / nbd0
rpc.py start_nbd_disk Nvme0n1 / dev / nbd0

＃创建GPT分区表。
parted -s / dev / nbd0 mklabel gpt

＃添加占用50％可用空间的分区。
parted -s / dev / nbd0 mkpart MyPartition'0％''50％'

＃将分区类型更改为SPDK GUID。
\#sgdisk是gdisk包的一部分。
sgdisk -t 1：7c5222bd-8f5d-4087-9c00-bf9843c7b58c / dev / nbd0

＃停止NBD设备（停止导出/ dev / nbd0）。
rpc.py stop_nbd_disk / dev / nbd0

\#Now现在Nvme0n1配置了GPT分区表，并且第一个分区将自动公开为SPDK应用程序中的＃Nvme0n1p1。

## **7.iSCSI bdev**

SPDK iSCSI bdev驱动程序依赖于libiscsi，因此默认情况下不启用。要使用它，请使用额外的`--with-iscsi-initiator`配置选项构建SPDK 。

以下命令`iSCSI0`从`iqn.2016-06.io.spdk:init`作为报告的启动器IQN的给定iSCSI URL公开的单个LUN 创建bdev 。

```
rpc.py construct_iscsi_bdev -b iSCSI0 -i iqn.2016-06.io.spdk:init --url iscsi://127.0.0.1/iqn.2016-06.io.spdk:disk1/0
```

URL的格式如下： `iscsi://[<username>[%<password>]@]<host>[:<port>]/<target-iqn>/<lun>`

## **8.Linux AIO bdev**

Linux 异步 I/O 是 Linux 内核中提供的一个相当新的增强。它是 2.6 版本内核的一个标准特性，但是我们在 2.4 版本内核的补丁中也可以找到它。AIO 背后的基本思想是允许进程发起很多 I/O 操作，而不用阻塞或等待任何操作完成。稍后或在接收到 I/O 操作完成的通知时，进程就可以检索 I/O 操作的结果。

SPDK AIO bdev驱动程序通过Linux AIO提供对Linux内核块设备的SPDK块层访问或Linux文件系统上的文件。请注意，使用了O_DIRECT，因此绕过了Linux页面缓存。此模式可能与基于内核的典型目标接近，因为用户空间目标可以在不使用用户空间驱动程序的情况下获得。要创建AIO，`construct_aio_bdev`应使用bdev RPC命令。

示例命令

```
rpc.py construct_aio_bdev /dev/sda aio0
```

此命令`aio0`将从/ dev / sda 创建设备。

```
rpc.py construct_aio_bdev /tmp/file file 8192
```

此命令`file`将从/ tmp / file 创建块大小为8192的设备。

要删除aio bdev，请使用delete_aio_bdev命令。

```
rpc.py delete_aio_bdev aio0
```

## 9.**Malloc bdev**

malloc bdevs是ramdisks。由于它的性质，它们是不稳定的。它们是从为SPDK应用程序提供的巨大页面内存创建的。

## 10.**NULL**

SPDK null bdev驱动程序是一个虚拟块I / O目标，它会丢弃所有写入并返回未定义的读取数据。它可以用最小的块设备开销对bdev I / O堆栈的其余部分进行基准测试，并用于测试使用Malloc bdev无法轻松创建的配置。要创建Null bdev，应该使用construct_null_bdevRPC命令。

示例命令

```
rpc.py construct_null_bdev Null0 8589934592 4096
```

此命令将创建一个`Null0`块大小为4096 的8 PB的设备。

要删除null bdev，请使用delete_null_bdev命令。

```
rpc.py delete_null_bdev Null0
```

## 11.**NVMe bdev**

有两种方法可以在SPDK中基于NVMe设备创建块设备。第一种方式是连接本地PCIe驱动器，第二种方法是连接NVMe-oF设备。在这两种情况下，用户都应使用`construct_nvme_bdev`RPC命令来实现这一点。

示例命令

```
rpc.py construct_nvme_bdev -b NVMe1 -t PCIe -a 0000:01:00.0
```

此命令将在系统中创建物理设备的NVMe bdev。

```
rpc.py construct_nvme_bdev -b Nvme0 -t RDMA -a 192.168.100.1 -f IPv4 -s 4420 -n nqn.2016-06.io.spdk:cnode1
```

此命令将创建NVMe-oF资源的NVMe bdev。

要删除NVMe控制器，请使用delete_nvme_controller命令。

```
rpc.py delete_nvme_controller Nvme0
```

此命令将删除名为Nvme0的NVMe控制器。

## 12.**逻辑卷**

Logical Volumes库是一个灵活的存储空间管理系统。它允许在其他bdev之上创建和管理具有可变大小的虚拟块设备。SPDK逻辑卷库建立在[Blobstore程序员指南](https://link.zhihu.com/?target=https%3A//spdk.io/doc/blob.html)之上。有关详细说明，请参阅[逻辑卷](https://link.zhihu.com/?target=https%3A//spdk.io/doc/logical_volumes.html%23lvol)。

## 13.**逻辑卷存储**

在创建任何逻辑卷（lvols）之前，必须首先在选定的块设备上创建lvol存储。Lvol商店是lvols容器，负责管理lvol bdevs的底层bdev空间分配和存储元数据。要创建lvol存储，用户应该使用`construct_lvol_store`RPC命令。

示例命令

```
rpc.py construct_lvol_store Malloc2 lvs -c 4096
```

这将创建`lvs`以簇大小4096 命名的`Malloc2`lvol 存储，构建在bdev之上。作为响应，将向用户提供uuid，这是唯一的lvol商店标识符。

用户可以使用`get_lvol_stores`RPC命令获取可用的lvol存储列表（没有可用的参数）。

response示例

{
"uuid": "330a6ab2-f468-11e7-983e-001e67edf35d",
"base_bdev": "Malloc2",
"free_clusters": 8190,
"cluster_size": 8192,
"total_data_clusters": 8190,
"block_size": 4096,
"name": "lvs"
}

要删除lvol store，用户应该使用`destroy_lvol_store`RPC命令。

示例命令

```
rpc.py destroy_lvol_store -u 330a6ab2-f468-11e7-983e-001e67edf35d
rpc.py destroy_lvol_store -l lvs
```

## **14.Lvols**

要在现有lvol存储上创建lvol，用户应使用`construct_lvol_bdev`RPC命令。每个创建的lvol都将由新的bdev表示。

示例命令

```
rpc.py construct_lvol_bdev lvol1 25 -l lvs
rpc.py construct_lvol_bdev lvol2 25 -u 330a6ab2-f468-11e7-983e-001e67edf35d
```

## 15.**Passthru**

SPDK Passthru虚拟块设备模块用作如何编写虚拟块设备模块的示例。它实现了vbdev模块所需的功能，并演示了一些其他基本功能，例如每个I / O上下文的使用。

示例命令

```
rpc.py construct_passthru_bdev -b aio -p pt
rpc.py delete_passthru_bdev pt
```

## 16.**Pmem**

SPDK pmem bdev驱动程序使用pmemblk池作为块I / O操作的目标。有关Pmem内存的详细信息，请参阅[http://pmem.io](https://link.zhihu.com/?target=http%3A//pmem.io/)网站上的PMDK文档。首先，用户需要配置SPDK以包含PMDK支持：

```
configure --with-pmdk
```

要创建用于SPDK用户的pmemblk池，应使用`create_pmem_pool`RPC命令。

示例命令

```
rpc.py create_pmem_pool /path/to/pmem_pool 25 4096
```

要获取有关已创建的pmem池文件的信息，用户可以使用`pmem_pool_info`RPC命令。

示例命令

```
rpc.py pmem_pool_info /path/to/pmem_pool
```

要删除pmem池文件，用户可以使用`delete_pmem_pool`RPC命令。

示例命令

```
rpc.py delete_pmem_pool /path/to/pmem_pool
```

要基于pmemblk池文件创建bdev，用户应使用`construct_pmem_bdev`RPC命令。

示例命令

```
rpc.py construct_pmem_bdev /path/to/pmem_pool -n pmem
```

要删除块设备表示，请使用delete_pmem_bdev命令。

```
rpc.py delete_pmem_bdev pmem
```

## **17.Virtio Block**

Virtio-Block驱动程序允许从Virtio-Block设备创建SPDK bdev。

以下命令创建一个Virtio-Block设备`VirtioBlk0`，该设备以`/tmp/vhost.0`SPDK [vhost Target](https://link.zhihu.com/?target=https%3A//spdk.io/doc/vhost.html)直接公开的vhost-user套接字命名。optional `vq-count`和`vq-size`params指定要使用的请求队列数和队列深度。

```
rpc.py construct_virtio_dev --dev-type blk --trtype user --traddr /tmp/vhost.0 --vq-count 2 --vq-size 512 VirtioBlk0
```

该驱动程序也可以在基于QEMU的VM中使用。以下命令创建一个名为`VirtioBlk0`Virtio PCI设备的Virtio Block设备`0000:00:01.0`。将从PCI配置空间自动读取整个配置。它将反映传递给QEMU的vhost-user-scsi-pci设备的所有参数。

```
rpc.py construct_virtio_dev --dev-type blk --trtype pci --traddr 0000:01:00.0 VirtioBlk1
```

可以使用以下命令删除Virtio-Block设备

```
rpc.py remove_virtio_bdev VirtioBlk0
```

## **18.Virtio SCSI**

Virtio-SCSI驱动程序允许从Virtio-SCSI LUN创建SPDK块设备。

Virtio-SCSI bdev的构造方式与Virtio-Block的构造方式相同。

```
rpc.py construct_virtio_dev --dev-type scsi --trtype user --traddr /tmp/vhost.0 --vq-count 2 --vq-size 512 VirtioScsi0
rpc.py construct_virtio_dev --dev-type scsi --trtype pci --traddr 0000:01:00.0 VirtioScsi0
```

每个Virtio-SCSI设备最多可以导出64个名为VirtioScsi0t0~VirtioScsi0t63的块设备，每个SCSI设备一个LUN（LUN0）。以上2个命令将输出所有暴露的bdev的名称。

可以使用以下命令删除Virtio-SCSI设备

```
rpc.py remove_virtio_bdev VirtioScsi0
```

删除Virtio-SCSI设备将破坏其所有bdev。







原文链接：[https://www.cnblogs.com/whl320124/a](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/whl320124/articles/10064186.html)  原文作者： 海之心1213
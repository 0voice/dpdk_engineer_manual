# spdk + fio测试nvme 设备的性能

## 1.背景

- spdk ：是一个基于dpdk的存储开发kit，这里主要利用了它提供的用户态nvme driver —— [spdk链接](https://link.jianshu.com?t=http%3A%2F%2Fwww.spdk.io%2F)
- fio: io测试工具，提供丰富的参数，可以构造复杂的io pattern

fio的测试对象可以是块设备、文件等，在spdk的使用过程中会unbind默认的nvme driver，所以在系统中是看不到nvme块设备的，在spdk中可以通过fio_plugin的方式，将spdk的用户态driver部分的io功能打包成一个ioengine提供给fio使用，可以综合spdk的高性能和fio提供的复杂场景。

## 2.使用步骤

### 2.1下载dpdk、spdk、fio并解压

用最新的版本即可

### 2.2编译fio

```go
cd fio_dir
./configure
make&&make install
```

### 2.3编译dpdk

```objectivec
vim <dpdk_dir>/config/defconfig_x86_64-native-linuxapp-gcc
#增加一行
EXTRA_CFLAGS=-fPIC
#回到<dpdk_dir>编译dpdk
make install T=x86_64-native-linuxapp-gcc DESTDIR=.
```

### 2.4编译spdk

```ruby
cd <spdk_dir>
./configure --with-fio=/root/Downloads/fio-fio-3.3/ --with-dpdk=/root/Downloads/dpdk-17.11/x86_64-native-linuxapp-gcc
#修改<spdk_dir>/CONFIG文件
CONFIG_FIO_PLUGIN=y
FIO_SOURCE_DIR=fio的目录
#编译spdk
make DPDK_DIR=/root/Downloads/dpdk-17.11/x86_64-native-linuxapp-gcc
```

### 2.5unbind nvme driver替换为vfio

```bash
cd <spdk_dir>/scripts
sh setup.sh
```

### 2.6使用fio执行测试

```jsx
LD_PRELOAD=/root/Downloads/spdk-18.01/examples/nvme/fio_plugin/fio_plugin /root/Downloads/fio-fio-3.3/fio example_config.fio
```

### 2.7 jobfile的例子

这个jobfile是spdk中提供的例子

```csharp
[global]
ioengine=spdk
thread=1
group_reporting=1
direct=1
verify=0
time_based=1
ramp_time=0
norandommap=1
runtime=60
iodepth=32
rw=randwrite
bs=3000b
[test]
numjobs=1
filename=trtype=PCIe traddr=0000.01.00.0 ns=1
```

## 3.注意

- 由于spdk已经提前从系统中unbind了nvme设备，所以/dev/下是没有nvme设备的，必须指定到具体的pcie的接口上，fio的jobfile中的filename需要使用key=val的模式（这个地方有点怪异）
  - trttype：有pcie和rdma两种
  - traddr：pcie的“**domain:bus:device:function**”格式，可以用**lspci**命令查看对应的nvme设备在那个总线上，一般单台机器的domain都是0000
  - ns：namespace的id
- spdk做性能测试时，对每个namespace会绑定一个lcore，所以fio的thread只能等于1
- fio测试random的io时，需要设置norandommap=1 ，防止fio的random map影响性能





原文链接：https://www.jianshu.com/p/eeaf81ffb7b5  原文作者：Stansosleepy
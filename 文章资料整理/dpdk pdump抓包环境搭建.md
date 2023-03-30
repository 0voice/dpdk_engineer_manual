# dpdk pdump抓包环境搭建

 intel的e1000网卡被uio驱动接管后， 在linux下通过ifconfig是看不到网卡信息的，也就无法使用tcpdump来抓包。 dpdk在16.7版本引入了pdump抓包工具，可以使用这个工具来抓包，抓到的包可以在windows下通过wireshark来分析。pdump抓包工具需要依赖libpcap和libpcap-dev两个库。 下面来看下如何安装pdump工具。

## 1.安装libpcap

        我这里安装的libpcap版本为1.4.0， 可以在官网选择任意一个版本进行下载。下载好后进行解压。

```c++
root@apelife:/home/apelife/work/packet# tar -zxvf libpcap-1.4.0.tar.gz -C /home/apelife/work/bin
```

 解压后在libpcap源码目录下执行configure 进行配置

```c++
root@apelife:/home/apelife/work/bin/libpcap# ./configure
```

​    如果出现下面这个错误，则说明缺少flex语法分析器，需要进行安装。

![img](https://img-blog.csdnimg.cn/2019063017305055.png)

 执行下面命令开始安装flex语法分析器。

```c++
root@apelife:/home/apelife/work/bin/libpcap# sudo apt-get install flex
```

 安装好后重新执行./configure, 之后执行make开始编译

```c++
root@apelife:/home/apelife/work/bin/libpcap# ./configure
root@apelife:/home/apelife/work/bin/libpcap# make
```

 如果在编译过程中出现下面这个错误，则需要安装byacc

![img](https://img-blog.csdnimg.cn/20190630173550254.png)

```c++
root@apelife:/home/apelife/work/bin/libpcap# sudo apt-get install -y byacc
```

最后重新执行make就好了，到此libpcap安装完成。

## **2.安装libpcap-dev**

​    可以直接通过apt-get install libpcap-dev来进行安装

```c++
root@apelife:/home/apelife/work/bin/libpcap# sudo apt-get install libpcap-dev
```

## **3.pdump的安装**

​    由于在dpdk 16.7版本才引入这个pdump工具， 我就从dpdk官网下载了17.02.1版本的代码来编译。先来看下pdump整体结构。在dpdk源码目录下会有一个pdump工具的源码。

![img](https://img-blog.csdnimg.cn/2019063017452959.png)

   编译好pdump后，就会生成一个dpdk-pdump工具。下面开始来看下如何安装pdump![img](https://img-blog.csdnimg.cn/2019063017463031.png)

### 3.1pdump安装目录的设置

        新建一个dpdkenv文件，里面设置好dpdk需要的环境变量，其中DESTDIR为dpdk生成的一些工具目录，其中就包含了dpdk-pdump工具。 mytool是我在dpdk源码目录手动创建，我将dpdk生成的工具都放在/home/apelife/work/bin/dpdk/mytool目录

```c++
root@apelife:/home/apelife/work/bin/dpdk# cat dpdkenv 
export RTE_SDK=`pwd`
export RTE_TARGET=i686-native-linuxapp-gcc
export EXTRA_CFLAGS="-O0 -g"
export DESTDIR=/home/apelife/work/bin/dpdk/mytool
```

  设置好环境变量后，执行source dpdkenv， 让这个文件里面设置的环境变量立即生效，而无需重启设备才生效。

### 3.2开启编译选项

        需要修改/home/apelife/work/bin/dpdk/config/common_base文件，将下面两个编译选项设置为y，使得这两个功能生效。

```c++
CONFIG_RTE_LIBRTE_PMD_PCAP=y
CONFIG_RTE_LIBRTE_PDUMP=y
```

### 3.3编译dpdk与pdump

       到这个步骤，所有需要设置的操作都已经完成了。接下里就可以编译dpdk了，编译dpdk的时候会生成dpdp-pdump工具，因此dpdk-pdump这个工具无需额外编译。 dpdk两种编译方式可以参考虚拟机dpdk环境搭建这篇文章。例如我的ubuntun操作系统是32位的，则执行make install T=i686-native-linuxapp-gcc。编译好后，就可以在/home/apelife/work/bin/dpdk/mytool/bin生成dpdk-pdump工具

## 4.pdump抓包实例

        在这里以l2fwd二层转发为例，看下如何抓包。简单说明下抓包原理。首先l2fwd作为主进程运行， 而dpdk-pdump做为从进程运行，两个进程通过ring队列进行通信。例如l2fwd将报文通过ring队列发给dpdk-pdump，进而可以抓包到。 详细的抓包原理在后续源码分析过程中再来详细分析，这里只需要会用就可以了。

1.修改l2fwd的源码，在rte_eal_init后面加上rte_pdump_init初始化抓包框架，使得l2fwd以主进程方式运行

![img](https://img-blog.csdnimg.cn/20190630180743978.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FwZUxpZmU=,size_16,color_FFFFFF,t_70)

2、运行l2fwd主进程

```c++
root@apelife:/home/apelife/work/bin/dpdk/examples/l2fwd# ./l2fwd -c 0xf -n 2 -- -q 2 -p 0xf
```

3、运行dpdk-pdump抓包进程(也就是从进程)

        执行下面命令开始抓包，其中port为需要抓包的那个端口。我这个例子中指定了4个网卡，因此port的范围从0-3； queue表示抓某个网卡的某个队列(通常一个网卡有多个接收队列)，如果为*则表示抓这个网卡所有队列的报文； rx-dev指定抓包存放路径。

```c++
root@apelife:/home/apelife/work/bin/dpdk/mytool/bin# ./dpdk-pdump -- --pdump 'port=0,queue=*,rx-dev=/home/apelife/mypacket/dpdk.pcap'
```

![img](https://img-blog.csdnimg.cn/20190630184437441.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FwZUxpZmU=,size_16,color_FFFFFF,t_70)

 ctrl+c停止抓包，抓到报文后，就可以把报文在windows下通过[wireshark](https://so.csdn.net/so/search?q=wireshark&spm=1001.2101.3001.7020)来分析。







原文链接：https://blog.csdn.net/ApeLife/article/details/94338446?spm

原文作者：ApeLife
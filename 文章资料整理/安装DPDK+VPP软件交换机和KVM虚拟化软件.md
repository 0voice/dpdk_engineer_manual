# 安装DPDK+VPP软件交换机和KVM虚拟化软件 

## 1. 安装 Vagrant and VirtualBox 虚拟机工具

在宿主机系统中卸载旧版xen-tool和virtualbox。

```shell
sudo apt-get remove -y xen-tools virtualbox
```

到[Oracel VirtualBox官方网站](https://www.virtualbox.org/wiki/Linux_Downloads)，下载安装最新版VirtualBox虚拟机系统。

VirtualBox Extension Pack 提供对USB2.0、USB3.0、NVMe等的支持。因此，对于提升VirtualBox的性能很有帮助。请到[VirtualBox的官网](https://www.virtualbox.org/wiki/Downloads)下载对应你的VirtualBox版本的Extension Pack。如何确定VirtualBox的版本？进入Virtual Box后，菜单栏上点击“帮助”，然后点关于。下载号是一个后缀名为.vbox-extpack的文件。直接双击，VirtualBox会打开。直接安装就可以了。具体参考[网页指南](https://blog.csdn.net/jumpingpig/article/details/104103838)。

参考链接：https://fd.io/docs/vpp/master/reference/vppvagrant/installingvboxvagrant

安装Vagrant：到[Vagrant官网](https://releases.hashicorp.com/vagrant/)下载最新版Vagrant deb安装文件，点击安装。下面测试vagrant安装。

```shell
$ vagrant --version
Vagrant 2.2.17
```

## 2. 安装QEMU虚拟化和libvirt虚拟机管理软件

安装依赖项目

```shell
sudo apt-get update
sudo apt-get install -y qemu qemu-system-x86 ovmf
sudo apt-get install -y virt-manager libvirt-daemon
```

为避免后面在非`/home`目录下启动QEMU-KVM虚拟机时 `Permission denied` 的问题，打开配置文件 `/etc/libvirt/qemu.conf`

```shell
sudo gedit /etc/libvirt/qemu.conf
```

在下方`user` 和 `group` 的位置处改为 `root` ，其余位置不变，修改后保存：

```shell
# The user for QEMU processes run by the system instance. It can be
# specified as a user name or as a user id. The qemu driver will try to
# parse this value first as a name and then, if the name doesn't exist,
# as a user id.
#
# Since a sequence of digits is a valid user name, a leading plus sign
# can be used to ensure that a user id will not be interpreted as a user
# name.
#
# Some examples of valid values are:
#
#       user = "qemu"   # A user named "qemu"
#       user = "+0"     # Super user (uid=0)
#       user = "100"    # A user named "100" or a user with uid=100
#
user = "root"

# The group for QEMU processes run by the system instance. It can be
# specified in a similar way to user.
group = "root"

# Whether libvirt should dynamically change file ownership
# to match the configured user/group above. Defaults to 1.
# Set to 0 to disable file ownership changes.
#dynamic_ownership = 1
```

之后重启 `libvirtd` 服务：

```shell
sudo service libvirtd restart
```

这样就可以在非/home目录下启动QEMU-KVM虚拟机。当然也有其他方法，可以参考[链接](https://github.com/jedi4ever/veewee/issues/996)。

## 3. 从源代码安装VPP（推荐）

参考官方文档：https://wiki.fd.io/view/VPP/Pulling,_Building,_Running,_Hacking_and_Pushing_VPP_Code

### 3.1 准备全局的配置项和安装依赖

注意：要卸载旧的从APT源装的vpp。

```shell
 sudo apt-get remove vpp
```

1. 在`~/.bashrc`或者`~/.zshrc`的末尾插入以下的环境变量配置项。

   ```shell
   export VPP_VERSION=v21.10
   ```

   执行`source ~/.bashrc`或者`source ~/.zshrc`命令。

2. 确认Python Pip的源配置，编辑`/home/user/.pip/pip.conf`，在全局将aliyun的源设置为可信任trusted-host。

   ```ini
   [global]
   index-url = http://mirrors.aliyun.com/pypi/simple/
   trusted-host = mirrors.aliyun.com
   ```

3. 下载最新版本的VPP源代码

   ```shell
   git clone https://github.com/FDio/vpp
   cd vpp
   git checkout $VPP_VERSION
   ```

4. 安装依赖的x86平台的nasm汇编语言编译程序。VPP需要至少2.14版本以上的nasm，较为老旧版本的ubuntu自带为2.13，因此需要手动导入第三方ppa源。

   ```shell
   sudo add-apt-repository ppa:team-xbmc/ppa
   sudo apt-get update
   sudo apt-get install nasm
   ```

5. 用如下命令安装VPP的其他依赖。

   ```shell
   make install-deps
   make install-ext-deps
   ```

   这一步的主要作用就是下载vpp环境所需的依赖包，这一步相当重要，一定要仔细查看执行情况，若有问题，一定要一一解决，不然在后面编译问题多，基本就是在环境搭建的第一次执行。

   我们遇到如下的版本兼容性问题：

   ```shell
   The following packages have unmet dependencies:
    libsctp-dev : Depends: libsctp1 (= 1.0.17+dfsg-2) but 1.0.18+dfsg-1 is to be installed
   E: Unable to correct problems, you have held broken packages.
   ```

   需要到[ubuntu packages搜索网站](https://pkgs.org/download/libsctp1)，下载libsctp1版本1.0.17+dfsg-2的Binary Package包（注意观察Ubuntu系统版本是18.04，CPU指令架构是amd64代表intel x86架构），发现链接都是来自[archive ubuntu官方网站](http://archive.ubuntu.com/ubuntu/pool/main/l/lksctp-tools/libsctp1_1.0.17+dfsg-2_amd64.deb)。然后，用如下命令卸载ubuntu缺省版本的libsctp1，并用dpkg命令安装指定1.0.17+dfsg-2版本的libsctp1。

   ```shell
   sudo apt-get remove libsctp1
   sudo dpkg -i libsctp1_1.0.17+dfsg-2_amd64.deb
   ```

### 3.2 编译VPP源代码

1. 用如下命令编译VPP源代码。

   ```shell
   make build        # or `make build-release`
   ```

   开发一般编译测试版，执行make build编译vpp主程序，make wipe清除，带有release的就是发型版本了。dpdk不用专门下载，在执行make build的时候会自动下载dpdk到vpp/dpdk目录下。只要执行make build后，vpp基本就编译成功。

   我们遇到报告如下错误。主要原因是Ubuntu系统默认的python是2.7版本。记着一定要按之前的步骤把python命令关联到python3的环境。

   ```vhdl
   -- Install configuration: "debug"
     File "home/wxg/vpp/vpp/build-root/install-vpp_debug-native/vpp/lib/python2.7/site-packages/vpp_papi/vpp_papi.py", line 391
       def __init__(self, *, apifiles=None, testmode=False, async_thread=True,
                           ^
   SyntaxError: invalid syntax
   
     File "build/bdist.linux-x86_64/egg/vpp_papi/vpp_papi.py", line 391
       def __init__(self, *, apifiles=None, testmode=False, async_thread=True,
                           ^
   SyntaxError: invalid syntax
   ```

2. 用python3解决上面的错误以后，重新用如下命令编译VPP源代码，编译通过。

   ```shell
   make build        # or `make build-release`
   ```

   再执行make run命令运行软件交换机，可以进入vpp命令行控制台。可以用`show version`命令查看目前的vpp版本号。

   ```shell
   $ make run
   
   WARNING: STARTUP_CONF not defined or file doesn't exist.
            Running with minimal startup config:  unix { interactive cli-listen /run/vpp/cli.sock gid 1000 } dpdk { no-pci } \n
   snort                [debug ]: initialized
   snort                [debug ]: snort listener /run/vpp/snort.sock
   dpdk/cryptodev       [warn  ]: dpdk_cryptodev_init: Failed to configure cryptodev
   vat-plug/load        [error ]: vat_plugin_register: oddbuf plugin not loaded...
       _______    _        _   _____  ___ 
    __/ __/ _ \  (_)__    | | / / _ \/ _ \
    _/ _// // / / / _ \   | |/ / ___/ ___/
    /_/ /____(_)_/\___/   |___/_/  /_/    
   
   DBGvpp# show version
   vpp v22.02-rc0~0-g192c55f2e built by wxg on inspur1 at 2021-10-25T03:33:21
   DBGvpp# quit
   ```

3. 安装vpp编译好的deb软件包。注意：这个要通过一定要Ubuntu系统默认的语言是英语，否则安装过程无法交互。

   ```shell
   sudo vim /etc/default/locale
   ```

   改为如下内容，然后重启Ubuntu系统:

   ```ini
   LANG="en_US.UTF-8"
   LANGUAGE="en_US:en"
   LC_ALL="en_US.UTF-8"
   ```

   再进到原先的vpp源代码目录。用如下命令安装软件包。

   ```shell
   cd build-root
   make
   sudo dpkg -i *.deb
   cd ..
   ```

   测试vpp安装。

   ```shell
   $ sudo vppctl "show version"
   vpp v22.02-rc0~0-g192c55f2e built by wxg on inspur1 at 2021-10-25T03:21:57
   ```

## 4. 在Ubuntu 18.04系统上从APT源安装VPP（可选）

### 4.1 从官网下载安装新版本的Intel DPDK源代码，编译安装

采用较新的DPDK `$DPDK_VERSION` (LTS)，或更早的19.11.8 (LTS)版本

```shell
export DPDK_VERSION=20.11
```

- [DPDK Release 20.11版本说明](https://blog.csdn.net/weixin_37097605/article/details/111602399)
- dpdk从20.02版本后默认不编译老版本的igb_uio.ko驱动，缺省只是编译vfio.ko新版驱动
- 新版本dpdk 20 仅支持meson和ninjia编译工具，移除了usertools下的数字脚本
- [Ninja构建系统入门](https://www.cnblogs.com/sandeepin/p/ninja.html)

安装ninjia的编译工具。

```shell
sudo apt-get install python3 python3-pip ninja-build -y
```

脚本安装需要 meson。

```shell
sudo pip3 install meson
pip3 install --user meson
```

下载编译dpdk。

```shell
export PATH=~/.local/bin:$PATH
wget http://fast.dpdk.org/rel/dpdk-$DPDK_VERSION.tar.xz
tar -xf dpdk-$DPDK_VERSION.tar.xz
cd dpdk-$DPDK_VERSION
meson build
ninja -C build
mkdir -p /dev/hugepages
mountpoint -q /dev/hugepages || mount -t hugetlbfs nodev /dev/hugepages
sudo sh -c 'echo 4096 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages'
sudo sh -c 'echo 4096 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages'
```

也可以在build的时候指定包含example

```shell
meson -Dexamples=all build
```

以接收模式跑一下testpmd进行测试（testpmd是一个dpdk转发工具，可以只接收），需要回环链接光口nic0和nic1。

```shell
build/app/dpdk-testpmd -c7 --vdev=net_pcap0,iface=nic0 --vdev=net_pcap1,iface=nic1 -i --nb-cores=2 --nb-ports=2 --total-num-mbufs=2048
```

### 4.2 用APT安装Prebuilt

参考链接：https://fd.io/vppproject/vppinstalling/

```shell
sudo sh -c 'echo "deb [trusted=yes] https://packagecloud.io/fdio/release/ubuntu bionic main" > /etc/apt/sources.list.d/99fd.io.list'
curl -L https://packagecloud.io/fdio/release/gpgkey | sudo apt-key add -
sudo apt-get update
sudo apt-get install -y vpp vpp-plugin-core vpp-plugin-dpdk
sudo apt-get install -y vpp-api-python python3-vpp-api vpp-dbg vpp-dev
```

测试vpp安装。

```shell
$ sudo vppctl "show version"
vpp v21.01-release built by root on fcb1bae62b24 at 2021-01-27T16:06:22
```

 





原文链接：https://www.cnblogs.com/cscshi/p/16584073.html  原文作者：人生有迹


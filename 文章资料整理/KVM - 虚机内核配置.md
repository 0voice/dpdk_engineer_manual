# KVM - 虚机内核配置

## 1.缘起

笔者最近分别购买了一台腾讯云和百度云的机器，都是一年期的，配置和价格分别如下：

|      | 腾讯云                  | 百度云                  |
| ---- | ----------------------- | ----------------------- |
| 配置 | 2 核，2G 内存，40G 硬盘 | 2 核，4G 内存，80G 硬盘 |
| 价格 | 50 元                   | 78 元                   |

似乎性价比都差不多，但是，用 "lscpu" 命令一看就会发现，腾讯云机器的 "Thread(s) per core" 这一项为 1，而百度云为 2，这意味着后者是开启了 hyper-threading 的，性能一般只相当于 1.1-1.2 个 CPU（大家注意以后避免踩坑），而腾讯的是真两核（要不然怎么叫良心云呢）。

另外从控制台上，虽然是轻量应用版本，但是腾讯云也支持 snapshot 功能，而百度云好像没有。不能打快照的话，都不敢在上面做过多的试验性操作。

有时想想，这么低的价格，怕是连电费都不够，还别说为了保持产品的竞争力，隔几年就需要对昂贵的 CPU 进行升级换代，这可是一笔不小的开销。

云服务器普遍存在“超卖”的情况，毕竟不可能售出的所有虚机都同时使用（银行也是无法同时满足每一个储户的取现需求）。会不会是像航空公司一样，多卖出去的一个座位，只是几十元的「边际成本」（比如这两年的“随心飞”产品）。当使用一年之后，由于数据迁移的不便，加上习惯和黏性，赌你还会接着续费使用？

采用虚拟机模式构建的云服务，算是一种共享经济了，但根据目前了解到的一些信息，在公司级项目的层面，由于各种各样的原因，目前购买云主机的成本并不见得比自建机房低。

不过“云”还有一个好处，就是随时随地的访问，周末写文章常常需要一个 Linux 的访问环境，买台云服务器的话，可以省去背电脑回家的麻烦。

在到手的这 2 台机器上，笔者都基于 5.17.0 内核，使用 menuconfig 默认配置进行了编译，结果生成的镜像占用空间较多。由于磁盘空间受限，笔者就想着能不能精简下配置，把不必要的模块都去掉。前提一是能在采用 KVM 的虚机上运行，二是虚机里能跑 docker。

## 2.动手

笔者最开始使用的是 defconfig，为了加快后续编译，make 时加上了 ccache（参考[这篇文章](https://link.zhihu.com/?target=http%3A//nickdesaulniers.github.io/blog/2018/06/02/speeding-up-linux-kernel-builds-with-ccache/)）。替换内核是高危操作，所以保险的做法是在 make install 之前打个快照，并且用 "grubby --set-default" 将老内核作为默认启动项（因为启动失败后，网页版的 VNC 界面很可能无法切换内核）。

### 2.1.支持 KVM

之后，切到新内核的启动卡在了 "System hang after output: Reached target Basic System"，这可能是 initramfs 里的磁盘驱动缺失导致。

因为目前 KVM 普遍采用 VIRTIO 作为虚机的驱动，所以得在配置里包含相关部分，主要是 BLK, SCSI, PCI 和 NET，以 block 为例，配置路径如下：

> Device Drivers ---> Block devices --> Virtio block driver

此外，作为 guest 虚机，还需加上 "KVM_GUEST" 的配置：

> Processor type and features ---> Linux guest support

注意，"Virtualization --> Kernel-based Virtual Machine (KVM) support" 这个并不需要，因为它是用来支持虚机运行的，是给宿主机内核用的。

煞费周章后，新内核终于可以正常启动了。其实后来才知道，Linux 针对 KVM 虚机提供了一个现成的配置，在拥有一个基础的 ".config" 文件后，直接 "make kvm_guest.config" 即可（"kvm_guest" 这个 config 文件并不是内核编译所需的完整配置文件，它只有几十行，所以中间其实自动进行了一个和基础配置文件 merge 的过程）。

### 2.2.支持 docker

启动后试着输入 "docker ps" 命令，dockerd 守护进程都没起来：

```python3
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. 
Is the docker daemon running?
```

那为啥没 run 呢，是没有设置开机自启么？用 "systemctl is-enabled docker"，结果显示为 "enabled"，再用 "systemctl status docker"：

![img](https://pic1.zhimg.com/80/v2-059b90cfea072cd85ff8b349146a5a14_720w.webp)

只有最后一部分日志，看不出来具体原因，完整的还得用 "journalctl -u docker"，然后从里面找到端倪：

```python3
msg="failed to mount overlay: no such device" storage-driver=overlay
```

"lsmod |grep overlay" 的输出为空，说明确实没有这个驱动，得增加 *CONFIG_OVERLAY_FS*。但重新编译后，docker 还是没能起来：

![img](https://pic3.zhimg.com/80/v2-67f3cb3b8c0969625220b9fb768fc5c2_720w.webp)

再把 [cgroup](https://zhuanlan.zhihu.com/p/143253843) 里面缺失的 controller 都补上，比如 *CONFIG_CGROUP_DEVICE。*现在有经验了，docker 的三要素嘛，还差一个 [namespace](https://zhuanlan.zhihu.com/p/149886216)，也一起给补了：

![img](https://pic3.zhimg.com/80/v2-8360e0113527993ac9b5075db9bd4902_720w.webp)

嗯，dockerd 总算是可以运行了，但是容器的网络不通，还是没法正常使用：

![img](https://pic2.zhimg.com/80/v2-7c3223934b87989e59e4cd9365488c49_720w.webp)

docker 网络这块的关系颇为复杂，传统上是 iptables，由于两台机器都采用和 RHEL-8 兼容的用户态，所以还涉及 nftables：

![img](https://pic4.zhimg.com/80/v2-1f89affb1457c77eb348115a7f923e7f_720w.webp)

图片来源见文末链接

折腾一番后，所幸找到了一个查找容器运行依赖项的工具（具体用法见[这篇文章](https://link.zhihu.com/?target=https%3A//blog.hypriot.com/post/verify-kernel-container-compatibility/)）。对着检查出的结果， 补齐 "Generally Necessary" 里面 "missing" 的部分（比如 *CONFIG_VETH*），最后总算勉强跑起来了。

## 3.小结

由于在此期间需要多次重编内核，ccache 的加速作用功不可没。在配置差异的比较中，源码 "scripts" 目录下的 diffconfig 工具也是必不可少。整个探（折）索（腾）的过程，除了工具的使用，对 KVM 和 docker 底层依托的组件，也有了更深的印象。

除了腾出有限的存储空间，更少的配置也意味着更短的编译时间（咱这只有 1 到 2 个 CPU 的小驴，编译内核可是非常耗时的），以及更快速的启动（可以用 "systemd-analyze" 来分析启动时间的构成）。

**参考：**

[iptables: The two variants and their relationship with nftables | Red Hat Developer](https://link.zhihu.com/?target=https%3A//developers.redhat.com/blog/2020/08/18/iptables-the-two-variants-and-their-relationship-with-nftables%3Fsource%3Dsso%23using_iptables_nft)







原文链接：https://zhuanlan.zhihu.com/p/523851273 原文作者:兰新宇
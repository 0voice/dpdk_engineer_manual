# ubuntu22.04 虚拟机 双网卡配置 桥接

## 1.添加一个新网卡

![在这里插入图片描述](https://img-blog.csdnimg.cn/9614094cbd784a1cac436de970a5f33b.png)

网卡类型选择桥接

![在这里插入图片描述](https://img-blog.csdnimg.cn/fc93336dc6cf459cbe1ecf187cab6c0a.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/c2ebdd39a3eb4b12b48f953b7c138c92.png)

windows端win+R ipconfig

![在这里插入图片描述](https://img-blog.csdnimg.cn/bcb60df310e84b1da9c9c91d5f55dd71.png)

## 2.修改虚拟机网络配置

输入命令

```
ip a
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/de5dd2580b154828a5e6050ef50380d5.png)

记录这里的ip地址

修改/etc/netplan/00-installer-config.yaml

```
sudo vim /etc/netplan/00-installer-config.yaml
```

修改为以下模样

```
network:
  ethernets:
    ens33:
      addresses: []
      dhcp4: true
    ens38:
      addresses: [你的ip地址/24（这个也是自己的）]
      dhcp4: false
      gateway4: 你的默认网关
      nameservers:
         addresses: [同上]
  version: 2
```

然后运行命令

```
sudo netplan apply
```

我的报了一个错误
如下

```
fabric@fabric:/etc/netplan$ sudo netplan apply

** (generate:15158): WARNING **: 20:49:17.450: `gateway4` has been deprecated, use default routes instead.
See the 'Default routes' section of the documentation for more details.

** (process:15156): WARNING **: 20:49:17.568: `gateway4` has been deprecated, use default routes instead.
See the 'Default routes' section of the documentation for more details.
```

解决方法：
修改/etc/netplan/00-installer-config.yaml

```
network:
   version: 2
   renderer: NetworkManager
   ethernets:
     ens34:
       dhcp4: no
       dhcp6: no
       addresses: [你的ip/24]
       routes:
         - to: default
           via: 172.16.13.1(默认网关)
       nameservers:
         addresses: [8.8.8.8,114.114.114.114]

```



原创：[correct！](https://blog.csdn.net/weixin_43701790)  本文链接：https://blog.csdn.net/weixin_43701790/article/details/124989806
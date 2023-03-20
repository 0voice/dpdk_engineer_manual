# 简述DHCP和DNS

## 1.DHCP简介

**DHCP(Dynamic Host Configuration Protocol，动态主机配置协议)**是一个[局域网](https://link.zhihu.com/?target=https%3A//baike.so.com/doc/3165868-3336420.html)的[网络协议](https://link.zhihu.com/?target=https%3A//baike.so.com/doc/5403497-5641193.html)，使用[UDP](https://link.zhihu.com/?target=https%3A//baike.so.com/doc/5418284-5656447.html)协议工作， 主要有两个用途:给内部网络或[网络服务](https://link.zhihu.com/?target=https%3A//baike.so.com/doc/520018-550563.html)供应商自动分配IP地址，给用户或者内部网络管理员作为对所有[计算机](https://link.zhihu.com/?target=https%3A//baike.so.com/doc/3435270-3615253.html)作中央管理的手段，在RFC 2131中有详细的描述。DHCP有3个端口，其中UDP67和UDP68为正常的DHCP服务端口，分别作为DHCP Server和DHCP Client的服务端口。

## 2.DHCP工作原理

![img](https://pic1.zhimg.com/80/v2-fd42406650ad2c2e58948a0f7e6369e0_720w.webp)

![img](https://pic1.zhimg.com/80/v2-06795cef8c49786295323612622d366c_720w.webp)

## 3.DHCP配置

```ada
Router（config）#service dhcp
//开启DHCP服务，Cisco设备默认开启
Router（config）#ip dhcp pool [pool name]
//定义DHCP地址池，一个网段对应一个地址池
Router（dhcp-config）#network  [network address] [subnet mask]
//定义地址池关联网段
Router（dhcp-config）#default-router [host address]
//定义分配给客户端的网关IP
Router（config）#ip dhcp excluded-address [network address] 
//将一个或多个地址排除在地址池之外（如网关IP等），以免分配给客户端
```

**配置示例：**

- Server配置

```ada
Server# conf ter
Server(config)# inter f0/0
Server(config-if)# ip add 192.168.1.254 255.255.255.0
Server(config-if)# no sh
Server(config-if)# exit
Server(config)# ip dhcp pool demon
Server(dhcp-config)# network 192.168.1.0/24
Server(dhcp-config)# default-router 192.168.1.254
Server(dhcp-config)# exit
Server(config)# ip dhcp excluded-address 192.168.1.254
```

- Client配置

通过DHCP获取IP地址

```ada
Client# conf ter
Client（config）# no ip routing
Client（config）# inter f0/0
Client（config-if）# no sh
Client（config-if）# ip address dhcp
```

## 4.DHCP中继

当Server和Client不在一个广播域内时，需要配置DHCP Relay(DHCP中继)。中继会将Client的广播转换为单播发送给Server

激活中继功能，并为UDP广播包设置中继目标IP（单播包的目标IP地址）

**注意是在接口上配置，该接口为沿途阻挡该广播消息的第一个接口**

![img](https://pic4.zhimg.com/80/v2-d3092db36528814ebbf252ca11cca167_720w.webp)

**思考IP helper-address部署在哪里？**

![img](https://pic2.zhimg.com/80/v2-8b53e3337a685461a83ae06a06acc21d_720w.webp)

配置在f0/0口上

![img](https://pic2.zhimg.com/80/v2-22dc156db7436413699950db22f43439_720w.webp)

配置在SVI口（VLAN10）上

## 5.DHCP中继配置

![img](https://pic2.zhimg.com/80/v2-5444e5ea18703e6ef0c6207235367361_720w.webp)

```ada
MSW1(config)# inter f0/0
MSW1(config)# switchport mode access
MSW1(config)# switchport access vlan 200
MSW1(config)# no sh
MSW1(config)# inter vlan 200
MSW1(config)# ip add 192.168.200.254 255.255.255.0
MSW1(config)# no sh
MSW1(config)# inter f0/1
MSW1(config)# switchport mode access
MSW1(config)# switchport access vlan 10
MSW1(config)# no sh
MSW1(config)# inter vlan 10
MSW1(config)# ip add 192.168.10.254 255.255.255.0
MSW1(config)# ip helper-address 192.168.200.200
MSW1(config)# no sh
```

## 6.DNS

**DNS(Domain Name System，域名系统)**，因特网上作为域名和[IP地址](https://link.zhihu.com/?target=https%3A//baike.so.com/doc/4252723-4455111.html)相互映射的一个[分布式数据库](https://link.zhihu.com/?target=https%3A//baike.so.com/doc/6591740-6805519.html)，能够使用户更方便的访问[互联网](https://link.zhihu.com/?target=https%3A//baike.so.com/doc/2011565-2128705.html)，而不用去记住能够被机器直接读取的IP数串。通过[主机](https://link.zhihu.com/?target=https%3A//baike.so.com/doc/5331327-5566564.html)名，最终得到该主机名对应的IP地址的过程叫做域名解析(或主机名解析)。DNS协议运行在[UDP](https://link.zhihu.com/?target=https%3A//baike.so.com/doc/5418284-5656447.html)协议之上，使用端口号53。

![img](https://pic2.zhimg.com/80/v2-69e55bed3e6025c8a15df2168eee40e5_720w.webp)

- **DNS的作用**

![img](https://pic4.zhimg.com/80/v2-40d658292ce6003e66bc2ca70e1a57df_720w.webp)

- **DNS的特点**

![img](https://pic3.zhimg.com/80/v2-3b47e23c0a04ba12e948def53ea29e3a_720w.webp)

- **DNS体系结构**

![img](https://pic3.zhimg.com/80/v2-c3fbcfdb85addf43f0b06f84ddcc9a3e_720w.webp)

- **递归服务器**

![img](https://pic1.zhimg.com/80/v2-54e891d569f361da6fb04298e7fe4628_720w.webp)

- **DNS数据**

![img](https://pic4.zhimg.com/80/v2-dfcc8c20955e743a74d0034bd8a1b923_720w.webp)

- **域名记录类型**

![img](https://pic3.zhimg.com/80/v2-2d0f80c0811c193f6515172d999ad812_720w.webp)

- **DNS资源记录类型**

![img](https://pic1.zhimg.com/80/v2-0231eb3e283cf6f8e48ca4ec3d82cd48_720w.webp)

- **域名解析过程**

域名解析总体可分为两大步骤

> 第一个步骤是本机向本地域名服务器发出一个DNS请求报文，报文里携带需要查询的域名。
> 第二个步骤是本地域名服务器向本机回应一个DNS响应报文，里面包含域名对应的IP地址。

从下面对[http://jocent.me](https://link.zhihu.com/?target=http%3A//jocent.me)进行域名解析的报文中可明显看出这两大步骤。

![img](https://pic1.zhimg.com/80/v2-dc23d034d2515234ddcafab8f811335c_720w.webp)

原文链接：https://zhuanlan.zhihu.com/p/152330355  原文作者：Demon 
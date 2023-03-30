# Docker容器如何与主机同网段其它主机互通？

## **1.前言**

### **1.1使用场景**

对开发者而言，随着容器的普遍使用，开发者可以很方便的搭建项目的简易测试环境。有时候为了项目可以在任意机器一键运行，不用配置连接IP等信息。所以希望可以提前固定容器的IP地址，而且一个项目有时候涉及多个容器，可能还会部署在多台机器上。所以如果容器间可以固定IP跨机器通信的话，会有很大方便。

### **1.2docker网络**

- [docker容器](https://www.zhihu.com/search?q=docker容器&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2587014274})默认是一个有自己独立网络空间的[虚拟系统](https://www.zhihu.com/search?q=虚拟系统&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2587014274})。
- docker安装后自动创建3中网络：bridge、host、none。
- bridge：网桥模式，默认创建docker0网桥，172.17.0.0/16，宿主机可访问，外部机器不可见。
- host：共享宿主机网络模式，外部主机与容器直接通信，容器缺少了隔离性。
- none：禁用[网络模式](https://www.zhihu.com/search?q=网络模式&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2587014274})。
- docker自定义网络
  docker提供了三种自定义网络驱动：bridge、overlay、macvlan。
- bridge驱动类似默认的bridge网络模式。
- overlay和macvlan是用于创建跨主机网络。
- 支持自定义网段、网关，docker network create --subnet 172.77.0.0/24 --gateway 172.77.0.1 my_net。
- docker创建容器使用默认docker0网络不支持自定义固定IP，都是动态的。

### **1.3自定义网络使用**

1. 自定义创建网段。docker network create --subnet=172.18.0.0/16 spark-net。
2. 指定网络驱动docker network create -d overlay --subnet 10.22.1.0/24 --gateway 10.22.1.1 spark-net-0。
3. 创建容器固定IP。

```text
docker run --name cloud1 \
--net spark-net --ip 172.18.0.2 \
-h cloud1 \
-it ubuntu
docker run --name cloud1_0 \
--network spark-net-0 --ip 10.22.1.26 \
-h cloud1 \
-it ubuntu
```

## **2.实践操作**

### **2.1Overlay网络模式详解**

- Overlay网络是目前比较主流的跨节点容器间数据传输和路由方案。
- Overlay网络模式在主机网络之上，在多个Docker主机之间实现[分布式网络](https://www.zhihu.com/search?q=分布式网络&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2587014274})，允许跨容器之间的交互。
- [Overlay网络](https://www.zhihu.com/search?q=Overlay网络&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2587014274})是指在不改变现有网络基础设施的前提下，通过某种约定通信协议，把二层报文封装在IP报文之上的新的数据格式。

### **2.2Consul[服务发现](https://www.zhihu.com/search?q=服务发现&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2587014274})**

- Consul是一个分布式、高可用性和多数据中心感知工具，用于服务发现、配置和编排。Consul 支持大规模快速部署、配置和维护面向服务的架构。
- 部署单节点的consul服务（可选择[公网服务器](https://www.zhihu.com/search?q=公网服务器&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2587014274})，或者能与其他部署容器通信的主机）。

```text
# 拉取镜像
docker pull progrium/consul
# 运行consul容器
docker run -d -p 8500:8500 -h consul --name consul --restart=always progrium/consul -server -bootstrap
# -h：表示consul的主机名
# --name consul：表示为该容器名
# --restart=always表示可以随着docker服务的启动而启动；
# 运行consul容器，该服务的默认端口是8500；-p：表示将容器的8500端口映射到宿主机的8500端口
# -serve -bootstarp：表示当在群集中，加上这两个选项可以使其以master的身份出现
```

- 管理访问地址
  [http://IP:8500/ui/#/dc1/kv/docker/nodes/](https://link.zhihu.com/?target=http%3A//IP%3A8500/ui/%23/dc1/kv/docker/nodes/)。

### **2.3修改docker配置**

```text
# 所有需要通信的机器都需要修改
 vim /etc/docker/daemon.json
 # 增加 cluster-store、cluster-advertise两个参数
 {
  "registry-mirrors": ["https://xxxx.xxxx.aliyuncs.com","https://registry.docker-cn.com"],
  "cluster-store": "consul://IP:8500",
  "cluster-advertise": "ens33:2376"
 }
 # cluster-store，是配置sonsul集群的访问地址
 # cluster-advertise，是广播通信地址和端口
 # 重启docker
 systemctl daemon-reload
 systemctl restart docker
 #如果有端口拒绝访问问题，可直接关掉防火墙
 #停止firewall
 systemctl stop firewalld.service
 #禁止firewall开机启动
 systemctl disable firewalld.service
 #查看开放端口列表
 firewall-cmd --list-ports
```

### **2.4实践机器规划**

本文实践创建了3台虚机：192.168.17.150 192.168.17.151 192.168.17.152。

### **2.5创建[overlay网络](https://www.zhihu.com/search?q=overlay网络&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2587014274})**

- 选其中一台机器执行，例如在192.168.17.150执行

```text
# 创建overlay网络，并自定义制定网段以及网关
# 可以通过制定不同的网段，以隔离不同的服务
docker network create -d overlay --subnet 10.22.1.0/24  --gateway 10.22.1.1 spark-net
# 每台机器查看创建的网络
docker network ls
# 查看具体信息
docker network inspect spark-net
```

- 删除网络命令

```text
# 删除自定义网络
docker network rm spark-net
# 如果有已连接的，先断开连接
docker network disconnect -f net-spark con1
```

- 注意
  例如：在cloud1机器上，可以执行docker network disconnect -f spark-net cloud2但是执行docker network disconnect -f spark-net cloud1就无效，必须跨机器执行断连。

### **2.6 跨主机创建容器验证**

本文依赖上篇文章创建了3个容器, 可查看  ***[Docker搭建大数据平台之Hadoop,Spark,Hive初探](https://link.zhihu.com/?target=https%3A//www.51cto.com/article/714441.html)\*** 。

192.168.17.150机器上执行。

```text
docker run --name cloud1 \
 -p 50070:50070 \
 -p 8088:8088 \
 -p 8080:8080 \
 -p 7077:7077 \
 -p 9000:9000 \
 -p 16010:16010 \
 --network net-spark --ip 10.22.1.26 \
 -h cloud1 \
 --add-host cloud1:10.22.1.26 \
 --add-host cloud2:10.22.1.27 \
 --add-host cloud3:10.22.1.28 \
 -it spark:v4
```

192.168.17.151机器上执行。

```text
docker run --name cloud2  \
 --network net-spark --ip 10.22.1.27 \
 -h cloud2 \
 --add-host cloud1:10.22.1.26 \
 --add-host cloud2:10.22.1.27 \
 --add-host cloud3:10.22.1.28 \
 -it spark:v4
```

192.168.17.152机器上执行。

```text
docker run --name cloud3 \
 --network net-spark --ip 10.22.1.28 \
 -h cloud3 \
 --add-host cloud1:10.22.1.26 \
 --add-host cloud2:10.22.1.27 \
 --add-host cloud3:10.22.1.28 \
 -it spark:v4
```

可分别在三个容器内互相ping IP10.22.1.26、10.22.1.27、10.22.1.28验证。

## **3.常见问题**

### **3.1如遇错误常用命令**

- 如果网络改动，需要重启docker

```text
systemctl daemon-reload
 systemctl restart docker
```

- 关掉防火墙

```text
# 停止firewall
 systemctl stop firewalld.service
 # 禁止firewall开机启动
 systemctl disable firewalld.service
 # 查看开放端口列表
 firewall-cmd --list-ports
 # 开放端口
 firewall-cmd --zone=public --add-port=2379/tcp --permanent 
 # 重新载入
 firewall-cmd --reload
```

### **3.2将容器以指定IP链接到自定义网络中**

```text
#容器cloud3以IP10.22.1.28链接到overlay网络spark-net
 docker network connect --ip 10.22.1.28 spark-net cloud3
```

### **3.3将容器从自定义网络中删除**

```text
# 注意不可在当前容器里执行断连
 # 例如 需要断连容器cloud2，则需要在容器cloud1中执行如下命令
 docker network disconnect -f spark-net cloud2
```

### **3.4manager节点无法接入**

docker.service配置 -H tcp://0.0.0.0:2376 --cluster-store=consul://121.4.138.199:8500 --cluster-advertise=ens33:2376 并不能正确执行，原理暂未知。





原文链接：https://www.zhihu.com/question/420004476/answer/2587014274  原文作者：程序员斯纳Java
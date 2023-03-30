# 给DPVS加上SESSION同步功能

## **1.前言**

DPVS是一款爱奇艺开源的基于DPDK的优秀软件([https://github.com/iqiyi/dpvs)](https://link.zhihu.com/?target=https%3A//github.com/iqiyi/dpvs)。利用DPDK工作在用户空间的特性，相比于内核空间的LVS,我们可以使用用户空间的一系列工具/中间件等完成很多在内核空间很难完成的功能。

**Just for fun**

虽然笔者日常工作中是搞Java中间件开发的，但一直都对底层技术尤其是在网络层面抱有很大的激情与好奇心。偶然接触到DPDK这个用户态数据平面开发套件,看了其官方文档和源码后，不禁技痒难耐，于是就尝试在DPVS上增加一个Session同步功能。虽然和工作关系不大，但搞技术的乐趣不就在于不停的折腾么,Just For Fun!当然，由于精力原因，只是写出了原型并测试成功，距离生产环境还有很大的距离，毕竟不靠这个吃饭^_^。

## **2.DPVS**

DPVS事实上就是一个负载均衡软件，源于LVS，我们常说的Virtual IP(VIP)就可以使用DPVS来支持，如下图所示:

![img](https://pic3.zhimg.com/80/v2-caf6431ef968fc8f251da7363babf37a_720w.webp)


这次笔者就是在DPVS在FullNAT模式下对于主从模式增加了Session同步功能。如下图所示:

![img](https://pic3.zhimg.com/80/v2-f39bd94a0c146212524f308b82e42072_720w.webp)



### 2.1**没有SESSION同步功能会如何**

由于DPVS的数据转发是通过内部的session表来分发数据包的,如果没有Session同步功能,那么对应的数据库由于找不到对应的Session进而被丢弃。如果Client端是通过tcp进行连接的话:

![img](https://pic3.zhimg.com/80/v2-e564bd39ba0e05429f231be12ca31662_720w.webp)


那么将会在配置的tcp重传超时之后报错。

![img](https://pic2.zhimg.com/80/v2-c3f62789b1d1444e59aebd32c4407a1d_720w.webp)



### 2.2**TCP Client RealServer**



![img](https://pic2.zhimg.com/80/v2-c3f62789b1d1444e59aebd32c4407a1d_720w.webp)



### 2.3**如果SESSION同步会如何**



![img](https://pic2.zhimg.com/80/v2-7d496e500655078e5404ca751ee23039_720w.webp)


如果Session同步后，由于新晋升的DPVS2 Master依旧能够知道将这个Packet发送到后面哪台RealServer，如果是采用TCP连接的话，在一次重传之后，依旧能够保证连接的稳定。

### 2.4**SESSION同步方法**

笔者这次尝试的是主从模式下FullNat的Session同步,事实上只需要将FullNat下的两张Session表(Session_IN和Session_OUT)从Master同步到Slave即可。

![img](https://pic2.zhimg.com/80/v2-659d7550d5962911471761fb5cbe502d_720w.webp)



### 2.5**如果工作在内核态的LVS如何同步**

由于LVS这一类的软件工作在内核态，那么就需要使用比较复杂且难于调试的问题进行主从之间的通信，如下图所示:

![img](https://pic4.zhimg.com/80/v2-9a2168c5f63662aa2ce7a9407947d5a3_720w.webp)


内核态的调试由于比起用户态来说相对复杂,而且没什么好用的中间件，笔者就没有做这方面的尝试。

### 2.6**在用户态笔者采用Redis Pub/Sub同步**

而在用户态，可用的工具就太多了,于是笔者就选择了使用Redis的订阅/发布(Pub/Sub)功能将Session表信息从Master同步到Slave,如下图所示:

![img](https://pic1.zhimg.com/80/v2-f37640d52c03123ccd10e34615b82bcc_720w.webp)


由于FullNat采用五元组，所以笔者在Redis中Pub的Key为:

```text
session_key_(af协议簇)
             _(proto协议)
             _(client源地址)
             _(client端口号)
             _(vip地址)
             _(vip端口号)
             _(localIP)
             _(localPort)
             _(RealServer目的地址)
             _(RS目的端口号)
             _(当前session所在CPUID)
```

## 3.**SESSION同步工作线程**

首先，笔者在DPVS启动的main函数除了DPVS的线程之外用pthread新建了两个线程，用于reids的Send(Pub)和Receive(Sub)。

![img](https://pic4.zhimg.com/80/v2-51df4ff57172de719c31d81c40a35b07_720w.webp)



### **3.1线程间通信**

#### 3.1.1**发布信息到Redis**

DPDK线程与Send/Recv线程间，同时ring_buffer进行通信。所以一开始创建的时候，就给每个DPDK线程创建了一个rte_ring(session_rings)。当每有新建连接动作时候，DPDK线程就会将新建连接的动作封装成一个消息扔到里面，然后由SendPub线程去消费。如下图所示:

![img](https://pic2.zhimg.com/80/v2-cb6c6ea6afa4c37eba540b75ed1fb96d_720w.webp)


由于ring_buffer是有限的，可能出现消息丢失的现象。
新建连接的DPVS运行栈为:

```text
__dp_vs_in
    |->conn_sched
        |->tcp_conn_sched (tcp协议)
            /* only TCP-SYN without other flag can be scheduled */
            /* 即只有TCP-SYN包才会走新建连接的逻辑 */
            |->dp_vs_schedule
                |->dp_vs_snat_schedule (FullNAT模式)
```

在最终的dp_vs_snat_schedule代码中，加入一段代码:

```text
static struct dp_vs_conn *dp_vs_snat_schedule(......)
{
    conn = dp_vs_conn_new(mbuf,iph,&param,dest,0);
    ......
    // 加入把conn信息放入session_buffer的逻辑
    session_info_enqueue(conn);
    return conn;
}
```

放入逻辑，其实就是将conn的信息组装成一个sesion_msg结构体，然后将之前session_key的9个信息从conn中提取:

```text
void session_info_enqueue(struct dp_vs_conn* conn){
    ......
    int cid = rte_lcore_id();
    struct session_msg* msg;
    if(rte_mempool_get(message_pool,(void**)&msg) < 0){
        ......
        return;
    }
    copy_conn_to_msg(conn,msg);
    if(rte_ring_enqueue(session_rings[cid],msg) != 0){
        ...
        rete_mempool_put(message_pool,msg);
        return;
    }
}
```

#### 3.1.2**从Redis订阅消息**

同样的，有一个Recv(Sub)线程从Redis订阅信息，然后Recv(Sub)线程和DPDK间的线程也用ring_buffer来同步，不过另用了一个session_subscribe_buffer。

![img](https://pic2.zhimg.com/80/v2-a2418593040791344d21e4041bb616b9_720w.webp)


如图中所示，从Redis订阅到信息之后，将消息重新塞到session_subscribe_buffer(每个线程都有)里面。然后利用DPVS的job回调方法在每个线程中处理subscribe消息并通过此消息重建session表:

```text
lcore_job_recv_fwd
    |->lcore_process_session_subscribe_ring

void lcore_process_session_subscribe_ring(...){
    struct rte_ring* ring = session_subscribe_rings[cid];
    ...
    struct session_msg* msg;
    if(rte_ring_dequeue(ring,(void**)&msg) < 0){
        return;
    }
    new_dpvs_conn(msg);
    rte_mempool_put(message_pool,msg);
}
```

笔者在new_dpvs_conn里面做了FullNAT的两张session表同步操作。

```text
void dp_vs_conn_new_from_session(struct session_msg* msg){
    ......
    /*init inbound conn tuple hash*/
    // SESSION IN 表项构建
    t->af = msg->af;
    t->proto = msg->proto;
    ......
    /*init outbound conn tuple hash*/
    // SESSION OUT 表项构建
    new->af = msg->af;
    new->proto = msg->proto;
    ......
    // 绑定dest
    err = dp_vs_conn_bind_dest(new,dest);
    ......
    // 绑定hash表
    dp_vs_conn_hash(new);
}
```

#### **3.1.3MQ消费重放**

用Redis做Pub/Sub只是笔者为了保持编码简单而做的选择。如果正式用在产线,笔者觉得还是要把这种Session发到Kafka这种queue里面，那么就可以将Session的变化落到本地。这样，在主备都宕机的情况下，可以通过消费Kafka中已有的消息重建Session表。

![img](https://pic2.zhimg.com/80/v2-77dabd9a847c92dd18b0857e1129931d_720w.webp)



## 4.**遇到的小坑**

在笔者进行测试的时候，遇到的一个问题时，在Session同步之后，虽然Session表项同步无误，但始终tcp连接被断开，在加了各种Print判断和TCP dump了一堆之后。才发现，DPVS本身会对TCP的sequence进行重写以增加toa字段，所以导致TCP sequence对不上，进而连接被断开。为了简单起见，笔者注掉了这段代码，然后终于成功了！

```text
static int tcp_fnat_in_handle(...)
{
    struct tcphdr *th;
    ......
    // tcp_in_add_toa(conn,mbuf,th);
    // tcp_in_adjust_seq(conn,th);
    th->source = conn->lport;
    th->dest = conn->dport;

    return tcp_send_csum(af,iphdrlen,th,conn,mbuf);
}
```

## 5.**不足之处**

当前笔者只做了Session新建动作的同步，Session删除等其它动作还需要慢慢斟酌。
另外，由于时间精力所限，笔者对DPVS的编码只相当于做了一次简单的原型验证，还远远达不到产线高可用的要求。
不过，当测试成功，Master宕机后另一台Slave立马接上后，长连接(用的MySQL Client做测试)保持不断,查询数据依旧丝滑,仿佛什么都没发生过的时候(如果没有这个功能，只能坐等25s左右的卡主超时了,tcp_retries2=5)，就感觉非常的有成就感！





原文链接：https://zhuanlan.zhihu.com/p/200892937  原文作者：无毁的湖光
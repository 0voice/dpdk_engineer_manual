# DPDK : 进程间通信以及在内存管理的应用

## 1.DPDK的进程间通信

DPDK将进程分为两种 : primary process 和secondary process。并且DPDK在单机上是一个集中式控制的系统，即主要由primary process对系统的资源(如内存，vfio container等)进行控制，而secondary process若需要申请资源，则向primary process发送申请的请求，由promary process处理请求后，将结果通知secondary process。

涉及到多个进程之间的相互协作，就必然会涉及到进程间通信(Inter-Process Communication, 缩写为IPC), 就目前阅读到的代码而言，主要采用了两种IPC的方式：共享内存通信(memory share)，Socket。

DPDK在初始化时创建一个控制线程用来监听来自其他进程的消息，接收到消息后，会根据消息的类型(同步或者异步)进行不同的处理，详情见rte_mp_channel_init(), process_msg()(位于文件
lib/librte_eal/common/eal_common_proc.c)。

primary process对内存初始化后，会将一部分初始化信息写入到文件中，而secondary process在初始化时能读取此文件，这就是使用了共享内存通信的方式。

这一篇博文主要介绍DPDK中采用Socket的进程间通信方式。

## 2.相关结构体的说明

下面还是先对一些结构体作简单的说明 :

```
********************rte_eal.h********************
struct rte_mp_msg {
    char name[RTE_MP_MAX_NAME_LEN]; //用来指明消息的类型
    int len_param; //指明param的长度
    int num_fds; //指明fds的长度
    uint8_t param[RTE_MP_MAX_PARAM_LEN]; //用来存放消息的相关参数
    int fds[RTE_MP_MAX_FD_NUM]; //用于进程间传递fd
};

struct rte_mp_reply {
    int nb_sent; //发送出去的消息的个数
    int nb_received; //收到回复的消息的个数
    struct rte_mp_msg *msgs; /* caller to free */ //用于存放收到回复的消息
};

********************eal_common_proc.c********************
//消息的四种类型
enum mp_type {
    MP_MSG, /* Share message with peers, will not block */
    MP_REQ, /* Request for information, Will block for a reply */
    MP_REP, /* Response to previously-received request */
    MP_IGN, /* Response telling requester to ignore this response */
};

struct mp_msg_internal {
    int type; //消息的类型
    struct rte_mp_msg msg; //对应的消息
};

//用于pending request中的async
struct async_request_param {
    rte_mp_async_reply_t clb;	 //收到异步消息的reply时，所调用的callback函数
	struct rte_mp_reply user_reply;	//用于记录消息的发送次数和接收次数，存放接收到的reply
    struct timespec end;  //用于记录异步消息超时的时间点
    int n_responses_processed; //用于记录已经处理的reply个数
};

//进程在发送一个消息时，需要将这个消息封装到pengding_request中， 并且保留在一个pending request list
//等收到reply后，再从list找到对应的pending_request，调用callback(异步通信机制)或者唤醒对应的线程(同步通信机制)
struct pending_request {
    TAILQ_ENTRY(pending_request) next;
    enum {
        REQUEST_TYPE_SYNC,
        REQUEST_TYPE_ASYNC
    } type;
    char dst[PATH_MAX];
    struct rte_mp_msg *request;//用于存放request消息
    struct rte_mp_msg *reply; 	 //用于存放一个reply消息
    int reply_received; 	//用于指示是否正确接收到了reply
    RTE_STD_C11
    union {
        struct {
            struct async_request_param *param;  //对应于异步通信，用于记录相关的信息
        } async;
        struct {
            pthread_cond_t cond;	//对应于同步通信，用于唤醒对应的线程
        } sync;
    };
};
```

## 3.通信过程的协议栈

下面给出进程间通信过程的协议栈：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/8pECVbqIO0yrtWrmYbFIPoar63xib050Rdcr4OXjHZceAFFWWc9GJ7MrsOnIODfp50ibWSNojwCCDTDfL3hiaHO4A/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

主要的过程如下:

1，将上层的消息(对于内存分配是struct malloc_mp_req)封装到struct rte_mp_msg的param字段中。

2，再继续封装在struct mp_msg_internal的msg字段。

3，最后将struct mp_msg_internal封装在 struct msghdr(而struct rte_mp_msg的fds作为msghdr控制信息)中。

4，使用系统提供的接口 sendmsg() 发送出去。

(由于是本机的IPC，所以会在msghdr中指定一个目的路径，这样接收者就能通过读取对应的文件描述符来获得信息)



## 4.DPDK的通信方式以及实现

DPDK的IPC通信过程实现了**同步通信**和**异步通信**

下面介绍一下pending_request的作用 :

> 在一次事务的过程中，发送方发送消息时会将未完成的请求封装到pending reuqest中，并且将其加入到pending request list。然后就收到回复时，根据对应的pending request采用对应的操作，然后将回复加入到对应的位置(如果是同步通信，则由用户在调用rte_mp_request_sync()，有一个参数用于指定位置，如果是异步通信，则会创建一个struct async_request_param作为消息存放的位置)。

在**同步通信**的场景下 :

> 1，发送消息时会阻塞，直到收到回复以后才继续执行后面的指令。2，若primary process同时给多个secondary process发送消息，则每发送一个消息，则会阻塞，直到收到回复后才给下一个secondary process发送消息。

**同步消息的实现**：

> 1，某个线程调用rte_mp_request_sync（）, 会创建一个类型为REQUEST_TYPE_SYNC的pending request。并加入到pending request list中。2，发送同步消息后，会阻塞在pending request中cond字段(条件变量)。3，在接收消息的线程接收到回复时，根据回复查找相应的penging request, 将回复消息存放在pending request的reply字段。并通过条件变量唤醒对应的线程。4，被唤醒的线程继续执行后面的指令，将回复消息(即pending request中的reply字段)复制到用户指定的位置。

在异步通信的场景下 :

> 发送消息时会将所有消息都发送出去，然后继续执行后面的指令，或者其他功能。

异步消息的实现 :

> 1，某个线程调用rte_mp_request_async()会传入一个callback和超时时长。接着创建一个类型为REQUEST_TYPE_ASYNC的pending request并加入到pending request list中。2，发送异步消息后（会设置一个alarm用于处理超时），线程可以继续执行后面的指令。3，在接收消息的线程接收到回复时，根据回复查找相应的pending request，将回复消息存放在pending request的reply字段。最后会将回复消息复制到async->param.user_reply中。4，若已经收到了全部的消息，则会触发callback函数，将user_reply作为函数，处理所有的回复消息。

(一个技巧，若全部消息发送失败,则会加入一个dummy pending request, 用于触发callback，保证异步通信一定能够触发callback函数的执行。但是本人目前尚未找到触发dummy pending request的代码在哪一块)

## 5.一个DPDK的IPC应用场景的说明

下面通过一个内存管理的场景(其他场景如hotplug设备移除时通知其他process，或者使用vfio中container，iommu type同步时都会使用IPC)说明DPDK中IPC的应用。

下面给出**secondary process向primary process申请内存**的全部过程：

一），若secondary process在调用rte_malloc()分配内存时，此时需要向OS申请更多的内存页，那么会**调用request_to_primary()发送一个RTE_TYPE_ALLOC请求给primary process**(下图第1步)。

二），**primary process接收到请求后，会向OS申请更多的内存页，并更新其rte_memseg_list的信息**(下图第2步)。若申请失败，则发送一个fail消息给secondary process, **否则发送一个sync request**(下图第3步)，让secondary process将其local rte_memseg_list 与 primary process的rte_memseg_list进行同步。

三），若secondary process收到失败的消息，直接退出。**若收到sync request, 则进行同步操作，若同步成功，则发送sync success的请求，否则发送sync failure的请求**(下图第4,5步)。

四) ，若primary process接收到sync success的请求，则发送alloc success的消息。**否则，primary process执行rollback操作，释放第二步向OS申请的内存(下图第6步)，然后继续向secondary proocess发送rollback request**(下图第7步)。

五）， 若secondary process收到sync success, 直接退出。**若收到rollback request, 则将其local rte_memseg_list 与 primary process的rte_memseg_list进行同步。然后将同步的结果发送给primary process**(下图第8,9步)。

六），**primary process收到同步的结果后，无论成功与否，将结果发送给secondary process**(下图第10步)。

七）， secondary process收到结果后，退出。

下面的图展示了上面加粗的字体的过程(其他过程比较简单，在此不给出图例)：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/8pECVbqIO0yrtWrmYbFIPoar63xib050RwAITmeYRRVpQ0hyCtibe47ibviaWUxNMQ6AMbwtpDWr4CphO9agZl6Sag/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



下面给出secondary process请求primary process释放内存页的过程：

一）， 若secondary process在调用rte_malloc()释放内存时，此时若能够将某些内存页归还给OS，那么会**调用request_to_primary()发送一个RTE_TYPE_FREE请求给primary process**(下图第1步)。

二）， primary process接收到请求后，会**将内存页释放给OS**(下图第2步)，并更新其rte_memseg_list的信息。若释放失败，则发送一个fail消息给secondary process, 否则**发送一个sync request**(下图第3步)，让secondary process将其local rte_memseg_list 与 primary process的rte_memseg_list进行同步。

三），若secondary process收到失败的消息，直接退出。**若收到sync request, 则进行同步操作，若同步成功，则发送sync success的请求，否则发送sync failure的请求**(下图第4,5步)。

四）， **primary process收到同步的结果后，无论成功与否，将结果发送给secondary process**(下图第6步)。

五）， secondary process收到结果后，退出.

下面的图展示了上面加粗的字体的过程：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/8pECVbqIO0yrtWrmYbFIPoar63xib050Ruxymj1XX2clZEqrz2T6ajOcicgC6FxfvOfF09D9oVnG3wOgd2x01IaQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



## 6.总结

本文简单地介绍了DPDK的进程间通信机制：

1，DPDK系统中使用了多种IPC方式，本文主要介绍了socket这一种方式。

2，DPDK通信过程自定义了一套消息格式，向用户隐藏了处理消息的方式。

3，DPDK中支持同步通信和异步通信，并且在secondary process向primary process申请/释放内存时，两种通信方式都会使用。

4，对内存管理的一个通信场景进行了说明（其他的通信场景相对于内存管理的通信场景而言比较简单）。
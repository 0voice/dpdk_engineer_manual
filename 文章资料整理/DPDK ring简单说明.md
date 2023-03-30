# DPDK ring简单说明

## 1.ring提供的接口

对于一个模块而言，其对外提供的接口直接表明了它所提供的功能，也是我们分析一个模块最初的入口。ring是一个环形无锁队列，支持多生产者多消费者操作，所以对于队列的操作构成了模块的主要接口。ring的实现在文件`rte_ring.c`和`rte_ring.h`中。

```c
rte_ring_create() //ring的创建
 
rte_ring_init() //ring的初始化
 
rte_ring_lookup() //ring的查找
 
rte_ring_free() //ring的释放
 
rte_ring_dump() //获取ring的使用信息
 
rte_ring_set_water_mark() //设置ring的水标
 
```

以上的几个大的接口提供了ring的开始和结束以及查找。同时对于一个队列来说，还有更多的入队，出队操作。如函数

```c
rte_ring_mp_enqueue_burst（）//多生产者批量入队
 
```

此处就省略其他很多单（多）生产者，单（多）消费者的操作函数接口了。

## 2.ring的创建及初始化

rte ring的创建是通过`rte_ring_create()`函数来实现的，这个函数的原型`struct rte_ring * rte_ring_create(const char *name, unsigned count, int socket_id,unsigned flags)`

这里需要注意的是`socket_id`和`flags`，在多个进程同时访问同一个ring时，要改善性能，可以创建多个ring,同时要注意多个进程绑定在同一个socket上。另一个参数flags则是表明创建的ring是否安全支持多生产者多消费者模型。接下来就来看看创建及初始化中的一些细节。

首先找到`ring_list`，`ring_list`是挂接在队列中的，根据ring不跨socket的原则，推断是每个socket都维护有一个这样的队列，具体就不去抠代码了，先主后次。

接下来就是两个准备操作：

- 获取创建的ring的空间大小，为后面分配空间做准备。

- 分配一个`struct rte_tailq_entry *te;`结构，在创建完成ring后，挂接这个队列元素到队列中去。此时不妨先看一下这个结构体的定义。

  ```c
  struct rte_tailq_entry {
  	TAILQ_ENTRY(rte_tailq_entry) next; 
  	/**< Pointer entries for a tailq list */
  	void *data; /**< Pointer to the data referenced by this 	tailq entry */
  };
  ```

其中的data指针就指向了创建的ring的地址。

然后就是为新创建的ring分配内存空间咯，使用了`rte_memzone_reserve()`函数分配，这个函数在内存部分详细说明，但是需要注意：

> `rte_memzone_reserve()`只能用于primary进程中内存分配，也暗含了对于多生产者多消费者使用的ring,其ring的创建要在 primary进程中。

最后就是把分配的ring初始化-`rte_ring_init()`,并填充te元素，把te挂接在队列中。

## 3.ring的出入队

ring的出入队操作，我们重点来关注几个接口:

```c
__rte_ring_mp_do_enqueue()
__rte_ring_sp_do_enqueue()
__rte_ring_mc_do_dequeue()
__rte_ring_sc_do_dequeue()
 
```

无论使用的哪个上层接口，最终调用的就是这4个函数。在使用多生产者多消费者时，函数中会有`rte_pause()`的操作，里面使用了`__mm_pause()`指令，看注释意思是避免忙等待，主要应用在短时的loops。至于具体的头和尾指针的移动，可以参考prog_guide中的ring一节，图文并茂。

## 4.ring的使用范围以及潜在问题

- 1.ring的调试信息在non_EAL线程中是不支持获取的。

- 2.ring支持多生产者入队和多消费者出队，但是都是不可抢占的。不可抢占的意思是：

  - 一个线程在做多生产者入队操作，此时，禁止被另一个做多生产者入队的线程抢占。
  - 一个线程在做多消费者出队操作，此时，禁止被另一个做多消费者出队的县城抢占。

  > 这意味着如果两个线程在同一个core上操作，那么2th线程则必须等到1th线程调度后才能访问，因此，尽量不要在同一个core上对同一个ring做多生产者同时入队或者出队。更详细的说明，请参考 prog_guide 3.3.4章节。

## 5.关于水标的使用

在初始化ring的时候，可以设置对应的水标位置，但感觉它并未提供设置接口，用的地方不是很多。比如，当入队已经达到水标位置时，就可以返回对应的错误值，上层调用就可以做些处理。

## 6.关于无锁队列的链接：

[无锁队列顶层设计](https://mp.weixin.qq.com/s?__biz=MzI3NDA4ODY4MA==&mid=2653334197&idx=1&sn=0f115d7dce93924f351fb4ed042d0945&chksm=f0cb5d32c7bcd424c1b79fce3d4da6c820a481e6ae5658f5c4d26b0d559c74496b54f312ae6e&scene=0&key=13226364e7503325532dbdffaf77dcf7261c39300b8d1ab55cf6902eb6d70105faf42d19d730f9a5cc95a93f0655f11a0091f8685be1c4744b2e0c0eb6ddc7d3a691f5db41febd72b247e03f648d42d9&ascene=7&uin=NjE1NTk4ODgy&devicetype=android-21&version=26050434&nettype=WIFI&abtest_cookie=AQABAAgAAQBGhh4AAAA%3D&pass_ticket=YQaFpDPG9attV1MHhVsh2EM%2FTzHQajwQRwM%2FVihb9jT1GzEnmKGmS%2BPfRR0dY0cs&wx_header=1)

[无锁队列到底有没有锁](https://mp.weixin.qq.com/s?__biz=MzI3NDA4ODY4MA==&mid=2653334197&idx=1&sn=0f115d7dce93924f351fb4ed042d0945&chksm=f0cb5d32c7bcd424c1b79fce3d4da6c820a481e6ae5658f5c4d26b0d559c74496b54f312ae6e&scene=0&key=13226364e7503325532dbdffaf77dcf7261c39300b8d1ab55cf6902eb6d70105faf42d19d730f9a5cc95a93f0655f11a0091f8685be1c4744b2e0c0eb6ddc7d3a691f5db41febd72b247e03f648d42d9&ascene=7&uin=NjE1NTk4ODgy&devicetype=android-21&version=26050434&nettype=WIFI&abtest_cookie=AQABAAgAAQBGhh4AAAA%3D&pass_ticket=YQaFpDPG9attV1MHhVsh2EM%2FTzHQajwQRwM%2FVihb9jT1GzEnmKGmS%2BPfRR0dY0cs&wx_header=1)





原创作者：[AISEED](https://home.cnblogs.com/u/yhp-smarthome/)       本文链接：https://www.cnblogs.com/yhp-smarthome/p/6910756.html
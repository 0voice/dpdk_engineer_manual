# dpdk多线程模型

## 1.多线程模型

一个cpu上可以运行多个线程， 由linux内核来调度各个线程的执行。内核在调度线程时，会进行上下文切换，保存线程的堆栈等信息， 以便这个线程下次再被调度执行时，继续从指定的位置开始执行。然而上下文切换是需要耗费cpu资源的的，多核体系的CPU，物理核上的线程来回切换，会导致L1/L2 cache命中率的下降。同时NUMA架构下，如果操作系统调度线程的时候，跨越了NUMA节点，将会导致大量的L3 cache的丢失。

 Linux对线程的亲和性是有支持的, 如果将线程和cpu进行绑定的话，线程会一直在指定的cpu上运行，不会被操作系统调度到别的cpu上，线程之间互相独立工作而不会互相扰完，节省了操作系统来回调度的时间。目前DPDK通过把线程绑定到cpu的方法来避免跨核任务中的切换开销。

现在开看下dpdk多线程的实现方式。 

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVLjJcgbic03M1dQRVkqcYwvkZibWwDw1hXtvQMmF8cQrN6VtU5x89cuQkutmVWmiaGxrF8KmzPGCpOQw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)      



 我的系统上有4个cpu， 刚开始运行dpdk进程的时候， 假设linux将调度这个dpdk主线程在cpu0上运行。dpdk主线程在运行过程中发现系统有4个cpu， 则会创建三个子线程。需要注意的是，此时这三个子线程还是运行在cpu0上的，直到每个子线程自己使用linux的亲和性功能，将子线程自己绑定到特定的cpu上。例如子线程1绑定到cpu1上运行；子线程2绑定到cpu2上运行；子线程3绑定到cpu3上运行。同时dpdk主线程自己也会进行绑定，例如绑定到cpu0上。



​       

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVLjJcgbic03M1dQRVkqcYwvkR67jox66sibzaUk2iaicYY1mkVaRibOLZFCVyuwxticCicGZIDk8t7ISqy0g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



从图中可以看出，dpdk主线程， 从线程分别绑定到了不同的cpu上。dpdk主线程， 也叫做master控制线程，通常绑定在master core上，用于接收用户配置信息，并传递配置参数给dpdk从线程等。dpdk从线程，也称为slave数据线程，用于高速处理报文转发任务，绑定到各个slave core上。每个dpdk线程都独立处理各自的业务逻辑，而不用担心被切换到其他cpu上。

一个cpu上只会运行一个dpdk线程。系统有多少个cpu，就会创建多少个子线程，然后绑定到cpu上。当然了这是可以配置的，而不是固定死的，例如我有4个cpu， 则可以配置为只创建一个主线程，2个从线程等。

下面从代码层面来分析dpdk的多线程模型，先从主线程这一层面入手分析。

主线程处理逻辑:



### 1.1子线程的创建

dpdk主线程运行的时候，会查看系统当前有多少cpu，然后为每个cpu都创建一个子线程。此时创建的子线程还是运行在和dpdk主线程同一个cpu上，也就是master core上。每个子线程内部会将自己绑定到不同的cpu上。



int rte_eal_init(int argc, char **argv)

{

​	//遍历所有的逻辑core， 除了master core之外

​	RTE_LCORE_FOREACH_SLAVE(i) 

​	{

​		//创建主到从线程的管道

​		pipe(lcore_config[i].pipe_master2slave);

​		//创建从到主线程的管道

​		pipe(lcore_config[i].pipe_slave2master);

​		//开始设置每个从线程为wait状态	

​		lcore_config[i].state = WAIT;

​		//创建子线程，此时每个子线程还是在master core上运行。内部会将每个子线程绑定到不同的从逻辑core上

​		ret = pthread_create(&lcore_config[i].thread_id, NULL, eal_thread_loop, NULL);

​	}

}



我们可以看到dpdk创建了两个方向的管道，一个是主线程到从线程方向的管道；另一个是从线程到主线程方向的管道。每个主从线程都有两个方向的管道。这两个方向的管道有什么作用呢？这两个方向的管道都是命令通道，用于主线程发送命令消息给从线程， 或者从线程发命令响应消息给主线程。每次主线程通过管道发消息给从线程后，从线程都会立即通过管道给主线程回响应消息， 之后从线程将会执行主线程通知从线程执行的特定的回调接口。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVLjJcgbic03M1dQRVkqcYwvkAViaLmaSVrmrLyUsjc0gdHYjk9NDOIIREfIp1v2OYBjYLL50UQvhxfw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

​     



### 1.2主线程通过管道发送命令消息给从线程

管道创建好后，主从线性就可以通过管道来进行消息交互了。那什么时候主线程可以发消息给从线程呢？主线程只有在从线程进入wait等待状态时，才会通过管道发送消息给从线程。从线程默认阻塞在read系统调用上，等待主线程的命令消息，此时从线程的状态为wait等待状态。因此只有从线程阻塞在read系统调用，也就是wait状态，表示从线程准备好接收消息了，主线程才能发消息到从线程， 如果从线程被设置为running运行状态, 或者finish状态，主线程是不会发消息给从线程的，此时主线程会等待，直到从线程进入wait状态才会发消息。从线程收到消息后，就会被唤醒，进而read系统调用返回，读取主线程通过管道发来的消息。

主线程发送消息给从线程的时候，还会传递一个回调接口，这个接口用来做什么的。其实就是用于通知从线程收到消息后，执行这个回调。例如l2fwd二层转发这个例子中，主线程传递给从线程的回调为l2fwd_launch_one_lcore， 每个从线程收到消息后，就会执行这个回调，进而处理报文的高速转发。因此也可以看出管道只是一个命令通道，而主线程传入的回调函数才是从线程真正执行的操作。



//master core发消息给所有的从线程, 让从线程执行相应的回调，并等待从线程的响应。

//只要有一个没进入wait状态，就会返回！此时由rte_eal_mp_wait_lcore接口来等待从线程进入wait状态

int rte_eal_mp_remote_launch(int (*f)(void *), void *arg,

​			 					  enum rte_rmt_call_master_t call_master)

{

​	//遍历所有的逻辑core， 除了master core之外。确保每个逻辑core都是处于wait状态

​	RTE_LCORE_FOREACH_SLAVE(lcore_id)

​	{

​		if (lcore_config[lcore_id].state != WAIT)

​		{

​			return -EBUSY;

​		}

​	}

​	//master core发送消息到每个从线程, 让从线程执行相应的回调。并接收从线程的响应

​	RTE_LCORE_FOREACH_SLAVE(lcore_id)

​	{

​		rte_eal_remote_launch(f, arg, lcore_id);

​	}

​	//标记master core是否也要执行相应的回调

​	if (call_master == CALL_MASTER) 

​	{

​		lcore_config[master].ret = f(arg);

​		lcore_config[master].state = FINISHED;

​	}

}



需要注意的是，刚开始的时候，主线程传递给从线程的回调为sync_func，这是一个空函数，从线程收到消息后将会执行这个空函数。dpdk之所以传递一个空函数，仅仅是为了测试否所有的从逻辑core已经进入wait状态，仅此而已，没有别的意图。

rte_eal_remote_launch是管道的实现方式，用于主线程通过管道发送命令给从线程，并通过管道接收从线程的响应。来看下这个接口的实现。



//master core发送消息到从线程, 并接收从线程的响应

int rte_eal_remote_launch(int (*f)(void *), void *arg, unsigned slave_id)

{

​	//设置从线程需要执行的回调

​	lcore_config[slave_id].f = f;

​	//master core发消息给从线程

​	while (n == 0 || (n < 0 && errno == EINTR))

​	{

​		n = write(m2s, &c, 1);

​	}

​	//master core等待从线程响应

​	do 

​	{

​		n = read(s2m, &c, 1);

​	} while (n < 0 && errno == EINTR);

}

 

### 1.3主线程等待从线程进入wait状态

来看下主线程的最后一个接口， 那就是等待从线程进入wait状态。上面已经提到了，只有在所有的从线程都进入wait状态，表示准备好了接收主线程的消息，主线程才能发消息给从线程，此时从线程才能够在read系统调用返回后接收到消息。那如果从线程处于running,或者finish状态呢，则主线程必须等待， 等待所有的从线程进入wait状态，然后发消息给所有从线程，否则因为从线程没有阻塞在read系统调用，此时主线程发消息给从线程，从线程是收不到的，相当于消息被丢弃了。例如l2fwd二层转发例子，如果rte_eal_init函数返回时，从线程都还没有进入wait状态，则主线程传递报文转发回调l2fwd_launch_one_lcore给从线程是没意义的，因为从线程收不到消息。



void rte_eal_mp_wait_lcore(void)

{

​	//遍历所有的逻辑core， 除了master core之外

​	RTE_LCORE_FOREACH_SLAVE(lcore_id) 

​	{

​		//使用循环的方式，等待从线程进入wait状态。

​		//如果从线程一直不进入wait状态，则master core的cpu占用率将会飙升

​		rte_eal_wait_lcore(lcore_id);

​	}

}

 

int rte_eal_wait_lcore(unsigned slave_id)

{

​	//如果从线程处于run状态，则循环等待从线程进入finish状态

​	while (lcore_config[slave_id].state != WAIT &&

​	    lcore_config[slave_id].state != FINISHED)

​	{

​		;

​	}

​	//将finish状态设置为wait状态

​	lcore_config[slave_id].state = WAIT;

}



需要注意的是，dpdk使用while循环方式等待所有从线程进入wait状态， 如果从线程迟迟不进入wait状态，则主线程将会卡在while死循环，这时会导致主线程所在的cpu利用率飙升。当然这只在dpdk进程运行瞬间才会，因为通常在rte_eal_init返回后，dpdk主线程只会发一次消息通知从线程，传递一个回调给从线程，这个回调一般也是一个死循环，用于从线程处理转发高速报文。

到此为止，主线程的处理逻辑已经分析完成了，现在来看下从线程的处理逻辑。

从线程处理逻辑

1、从线程cpu绑定

从线程的入口为eal_thread_loop，从线程首先就进行cpu的绑定，将自己绑定到某个cpu上。这样每个cpu都只运行一个dpdk线程，避免上下文切换消耗资源。



__attribute__((noreturn)) void * eal_thread_loop(__attribute__((unused)) void *arg)

{

​	//将某个线程绑定到从逻辑core上

​	if (eal_thread_set_affinity() < 0)

}



dpdk是通过调用pthread_setaffinity_np接口，将线程与cpu进行绑定的。关于这个接口使用方法，读者自行查找资料吧！这里就不再展开分析了。



2、接收主线程的消息

每个从线程都可以通过管道，接收来自主线程的消息，然后也是通过管道给主线程发送命令响应。子线程阻塞在read系统调用，等待来自主线程的命令消息。当收到主线程的消息后，从线程被唤醒，read返回后读取管道消息，并立即通过管道给主线程发送响应消息。从线程给主线程回了响应消息后，立即执行回调函数，真正执行回调函数里面的业务逻辑，例如上面提到的l2fwd二层转发l2fwd_launch_one_lcore高速报文转发接口。



__attribute__((noreturn)) void * eal_thread_loop(__attribute__((unused)) void *arg)

{

​	//从逻辑core线程的while死循环

​	while (1) 

​	{

​		//从线程阻塞读取master core发来的命令

​		do 

​		{

​			n = read(m2s, &c, 1);

​		} while (n < 0 && errno == EINTR);

​		//设置为run状态

​		lcore_config[lcore_id].state = RUNNING;

​		//从线程给主线程发送命令响应

​		while (n == 0 || (n < 0 && errno == EINTR))

​		{

​			n = write(s2m, &c, 1);

​		}

​		//从线程执行回调

​		fct_arg = lcore_config[lcore_id].arg;

​		ret = lcore_config[lcore_id].f(fct_arg);

​		lcore_config[lcore_id].ret = ret;

​		//设置为finish状态

​		lcore_config[lcore_id].state = FINISHED;

​	}

}



到此为止，dpdk主从线程模型已经分析完成了。

## 2.多进程模型

dpdk除了主从线程运行方式外， 还支持主从进程的运行方式。一个dpdk主进程对应多个dpdk从进程。dpdk主从进程共享大页内存(包括段内存，内存区，malloc堆空间，内存池)， 以及共享队列，例如中断队列等。 dpdk主进程用于创建大页内存，创建内存池，创建队列；而dpdk从进程是不会重复创建的，而是通过mmap进行映射。



​          

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVLjJcgbic03M1dQRVkqcYwvkSpcneyatMjS8n0YhpCBYFVQvEDThiapHWibZWCGAu1NWxbW1qBooaIkA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



需要注意的是，不管是dpdk主线程，还是dpdk从进程， 内部都是支持多线程的。上面提到的主从线程，都是适用于dpdk主从进程。也就是说，dpdk主进程，内部会创建多个子线；dpdk从进程，内部也会创建多个子线程。

如何运行dpdk主从进程呢？可以在运行dpdk程序时，指定主从进程模式，设置proc-type为primary或者为secondary。例如下面的例子，将会运行4个dpdk进程，其中一个为dpdk主进程，另三个为dpdk从进程



./l2fwd -c 0xf -n 2 -- -q 2 -p 0xf -proc-type primary

./l2fwd -c 0xf -n 2 -- -q 2 -p 0xf -proc-type secondary

./l2fwd -c 0xf -n 2 -- -q 2 -p 0xf -proc-type secondary

./l2fwd -c 0xf -n 2 -- -q 2 -p 0xf -proc-type secondary 





原文链接：https://mp.weixin.qq.com/s/1uas2RgaWNLbQot8jBYmlQ
原文作者：IEEE
# dpdk中断机制

## 1.中断事件管理

dpdk实现了uio, 定时器alarm, vfio三种中断，且用链表来管理这些中断源。当应用层需要设置中断时， 设置好中断的触发回调后就可以调用rte_intr_callback_register接口注册一个中断源到中断链表中；当应用层想取消某个中断源时，调用rte_intr_callback_unregister接口从中断源链表中移除一个中断源。内部会将中断源链表中的所有中断源描述符都加入到epoll实现的红黑树中， 当相应中断源有事件发生时，epoll会调用这些中断源注册的回调函数。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVLDrHyO9RlD1xRTRLXiaZ8ZS1GZQicibosxZ4epZ1x31A73cmkAAAiaC3DztJI2q590yxNTrRgfgawaqA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 1.1中断源的设置

应用层通过调用rte_intr_callback_register接口，就可以将一个中断源注册到中断源链表intr_sources中。中断源事件回调callback也是一个链表，意思是同一个中断源可以重复注册，且每次注册都可以指定不同的回调函数，因此这也是一个链表。在这个中断源有事件触发时， 会调用这个中断源上注册的所有中断回调函数。

在中断源加入到中断源链表后，还会通过写管道的方式，往管道里面写入一个数值，此时读管道事件就会从epoll中触发，这个读管道事件的实现方式是退出epoll机制，重新将这个新加入的中断源注册到epoll事件机制中。



int rte_intr_callback_register(struct rte_intr_handle *intr_handle,rte_intr_callback_fn cb, void *cb_arg)

{

​	//创建中断源链表节点，并设置好中断触发时的回调函数

​	callback = rte_zmalloc("interrupt callback list", sizeof(*callback), 0);

​	callback->cb_fn = cb;

​	callback->cb_arg = cb_arg;

​	//将中断回调插入到已经存在的中断回调链表。中断事件回调也是一个链表，同一个中断源

​	//可以重复注册，且每次注册都可以指定不同的回调

​	TAILQ_FOREACH(src, &intr_sources, next) 

​	{

​		if (src->intr_handle.fd == intr_handle->fd) 

​		{

​			TAILQ_INSERT_TAIL(&(src->callbacks), callback, next);

​			break;

​		}

​	}

​	//不存在，则重新创建一个中断源，并插入到链表

​	if (src == NULL)

​	{

​		src = rte_zmalloc("interrupt source list", sizeof(*src), 0))；

​		src->intr_handle = *intr_handle;

​		TAILQ_INIT(&src->callbacks);

​		TAILQ_INSERT_TAIL(&(src->callbacks), callback, next);

​		TAILQ_INSERT_TAIL(&intr_sources, src, next);

​	}

​	//写管道的方式，通知epoll事件机制退出事件循环， 重新将新加入事件加入到epoll.

​	write(intr_pipe.writefd, "1", 1);

}



### 1.2中断源的卸载

如果应用层不再需要中断了，则调用rte_intr_callback_unregister接口将中断源卸载。中断源卸载那就很简单了，将中断源链表上的节点移除就好了。需要注意的是，如果一个中断源注册了多个中断回调，则只有在中断回调链表都卸载后，才会将这个中断源节点也给移除。



int rte_intr_callback_unregister(struct rte_intr_handle *intr_handle,rte_intr_callback_fn cb_fn, void *cb_arg)

{

​	//卸载中断回调链表中的节点

​	TAILQ_REMOVE(&src->callbacks, cb, next);

​	rte_free(cb);

​	//在这个中断源上的中断回调链表没有元素时，也卸载中断源节点

​	TAILQ_REMOVE(&intr_sources, src, next);

​	rte_free(src);

​	//通知epoll事件循环， 将已经删除的中断源也从epoll中移除

​	write(intr_pipe.writefd, "1", 1);

}



## 2.中断源事件的实现

### 2.1中断初始化

rte_eal_intr_init接口用于中断的初始化，内部会创建一个读写管道，用来控制是否退出epoll机制。当应用层添加了新的中断源或者卸载了中断源， 在上面提到的两个注册与卸载函数里面，会往管道写入数据，此时epoll读管道事件将会被触发，读取这个管道的内容后，从epoll中退出后，将新加的中断源注册到epoll, 或者将卸载的中断源从epoll移除。这些操作都是在子线程中完成的，由子线程来处理中断事件，主线程则处理报文的高速转发。



int rte_eal_intr_init(void)

{

​	//创建读写管道，用来控制器是否退出epoll机制。当应用层添加了新的源或者卸载了中断源，

​	//用来通知epoll返回，将新加的中断源注册到epoll, 或者将卸载的中断源从epoll移除

​	pipe(intr_pipe.pipefd);

​	//创建子线程，子线程处理所有的中断事件，主线性继续执行其他业务逻辑

​	pthread_create(&intr_thread, NULL, eal_intr_thread_main, NULL);

}



### 2.2epoll事件机制创建

eal_intr_thread_main是中断子线程的入口函数，内部会创建一个epoll句柄，并将管道描述符， 中断源链表中的所有描述符都加入到epoll事件机制中。需要注意的是这个子线程是一个死循环，永远都不会退出。如果有新的中断源加入或者移除，则会关闭epoll句柄，然后重新创建epoll对象，重新将管道以及中断源链表中的描述符加入到epoll中，这是一个循环的过程。一句话：这个死循环是为了在有新的中断源加入或者移除时，能够重复创建epoll句柄以及将中断源加入到epoll中



//中断子线程入口

static __attribute__((noreturn)) void * eal_intr_thread_main(__rte_unused void *arg)

{

​	//子线程死循环，永远不会退出

​	for (;;)

​	{

​		//创建epoll句柄

​		int pfd = epoll_create(1);

​		//将管道加入到epoll事件中

​		epoll_ctl(pfd, EPOLL_CTL_ADD, intr_pipe.readfd, &pipe_event);

​		//将中断源链表中的中断元素加入到epoll事件中

​		TAILQ_FOREACH(src, &intr_sources, next)

​		{

​			ev.events = EPOLLIN | EPOLLPRI;

​			ev.data.fd = src->intr_handle.fd;

​			epoll_ctl(pfd, EPOLL_CTL_ADD, src->intr_handle.fd, &ev);

​		}

​		eal_intr_handle_interrupts(pfd, numfds);

​		//执行到这里，说明异常了或者有新的中断事件加入或者中断源被移除。先关闭epoll后重新创建

​		close(pfd);

​	}

}



### 2.3epoll_wait等待中断事件发生

在将管道描述符，中断源链表中的所有描述符注册到epoll红黑树后，eal_intr_handle_interrupts内部会调用epoll_wait等待中断事件，管道事件的触发，这是一个异步的过程。需要注意的是这也是一个死循环，什么时候会退出呢? 还是上面提到的，在有新的中断源加入或者移除，会退出这个这个死循环。想想如果没有这个for死循环会发生什么事件呢？相当于每次epol_wait返回后，都需要关闭epoll句柄，重新创建epoll句柄，然后重新将管道以及中断源链表中的所有描述符加入到epoll中，这些系统调用也是要消耗性能的，只有在新加入中断源或者移除中断源时才需要这么做。如果都没有新加或者移除的中断源就没有必要这么做了。一句话：这个死循环是为了等待中断以及管道事件触发。



static void eal_intr_handle_interrupts(int pfd, unsigned totalfds)

{

​	//这又是一个死循环，循环等待已经加入到epoll的事件被触发

​	for(;;)

​	{

​		nfds = epoll_wait(pfd, events, totalfds, EAL_INTR_EPOLL_WAIT_FOREVER);

​		/* epoll_wait has at least one fd ready to read */

​		//处理所有已经发生的中断事件

​		if (eal_intr_process_interrupts(events, nfds) < 0)

​		{

​			return;

​		}	

​	}

}



### 2.4中断事件的处理

当epoll_wait返回后，说明有中断事件发生或者管道事件的发生，此时调用eal_intr_process_interrupts开始处理已经发生的事件。如果是管道事件，则直接退出epoll事件循环，这是优先级最高的事件。如果是中断源事件，则查找到相应的中断源，然后调用这个中断源注册的所有中断回调。



static int eal_intr_process_interrupts(struct epoll_event *events, int nfds)

{

​	//循环处理多个已经触发的中断事件

​	for (n = 0; n < nfds; n++) 

​	{

​		//有新的中断事件加入或者移除时，退出事件循环，之后会重新创建epoll， 将新事件加入到epoll中

​		if (events[n].data.fd == intr_pipe.readfd)

​		{

​			int r = read(intr_pipe.readfd, buf.charbuf,	sizeof(buf.charbuf));

​			return -1;

​		}

​		//查找中断源

​		TAILQ_FOREACH(src, &intr_sources, next)

​			if (src->intr_handle.fd == events[n].data.fd)

​			{

​				break;

​			}

​		//读取内容

​		bytes_read = read(events[n].data.fd, &buf, bytes_read);

​		//读取内容后，调用这个中断源注册的所有中断回调

​		TAILQ_FOREACH(cb, &src->callbacks, next)

​		{

​			active_cb.cb_fn(&src->intr_handle, active_cb.cb_arg);

​		}

​	}

}



## 3.中断的使用

以一个定时器中断的例子来说明中断的使用。

### 3.1定时器初始化

rte_eal_alarm_init函数用来初始化定时器，里面会创建一个定时器中断源。这个中断源在后面会加入到中断源链表中



//定时器初始化

int rte_eal_alarm_init(void)

{

​	//创建定时器中断源

​	intr_handle.type = RTE_INTR_HANDLE_ALARM;

​	intr_handle.fd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK);

}



### 3.2创建定时器

当需要创建定时器时，调用rte_eal_alarm_set创建定时器，可以多次调用这个接口来创建多个定时器。内部会调用rte_intr_callback_register接口将定时器中断源加入到中断源链表中，定时器中断源的定时时间为定时器链表中第一个元素的时间，因为定时器链表是按照时间从小到达排序的，因此第一个元素时间是最小的。

顺便说下这个接口的实现吧，每次调用rte_eal_alarm_set这个接口，都会创建一个定时器节点struct alarm_entry，并将这个定时器节点按照时间从小到大的顺序添加到定时器链表alarm_list中。定时器链表中有多个定时器节点，但只有一个中断源，这个中断源来管理所有的定时器。也就是说当中断源被触发时，会调用alarm_time定时器中断源回调， 这个回调里面会遍历定时器链表中的所有定时器节点，进而调用每个定时器回调。

![图片](https://mmbiz.qpic.cn/mmbiz_png/nrgibEGYIoVLDrHyO9RlD1xRTRLXiaZ8ZSH08gb9ug7uotACrpdyR2f2PlYVPjtLKjicLrRfnwQQUjgoJo3SY9bhA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

//创建一个定时器

int rte_eal_alarm_set(uint64_t us, rte_eal_alarm_callback cb_fn, void *cb_arg)

{

​	//首次注册一个定时器中断源，中断回调为eal_alarm_callback。

​	//当定时器中断被触发时，这个回调会调用定时器链表中的所有已经到期的定时器回调

​	if (!handler_registered) 

​	{

​		ret |= rte_intr_callback_register(&intr_handle, eal_alarm_callback, NULL);

​	}

​	//设置定时器链表中是按照时间从小到达排除的，因此第一个元素时间是最小的

​	if (LIST_FIRST(&alarm_list) == new_alarm)

​	{

ret |= timerfd_settime(intr_handle.fd, 0, &alarm_time, NULL);

​	}

}

  当定时器中断源定时时间到后，定时器中断源事件会被触发，进而调用定时器中断源回调eal_alarm_callback。这个函数里面会遍历已经注册到定时器链表中的各个定时器，然后调用每个定时器的处理函数。

static void eal_alarm_callback(struct rte_intr_handle *hdl __rte_unused,void *arg __rte_unused)

{

​	//遍历定时器链表，调用各个定时器的回调函数

​	while ((ap = LIST_FIRST(&alarm_list)) !=NULL &&

​			gettimeofday(&now, NULL) == 0 &&

​			(ap->time.tv_sec < now.tv_sec || (ap->time.tv_sec == now.tv_sec &&

​						ap->time.tv_usec <= now.tv_usec)))

​	{

​		ap->cb_fn(ap->cb_arg);

​	}

}



### 3.3定时器删除

调用rte_eal_alarm_cancel接口可以将定时器从定时器链表中删除，函数实现很简单，这里就不再贴代码了。

到此为止，dpdk中断机制的实现已经分析完成了。



原文链接：https://mp.weixin.qq.com/s/e7I8gGQ1u5wByXokAWcqCA

原文作者：IEEE
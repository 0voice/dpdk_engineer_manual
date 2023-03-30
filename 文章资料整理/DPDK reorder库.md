# DPDK reorder库

前言：对于有些任务比较重的工作，我们通常都会采用负载均衡的方法用以提高任务效率。然而有些应用报文对于报文顺序要求比较严格，这就要求在报文在经过负载均衡处理后，进一步处理前，重新调整报文的顺序，保证和进入负载均衡前的顺序一致。DPDK reorder库就是这样一个东西，用以在进一步处理前重新对报文进行排序。

## 1. DPDK reorder库的工作原理

对于DPDK的原理，其实是比较简单的，这里直接翻译官方文档的部分内容来说明（有调整顺序，意译）：

> 16.1 操作
>
> reorder库本质上是一个重新排序mbufs的缓冲区。用户把乱序的mbufs插入到reorder缓冲区，然后从in-order缓冲区取出来有序的报文。
>
> 在一个给定的时间内，reorder缓冲区包含了那些序列号在序列窗口内的mbufs。序列窗口值的大小是由配置的最小序列号和缓冲区所能容纳的条目决定的。比如说，给定一个reorder缓冲区有200个条目，最小序列号是350，那么序列窗口值范围则在350-550。
>
> 当插入mbufs，reorder库根据要插入的报文的序列号，把它们分为三类：
>
> - valid 序列号在序列窗口之内
> - late 序列号在序列窗口之外，同时小于最小窗口值。
> - early 序列号在序列窗口之外，同时大于最大窗口值。
>
> reorder库直接返回late的mbufs,试着容纳early的mbufs。

这段话说明了order的操作过程，简而言之，就是把到来的mbufs分类处理，在窗口内的放进去，late的，已经晚了，就没办法了；early的，尽力放进去。

接下来看看具体工作细节：

> 16.2 实现细节
>
> order库的实现是由一对缓冲区完成的：Order缓冲区和Ready缓冲区。
>
> 在一个插入的调用中，valid报文被直接插入到Order缓冲区，而late的报文则直接返回调用者错误。
>
> 如果插入的是early报文，Order缓冲区会试着移动窗口（增加最小序列窗口值），这样的话，这个early报文就变成了valid报文，就能被插入到Oorder缓冲区。最后，Order缓冲区的报文就会被移动到Ready缓冲区。还没到来的报文就会被忽略，自然，以后就会变成late报文。这意味着只要Ready缓冲区有空间，那么当early报文到来时，就会移动mbufs到Ready缓冲区去。
>
> 例如：假定我们有一个Order缓冲区，包含200个条目，最小序列号现在是350，我们需要插入一个early报文，序列号是565。这意味着，我们需要至少需要把序列窗口移动15个条目才能放下这个报文（因为最大窗口是550）。Order缓冲区便会试着移动至少15个条目到Ready缓冲区中。如果此时Order缓冲区中有gap,就会被跳过。什么时候移动停止呢？也就是移动多少个呢？最少15个！最多呢？直到移动到碰到gap为止。
>
> 当取出排序后的报文时，先从Ready缓冲区中取出，然后从Order缓冲区中取出报文，直到遇到gap。

以上就是官网上关于这个reorder库的说明。

## 2. 代码实现

解下来我们简单看一下在代码中是怎么使用的：

reorder库的例子使用是在栗子`packer_reordering`目录下的，例子使用的是三个核来演示：一个核从网卡瘦数据报，一个负责处理均衡上来的报文，另一个核负责重新排序，发送出去。

整体逻辑非常简单，就是三个线程的处理

### 2.1 接收线程

接收线程是从`rx_thread()`开始的，我们重点看这一句：

```c
for (i = 0; i < nb_rx_pkts; )
	pkts[i++]->seqn = seqn++;
```

这是给接收到的每一个包标上序号，这也是后面进行排序的基础。添完序号后，就发送到对应的工作核上，进行报文处理。

```c
ret = rte_ring_enqueue_burst(ring_out, (void *) pkts,
								nb_rx_pkts);
```

### 2.2 工作线程

在示例中，工作线程并没有做什么事情，只是更换一个接收和发送端口后，就由发送出去，等待着最后的发送线程排序进行处理。

### 2.3 发送线程

在发送线程中，主要就对要发送的报文先进行排序，然后发送。那么在使用reorder库前，先要对其进行初始化，其工作是在`main.c`中做的：

```c
if (!disable_reorder) {
  send_args.buffer = rte_reorder_create("PKT_RO", rte_socket_id(),
                                        REORDER_BUFFER_SIZE);
  if (send_args.buffer == NULL)
    rte_exit(EXIT_FAILURE, "%s\n", rte_strerror(rte_errno));
}
```

这里就是以名字`PKT_RO`为标识创建一个reorder排序框架。注意一下这个`struct rte_reorder_buffer`结构体，

```c
struct rte_reorder_buffer {
	char name[RTE_REORDER_NAMESIZE];
	uint32_t min_seqn;  /**< Lowest seq. number that can be in the buffer */
	unsigned int memsize; /**< memory area size of reorder buffer */
	struct cir_buffer ready_buf; /**< temp buffer for dequeued entries */
	struct cir_buffer order_buf; /**< buffer used to reorder entries */
	int is_initialized;
} __rte_cache_aligned;
```

从这个结构可以看出，其中包含了两个缓冲区。

其实创建reorder也就是创建一个内存区：

```c
b = rte_zmalloc_socket("REORDER_BUFFER", bufsize, 0, socket_id);
	if (b == NULL) {
		RTE_LOG(ERR, REORDER, "Memzone allocation failed\n");
		rte_errno = ENOMEM;
		rte_free(te);
	} else {
		rte_reorder_init(b, bufsize, name, size);
		te->data = (void *)b;
		TAILQ_INSERT_TAIL(reorder_list, te, next);
	}
```

之后，我们就有了这个缓冲区，就可以进行重排序操作了，我们接着看在发送线程中是怎么做的`send_thread()`：

```c
for (i = 0; i < nb_dq_mbufs; i++) {
			/* send dequeued mbufs for reordering */
			ret = rte_reorder_insert(args->buffer, mbufs[i]);
 
			if (ret == -1 && rte_errno == ERANGE) {
				/* Too early pkts should be transmitted out directly */
				RTE_LOG_DP(DEBUG, REORDERAPP,
						"%s():Cannot reorder early packet "
						"direct enqueuing to TX\n", __func__);
				outp = mbufs[i]->port;
				if ((portmask & (1 << outp)) == 0) {
					rte_pktmbuf_free(mbufs[i]);
					continue;
				}
				if (rte_eth_tx_burst(outp, 0, (void *)mbufs[i], 1) != 1) {
					rte_pktmbuf_free(mbufs[i]);
					app_stats.tx.early_pkts_tx_failed_woro++;
				} else
					app_stats.tx.early_pkts_txtd_woro++;
			} else if (ret == -1 && rte_errno == ENOSPC) {
				/**
				 * Early pkts just outside of window should be dropped
				 */
				rte_pktmbuf_free(mbufs[i]);
			}
		}
```

把每一个报文都进行插入操作，如果失败了，并且没有Ready缓冲区没有空间了，那么就释放报文，进一步的，看看插入操作：

```c
offset = mbuf->seqn - b->min_seqn;
if (offset < b->order_buf.size) 
{
  position = (order_buf->head + offset) & order_buf->mask;
  order_buf->entries[position] = mbuf;
} 
else if (offset < 2 * b->order_buf.size) 
{
  if (rte_reorder_fill_overflow(b, offset + 1 - order_buf->size)
      < (offset + 1 - order_buf->size)) 
  {
    /* Put in handling for enqueue straight to output */
    rte_errno = ENOSPC;
    return -1;
  }
  offset = mbuf->seqn - b->min_seqn;
  position = (order_buf->head + offset) & order_buf->mask;
  order_buf->entries[position] = mbuf;
} 
else 
{
  /* Put in handling for enqueue straight to output */
  rte_errno = ERANGE;
  return -1;
}
return 0;
```

先计算序列号的偏移，如果在窗口内，那么找到对应的位置插入；不在窗口内则计算是否能够放置到Ready缓冲区内，如果不能，则会有两种错误：ENOSPC和ERANGE错误。后者代表包太超前，移动窗口也不能放下。

把所有的包都插入完毕后，就可以从取出顺序的报文了：`rte_reorder_drain()`:

```c
unsigned int
rte_reorder_drain(struct rte_reorder_buffer *b, struct rte_mbuf **mbufs,
		unsigned max_mbufs)
{
	unsigned int drain_cnt = 0;
 
	struct cir_buffer *order_buf = &b->order_buf,
			*ready_buf = &b->ready_buf;
 
	/* Try to fetch requested number of mbufs from ready buffer */
	while ((drain_cnt < max_mbufs) && (ready_buf->tail != ready_buf->head)) {
		mbufs[drain_cnt++] = ready_buf->entries[ready_buf->tail];
		ready_buf->tail = (ready_buf->tail + 1) & ready_buf->mask;
	}
 
	/*
	 * If requested number of buffers not fetched from ready buffer, fetch
	 * remaining buffers from order buffer
	 */
	while ((drain_cnt < max_mbufs) &&
			(order_buf->entries[order_buf->head] != NULL)) {
		mbufs[drain_cnt++] = order_buf->entries[order_buf->head];
		order_buf->entries[order_buf->head] = NULL;
		b->min_seqn++;
		order_buf->head = (order_buf->head + 1) & order_buf->mask;
	}
 
	return drain_cnt;
}
```

这个取出来的逻辑就恨简单了：先从Ready缓冲区中取出来排序好的报文，如果数量不够要求取出的报文数量，那么再去Order缓冲区中去取。



转载自： 作者 [AISEED](https://home.cnblogs.com/u/yhp-smarthome/)  本文链接https://www.cnblogs.com/yhp-smarthome/p/7482290.html
# DPDK数据包与内存专题 mempool内存池 

前言：DPDK提供了内存池机制，使得内存的管理的使用更加简单安全。在设计大的数据结构时，都可以使用mempool分配内存，同时，mempool也提供了内存的获取和释放等操作接口。对于数据包mempool甚至提供了更加详细的接口-`rte_pktmbuf_pool_create()`，接下来重点分析通用的内存池相关内容。使用DPDK-17.02版本。

## 1. mempool的创建

内存池的创建使用的接口是`rte_mempool_create()`。在仔细分析代码之前，先说明一下mempool的设计思路：在DPDK-17.02版本中（和2.1等先前版本在初始化略有差异），总体来说，mempool的组织是通过3个部分实现的

- 1.mempool头结构。mempool由名字区分，挂接在`struct rte_tailq_elem rte_mempool_tailq`全局队列中，可以根据mempool的名字进行查找，使用`rte_mempool_lookup()`接口即可。这只是个mempool的指示结构，mempool分配的内存区并不在这里面，只是通过物理和虚拟地址指向实际的内存地址。
- 2.mempool的实际空间。这就是通过内存分配出来的地址连续的空间，用来存储mempool的obj对象。
- 3.ring队列。ring是个环形无锁队列，关于这个话题，可以参考官方文档来了解。其作用就是存放mempool中的对象指针，提供了方便存取使用mempool的空间的办法。

------

接下来，就来具体看看mempool的创建和初始化过程。
先注意一下`rte_mempool_create`的参数中的两个-`mp_init` 和`obj_init`，前者负责初始化mempool中配置的私有参数，如在数据包中加入的我们自己的私有结构；后者负责初始化每个mempool对象。我们然后按照mempool的3个关键部分展开说明。

- 1.mempool头结构的创建
  在DPDK-17.02中，mempool头结构包含3个部分：`struct rte_mempool`，cache和mempool private。创建是在`rte_mempool_create_empty()`中完成的，看这个函数，先进行了对齐的检查

```c
RTE_BUILD_BUG_ON((sizeof(struct rte_mempool) & RTE_CACHE_LINE_MASK) != 0); 
```

然后从mempool队列中取出头节点，我们创建的mempool结构填充好，就挂接在这个节点上。接下来做一些检查工作和创建flag的设置。

`rte_mempool_calc_obj_size()`计算了每个obj的大小，这个obj又是由三个部分组成的，objhdr,elt_size,objtlr，即头，数据区，尾。在没有开启`RTE_LIBRTE_MEMPOOL_DEBUG`调试时，没有尾部分；头部分的结构为：`struct rte_mempool_objhdr`，通过这个头部，mempool中的obj都是链接到队列中的，所以，提供了遍历obj的方式（尽管很少这么用）。函数返回最后计算对齐后的obj的大小，为后面分配空间提供依据。

然后分配了一个mempool队列条目，为后面挂接在队列做准备。

```c
te = rte_zmalloc("MEMPOOL_TAILQ_ENTRY", sizeof(*te), 0);
	if (te == NULL) {
		RTE_LOG(ERR, MEMPOOL, "Cannot allocate tailq entry!\n");
		goto exit_unlock;
	}
```

接下来，就是计算整个mempool头结构多大，吐槽这里的命名！

```c
	mempool_size = MEMPOOL_HEADER_SIZE(mp, cache_size);
	mempool_size += private_data_size;
	mempool_size = RTE_ALIGN_CEIL(mempool_size, RTE_MEMPOOL_ALIGN);
```

`mempool_size`这个名字太有误导性，这里指的是计算mempool的头结构的大小。而不是内存池实际的大小。在这里可以清晰的看出这个mempool头结构是由三部分组成的。cache计算的是所有核上的cache之和。

然后，分配这个mempool头结构大小的空间，填充mempool结构体，并把mempool头结构中的cache地址分配给mempool。初始化这部分cache.

最后就是挂接mempool结构。`TAILQ_INSERT_TAIL(mempool_list, te, next);`

- 2.mempool实际空间的创建和ring的创建

这部分的创建是在函数`rte_mempool_populate_default()` 中完成的。

首先计算了每个elt的总共的大小

```c
total_elt_sz = mp->header_size + mp->elt_size + mp->trailer_size;
```

然后计算为这些元素需要分配多大的空间，`rte_mempool_xmem_size(n, total_elt_sz, pg_shift);`

接着`rte_memzone_reserve_aligned()`分配空间。
终于到关键的一步了，`rte_mempool_populate_phys()`把元素添加到mempool,实际上就是把申请的空间分给每个元素。

先看到的是这么一段代码：

```c
 
if ((mp->flags & MEMPOOL_F_POOL_CREATED) == 0) {
		ret = rte_mempool_ops_alloc(mp);
		if (ret != 0)
			return ret;
		mp->flags |= MEMPOOL_F_POOL_CREATED;
} 
```

这就是创建ring的过程咯，其中的函数rte_mempool_ops_alloc()就是实现。那么，对应的ops->alloc()在哪注册的呢？

```c
if ((flags & MEMPOOL_F_SP_PUT) && (flags & MEMPOOL_F_SC_GET))
		rte_mempool_set_ops_byname(mp, "ring_sp_sc", NULL);
	else if (flags & MEMPOOL_F_SP_PUT)
		rte_mempool_set_ops_byname(mp, "ring_sp_mc", NULL);
	else if (flags & MEMPOOL_F_SC_GET)
		rte_mempool_set_ops_byname(mp, "ring_mp_sc", NULL);
	else
		rte_mempool_set_ops_byname(mp, "ring_mp_mc", NULL);
```

就是根据ring的类型，来注册对应的操作函数，如默认的就是ring_mp_mc，多生产者多消费者模型，其操作函数不难找到：

```c
static const struct rte_mempool_ops ops_mp_mc = {
	.name = "ring_mp_mc",
	.alloc = common_ring_alloc,
	.free = common_ring_free,
	.enqueue = common_ring_mp_enqueue,
	.dequeue = common_ring_mc_dequeue,
	.get_count = common_ring_get_count,
};
```

接下来，又分配了一个`struct rte_mempool_memhdr *memhdr;`结构的变量，就是这个变量管理着mempool的实际内存区，它记录着mempool实际地址区的物理地址，虚拟地址，长度等信息。

再然后，就是把每个元素对应到mempool池中了：`mempool_add_elem()`。在其中，把每个元素都挂在了elt_list中，可以遍历每个元素。最后`rte_mempool_ops_enqueue_bulk(mp, &obj, 1);`，最终，把元素对应的地址入队，这样，mempool中的每个元素都放入了ring中。

创建完成！！！

## 2. mempool的使用

mempool的常见使用是获取元素空间和释放空间。

- `rte_mempool_get`可以获得池中的元素，其实就是从ring取出可用元素的地址。
- `rte_mempool_put`可以释放元素到池中。
- `rte_mempool_in_use_count`查看池中已经使用的元素个数
- `rte_mempool_avail_count` 查看池中可以使用的元素个数

## 3. 后记

mempool是DPDK内存管理的重要组件，这篇重点介绍了 mempool创建使用的过程，对于系统如何做大页映射，统一地址并没有涉及，希望在后面的篇幅中，关注一下大页的映射和共享内存等。再往后，会介绍驱动与收发包等联系较大的内容。



原创：[AISEED](https://home.cnblogs.com/u/yhp-smarthome/)     本文链接：https://www.cnblogs.com/yhp-smarthome/p/6687175.html
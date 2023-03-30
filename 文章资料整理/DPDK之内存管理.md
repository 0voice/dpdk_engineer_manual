# DPDK之内存管理

前言：DPDK的内存管理工作主要分布在几个大的部分：大页初始化与管理，内存管理。使用大页可以减少页表开销，是为了尽量减少TBL miss导致的性能损失。基于大页，DPDK又进一步细化管理这部分内存，使得分配，回收更加方便。

## 1.内存管理的对象说明

### 1.1. 从大页(hugepage)说起

linux内存是按照页来划分的，默认的每页为4K大小，对应的就存在页表（TBL）来记录每个页的地址等该单元的信息。这样就存在一个问题，当访问的内容不在本页时，就会触发 tbl miss,导致页换出换入，很影响性能。而一个解决办法就是使用hugepage，大页的每页大小可以设置，常用设置如2M，1G等，比如1G大小的内存，使用4k的页面，需要256K个，而使用1G的大页，只需要一个。这样子就能大大减少tbl miss的概率。 更加详细的大页的相关内容，请参考下面的链接：

http://www.tuicool.com/articles/vYZJ3i3

## 2. DPDK内存的初始化

内存的初始化在`rte_eal_init()`中完成，由于DPDK的进程分为primary和secondary，内存的初始化工作只能在primory进程中完成。主要的步骤如下：

1. `eal_hugepage_info_init()`；获取大页的信息，并初始化内部的结构。
2. `rte_config_init()`;创建配置文件,并做内存映射。
3. `rte_eal_memory_init()`；大页的内存初始化，并连接成连续的内存区。
4. `rte_eal_memzone_init()`；初始化memzone子系统。

### 2.1 `eal_hugepage_info_init()`

这一步是获取系统中已配置的大页的信息，以及大页的挂载点（在DPDK的参数中可以指定大页的挂载点，默认应该是`/mnt/huge`）。
`dir = opendir(sys_dir_path);`先打开`"/sys/kernel/mm/hugepages"`目录，读取系统中的大页目录，存储在`internal_config.hugepage_info[]`结构中，页面的大小在目录名中。然后获取大页的大小和挂载点：

```c
hpi = &internal_config.hugepage_info[num_sizes];
		hpi->hugepage_sz =
			rte_str_to_size(&dirent->d_name[dirent_start_len]);
		hpi->hugedir = get_hugepage_dir(hpi->hugepage_sz);
```

最后获取空闲页面数量,并且都先放在第一个核上：

```c
hpi->num_pages[0] = get_num_hugepages(dirent->d_name);
```

可以通过设置`MAX_HUGEPAGE_SIZES`宏的值来调整DPDK允许配置的大页的页面值个数。默认是3个。

之后，把这些大页按照大小顺序排一下序，大的页面在前面。

```c
qsort(&internal_config.hugepage_info[0], num_sizes,
	      sizeof(internal_config.hugepage_info[0]), compare_hpi);
```

最后做一下检查，这样，对于大页的信息的获取就做完了。

### 2.2 `rte_config_init()`

因为DPDK支持primary进程和secondary进程，他们都需要内存的配置信息，进程间通信使用了共享内存的方法，把`struct rte_mem_config *mem_config`结构做内存映射。

```c
switch (rte_config.process_type){
case RTE_PROC_PRIMARY:
	rte_eal_config_create();
	break;
case RTE_PROC_SECONDARY:
	rte_eal_config_attach();
	rte_eal_mcfg_wait_complete(rte_config.mem_config);
	rte_eal_config_reattach();
	break;
```

开始就根据进程的类型决定启动顺序问题，如果是primary进程，下面看看他的处理过程：

```c
if (internal_config.base_virtaddr != 0)
		rte_mem_cfg_addr = (void *)
			RTE_ALIGN_FLOOR(internal_config.base_virtaddr -
			sizeof(struct rte_mem_config), sysconf(_SC_PAGE_SIZE));
else
	rte_mem_cfg_addr = NULL;
 
if (mem_cfg_fd < 0){
	mem_cfg_fd = open(pathname, O_RDWR | O_CREAT, 0660);
	if (mem_cfg_fd < 0)
		rte_panic("Cannot open '%s' for rte_mem_config\n", pathname);
}
 
retval = ftruncate(mem_cfg_fd, sizeof(*rte_config.mem_config));
if (retval < 0){
	close(mem_cfg_fd);
	rte_panic("Cannot resize '%s' for rte_mem_config\n", pathname);
}
```

先根据启动的参数选择内存配置文件共享内存开始的地址，如果配置了`base_viraddr`，这个地址应该是可以指定大页开始的地址。在大页开始地址的前面映射内存配置文件。然后打开内存配置文件，裁剪大小。

然后选择地址，映射`sizeof(*rte_config.mem_config)`大小的内存到内存配置文件。

```c
rte_config.mem_config = (struct rte_mem_config *) rte_mem_cfg_addr;
 
/* store address of the config in the config itself so that secondary
 * processes could later map the config into this exact location */
rte_config.mem_config->mem_cfg_addr = (uintptr_t) rte_mem_cfg_addr;
```

填充映射后的地址，这里最后一句比较有意思，把primary进程中映射的地址保存下来，后面我们就会看到，是为了让secondary进程也映射同样的逻辑地址。

接下来就看看secondary进程的地址映射情况：

首先，做了一个attach操作，就是先对共享文件做了映射，记录了映射后的地址。

```c
rte_eal_config_attach()
```

之后，就等待primary进程完整eal层的初始化完成。等初始化完成后，魔数就会填充，`rte_eal_mcfg_complete()`。secondary进程会再次进行内存映射，这次映射的目的就是使得secondary进程中对内存配置文件映射后的逻辑地址和primary进程一样，这样做有什么好处我们后面再仔细说。

```c
rte_mem_cfg_addr = (void *) (uintptr_t) rte_config.mem_config->mem_cfg_addr;
 
munmap(rte_config.mem_config, sizeof(struct rte_mem_config));
mem_config = (struct rte_mem_config *) mmap(rte_mem_cfg_addr,
		sizeof(*mem_config), PROT_READ | PROT_WRITE, MAP_SHARED,
		mem_cfg_fd, 0);
```

最后需要说明的一点是：在DPDK中，创建的mempool，ring等可以在多个进程间访问，也是因为在`rte_config.mem_config`中有个成员是`struct rte_tailq_head tailq_head[RTE_MAX_TAILQ]`，创建的ring等队列头都是挂在其中，是通过构造函数在main函数之前就挂接上的。

### 2.3 `rte_eal_memory_init()`

这个函数是初始化内存子系统，任务很多，对于primary进程，则映射大页内存，而对于secondary进程，则把大页attach到primary进程。

#### 2.3.1 `rte_eal_hugepage_init()`

这就是在primary进程中进行大页的映射。非常有趣，来看看他的主要工作吧！下面直接引用函数原型中的说明:

```c
/*
 * Prepare physical memory mapping: fill configuration structure with
 * these infos, return 0 on success.
 *  1. map N huge pages in separate files in hugetlbfs
 *  2. find associated physical addr
 *  3. find associated NUMA socket ID
 *  4. sort all huge pages by physical address
 *  5. remap these N huge pages in the correct order
 *  6. unmap the first mapping
 *  7. fill memsegs in configuration with contiguous zones
 */
```

首先，获取全局的配置信息：

```c
mcfg = rte_eal_get_configuration()->mem_config;
```

这里比较有意思的地方是，primary进程和secondary进程中配置信息映射的逻辑地址是一样的。

然后获取当前使用的大页的大小和页数。

```c
for (i = 0; i < (int) internal_config.num_hugepage_sizes; i++) {
		/* meanwhile, also initialize used_hp hugepage sizes in used_hp */
		used_hp[i].hugepage_sz = internal_config.hugepage_info[i].hugepage_sz;
 
nr_hugepages += internal_config.hugepage_info[i].num_pages[0];
	}
```

分配大页页表，

```c
tmp_hp = malloc(nr_hugepages * sizeof(struct hugepage_file));
	if (tmp_hp == NULL)
		goto fail;
 
memset(tmp_hp, 0, nr_hugepages * sizeof(struct hugepage_file));
```

然后就到了非常重要的一步：内存映射大页。主要分为三步

1. 第一次映射大页。
2. 按大页的物理地址重新排序。
3. 第二次映射大页。

先看第一次映射大页：`map_all_hugepages(&tmp_hp[hp_offset], hpi, 1)`，最后一个参数就是指明是第一次映射。由于是第一次映射，所以，先填充大页的文件信息

```c
if (orig) {
	hugepg_tbl[i].file_id = i;
	hugepg_tbl[i].size = hugepage_sz;
	eal_get_hugefile_path(hugepg_tbl[i].filepath,
			sizeof(hugepg_tbl[i].filepath), hpi->hugedir,
			hugepg_tbl[i].file_id);
	hugepg_tbl[i].filepath[sizeof(hugepg_tbl[i].filepath) - 1] = '\0';
}
 
```

之后，就在`/mnt/huge`目录下创建每个大页文件，并映射每个大页到内存中。为什么是`/mnt/huge目录`？因为这是挂载大页文件系统的位置，挂载大页文件系统，可以通过`mount -t hugetlbfs nodev /mnt/huge`来挂载。

```c
fd = open(hugepg_tbl[i].filepath, O_CREAT | O_RDWR, 0600);
if (fd < 0) {
	RTE_LOG(DEBUG, EAL, "%s(): open failed: %s\n", __func__,
			strerror(errno));
	return i;
}
 
/* map the segment, and populate page tables,
 * the kernel fills this segment with zeros */
virtaddr = mmap(vma_addr, hugepage_sz, PROT_READ | PROT_WRITE,
		MAP_SHARED | MAP_POPULATE, fd, 0);
```

在这里，新创建的大页文件并没有大小，但是在映射后，文件大小就变成了映射的大小，貌似只能映射页大小的整数倍。
第一次映射，填充`orig_va`地址：
`hugepg_tbl[i].orig_va = virtaddr;`
然后计算下一个页面映射的地址：
`vma_addr = (char *)vma_addr + hugepage_sz;`

等把所有的页面都映射完了后，这部分对应的物理内存就不会被换出到磁盘。此时，我们映射的这部分内存，逻辑地址是连续的，但是物理地址不一定是连续的。

接下来查找已经映射的每个大页的物理地址，并填充其结构。

```c
find_physaddrs()
```

具体的虚拟地址到物理地址的查找关系

```c
rte_mem_virt2phy()
```

然后找到映射的大页内存被放在哪个NUMA node上。

```c
if (find_numasocket(&tmp_hp[hp_offset], hpi) < 0){
	RTE_LOG(DEBUG, EAL, "Failed to find NUMA socket for %u MB pages\n",
			(unsigned)(hpi->hugepage_sz / 0x100000));
	goto fail;
}
```

把映射的大页的物理地址按照从小到大的顺序进行排序。

```c
qsort(&tmp_hp[hp_offset], hpi->num_pages[0],
		      sizeof(struct hugepage_file), cmp_physaddr);
```

接下来就是第二次对大页进行映射：

```c
if (map_all_hugepages(&tmp_hp[hp_offset], hpi, 0) !=
		    hpi->num_pages[0])
```

这里我们看到最后一个参数就已经是0了。
这样进来函数之后，第一个循环时，`vma_len`就是0，然后就去查找物理地址连续的页：

```c
for (j = i+1; j < hpi->num_pages[0] ; j++) {
#ifdef RTE_ARCH_PPC_64
				/* The physical addresses are sorted in
				 * descending order on PPC64 */
				if (hugepg_tbl[j].physaddr !=
				    hugepg_tbl[j-1].physaddr - hugepage_sz)
					break;
#else
				if (hugepg_tbl[j].physaddr !=
				    hugepg_tbl[j-1].physaddr + hugepage_sz)
					break;
#endif
			}
			num_pages = j - i;
			vma_len = num_pages * hugepage_sz;
```

这样，就能确定连续的物理页有几个，然后，去尝试分配和连续物理页一样大的虚拟地址空间，如果不能，就减小一个页再尝试，直到成功（返回地址）或者失败（返回NULL）。如果能拿到地址，那么就以这个地址开始，依次映射物理地址连续的几个页。如果不能拿到这么大且连续的虚拟地址，那么，就让内核自己去分配地址，然后映射这一页。

第二次映射后，就填充`final_va`地址了：`hugepg_tbl[i].final_va = virtaddr;`。

既然已经重新映射了大页的虚拟地址，那么就应该撤销原来的映射。

```c
if (unmap_all_hugepages_orig(&tmp_hp[hp_offset], hpi) < 0)
			goto fail;
```

这样过后，对于大页内存的映射工作就完成了。

接下来就是分配映射的大页内存咯。

首先，清空配置信息中的每个socket中大页的数量，等待重新分配。

```c
for (i = 0; i < (int)internal_config.num_hugepage_sizes; i++)
	for (j = 0; j < RTE_MAX_NUMA_NODES; j++)
		internal_config.hugepage_info[i].num_pages[j] = 0;
```

然后获取每个socket上的大页数量，

```c
for (i = 0; i < nr_hugefiles; i++) {
	int socket = tmp_hp[i].socket_id;
 
	/* find a hugepage info with right size and increment num_pages */
	const int nb_hpsizes = RTE_MIN(MAX_HUGEPAGE_SIZES,
			(int)internal_config.num_hugepage_sizes);
	for (j = 0; j < nb_hpsizes; j++) {
		if (tmp_hp[i].size ==
				internal_config.hugepage_info[j].hugepage_sz) {
			internal_config.hugepage_info[j].num_pages[socket]++;
		}
	}
}
```

重新计算调整每个socket上的大页的分布，最后返回大页个数。

```c
nr_hugepages = calc_num_pages_per_socket(memory,
			internal_config.hugepage_info, used_hp,
			internal_config.num_hugepage_sizes);
```

默认每个socket上的大页数量是按核心数量的比例分配的。

然后为大页映射信息文件创建共享内存，用于secondary进程来映射地址。

先撤销不用的大页映射，然后把临时大页信息文件拷贝到创建的共享内存中。

```c
if (unmap_unneeded_hugepages(tmp_hp, used_hp,
			internal_config.num_hugepage_sizes) < 0) {
		RTE_LOG(ERR, EAL, "Unmapping and locking hugepages failed!\n");
		goto fail;
	}
 
 
if (copy_hugepages_to_shared_mem(hugepage, nr_hugefiles,
			tmp_hp, nr_hugefiles) < 0) {
		RTE_LOG(ERR, EAL, "Copying tables to shared memory failed!\n");
		goto fail;
	}
```

最后把大页内存切成段保存在内存管理结构中。大页内存切段的条件是：

- 不在同一个socket上。
- 页的大小不相同
- 物理地址不连续
- 虚拟地址不连续

然后把切好的内存段放入mcfg配置表中：

```c
mcfg->memseg[j].phys_addr = hugepage[i].physaddr;
mcfg->memseg[j].addr = hugepage[i].final_va;
mcfg->memseg[j].len = hugepage[i].size;
mcfg->memseg[j].socket_id = hugepage[i].socket_id;
mcfg->memseg[j].hugepage_sz = hugepage[i].size;
```

这样，大页的初始化就完成了！

2.3.2 `rte_eal_hugepage_attach()`

对于secondary进程而言，它并不能创建大页的共享内存，而只能attach上去。

开始大页内存attach的前提是先attach内存配置文件，我们再来看一下attach配置的过程：

```c
rte_eal_config_attach();
rte_eal_mcfg_wait_complete(rte_config.mem_config);
rte_eal_config_reattach();
```

第一个函数中，先映射一下`/var/run/.rte_config`文件，拿到内存配置的结构信息，就是为了第二个函数的等待判断用的。第三个函数中，既然主进程已经初始化完成，那么，就先解除第一个函数的映射，以primary进程中映射的内存配置文件地址作为新的映射地址，重新映射，映射完成后，primary进程和secondary进程中，对于`/var/run/.rte_config`映射的虚拟地址是一样的。（虽然，对于配置文件映射地址一样，感觉并没什么卵用~，但后面的大页映射也是这么做的，映射地址的一致，就有用啦）。

接下来就来看大页内存的attach，首先打开`/dev/zero`文件，按照primary的段的虚拟地址依次映射所有的内存段，这一步相当于先测试一下是否能分配这样的连续地址空间。

```c
base_addr = mmap(mcfg->memseg[s].addr, mcfg->memseg[s].len,
				 PROT_READ, MAP_PRIVATE, fd_zero, 0);
```

然后映射大页信息共享文件`/var/run/.rte_hugepage_info`，并计算页个数等。

```c
size = getFileSize(fd_hugepage);
hp = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd_hugepage, 0);
if (hp == MAP_FAILED) {
	RTE_LOG(ERR, EAL, "Could not mmap %s\n", eal_hugepage_info_path());
	goto error;
}
 
num_hp = size / sizeof(struct hugepage_file);
```

最后解除映射到`/dev/zero`，重新映射到各个大页文件中，

```c
for (i = 0; i < num_hp && offset < mcfg->memseg[s].len; i++){
	if (hp[i].memseg_id == (int)s){
		fd = open(hp[i].filepath, O_RDWR);
		if (fd < 0) {
			RTE_LOG(ERR, EAL, "Could not open %s\n",
				hp[i].filepath);
			goto error;
		}
		mapping_size = hp[i].size;
		addr = mmap(RTE_PTR_ADD(base_addr, offset),
				mapping_size, PROT_READ | PROT_WRITE,
				MAP_SHARED, fd, 0);
		close(fd); /* close file both on success and on failure */
		if (addr == MAP_FAILED ||
				addr != RTE_PTR_ADD(base_addr, offset)) {
			RTE_LOG(ERR, EAL, "Could not mmap %s\n",
				hp[i].filepath);
			goto error;
		}
		offset+=mapping_size;
	}
}
```

到这里我们仔细看一下，进程中是以primary中的虚拟地址作为映射地址来映射的，也就是说在映射完成后，primary进程和secondary进程中映射的大页地址是一样的。这很关键，这正是实现零拷贝的原理。虚拟地址一样，那么从大页内存中拿到的数据包，就可以不经过拷贝，直接把地址传到secondary进程中。

这些都映射完了后，就完成了attach工作。

### 2.4 `rte_eal_memzone_init()`

memzone是内存分配器，上一步中，我们已经把大页内存分段放好了，但是在使用的时候，怎么来分配呢？自然需要内存分配器，就是memzone。而`memzone_init`主要就是把内存放到空闲链表中，等需要的时候，能够分配出去。

在看初始化前，先看一个结构，`struct malloc_elem`，这个结构表示一个内存对象，

```c
struct malloc_elem {
	struct malloc_heap *heap;
	struct malloc_elem *volatile prev;      /* points to prev elem in memseg */
	LIST_ENTRY(malloc_elem) free_list;      /* list of free elements in heap */
	const struct rte_memseg *ms;
	volatile enum elem_state state;
	uint32_t pad;
	size_t size;
#ifdef RTE_LIBRTE_MALLOC_DEBUG
	uint64_t header_cookie;         /* Cookie marking start of data */
	                                /* trailer cookie at start + size */
#endif
} __rte_cache_aligned;
 
```

然后看初始化

```c
rte_eal_malloc_heap_init()
```

依次把每一段都添加到heap中，段属于哪个socket，就添加到哪个socket的heap中，分配就从这里拿。

```c
for (ms = &mcfg->memseg[0], ms_cnt = 0;
			(ms_cnt < RTE_MAX_MEMSEG) && (ms->len > 0);
			ms_cnt++, ms++) {
		malloc_heap_add_memseg(&mcfg->malloc_heaps[ms->socket_id], ms);
	}
```

把每一段做初始化，并挂在空闲链表中:

```c
malloc_elem_init(start_elem, heap, ms, elem_size);
malloc_elem_mkend(end_elem, start_elem);
malloc_elem_free_list_insert(start_elem);
 
heap->total_size += elem_size;
```

然后就初始化完了！

## 3. DPDK内存的分配

内存分配有一系列的接口：大多定义在`rte_malloc.c`文件中。我们重点挑两个来看一下。

- `rte_malloc_socket()`
  这个是一个基础函数，可以在这个函数的基础上进行封装，主要参数是类型，大小，对齐，以及从哪个socket上分。一般来说，分配内存从当前线程运行的socket上分配，可以避免内存跨socket访问，提高性能。

```c
ret = malloc_heap_alloc(&mcfg->malloc_heaps[socket], type,
				size, 0, align == 0 ? 1 : align, 0);
if (ret != NULL || socket_arg != SOCKET_ID_ANY)
	return ret;
```

先在指定的socket上分配，如果不能成功，再去尝试其他的socket。我们接着看函数`malloc_heap_alloc()`:

```c
void *
malloc_heap_alloc(struct malloc_heap *heap,
		const char *type __attribute__((unused)), size_t size, unsigned flags,
		size_t align, size_t bound)
{
	struct malloc_elem *elem;
 
	size = RTE_CACHE_LINE_ROUNDUP(size);
	align = RTE_CACHE_LINE_ROUNDUP(align);
 
	rte_spinlock_lock(&heap->lock);
 
	elem = find_suitable_element(heap, size, flags, align, bound);
	if (elem != NULL) {
		elem = malloc_elem_alloc(elem, size, align, bound);
		/* increase heap's count of allocated elements */
		heap->alloc_count++;
	}
	rte_spinlock_unlock(&heap->lock);
 
	return elem == NULL ? NULL : (void *)(&elem[1]);
```

先去空闲链表中找是否有满足需求的内存块，如果找到，就进行分配，否则返回失败。进一步的，在函数`malloc_elem_alloc()`分配的的时候，如果存在的内存大于需要的内存时，会对内存进行切割，然后把用不完的重新挂在空闲链表上。就不细致的代码分析了。

- `rte_memzone_reserve_aligned()`
  这个函数的返回值类型是`struct rte_memzone`型的，这是和上一个分配接口的不同之处，同时注意分配时的flag的不同。分配出来的memzone可以直接使用名字索引到。这个函数最终也是会调用到`malloc_heap_alloc()`,就不多说了，可以看看代码。

除此以外，需要额外提到的内存分配的地方是创建内存池。在创建内存池时，会创建一个ring来存储分配的对象，同时，为了减少多核之间对同一个ring的访问，每一个核都维护着一个cache，优先从cache中取。

## 4. DPDK内存的回收

说完了DPDK的内存分配，最后来说一下内存回收。跟分配的接口对应，也有多个回收函数。

- `rte_free()`
  同样这个函数，在上层封装了多种接口。如`rte_memzone_free()`等。主要的过程也是重新把elem放进free list上，如果有能够合并的，还会对其进行合并。
- `rte_memzone_free()`
  上面都说过了，这个里面也是对`rte_free()`的封装，不多说了，just see the code!

同样，关于回收也有点注意的，对于内存池中的元素的回收，不是释放回空闲链表，而是重新放到ring或者cache中，就这么多了。







原创作者：[AISEED](https://home.cnblogs.com/u/yhp-smarthome/)    本文链接：https://www.cnblogs.com/yhp-smarthome/p/6995292.html
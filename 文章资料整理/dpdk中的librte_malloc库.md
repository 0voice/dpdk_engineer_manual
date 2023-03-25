# dpdk中的librte_malloc库

dpdk中的librte_malloc库提供了能够分配任意大小内存的API。

该库的目标是提供类似malloc的函数从hugepage中分配内存，以及帮助应用程序移植。

通常情况下，这种类型的分配不应该在数据平面处理，因为其比基于内存池的分配更慢，

并且在分配和释放时会使用锁。

不过，可以将其用在配置代码中。

## **1. Cookies**

如果在配置文件中打开CONFIG_RTE_MALLOC_DEBUG，

分配的内存会包含覆盖保护区域，以识别缓冲区溢出问题。

## **2. 对齐与NUMA Constraints**

rte_malloc()函数包含一个align参数，用来要求内存区域对齐到该值的倍数（必须是2的倍数）。

在支持NUMA的系统中，调用rte_malloc()函数时，会在调用该函数的进程所在的socket上分配内存。

同时该库也提供了一组API，使用户可以直接在指定的NUMA socket上分配内存，

或者在另一个core所在的NUMA socket上分配内存。

## **3. 用例**

应用程序在初始化时使用类似malloc这样的函数时，可以使用该库。

要在运行时分配/释放内存数据，如果应用程序对速度有要求，

请用内存池库代替本库。

如果要使用一块需要知道物理地址的内存块，如硬件设备使用的内存块，

则应该使用memory zone。

 

## **4. 数据结构**

在malloc库的内部使用两种数据结构类型：

struct malloc_heap: 用来管理每个socket上的空闲空间

struct malloc_elem: 分配的基本元素，由库内部管理的空闲空间。

 

### 4.1struct malloc_heap

该结构体用来管理每个socket上的空闲空间。

在库的内部，每个NUMA node上包含一个 heap结构体，

使我们可以根据线程运行所在的NUMA node，在对应的结点分配内存。

虽然不能保存一定会在指定的结点上分配内存，但比总在某个固定的的结点或随机结点分配要好。

heap的关键成员变量和成员函数描述如下：

mz_count: 保存本结点已经为heap内存分配的memory zone的数量。该值的唯一用途就是与numa_socket值组合为每个memory zone生成一个唯一的名字。

lock: 该变量用来做对heap访问的同步。考虑到heap中的空闲空间是由一个list管理的，所以我们需要一个锁来防止两个线程同时访问该list。

free_head: 该变量该malloc heap的free nodes list中的第一个元素。

 

注意: malloc_heap结构体不会管理已经分配的memzones，这么做是毫无意义的，因为它们不会被释放。

也不会管理使用中的内存块，因为除非它们被释放，否则是不会再次接触到这些内存块的。

在释放时，指向这些内存块的指针会作为free()函数的参数。

![img](https://sysight.com/?qa=blob&qa_blobid=583232675911585503)

### **4.2 struct malloc_elem结构体**

malloc_elem结构体被用作memzone中各种内存块的头部结构。

有三种不同的用法：

1、分配或释放内存块时的头部 - 普通情况

2、在内存块中作为padding头部

3、作为memzone结尾处的标记

下文描述了结构中最重要的部分以及用法。

 

注意：如果某种用法不属于上面描述的三种中的任何一种，则认为对应的变量是未定义的。

例如，只有当"state"和"pad"两个变量的值是有效值是，才认为其是一个padding header。

 

head：该指针是已经分配的内存块中指向heap结构的反向引用，即指向对应的heap。

​    普通内存块在释放时会使用该指针，将当前释放的内存块添加到heap的free list中。

 

prev：该指针指向memzone中当前内存块紧前面的内存块的header element/block。

​    当释放一个内存块时，该指针用来引用前一个内存块，看其是否也需要释放。

​    如果需要，则两块内存组合成一块更大的内存块。

 

next_free：该指针用来将未分配的内存块链接到一起。

​    同样，该变量只在普通内存块中使用，在malloc()函数中找到一块符合需求的内存块来分配，

​    并且在调用free()函数将新释放的内存添加到free-list中。

 

state：该变量可以是以下三个值之一：“Free”， “Busy”或“Pad”。

​    前两个用业表示普通内存块的分配状态，

​    第三个用来表示在start-of-block padding的结尾处的元素结构体是一个dummy结构体。

​    （例如，由于强制对齐，内存块中数据的开始处不在内存块中。？？？）

​    在这种情况下，pad header用来定位实际分配的元素header。

​    对于end-of-memzone结构体，该值总是“busy”，

​    以确保在释放时没有元素为了整合成一个更大的内存块，而在memzone的结尾外面查找其它内存块。

 

pad：该变量保存内存块开始处的padding区域的长度。

​    如果是普通内存块header，该值会被加到header的结尾处的地址，以给出数据区域的正确地址。

​    例如，在调用malloc函数时传回的值。

​    在padding中的dummy header的内部，该值也会被保存，

​    and is subtracted from the address of the dummy header to yield  the address of the actual block header.

 

size：表示数据内存块的大小，包含header自身。对于end-of-memzone结构，该值为0，虽然从不会检查该值。

​    对于被释放的普通内存块，该值用来代替“next”指针，用来计算下一个内存块所在的地址。

​    （因此如果下一个内存块也是free的，两个内存块可以整合成一个）。

 

### **4.3 内存分配**

应用程序调用类似malloc的函数时，malloc函数首先会根据调用线程索引lcore_config结构，

以及根据该线程确定其所在的NUMA结点。

即用来索引malloc_head结构数组，之后以该数组为参数调用heap_alloc()函数，

同时作为参数的还有要分配的大小，类型和对齐。

heap_alloc()函数会扫描heap的free_list，并尝试找到一个合适大小的内存块来存储数据，同时强制对齐。

如果没有找到合适大小的内存块，例如，第一次在某结点上调用malloc函数时free-list是空的，

则会创建一个新的memzone并配置为heap元素，其会将一个dummy结构放置到memzone的结尾处，

作为一个标记，防止访问超出这块内存之外（由于该标记被置为“BUSY”，malloc库永远无法将这块内存分配出去）。

同时在memzone的开始处放置一个合适的element header。这个header标记了memzone中的所有空间，

bar the sentinel value at the end，end, as a single free  heap element, and it is then added to the free_list for the heap.

 

新的memzone配置好之后，会重新对heap的free-list进行描述，这次描述会找到新添加的合适大小的元素，

将其作为memzone中保留内存的大小，至少是调用函数中指定的大小的数据内存块加上对齐，

至少是Intel DPDK运行时配置中指定的最小大小。

 

找到一个合适大小的空闲元素之后，会计算返回到用户的指针，包含提供给用户的空闲内存块结尾处的空间。

紧跟着这块内存的cache-line被填充一个struct malloc_elem头：

如果内存块中余下的空间比较小，如<=128字节，就会使用一个pad header，余下的空间就浪费了。

不过，如果余下的空间大于128字节，则这块空闲内存块就被分成两份，

一个新的，合适的malloc_elem头被放到返回的数据空间之前。

从已经存在的元素的结尾分配内存的好处是，在这种情况下，不需要调整free list——

free list中已经存在的元素已经调整过尺寸指针了，后面element的“prev”指针已经重新指向这个新创建的element了。



### **4.4 释放内存**

要释放内存，需要将指向数据区域起始地址的指针传递给free函数。

函数会从指针中减去malloc_elem结构的大小以获取内存块的element header。

如果header的类型是“PAD”，则再从指针中减去pad的长度。

 

从该element指针中，可以获取到指向堆的来源和需要释放到哪里的指针，

以及指向前一个元素的，并且通过size变量，可以计算下一个元素的指针。

之后也会检查后面的和前面的元素，看其是否也需要被释放。

这意味着永远不会发生两个空闲内存块相邻的情况，这样的内存块总是会被整合成一个更大的内存块。



原文链接：https://sysight.com/index.php?qa=20&qa_1=dpdk%E4%B8%AD%E7%9A%84librte_malloc%E5%BA%93     原文作者“ sysight
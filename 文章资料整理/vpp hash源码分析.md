# vpp hash源码分析

## 1.**概述**

vpp的hash结构分为hash头、桶（_hash_create或hash_resize申请）和桶下元素（clib_mem_realloc申请），总共3个部分组成。

根据元素key的hash值不同，分配到不同的桶下，与其他hash表原理相同。

vpp的hash结构默认是支持动态扩容的，即当hash表存放键值对大于3/4哈希桶的个数时，会2倍扩容。

哈希冲突时，vector长度认为是没有限制的。

如非特殊说明，以存储key和value来进行介绍，即log2_pair_size > 0。

## 2.**hash结构图**

![img](https://pic3.zhimg.com/80/v2-2e6718c44da84803502161d1efa3da5a_720w.webp)

hash struct (原图更清晰)

1. hash_t：hash_header

```c
/* Vector header for hash tables. */
typedef struct hash_header
{
  /* 哈希表元素数量，包括所有桶下的元素 */
  uword elts;

  /* 标记位，包括以下3种取值 */
  u32 flags;

  /* 自动扩容，默认打开 */
#define HASH_FLAG_NO_AUTO_GROW		(1 << 0)
  /* 自动缩减，默认关闭 */
#define HASH_FLAG_NO_AUTO_SHRINK	(1 << 1)
  /* 当hash_next在遍历此散列表的过程中进行时设置 */
#define HASH_FLAG_HASH_NEXT_IN_PROGRESS (1 << 2)

  /* 0：不存储key  >0：存储key+value大小的log2值 */
  u32 log2_pair_size;

  /* 哈希函数，包括以下5种取值 */
  hash_key_sum_function_t *key_sum;

  /* Special values for key_sum "function". */
#define KEY_FUNC_NONE		(0)	/*< sum = key */
#define KEY_FUNC_POINTER_UWORD	(1)	/*< sum = *(uword *) key */
#define KEY_FUNC_POINTER_U32	(2)	/*< sum = *(u32 *) key */
#define KEY_FUNC_STRING         (3)	/*< sum = string_key_sum, etc. */
#define KEY_FUNC_MEM		(4)	/*< sum = mem_key_sum */

  /* 比较函数 */
  hash_key_equal_function_t *key_equal;

  /* 用户自定义数据 */
  any user;

  /* Format函数 */
  format_function_t *format_pair;

  /* Format函数参数 */
  void *format_pair_arg;

  /* 1：有一个元素   0：无元素或有冲突元素 */
  uword is_user[0];
} hash_t;
```

\2. hash结构内存分布：

hash表的handler是hash_t指针，经过初始化、插入元素后，内存结构如上图所示。

（1）红色部分（hash结构）是一块连续内存，在_hash_create后形成

（2）每个桶下的元素，是在lookup--->set_indirect--->clib_mem_realloc中申请的（2个及以上元素时）

\3. 存储key和value：

（1）log2_pair_size=0时，不存储value，hash_pair_indirect_t中的pairs是vec结构，上图黄色所示

（2）log2_pair_size>0时，存储value，value_len = 2^log2_pair_size - 1

（3）key的大小是uword的，在x64系统是8字节。如果key超过8字节，只能存储指针，创建时使用hash_create_mem函数，key会被保存，但无法访问，是hash表的私有内存。mhash可以存储key，hash常常与pool配合使用

\4. hash无冲突：

使用hash_pair_t记录key和value。如果is_user[i]为0，没有元素；为1，则表示有1个元素

\5. hash冲突：

使用hash_pair_indirect_t记录key和value，且is_user[i]为0。

在_hash_create时，桶创建的结构体是hash_pair_t。在冲突时，将hash_pair_t*转成hash_pair_union_t*，使用其中的hash_pair_indirect_t*指针，获取hash_pair_indirect_t中内容。如果不理解此段，可看下面的示例

仔细分析lookup（SET）---> set_indirect_is_user流程：当前桶下1个元素扩充到2个元素的过程。

（1）首先，调用key_sum函数，获取桶的index

（2）通过get_pair拿到桶下唯一元素，此时 hash_pair_t *强转成hash_pair_union_t *，实际内容是 hash_pair_t

（3）hash_is_user函数的返回值是1，因为只有一个元素，准备SET第二个元素

（4）found_key当然是没找到了，准备加入新元素，否则，覆盖之后还是一个元素

（5）调用set_indirect_is_user函数，hash_pair_indirect_t *pi = &p->indirect，在(2)中，p是hash_pair_t *强转成hash_pair_union_t *的，p只有hash_pair_t内容。pi的第一个元素是hash_pair_t *pairs，那么pi->pairs的地址与之前p->direct的地址是一致的，这样就把最初的hash_pair_t*转换成了hash_pair_indirect_t的pairs了。用clib_memcpy_fast将第一个元素拷贝到新申请的内存中，然后将pi->pairs指向新申请的内存，更新长度、位图等标记位，最终返回新元素，即第二个元素

## 3.**主要函数**

\1. hash_create：

有7种hash创建函数，hash_create_mem（key是指针）、hash_create_vec（key是vec数组）、hash_create_shmem（同mem）、hash_create_string（key是string）、hash_create、hash_create_uword（key是8字节指针）、hash_create_u32（key是4字节指针）。

所有函数最终会调用 hash_create2函数，初始化format、hash等参数，然后调用_hash_create函数，申请所有桶，再初始化剩余参数，elts、log2_pair_size等。

hash表使用前只需要调用 hash_create_*** 一个函数即可。

![img](https://pic4.zhimg.com/80/v2-a6d3050fe008c5f99d49f21747f9e8bb_720w.webp)

hash create后内存结构

\2. hash_set：

hash_set最终会调用 lookup函数，op参数是SET。

（1）key_sum：获取桶的index，hash_create_***确定使用哪种hash 函数来计算

（2）get_pair：通过index，找到对应桶

（3）hash_is_user：判断当前桶是否有元素，是否冲突。取值是1（无冲突），桶中有1个元素，调用set_indirect_is_user函数，再添加一个，变成2个，上面已进行过分析；取值是0，分两种情况：

【1】空桶，直接赋值，调用set_is_user函数设置位图

【2】冲突，set_indirect函数处理冲突，即元素挂在桶的末尾

![img](https://pic4.zhimg.com/80/v2-962a74274495c140cf2e72c71a585ca3_720w.webp)

hash桶中1个元素或多个元素内存结构

\3. hash_unset：

hash_unset最终会调用 lookup函数，op参数是UNSET。

（1）key_sum函数获取桶的index，get_pair函数找到对应桶

（2）如果桶下只有一个元素（hash_is_user返回值是1），set_is_user设置位图，标记为无元素，zero_pair将元素清空；如果桶下有多个元素或无元素（hash_is_user返回值是0），get_indirect函数逐个key对比，找到对应元素或未找到元素，找到情况下，unset_indirect进行unset设置

（3）unset_indirect：找到末尾元素e，桶下元素长度len，分别处理len <=2 和 len > 2两种情况。len <=2时，分为=2和=1两种情况，=2时判断删除的是桶下第一个还是第二个元素；=1时，直接删除这个元素即可。 len > 2时，拷贝末尾元素到待删除元素位置，然后更新len，如下图所示

![img](https://pic2.zhimg.com/80/v2-0d4383e42b196b4eb9bd704273c78ffd_720w.webp)

unset_indirect函数 len &amp;amp;amp;amp;amp;amp;gt; 2 情况

（4）更新elts等变量

\4. hash_get：

hash_get最终会调用 lookup函数，op参数是GET。

key_sum函数获取桶的index，get_pair函数找到对应桶，通过key_equal或get_indirect函数找到对应的元素，返回；未找到，则返回空。

\5. hash_free：

通过vec_free或clib_mem_free函数释放每个桶下的元素，最后释放桶和所有vec内存。

\6. hash_foreach_pair：

循环每个桶下的每个元素，调用body动作，与 hash_free 循环流程相似。

## 4.**函数调用**

以ipiptunne_key_t 为例，介绍接口使用情况。

```text
ipip_main_t *gm = &ipip_main;
/* 创建 */
gm->tunnel_by_key =
    hash_create_mem (0, sizeof (ipip_tunnel_key_t), sizeof (uword));

/* 添加 */
hash_set_mem_alloc (&gm->tunnel_by_key, key, t->dev_instance);

/* 删除 */
hash_unset_mem_free (&gm->tunnel_by_key, key);

/* 查询 */
uword *p = hash_get_mem (gm->tunnel_by_key, key);
```

原文链接：https://zhuanlan.zhihu.com/p/390870663  原文作者： 大白鲨


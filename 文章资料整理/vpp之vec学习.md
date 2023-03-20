# vpp之vec学习

## 概念理解

vpp的vec（向量）是一个允许动态调整大小的数组，数组的类型可以是任意的c所有数据结构类型，
通过vec来操作数组数据非常方便.

一个vec包含以下几部分：

```c++
				 用户自定义头部(aligned to uword boundary) 可选
				 vec头部(vec_header_t, 主要是定义了vector length: number of elements)
user's pointer-> vector element #0
				 vector element #1
				 ...
				 

typedef struct
{
#if CLIB_VEC64 > 0
  u64 len;
#else
  u32 len; /**< Number of elements in vector (NOT its allocated length). */
  u32 dlmalloc_header_offset;	/**< offset to memory allocator offset  */
#endif
  u8 vector_data[0];  /**< Vector data . */
} vec_header_t;				 

```

vpp使用vec_resize_allocate_memory来申请内存,申请到的返回值vec是元素0的地址，不是vec头部地址，需要注意

```c++
void *
vec_resize_allocate_memory (void *v,
			    word length_increment,
			    uword data_bytes,
			    uword header_bytes, uword data_align)
{
  vec_header_t *vh = _vec_find (v);
  uword old_alloc_bytes, new_alloc_bytes;
  void *old, *new;
  
  # 获取vec头部大小，这里的header_bytes传进来的时候是私有头部的大小，需要加上sizeof (vec_header_t), 并对齐为2的指数次方
  header_bytes = vec_header_bytes (header_bytes);
  
  # 更新内存总的长度
  data_bytes += header_bytes;
  
  # 如果v是空的，则申请内存new，并把v设置为new + header_bytes，即用户数据开始位置
  if (!v)
    {
      new = clib_mem_alloc_aligned_at_offset (data_bytes, data_align, header_bytes, 1	/* yes, call os_out_of_memory */
	);
      data_bytes = clib_mem_size (new);
      clib_memset (new, 0, data_bytes);
      v = new + header_bytes;
	  # 保存vec元素的数量到_vec_len (v)
      _vec_len (v) = length_increment;
      return v;
    }
  
  # 如果v不为空，表示已经存在，需要在原有内存的基础上考虑是否增加
  vh->len += length_increment;
  # 旧的vec头部指针
  old = v - header_bytes;
   
  # 这里应该是判断旧的vec头部地址在合理的范围内
  /* Vector header must start heap object. */
  ASSERT (clib_mem_is_heap_object (old));
  
  # 获取旧的vec总的内存大小
  old_alloc_bytes = clib_mem_size (old);
  
  # 如果旧的vec总的内存够用，则直接返回v即可，不需要申请新的内存
  /* Need to resize? */
  if (data_bytes <= old_alloc_bytes)
    return v;

  # 如果新的内存的大小大于旧的内存的3/2的话，则直接按照新的内存大小申请内存，否则按照旧的内存的3/2大小申请内存
  new_alloc_bytes = (old_alloc_bytes * 3) / 2;
  if (new_alloc_bytes < data_bytes)
    new_alloc_bytes = data_bytes;
  
  # 下面就是申请新的new后拷贝old的数据到new，在返回v + header_bytes给用户
  new =
    clib_mem_alloc_aligned_at_offset (new_alloc_bytes, data_align,
				      header_bytes,
				      1 /* yes, call os_out_of_memory */ );

  /* FIXME fail gracefully. */
  if (!new)
    clib_panic
      ("vec_resize fails, length increment %d, data bytes %d, alignment %d",
       length_increment, data_bytes, data_align);

  clib_memcpy_fast (new, old, old_alloc_bytes);
  clib_mem_free (old);

  /* Allocator may give a bit of extra room. */
  new_alloc_bytes = clib_mem_size (new);
  v = new;

  /* Zero new memory. */
  memset (v + old_alloc_bytes, 0, new_alloc_bytes - old_alloc_bytes);

  return v + header_bytes;
}

```

## 实际使用举例说明

创建一个u64的vec指针变量，
第1步、使用vec_add1增加一个elm元素，
第2步、使用vec_add1增加一个elm元素，
第3步、使用vec_add批量增加10个元素，
第4步、使用vec_add2增加10个元素，这个时候，vec里面有22个元素，具体代码如下：

```c++
u64 elm[1024];
u64 *vec_u64 = 0;
u64 *elm_add2 = 0;
int index = 0;
u64 *tmp_elm;

for (index = 0; index < 1024; index++) {
	elm[index] = index;
}

index = 0;
vec_add1(vec_u64, elm[index]);
index++;
Log(CM, DEBUG, "vec_u64:%p", vec_u64);
Log(CM, DEBUG, "vec_len:%d", vec_len(vec_u64));
Log(CM, DEBUG, "vec_bytes:%d", vec_bytes(vec_u64));
Log(CM, DEBUG, "vec_capacity:%d", vec_capacity(vec_u64,0));
Log(CM, DEBUG, "vec_max_len:%d", vec_max_len(vec_u64));
Log(CM, DEBUG, "vec_end:%p", vec_end(vec_u64));
vec_add1(vec_u64, elm[index]);
index++;
Log(CM, DEBUG, "vec_u64:%p", vec_u64);
Log(CM, DEBUG, "vec_len:%d", vec_len(vec_u64));
Log(CM, DEBUG, "vec_bytes:%d", vec_bytes(vec_u64));
Log(CM, DEBUG, "vec_capacity:%d", vec_capacity(vec_u64,0));
Log(CM, DEBUG, "vec_max_len:%d", vec_max_len(vec_u64));
Log(CM, DEBUG, "vec_end:%p", vec_end(vec_u64));
vec_add(vec_u64, &elm[index], 10);
index+=10;
Log(CM, DEBUG, "vec_u64:%p", vec_u64);
Log(CM, DEBUG, "vec_len:%d", vec_len(vec_u64));
Log(CM, DEBUG, "vec_bytes:%d", vec_bytes(vec_u64));
Log(CM, DEBUG, "vec_capacity:%d", vec_capacity(vec_u64,0));
Log(CM, DEBUG, "vec_max_len:%d", vec_max_len(vec_u64));
Log(CM, DEBUG, "vec_end:%p", vec_end(vec_u64));
vec_add2(vec_u64, elm_add2, 10);
for (index = 0; index < 10; index++) {
	elm_add2[index] = index;
}
Log(CM, DEBUG, "vec_u64:%p, elm_add2:%p", vec_u64, elm_add2);
Log(CM, DEBUG, "vec_len:%d", vec_len(vec_u64));
Log(CM, DEBUG, "vec_bytes:%d", vec_bytes(vec_u64));
Log(CM, DEBUG, "vec_capacity:%d", vec_capacity(vec_u64,0));
Log(CM, DEBUG, "vec_max_len:%d", vec_max_len(vec_u64));
Log(CM, DEBUG, "vec_end:%p", vec_end(vec_u64));

index = 0;
vec_foreach(tmp_elm, vec_u64) {
	Log(CM, DEBUG, "vec_u64[%d]:%d", index++, *tmp_elm);
}

if (vec_u64)
	vec_free(vec_u64);

```

这里分别使用了vec_add1、vec_add、vec_add2来往vec里面增加元素，他们使用上的参数有区别，需要注意：
vec_add1(V,E) 第二个参数是elm变量，不是使用变量的地址
vec_add(V,E,N) 第二个参数是elm数组的头地址，不是变量的值，需要注意，虽然这里都用E来表示
vec_add2(V,P,N) 第二个参数是一个指针变量，不需要初始化，vec_add2以后，vec会把申请到的空间地址保存到P参数，
这个时候我们需要把我们要保存的变量保存到P指针的位置。

结果如下:

```c++
vec_u64:0x7fc66448f82c
vec_len:1
vec_bytes:8
vec_capacity:28
vec_max_len:3
vec_end:0x7fc66448f834

vec_u64:0x7fc66448f82c
vec_len:2
vec_bytes:16
vec_capacity:28
vec_max_len:3
vec_end:0x7fc66448f83c

vec_u64:0x7fc66446c08c
vec_len:12
vec_bytes:96
vec_capacity:108
vec_max_len:13
vec_end:0x7fc66446c0ec

vec_u64:0x7fc66330eecc, elm_add2:0x7fc66330ef2c
vec_len:22
vec_bytes:176
vec_capacity:188
vec_max_len:23
vec_end:0x7fc66330ef7c
vec_u64[0]:0
vec_u64[1]:1
vec_u64[2]:2
vec_u64[3]:3
vec_u64[4]:4
vec_u64[5]:5
vec_u64[6]:6
vec_u64[7]:7
vec_u64[8]:8
vec_u64[9]:9
vec_u64[10]:10
vec_u64[11]:11
vec_u64[12]:0
vec_u64[13]:1
vec_u64[14]:2
vec_u64[15]:3
vec_u64[16]:4
vec_u64[17]:5
vec_u64[18]:6
vec_u64[19]:7
vec_u64[20]:8
vec_u64[21]:9
```

从上面可以看出来，每一个add后，如果vec预留的大小不够用的话，vec会申请新的内存，
否则直接使用旧的内存(上面例子第一步和第二步的时候，vec_u64地址没有变化)

## api说明

1.vec_new

```c++
/** \brief Create new vector of given type and length
    (unspecified alignment, no header).

    @param T type of elements in new vector
    @param N number of elements to add
    @return V new vector
*/
#define vec_new(T,N)           vec_new_ha(T,N,0,0)

```

创建指定类型和数量的vec变量，不包含头部和没有指定对齐，
返回值是得到的vec变量

2.vec_free

```c++
/** \brief Free vector's memory (no header).
    @param V pointer to a vector
    @return V (value-result parameter, V=0)
*/
#define vec_free(V) vec_free_h(V,0)

```

释放vec变量

3.vec_add1

```
/** \brief Add 1 element to end of vector (unspecified alignment).

    @param V pointer to a vector
    @param E element to add
    @return V (value-result macro parameter)
*/
#define vec_add1(V,E)           vec_add1_ha(V,E,0,0)

```

增加一个元素到指定的vec里面，E参数是元素的值而不是元素的地址
返回值是vec变量

4.vec_add2

```c++
/** \brief Add N elements to end of vector V,
    return pointer to new elements in P. (no header, unspecified alignment)

    @param V pointer to a vector
    @param P pointer to new vector element(s)
    @param N number of elements to add
    @return V and P (value-result macro parameters)
*/

#define vec_add2(V,P,N)           vec_add2_ha(V,P,N,0,0)

```

增加N个元素到指定的vec里面，P参数是输入的时候是空指针
返回值是vec变量和P指针，vec会申请的空间地址保存到P参数，
这个时候我们需要把我们要保存的变量保存到P指针的位置

5.vec_add

```c++
/** \brief Add N elements to end of vector V (no header, unspecified alignment)

    @param V pointer to a vector
    @param E pointer to element(s) to add
    @param N number of elements to add
    @return V (value-result macro parameter)
*/
#define vec_add(V,E,N)           vec_add_ha(V,E,N,0,0)

```

增加N个元素到指定的vec里面，E参数是插入元素数组的头地址，不是变量的值
返回值是vec变量

6._vec_find

```c++
/** \brief Find the vector header

    Given the user's pointer to a vector, find the corresponding
    vector header

    @param v pointer to a vector
    @return pointer to the vector's vector_header_t
*/
#define _vec_find(v)	((vec_header_t *) (v) - 1)

```

根据vec变量获取vec头部地址

7.vec_len

```c++
/** \brief Number of elements in vector (rvalue-only, NULL tolerant)

    vec_len (v) checks for NULL, but cannot be used as an lvalue.
    If in doubt, use vec_len...
*/

#define vec_len(v)	((v) ? _vec_len(v) : 0)

```

获取vec内部元素的数量

8.vec_reset_length

```c++
/** \brief Reset vector length to zero
    NULL-pointer tolerant
*/

#define vec_reset_length(v) do { if (v) _vec_len (v) = 0; } while (0)

```

复位vec元素的长度，内存空间不变化，只是把len设置为0而已

9.vec_bytes

```c++
/** \brief Number of data bytes in vector. */

#define vec_bytes(v) (vec_len (v) * sizeof (v[0]))

```

获取vec内部数据的长度，即元素的数量乘以每个元素的大小

10.vec_capacity

```c++
/** \brief Total number of bytes that can fit in vector with current allocation. */

#define vec_capacity(v,b)							\
({										\
  void * _vec_capacity_v = (void *) (v);					\
  uword _vec_capacity_b = (b);							\
  _vec_capacity_b = sizeof (vec_header_t) + _vec_round_size (_vec_capacity_b);	\
  _vec_capacity_v ? clib_mem_size (_vec_capacity_v - _vec_capacity_b) : 0;	\
})

```

计算动态数组占用的内存空间，也包含2种类型的头部，不仅仅是数据部分

11.vec_max_len

```c++
/** \brief Total number of elements that can fit into vector. */
#define vec_max_len(v) (vec_capacity(v,0) / sizeof (v[0]))

```

获取当前已分配的内存最多容纳的元素个数

12.vec_end

```c++
/** \brief End (last data address) of vector. */
#define vec_end(v)	((v) + vec_len (v))

```

当前数组元素的最后一个元素的下一个位置，新元素在这里追加

13.vec_is_member

```c++
/** \brief True if given pointer is within given vector. */
#define vec_is_member(v,e) ((e) >= (v) && (e) < vec_end (v))

```

判断一个元素是否合法的数组元素，是否越界

14.vec_elt

```c++
/** \brief Get vector value at index i checking that i is in bounds. */
#define vec_elt_at_index(v,i)			\
({						\
  ASSERT ((i) < vec_len (v));			\
  (v) + (i);					\
})

/** \brief Get vector value at index i */
#define vec_elt(v,i) (vec_elt_at_index(v,i))[0]

```

vec_elt读取vec第i元素的值
vec_elt_at_index读取vec第i元素的地址

15.vec_foreach and vec_foreach_backwards

```c++
/** \brief Vector iterator */
#define vec_foreach(var,vec) for (var = (vec); var < vec_end (vec); var++)

/** \brief Vector iterator (reverse) */
#define vec_foreach_backwards(var,vec) \
for (var = vec_end (vec) - 1; var >= (vec); var--)

```

循环变量vec所有元素，var得到的是地址，
而vec_foreach是从小到大循环遍历，而vec_foreach_backwards
是从大到小循环遍历

16.vec_foreach_index and vec_foreach_index_backwards

```c++
/** \brief Iterate over vector indices. */
#define vec_foreach_index(var,v) for ((var) = 0; (var) < vec_len (v); (var)++)

/** \brief Iterate over vector indices (reverse). */
#define vec_foreach_index_backwards(var,v) \
  for ((var) = vec_len((v)) - 1; (var) >= 0; (var)--)

```

循环变量vec所有元素，var得到的是元素的索引，
而vec_foreach_index是从小到大循环遍历，而vec_foreach_index_backwards
是从大到小循环遍历
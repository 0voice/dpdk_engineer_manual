# 【No.】DPDK IPV4 LPM(路由表实现)详解

## 1. 研究目的

1.DPDK 如何实现高速路由查找

## 2.技术背景知识

> 阅读本文要求读者具备如下的背景知识

1. ipv4 基础知识
   - ipv4地址组成
   - ipv4路由管理
     - 路由表的组成
     - 协议栈如何根据目的IP 进行路由寻路
     - 添加, 删除，更新　路由条目
2. dpdk 基础知识
   - 基本工作原理

## 3 环境描述

| 软件名                                                     | 版本号        | 描述                 | 其他              |
| :--------------------------------------------------------- | :------------ | :------------------- | :---------------- |
| [ubuntu kylin 64位](http://www.ubuntukylin.com/downloads/) | 16.04         | 笔者使用PC机操作系统 |                   |
| [dpdk](https://github.com/DPDK/dpdk)                       | 17.08         | 高速转发框架         |                   |
| [sublimetext](http://www.sublimetext.com/3)                | 3.0 build3143 | 代码编辑器           | build3143相对稳定 |

## 4. ipv4 路由查找实例

1.假设路由转发设备上存在两个接口, 分别为ge0和ge1．

> 本文以ge表示1000M光口．

2.ge0上ipv4地址的配置如下:

```cobol
 address: 192.168.1.1 netmask: 255.255.255.0
```

3.ge1上ipv4地址的配置如下:

```cobol
 address: 192.168.2.1 netmask: 255.255.255.0
```

4.在该系统上配置如下的静态路由:

> ```cobol
> 1. netaddr: 192.168.3.128 netmask:255.255.255.128 nexthop: 192.168.1.2
> 2. netaddr: 192.168.4.0   netmask:255.255.255.0   nexthop: 192.168.2.2
> 3. netaddr: 192.168.8.0   netmask:255.255.254.0   nexthop: 192.168.2.3
> 4. netaddr: 0.0.0.0       netmask:0.0.0.0         nexthop: 192.168.1.3
> ```

观察上述的路由:

- 1的掩码长度为25位
- 2的掩码长度为24位
- 3的掩码长度为23位
- 4的掩码长度为0位

> 4即默认路由

5.报文选路的原理

- 假设报文的目的IP地址为192.168.3.1, 在查找路由表项的时候, 将目的IP和路由表项中的掩码执行按位与运算, 如果等于netaddress则表明匹配．

> 携带优先级的路由选择暂且不表

- 上述的例子最终会选择路由表项1, 因为表项1的掩码长度为25．

> 这就是路由匹配的最长掩码匹配, 即LPM.

## 5 dpdk中的ipv4 lpm算法

lpm的具体实现位于`dpdk/lib/librte_lpm`目录中．

dpdk提供对外的api的时候采用了api的接口版本管理机制.

> 关于api的版本管理请查看7.1章节

编译dpdk的时候默认是采用静态库来进行编译, 此时各接口实际实现表如下:

| api接口                 | 实际实现                      | 其他                 |
| :---------------------- | :---------------------------- | :------------------- |
| rte_lpm_add             | rte_lpm_add_v1604             | 添加路由             |
| rte_lpm_create          | rte_lpm_create_v1604          | 创建路由表           |
| rte_lpm_delete          | rte_lpm_delete_v1604          | 删除路由             |
| rte_lpm_delete_all      | rte_lpm_delete_all_v1604      | 销毁路由表           |
| rte_lpm_find_existing   | rte_lpm_find_existing_v1604   | 根据名字找到路由表   |
| rte_lpm_free            | rte_lpm_free_v1604            | 释放路由表占用的空间 |
| rte_lpm_is_rule_present | rte_lpm_is_rule_present_v1604 | 检查是否条目存在     |
| rte_lpm_lookup          |                               | 路由查找实现         |

> 上述的对应关系也可以直接通过nm librte_lpm.a来查看 如:

```cobol
0000000000000d60 T rte_lpm_add
0000000000000d60 T rte_lpm_add_v1604
```

> 可以看到实际上rte_lpm_add和rte_lpm_add_v1604的地址是一样的.

## 5.1 数据结构以及算法

DPDK中涉及的主要路由数据结构的概念如下:

- 路由条目表项

  ```
  rte_lpm_rule
  ```

  - 用来存储的具体的路由信息

- 路由条目描述信息 rte_lpm_rule_info

  - 不同掩码长度路由条目表的管理结构
  - 通过此数据结构可以访问到各种掩码长度的所有的路由条目表项的信息

- 路由hash表项`rte_lpm_tbl_entry`

  - DPDK中的LPM算法是空间换时间的典型, 核心算法是hash.
  - DPD中将32的ip地址分为24位和8位两部分：
    - 一张2^24大小的hash表,称之为tbl24
      - tbl24 hash key = 目的IP >> 8
    - N张2^8大小的hash表, 称之为tbl8
      - 极限情况下每一个tbl24的表项都会存在一个2^8大小的hash表

### 5.1.1 主要数据结构

1.路由条目描述信息-`rte_lpm_rule_info`

```cobol
struct rte_lpm_rule_info {
	uint32_t used_rules; 
	uint32_t first_rule;
};
```

- 路由条目是按照掩码位数来组织的, 同样的掩码位数的由条目从索引`first_rule`开始到`first_rule+used_rules-1`结束
- 路由条目的数据存储在路由表rte_lpm中的rules_tbl中
- 对于ipv4的地址掩码, 一个路由表中总共存在32个rte_lpm_rule_info元素

下面举例来描述　rte_lpm_rule_info[32]的维护操作

> 关于错误处理部分在此不描述, 请参考具体代码

- 插入

  - 假设原有路由条目表中除了掩码为N的路由条目没有插入, 其他各掩码均有路由条目存在.

  - 插入掩码为N的路由表目

  - 插入的时候会将>N的条目往后顺移一个位置, 同时>N的路由描述信息的first_rule+1

- 删除
  - 假设原有路由条目表中所有的掩码的路由条目都存在
  - 删除掩码为N的路由条目一条
  - 删除的时候会将>N的条目向前顺移一个位置,同时>N的路由描述信息的frist_rule-1

> 上述移动的时候采用了交换的算法, 具体请参考代码

2.路由条目表项

```cobol
struct rte_lpm_rule {
	uint32_t ip; /**< Rule IP address. */
	uint32_t next_hop; /**< Rule next hop. */
};
```

- ip 存储路由条目
- next_hop 下一跳的ip地址

3.路由表-`rte_lpm`

```cobol
struct rte_lpm {
	/* LPM metadata. */
	char name[RTE_LPM_NAMESIZE];        /**< Name of the lpm. */
	uint32_t max_rules; /**< Max. balanced rules per lpm. */
	uint32_t number_tbl8s; /**< Number of tbl8s. */
	struct rte_lpm_rule_info rule_info[RTE_LPM_MAX_DEPTH]; /**< Rule info table. */
 
	/* LPM Tables. */
	struct rte_lpm_tbl_entry tbl24[RTE_LPM_TBL24_NUM_ENTRIES]
			__rte_cache_aligned; /**< LPM tbl24 table. */
	struct rte_lpm_tbl_entry *tbl8; /**< LPM tbl8 table. */
	struct rte_lpm_rule *rules_tbl; /**< LPM rules. */
};
```

- name
  - 系统可能存在多张路由表(特别是vpn存在的情况下), 所以需要用name来区分不同的路由表.
- max_rules
  - 存储的最大路由条目个数, 在创路由表的时候使用rte_lpm_config指定
- number_tbl8s
  - 存储的最大的tbl8表个, 每一个tbl8是255个元素(即2^8)
  - 创建路由表的时候使用rte_lpm_config指定
- rule_info
  - 存储路由描述信息的机构
  - `RTE_LPM_MAX_DEPTH=32`
  - 假设rule_info
- tbl24
  - hash24的节点
  - `RTE_LPM_TBL24_NUM_ENTRIES=2^24`
  - `hash key=目的ip>>8`
- tbl8
  - hash8的节点
  - tbl8空间=255*number_tbl8s
- rules_tbl
  - 存储具体路由条目的空间
  - 其存储路由条目的个数=max_rules

3.路由hash表项

```cobol
struct rte_lpm_tbl_entry {
	/**
	 * Stores Next hop (tbl8 or tbl24 when valid_group is not set) or
	 * a group index pointing to a tbl8 structure (tbl24 only, when
	 * valid_group is set)
	 */
	uint32_t next_hop    :24;
	/* Using single uint8_t to store 3 values. */
	uint32_t valid       :1;   /**< Validation flag. */
	/**
	 * For tbl24:
	 *  - valid_group == 0: entry stores a next hop
	 *  - valid_group == 1: entry stores a group_index pointing to a tbl8
	 * For tbl8:
	 *  - valid_group indicates whether the current tbl8 is in use or not
	 */
	uint32_t valid_group :1;
	uint32_t depth       :6; /**< Rule depth. */
};
```

- rte_lpm_tbl_entry是tbl24和tbl8共用的 hash 节点
- next_hop
  - 下一跳
  - 当为tbl24节点且此节点上挂载tbl8的时候, 此数据表示tbl8的开始索引
  - 当为tbl24节点, 但是此节点上不存在tbl8的时候, 此数据为真实的下一跳
- valid
  - 表明此节点是否有效
- valid_group
  - 当为tbl24节点的时候 valid_group为1表明next_hop为tbl8的开始索引,为０表示下一跳
- depth
  - 此hash节点的掩码位

### 5.1.2 路由表的具体算法

在对具体算法进行介绍之前, 做如下约定:

- 设定网段ip为netaddr, 掩码位数为depth, 掩码为mask

> mask和depth可以转换, 为了方便文字描述增加mask的定义

- 设定需要查找路由的报文的目的ip为dest_ip

#### 1. 添加

- `key = netaddr>>8`

`rte_lpm_add_v1604`执行具体的添加算法, 主要分为三种情况

1.小于等于24位掩码的时候

- 计算得到访问tbl24的索引范围为 [ key, key+ 1<<(24-depth)]
- 遍历这个范围的tbl24的hash表项, 执行如下操作
  - 此节点不存在tbl8的时候, 执行如下过程, 命名为A
    - 节点无效, 更新路由信息
    - 节点有效, 掩码长度大于depth, 放弃更新, 保持lpm特性
    - 节点有效, 掩码长度小于等于depth, 更新路由信息
  - 此节点存在tbl8的时候
    - 循环tbl8的节点, 同样执行过程A

2.大于24位掩码的时候

- 大于24位掩码的时候tbl24上的节点不存储路由数据, next_hoop为tbl8的索引
- 根据key找到tbl24的hash表项
  - 如果没有tbl8的表项
    - 为该tbl24 hash节点分配tbl8的表项
    - 更新该tbl8所有表项的路由为插入的路由信息
  - 如果存在tbl8的表项
    - 节点无效, 更新路由信息
    - 节点有效, 掩码长度大于depth, 放弃更新, 保持lpm特性
    - 节点有效, 掩码长度小于等于depth, 更新路由信息

#### 2. 查询

- tbl24-key = dest_ip >> 8
- tbl8-key = dest_ip & 0xFFFFFF00
- 根据tbl24-key访问tbl24 的节点, 并得到tbl8的开始索引
- 根据tbl8-key+tabl8的开始索引访问tbl8的节点
  - 找到 下一跳信息

从上述的流程可以看到, 如果掩码小于等于24则只需要一次hash访问即可得到路由信息． 如果掩码大于24则最多需要两次hash访问即可得到路由信息.

#### 3. 删除

> 从路由条目表和路由表中删除一个指定的路由的时候, 需要执行到一个回溯的动作．

- 从路由条目表中删除路由条目 netaddr,depth
- 设定掩码位数范围为[depth-1, 0]
- 按照掩码范围遍历路由条目表, 找到包含netaddr的路由条目X
  - 找不到则仅需要删除tbl24和tbl8中的对应路由hash节点即可
  - 找到则需要将tbl24和tbl8中netaddr对应路由hash节点全部修改成路由条目X的信息

## 6 总结

DPDK中的ipv4路由算法, 查找的时间复杂度 能够达到O(1)的效果．

对于研发路由器快速转发产品而言, 存在如下的问题:

1. 默认路由在配置的时候会造成存储hash表的极端情况
2. 上述路由表结构无法支持路由优先级以及等价路由或者负载均衡

## 7. 附录

### 7.1 源代码api的版本控制

1.`rte_lpm_version.map`

- 当dpdk的编译配置文件中启用了`CONFIG_RTE_BUILD_SHARED_LIB` 的时候, dpdk将产生动态链接库文件, `rte_lpm_version.map`用于不同版本的库中发布不同的函数abi的描述文件．

2.宏`VERSION_SYMBOL`

```lisp
#define VERSION_SYMBOL(b, e, n) __asm__(".symver " RTE_STR(b) RTE_STR(e) ", " RTE_STR(b) "@DPDK_" RTE_STR(n))
```

- 动态库有效
- rte_lpm.c中有如下定义:

```cobol
VERSION_SYMBOL(rte_lpm_create, _v20, 2.0);
```

```perl
__asm__(".symver rte_lpm_create_v20, rte_lpm_create@DPDK_2.0")
```

- 这个含义就是2.0的library中`rte_lpm_create`对应的实现是`rte_lpm_create_v20`

3.宏`BIND_DEFAULT_SYMBOL`

```lisp
#define BIND_DEFAULT_SYMBOL(b, e, n) __asm__(".symver " RTE_STR(b) RTE_STR(e) ", " RTE_STR(b) "@@DPDK_" RTE_STR(n))
```

- 动态库有效
- 整个宏和VERSION_SYMBOL的区别在于@变成了@@, 那么@@是动态链接库中的默认符号．即如果链接so的程序没有明确指定api的版本号, 使用@@这个版本的api．

4.宏`MAP_STATIC_SYMBOL`

```lisp
#define MAP_STATIC_SYMBOL(f, p) f __attribute__((alias(RTE_STR(p))))
```

- 静态链接才有效
- 这个宏使用了gcc的__attribute__属性, 为函数做了重命名

### 7.2 相关的数据结构

1.路由表配置

```cobol
/** LPM configuration structure. */
struct rte_lpm_config {
	uint32_t max_rules;      /**< Max number of rules. */
	uint32_t number_tbl8s;   /**< Number of tbl8s to allocate. */
	int flags;               /**< This field is currently unused. */
};
```

- max_rules是用来分配rte_lpm结构中rules_tbl的存储空间的
- number_tbl8s是用来分配rte_lpm结构中tbl8的存储空间的．
  - tbl8需要分配的元素个数为number_tbl8s*255

————————————————
转载自  原文链接：https://blog.csdn.net/tekjin/article/details/78466311

# DPDK LPM路由存储与查找

前言：DPDK的LPM模块实现了一种最长前缀匹配，其中的KEY是32位的，可以说是为查找路由量身定做的，为了实现快速查找，实现上使用了用空间换时间的思路。同时为了最大限度的减少查询次数，把32位的KEY值划分为24位和8位两张表中。这样的设计思路可以用于以后的前缀查找。本篇分析以16.07版本为例。

## 1. LPM的设计概览

对于路由的查找，有多种方法，前缀匹配，红黑树，各有优劣。这都在昭示着上帝创世界是均衡的。从统计上看，只有少数的路由网段深度大于24，因此使用了DIR-24-8表，大部分匹配在第一次查找时即可命中。

对于二级表tbl8，理论上有224个，但是我们不能存储这个多个表，否则总共的表项不就是232了嘛，相当于每个ip地址都有一个条目对应。所以，可以通过限定tbl8表中的组的数目，减少空间占用。每个组也可以再次限定条目的数目，最大自然就是256了。直接使用下面的图来表示：

![img](https://images2015.cnblogs.com/blog/614525/201705/614525-20170507201508414-1578884301.png)

> 在16.04版本后，比如16.07版本中，就出现了LPM操作的两套接口，如`rte_lpm_add_v20()`和`rte_lpm_add_v1604()`,区别在于下一跳的值范围，v20版本是8位的，v1604是32位的。这里的下一跳并不是地址的下一跳，仅仅是表示一个索引，可以用来指示自己的表项。

## 2. LPM的代码实现

在代码分析上，我们以v20版本做分析，v1604的版本基本一样。

### 2.1 LPM路由的存储

路由的存储是从`rte_lpm_add_v20()`开始的：
首先，获得网段，

```c
ip_masked = ip & depth_to_mask(depth);
```

然后是把路由信息添进规则表--`rule_add_v20()`,
规则表是按照掩码深度排的，共有32个组，分别对应不同的掩码深度。

既然是添加rule，首先要看是否已经有了这个rule。这里使用了`lpm->rule_info`结构来存储每个组的第一个rule在rule table的index。所以，查找过程就是：先看对应深度的组是否有元素，如果没有，直接下一步插入，如果有，就找到对应组的第一个元素，然后遍历，找到就返回index,没找到就下一步插入。

主要来说一下这个插入位置，由于各个组的rule存储是连着的，没有空余位置，因此当要添加一个20位深度的rule时，如果已经添加了大于20的rule，就要挪出位置来。

```c
 
rule_index = 0;
 
for (i = depth - 1; i > 0; i--) {
	if (lpm->rule_info[i - 1].used_rules > 0) {
		rule_index = lpm->rule_info[i - 1].first_rule
				+ lpm->rule_info[i - 1].used_rules;
		break;
	}
}
if (rule_index == lpm->max_rules)
	return -ENOSPC;
 
lpm->rule_info[depth - 1].first_rule = rule_index;
```

这是计算出新的rule要在表中的位置。然后接下来就是为这个rule挪出位置来：

```c
/* Make room for the new rule in the array. */
for (i = RTE_LPM_MAX_DEPTH; i > depth; i--) {
	if (lpm->rule_info[i - 1].first_rule
			+ lpm->rule_info[i - 1].used_rules == lpm->max_rules)
		return -ENOSPC;
 
	if (lpm->rule_info[i - 1].used_rules > 0) {
		lpm->rules_tbl[lpm->rule_info[i - 1].first_rule
			+ lpm->rule_info[i - 1].used_rules]
				= lpm->rules_tbl[lpm->rule_info[i - 1].first_rule];
		lpm->rule_info[i - 1].first_rule++;
	}
}
```

从32位深度的组开始，大于depth的组往后依次挪一个位置。

最后把填充规则信息：

```c
lpm->rules_tbl[rule_index].ip = ip_masked;
lpm->rules_tbl[rule_index].next_hop = next_hop;
 
/* Increment the used rules counter for this rule group. */
lpm->rule_info[depth - 1].used_rules++;
```

这样，规则表就添加完成了！

接下来就是重要的往DIR-24-8表中添加条目了。
对于掩码深度<=24位的，只需要添加tbl24表，而掩码深度大于24位的，还需要添加tbl8表。因此，这里有两个入口函数，先看掩码深度<=24的处理：`add_depth_small_v20()`

首先要知道的是，对于tbl24表，存的是地址前24位的排列，最大共224个条目。此时想象一下，如果配置了一个20位深度的路由，那么前24位就有2(24-20)个可能，这些条目也都要填充为这个条目。

因此，先算出来对于这样深度的路由，其前24位可能的范围。

```c
tbl24_index = ip >> 8;
tbl24_range = depth_to_range(depth);
```

然后对这个范围内的条目操作：

```c
for (i = tbl24_index; i < (tbl24_index + tbl24_range); i++)
```

更新或者新增新的tbl24条目。如果有tbl8表的，找到对应的表，更新或者添加tbl8表。

接下来我们看一下当深度>24时，对于这两张表是怎么操作来插入路由的：`add_depth_big_v20()`
首先也是确定他的高24位的段和后8位的范围，

```c
tbl24_index = (ip_masked >> 8);
tbl8_range = depth_to_range(depth);
```

然后主要分了3类情况处理，
第一种，连对应的tbl24条目都不存在的，自然是要先增添tbl24条目

```c
if (!lpm->tbl24[tbl24_index].valid) {
	/* Search for a free tbl8 group. */
	tbl8_group_index = tbl8_alloc_v20(lpm->tbl8);
 
	/* Check tbl8 allocation was successful. */
	if (tbl8_group_index < 0) {
		return tbl8_group_index;
	}
 
	/* Find index into tbl8 and range. */
	tbl8_index = (tbl8_group_index *
			RTE_LPM_TBL8_GROUP_NUM_ENTRIES) +
			(ip_masked & 0xFF);
 
	/* Set tbl8 entry. */
	for (i = tbl8_index; i < (tbl8_index + tbl8_range); i++) {
		lpm->tbl8[i].depth = depth;
		lpm->tbl8[i].next_hop = next_hop;
		lpm->tbl8[i].valid = VALID;
	}
 
	/*
	 * Update tbl24 entry to point to new tbl8 entry. Note: The
	 * ext_flag and tbl8_index need to be updated simultaneously,
	 * so assign whole structure in one go
	 */
 
	struct rte_lpm_tbl_entry_v20 new_tbl24_entry = {
		{ .group_idx = (uint8_t)tbl8_group_index, },
		.valid = VALID,
		.valid_group = 1,
		.depth = 0,
	};
 
	lpm->tbl24[tbl24_index] = new_tbl24_entry;
 
}
```

这里的逻辑是比较容易理解的，同时创建tbl24和tbl8的条目，这里注意到tbl8表组索引的分配，就是在最大限定的组数量范围内，找出没使用的组，默认最大为256个组，所以，默认只能存256个大于32位深度的路由，在16.04版本中，这个组数量是在初始化可配置的。

第二种情况，tbl24表存在，但是没有tbl8的标识，比如先配了192.168.1.0/24，此时，tbl24已经创建了一个条目，如果再配置192.168.1.0/28，那么，先检查tbl24时，已经创建，但是没有tbl8标识，现在要做的就是更新tbl24条目，同时创建tbl8条目。

这里就很有意思了，既然更新tbl24，那么主要就是nexthop，valid标识，此时，因为查询大于24位深度的路由时要使用tbl8，而如果查询小于24位的又不查tbl8，这样怎么定nexthop呢？所以，就把先把tbl8条目都设置成tbl24的内容，也就是把对应网断的tbl24条目搬到tbl8中，然后再根据tbl8的范围，刷新tbl8，这样，老的tbl24的条目和新添加tbl8的条目都存在这个tbl8中了。

```c
for (i = tbl8_group_start; i < tbl8_group_end; i++) {
	lpm->tbl8[i].valid = VALID;
	lpm->tbl8[i].depth = lpm->tbl24[tbl24_index].depth;
	lpm->tbl8[i].next_hop =
			lpm->tbl24[tbl24_index].next_hop;
}
 
tbl8_index = tbl8_group_start + (ip_masked & 0xFF);
 
/* Insert new rule into the tbl8 entry. */
for (i = tbl8_index; i < tbl8_index + tbl8_range; i++) {
	lpm->tbl8[i].valid = VALID;
	lpm->tbl8[i].depth = depth;
	lpm->tbl8[i].next_hop = next_hop;
}
```

最后更新tbl24表

```c
struct rte_lpm_tbl_entry_v20 new_tbl24_entry = {
		{ .group_idx = (uint8_t)tbl8_group_index, },
		.valid = VALID,
		.valid_group = 1,
		.depth = 0,
};
 
lpm->tbl24[tbl24_index] = new_tbl24_entry;
```

第三种情况是如果已经有对应的tbl8组了，那自然就是更新啦。如果理解了第二种情况，这种也很好理解

```c
tbl8_group_index = lpm->tbl24[tbl24_index].group_idx;
tbl8_group_start = tbl8_group_index *
	RTE_LPM_TBL8_GROUP_NUM_ENTRIES;
tbl8_index = tbl8_group_start + (ip_masked & 0xFF);
 
for (i = tbl8_index; i < (tbl8_index + tbl8_range); i++) {
 
if (!lpm->tbl8[i].valid ||
		lpm->tbl8[i].depth <= depth) {
	struct rte_lpm_tbl_entry_v20 new_tbl8_entry = {
		.valid = VALID,
		.depth = depth,
		.valid_group = lpm->tbl8[i].valid_group,
	};
	new_tbl8_entry.next_hop = next_hop;
	/*
	 * Setting tbl8 entry in one go to avoid race
	 * condition
	 */
	lpm->tbl8[i] = new_tbl8_entry;
```

就是把最新添加的深度的条目更新一下，如老的是192.168.1.0／28的，新添加的是192.168.1.0/30的，根据最长匹配，那么就把最长网段（30掩码的）中的4个路由更新一下。

这样，路由添加就完成了！

### 2.2 LPM路由的查找

对应条目的查找就很简单了，只要找到nexthop就行了，但要注意的是，这里的nexthop可不是最后的下一跳，因为看结构定义，只有8位

```c
struct rte_lpm_tbl_entry_v20 {
	uint8_t depth       :6;
	uint8_t valid_group :1;
	uint8_t valid       :1;
	union {
		uint8_t group_idx;
		uint8_t next_hop;
	};
};
```

因此，这个nexthop实际上也只是个索引，根据nexthop的索引，我们可以自己进行后续处理。

## 3. 后记：

LPM模块是DPDK的重要组成部分，主要用于查找路由，其使用的以空间换时间的方式，同时为了尽量减少空间消耗，采用DIR-24-8的二级表，再加上可配置的条目总数限制，在保证速度的前提下，最大可能降低了资源消耗，是一种很值得借鉴的设计方式。





原创：[AISEED](https://www.cnblogs.com/yhp-smarthome/)    本文链接：https://www.cnblogs.com/yhp-smarthome/p/6822085.html


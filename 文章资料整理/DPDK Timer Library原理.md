# DPDK Timer Library原理

## 前置知识学习跳表（SkipList）

**跳表应具有以下特征：**

1）一个跳表应该有多个层（level）组成，通常是10-20层。

2）跳表的第0层包含所有的元素。

3）每一层都是一个有序的链表。层数越高应越稀疏，这样在高层次中能跳过许多不符合条件的数据。

4）如果元素x出现在第i层，则所有比i小的层都包含x；

5）每个节点包含key及其对应的value和一个指向下一层相同内容的节点位置。

**skiplist的查询过程示例：**

以已有数据13、22、75、80、99为例，查找80。从最高层（此例为2）开始：

1）level2找到结点Node75小于80，且level2.Node75->next 大于80，则进入level1查找(此处已经跳过了13~75中间的结点（22)。

2）level1.Node75 < 80 < level1.Node75->next，进入level0。

3）level0.Node75->next 等于80，找到结点。

![img](https://img2020.cnblogs.com/blog/1926214/202005/1926214-20200518202057533-1146643008.png)

## 1. 定时器库

**定时器库为DPDK执行单元提供定时器服务，以异步执行回调函数。该库的功能有：**

- 定时器可以是周期性执行或单次执行。
- 定时器可以从一个内核加载并在另一个内核上执行。但必须在对rte_timer_reset（）的调用中指定它。
- 定时器提供精度高计数（取决于对rte_timer_manage（）的调用频率，该调用频率检查本地内核的定时器到期情况）。
- 如果应用程序中不需要，可以通过不调用rte_timer_manage（）来提高性能，从而在编译时禁用定时器。

定时器库使用rte_get_timer_cycles（）函数，该函数使用高精度事件定时器（HPET）或CPU时间戳计数器（TSC）提供可靠的时间参考。

该库提供添加，删除和重新启动定时器的接口。该API基于BSD callout（），但有一些区别。请参阅标注手册。

## 2. 实施细节

定时器以每个逻辑核为基础进行跟踪，并在跳表（skiplist）数据结构中按定时器到期的顺序维护核内的所有待处理定时器。使用的跳表有十个级别，并且表中的每个条目以（¼^级）的概率出现在每个级别中。这意味着所有条目都存在于级别0中，每4个条目中就有1个存在于级别1中，每16个条目中就有1个存在于级别2中，依此类推，直到第9级为止。这意味着从一个定时器列表中添加和删除条目的时间复杂度为log(n)，每个逻辑核最多能容纳4^10个条目，即约1,000,000个定时器。

定时器结构包含一个称为状态的特殊字段，该字段是定时器状态（已停止，挂起，正在运行，配置）和所有者（lcore id）的联合体。

```
union rte_timer_status {
    RTE_STD_C11
    struct {
        uint16_t state;  /**< Stop, pending, running, config. */
        int16_t owner;   /**< The lcore that owns the timer. */
    };
    uint32_t u32;            /**< To atomic-set status + owner. */
};
```

**根据定时器的状态，我们知道列表中是否存在定时器：**

- STOPPED停止：没有所有者，不在列表中
- CONFIG配置：由一个核拥有，不能被另一个核修改，在不在列表中未知，取决于先前的状态。
- PENDING待处理：由一个核拥有，在列表中
- RUNNING运行：由一个核拥有，不能被另一个核修改。在列表中。

不允许在CONFIG或RUNNING状态下重置或停止定时器。修改定时器的状态时，应使用CAS（比较并交换）指令来保证状态及所有者被原子地修改。

在rte_timer_manage（）函数内部，跳表将作为一个常规链表来遍历级别0的列表（包含所有定时器条目）直到遇到尚未到期的条目。当列表中有条目，但是没有任何定时器到期时，为了提高性能，第一个定时器条目的到期时间保存在每个逻辑核的定时器列表结构中。在64位平台上，可以直接检查该值，而无需对整个结构进行锁定。（由于将到期时间是一个64位值，32位平台上如果不使用比较交换（CAS）指令或使用锁，将无法直接对该值进行检查。因此将跳过此附加检查，而在某次获得锁后再进行检查）。在64位和32位平台上，调用核的定时器列表为空的情况下对rte_timer_manage()的调用将不带锁返回。

## 3. 用例

在垃圾收集器或某些状态机（ARP，桥接等）下，定时器库被用于定期调用。

## 4. 参考文献

- DPDK官方编程指南 - http://doc.dpdk.org/guides-20.02/prog_guide/timer_lib.html
- [callout manual](http://www.daemon-systems.org/man/callout.9.html) - The callout facility that provides timers with a mechanism to execute a function at a given time.
- [HPET](http://en.wikipedia.org/wiki/HPET) - Information about the High Precision Event Timer (HPET).



原创：[一觉醒来写程序](https://home.cnblogs.com/u/realjimmy/)   本文链接：https://www.cnblogs.com/realjimmy/p/12912751.html
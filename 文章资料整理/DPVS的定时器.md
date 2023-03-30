# DPVS的定时器

## 1.时间轮

DPDK的定时器使用跳表实现的，在处理大量比如会话时间时，session老化啥的，效率不高，DPVS的使用的是复杂度 O(1)的时间轮算法。

![img](https://pic1.zhimg.com/80/v2-9d70d9ed7aeddd10c3491708696e757c_720w.webp)

## 2.DPVS的定时器

### 2.1数据结构

```c
struct timer_scheduler {
    /* wheels and cursors */
    rte_spinlock_t      lock;
    uint32_t            cursors[LEVEL_DEPTH];
    struct list_head    *hashs[LEVEL_DEPTH];

    /* leverage dpdk rte_timer to drive us */
    struct rte_timer    rte_tim;
};
static struct timer_scheduler g_timer_sched;
```

lock是锁，增删改查都要锁。

rte_tim负责时间滴答，也就是tick++。

cursors和hashs 配合使用，构成时间轮。

从 0 到 LEVEL_SIZE (2<<18), LEVEL_DEPTH 值为 2，也就是说 cursors[0] 可以保存 2<<18 个嘀嗒，如果 DPVS_TIMER_HZ = 1000，那么就是 524s. cursors[0] 驱动 cursors[1] 时间轮，两个轮一共 8.7 年时间。hashs 是一个长度为 2 的数组，每个元素是一个链表数组，长度是 LEVEL_SIZE (2<<18), 链表成员就是具体的定时器消息。

### 2.2初始化

```c
int dpvs_timer_init(void)
{
    lcoreid_t cid;
    int err;

    /* per-lcore timer */
    rte_eal_mp_remote_launch(timer_lcore_init, NULL, SKIP_MAIN);
    RTE_LCORE_FOREACH_WORKER(cid) {
        err = rte_eal_wait_lcore(cid);
        if (err < 0) {
            RTE_LOG(ERR, DTIMER, "%s: lcore %d: %s.\n",
                    __func__, cid, dpvs_strerror(err));
            return err;
        }
    }

    /* global timer */
    return timer_init_schedler(&g_timer_sched, rte_get_main_lcore());
}
```

每个slave lcore 都要调 timer_lcore_init初始化自己的，然后初始化全局的_timer_sched。

### 23timer_lcore_init

```c
static int timer_lcore_init(void *arg)
{
    if (!rte_lcore_is_enabled(rte_lcore_id()))
        return EDPVS_DISABLED;

    return timer_init_schedler(&RTE_PER_LCORE(timer_sched), rte_lcore_id());
}
```

### 2.4timer_init_schedler

```c
static int timer_init_schedler(struct timer_scheduler *sched, lcoreid_t cid)
{
    int i, l;

    rte_spinlock_init(&sched->lock);


    timer_sched_lock(sched);
    for (l = 0; l < LEVEL_DEPTH; l++) {
        sched->cursors[l] = 0;

        sched->hashs[l] = rte_malloc(NULL,
                                     sizeof(struct list_head) * LEVEL_SIZE, 0);
        if (!sched->hashs[l]) {
            RTE_LOG(ERR, DTIMER, "[%02d] no memory.\n", cid);
            timer_sched_unlock(sched);
            return EDPVS_NOMEM;
        }

        for (i = 0; i < LEVEL_SIZE; i++)
            INIT_LIST_HEAD(&sched->hashs[l][i]);
    }
    timer_sched_unlock(sched);

    rte_timer_init(&sched->rte_tim);
    /* ticks should be exactly same with precision */
    if (rte_timer_reset(&sched->rte_tim, g_cycles_per_sec / DPVS_TIMER_HZ,
                        PERIODICAL, cid, rte_timer_tick_cb, sched) != 0) {
        RTE_LOG(ERR, DTIMER, "[%02d] fail to reset rte timer.\n", cid);
        return EDPVS_INVAL;
    }

    RTE_LOG(DEBUG, DTIMER, "[%02d] timer initialized %p.\n", cid, sched);
    return EDPVS_OK;
}
```

hashs是一个二维数组，大小是sizeof(structlist_head) * LEVEL_SIZE。

利用了dpdk的定时器去维护滴答，通过rte_timer_reset来注册，回调函数是rte_timer_tick_cb间隔是

rte_get_timer_hz()/DPVS_TIMER_HZ ticks。

### 2.5回调函数rte_timer_tick_cb

```c
static void rte_timer_tick_cb(struct rte_timer *tim, void *arg)
{
    struct timer_scheduler *sched = arg;
    struct dpvs_timer *timer, *next;
    uint64_t left, hash, off, remainder;
    int level, lower;
    uint32_t *cursor;
    bool carry;

    assert(tim && sched);

#ifdef CONFIG_TIMER_MEASURE
    deviation_measure();
#endif

    /* drive timer to move and handle expired timers. */
    timer_sched_lock(sched);
    for (level = 0; level < LEVEL_DEPTH; level++) {
        cursor = &sched->cursors[level];
        (*cursor)++;

        if (likely(*cursor < LEVEL_SIZE)) {
            carry = false;
        } else {
            /* reset the cursor and handle next level later. */
            *cursor = 0;
            carry = true;
        }

        list_for_each_entry_safe(timer, next,
                                 &sched->hashs[level][*cursor], list) {
            /* is all lower levels ticks empty ? */
            left = timer->delay % get_level_ticks(level);
            if (!left) {
                timer_expire(sched, timer);
            } else {
                /* drop to lower level wheel, note it may not drop to
                 * "next" lower level wheel. */
                list_del(&timer->list);

                lower = level;
                remainder = timer->delay % get_level_ticks(level);
                while (--lower >= 0) {
                    off = remainder / get_level_ticks(lower);
                    if (!off)
                        continue; /* next lower level */

                    hash = (sched->cursors[lower] + off) % LEVEL_SIZE;
                    list_add_tail(&timer->list, &sched->hashs[lower][hash]);
                    break;
                }

                assert(lower >= 0);
            }
        }

        if (!carry)
            break;
    }
    timer_sched_unlock(sched);

    return;
}
```

- 当 level 0 时，使用第一个轮，并调用`(*cursor)++`将嘀嗒加一。如果嘀嗒值大于 LEVEL_SIZE 说明第一个轮用完了，重置为 0 并设置 carry 进位标记。
- 遍历当前轮的 hash 数组，`get_level_ticks` 获取每一层的轮步进一个单位代表多少嘀嗒，当 level=0 时返回 1， 当 level=1时返回 LEVEL_SIZE。`left = timer->delay % get_level_ticks(level)` 如果 left 为 0 说明这个定时任务属于当前时间轮，并且己经过期了，触发 timer_expire 回调。
- 如果 left 有值，说明还没到超时时间。但是时间轮己经触发一次了，所以要降级到 --lower 层。从原有的链表中调用 list_del 删除，然后 `timer->delay / get_level_ticks(lower)` 求出在当前层步进单位个数，`(*cursor + off) % LEVEL_SIZE` 求出 hash 索引，然后 list_add_tail 添加到对应链表中。
- 如果没有进位，也就是 carry 未标记，那么退出循环。否则处理下一层的轮。

### 2.6添加超时任务

调用dpvs_timer_sched添加超时任务

其中主要调用了__dpvs_timer_sched，来看期主要实现

```c
static int __dpvs_timer_sched(struct timer_scheduler *sched,
                              struct dpvs_timer *timer, struct timeval *delay,
                              dpvs_timer_cb_t handler, void *arg, bool period)
{
    uint32_t off, hash;
    int level;

    assert(timer);

#ifdef CONFIG_TIMER_DEBUG
    /* just for debug */
    if (unlikely((uint64_t)handler > 0x7ffffffffULL)) {
        char trace[8192];
        dpvs_backtrace(trace, sizeof(trace));
        RTE_LOG(WARNING, DTIMER, "[%02d]: timer %p new handler possibly invalid: %p -> %p\n%s",
                rte_lcore_id(), timer, timer->handler, handler, trace);
    }
    if (unlikely(timer->handler && timer->handler != handler)) {
        char trace[8192];
        dpvs_backtrace(trace, sizeof(trace));
        RTE_LOG(WARNING, DTIMER, "[%02d]: timer %p handler possibly changed maliciously: %p ->%p\n%s",
                rte_lcore_id(), timer, timer->handler, handler, trace);
    }
#endif

    assert(delay && handler);

    if (timer_pending(timer))
        RTE_LOG(WARNING, DTIMER, "schedule a pending timer ?\n");

    timer->handler = handler;
    timer->priv = arg;
    timer->is_period = period;
    timer->delay = timeval_to_ticks(delay);

    if (unlikely(timer->delay >= TIMER_MAX_TICKS)) {
        RTE_LOG(WARNING, DTIMER, "exceed timer range\n");
        return EDPVS_INVAL;
    }

    /*
     * to schedule a 0 delay timer is not make sence.
     * and it will never stopped (periodic) or never triggered (one-shut).
     */
    if (unlikely(!timer->delay)) {
        RTE_LOG(INFO, DTIMER, "trigger 0 delay timer at next tick.\n");
        timer->delay = 1;
    }

    /* add to corresponding wheel, from higher level to lower. */
    for (level = LEVEL_DEPTH - 1; level >= 0; level--) {
        off = timer->delay / get_level_ticks(level);
        if (off > 0) {
            hash = (sched->cursors[level] + off) % LEVEL_SIZE;
            list_add_tail(&timer->list, &sched->hashs[level][hash]);
#ifdef CONFIG_TIMER_DEBUG
            assert(timer->handler == handler);
#endif
            return EDPVS_OK;
        }
    }

    /* not adopted by any wheel (never happend) */
    RTE_LOG(WARNING, DTIMER, "unexpected error\n");
    return EDPVS_INVAL;
}
```

- 根据timer，调用timeval_to_ticks生成需要多少个滴答时间，delay多少个 滴答时间后超时。
- 参数检查，timer的时间范围。
- 从最大层开始遍历时间轮，添加到指定链表中。off 是在当前轮中需要多少步进单位，hash 求出索引。

### 2.7刷新定时器

```c
int dpvs_timer_update(struct dpvs_timer *timer, struct timeval *delay, bool global)
{
    struct timer_scheduler *sched = this_lcore_sched(global);
    int err;

    if (!sched || !timer || !delay)
        return EDPVS_INVAL;

    timer_sched_lock(sched);
    if (timer_pending(timer))
        list_del(&timer->list);
    err = __dpvs_timer_sched(sched, timer, delay,
            timer->handler, timer->priv, timer->is_period);
    timer_sched_unlock(sched);

    return err;
}
```

也是调用__dpvs_timer_sched实现，需要先判断是否删除timer 的list。





原文链接：https://zhuanlan.zhihu.com/p/544932631  原文作者：木木女神经
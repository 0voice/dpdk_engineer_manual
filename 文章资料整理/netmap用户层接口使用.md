# netmap用户层接口使用

## 1.netmap初始化

打开网卡，

```c++
struct nmreq base_nmd;
bzero(&base_nmd, sizeof(base_nmd));
g.nmd = nm_open(g.ifname, &base_nmd, 0, NULL);if (g.nmd == NULL) {
    D("Unable to open %s: %s", g.ifname, strerror(errno));        goto out;
}
```

由于网卡内部可能会存在多个ring，netmap可以只操作指定ring，

```c++
//follow upper code
struct nm_desc saved_desc = *g.nmd;saved_desc.self = &saved_desc;saved_desc.mem = NULL;nm_close(g.nmd);saved_desc.req.nr_flags &= ~NR_REG_MASK;saved_desc.req.nr_flags |= NR_REG_ONE_NIC;saved_desc.req.nr_ringid = 0;g.nmd = nm_open(g.ifname, &base_nmd, NM_OPEN_IFNAME, &saved_desc);if (g.nmd == NULL) {
    D("Unable to open %s: %s", g.ifname, strerror(errno));
    goto out;}
```

## 2.netmap发送

有两种模式，一直是同步死等，

```c++
if (ioctl(pfd.fd, NIOCTXSYNC, NULL) < 0) {
    D("ioctl error on queue %d: %s", targ->me,
            strerror(errno));    goto quit;
}
```

第二种用poll异步超时等待，poll可能是超时，也可能是内部错误，

```c++
if (poll(&pfd, 1, 2000) <= 0) {    if (targ->cancel)        break;
    D("poll error/timeout on queue %d: %s", targ->me,
        strerror(errno));    // goto quit;
}if (pfd.revents & POLLERR) {
    D("poll error on %d ring %d-%d", pfd.fd,
        targ->nmd->first_tx_ring, targ->nmd->last_tx_ring);    goto quit;
}
```

等待通过之后，表明网卡有硬件资源可以发包，循环检测当前文件句柄的netmap ring的剩余空间，如果为空则查找下一个ring。发包过程，主要调用两个接口，nm_ring_space和NETMAP_BUF宏。

```c++
for (i = targ->nmd->first_tx_ring; i <= targ->nmd->last_tx_ring; i++) {
    txring = NETMAP_TXRING(nifp, i);        if (nm_ring_empty(txring))
        continue;
    n = nm_ring_space(ring);        if (n < count)
        count = n;        for (fcnt = nfrags, sent = 0; sent < count; sent++) {
        struct netmap_slot *slot = &ring->slot[cur];
        char *p = NETMAP_BUF(ring, slot->buf_idx);
        int buf_changed = slot->flags & NS_BUF_CHANGED;        ...
        } else if ((options & OPT_COPY) || buf_changed) {
            nm_pkt_copy(frame, p, size);                    if (fcnt == nfrags)
                update_addresses(pkt, g);
        } else if (options & OPT_MEMCPY) {                ...
        cur = nm_ring_next(ring, cur);
    }
    ring->head = ring->cur = cur;
}
```

netmap退出时检测所有包都已发送，

```c++
/* flush any remaining packets */ioctl(pfd.fd, NIOCTXSYNC, NULL);/* final part: wait all the TX queues to be empty. */for (i = targ->nmd->first_tx_ring; i <= targ->nmd->last_tx_ring; i++) {
    txring = NETMAP_TXRING(nifp, i);        while (!targ->cancel && nm_tx_pending(txring)) {
        RD(5, "pending tx tail %d head %d on ring %d",
            txring->tail, txring->head, i);
        ioctl(pfd.fd, NIOCTXSYNC, NULL);
        usleep(1); /* wait 1 tick */
    }
}
```

## 3.netmap接收

和发送一样有两个方式，同步，

```c++
if (ioctl(pfd.fd, NIOCRXSYNC, NULL) < 0) {
    D("ioctl error on queue %d: %s", targ->me,
            strerror(errno));    goto quit;
}
```

异步，

```c++
if (poll(&pfd, 1, 1 * 1000) <= 0 && !targ->g->forever) {
    clock_gettime(CLOCK_REALTIME_PRECISE, &targ->toc);
    targ->toc.tv_sec -= 1; /* Subtract timeout time. */
    goto out;
}if (pfd.revents & POLLERR) {
    D("poll err");
    goto quit;
}
```

接收过程，

```c++
for (i = targ->nmd->first_rx_ring; i <= targ->nmd->last_rx_ring; i++) {
    rxring = NETMAP_RXRING(nifp, i);        if (nm_ring_empty(rxring))
        continue;
    cur = ring->cur;
    n = nm_ring_space(ring);
    for (rx = 0; rx < limit; rx++) {
        struct netmap_slot *slot = &ring->slot[cur];
        char *p = NETMAP_BUF(ring, slot->buf_idx);
        *bytes += slot->len;
        //your recv code here
        cur = nm_ring_next(ring, cur);
    }
    ring->head = ring->cur = cur;
}
```

原文链接：https://mp.weixin.qq.com/s/LqHZ2F6rhalv1oeSuq_RZg
原文作者： 黑客三遍猪


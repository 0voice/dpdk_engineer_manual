# nvme 驱动架构分析

4.97内核nvme驱动通过多队列方式实现，nvme盘驱动路径：drivers/nvme/host，

drivers/nvme/target端是针对设备的驱动，不涉及。

和nvme盘操作直接相关的只有两个文件pci.c、core.c。

## Nvme驱动入口：

Nvme盘是一个pcie设备，因此入口是pcie驱动probe函数：nvme_probe

Probe函数通过 queue_work(nvme_workq, &dev->reset_work);

## 调用nvme_reset_work函数

这个函数初始化nvme盘的admin和io队列（struct nvme_queue），同时初始化nvme盘的管理队列和请求队列对应的硬件队列描述结构blk_mq_tag_set，注意：这里的请求队列结构是struct request_queue，并不是nvme盘收发命令的admin和io队列，每个nvme逻辑盘只有一个请求队列，一个该请求队列对应多个nvme盘硬件io队列。nvme逻辑盘用struct nvme_ns结构表示，该结构包含通用盘设备结构：struct gendisk。

blk_mq_tag_set结构包含一个物理nvme盘硬件队列数、队列深度、io请求及处理等信息，该结构包含物理块设备所有描述信息，是块设备软件请求队列和硬件物理存储设备队列之间的纽带，建立了系统软件层面的io请求队列和物理存储设备硬件队列的映射关系。通过该结构文件系统读写操作发送的io请求最终到达物理存储设备。

创建完blk_mq_tag_set后，通过运行nvme_scan_work创建nvme_ns，nvme_ns包含struct gendisk接口，表示一个通用数据存储盘。同时用已经创建的blk_mq_tag_set创建一个io请求队列request_queue。

## pci.c相关代码：

```text
static void nvme_reset_work(struct work_struct *work)

{

         struct nvme_dev *dev = container_of(work, struct nvme_dev, reset_work);

         int result = -ENODEV;

         if (WARN_ON(dev->ctrl.state == NVME_CTRL_RESETTING))

                   goto out;

         /*

          * If we're called to reset a live controller first shut it down before

          * moving on.

          */
         if (dev->ctrl.ctrl_config & NVME_CC_ENABLE)

                   nvme_dev_disable(dev, false);

         if (!nvme_change_ctrl_state(&dev->ctrl, NVME_CTRL_RESETTING))

                   goto out;

         result = nvme_pci_enable(dev);

         if (result)

                   goto out;

         //nvme盘队管理队列申请

         result = nvme_configure_admin_queue(dev);

         if (result)

                   goto out;

         //nvme盘管理队列初始化

         nvme_init_queue(dev->queues[0], 0);

         //管理队列blk_mq_tag_set初始化

         result = nvme_alloc_admin_tags(dev);

         if (result)

                   goto out;

         result = nvme_init_identify(&dev->ctrl);

         if (result)

                   goto out;

         result = nvme_setup_io_queues(dev);

         if (result)

                   goto out;

         /*

          * A controller that can not execute IO typically requires user

          * intervention to correct. For such degraded controllers, the driver

          * should not submit commands the user did not request, so skip

          * registering for asynchronous event notification on this condition.

          */

         if (dev->online_queues > 1)

                   nvme_queue_async_events(&dev->ctrl);

         mod_timer(&dev->watchdog_timer, round_jiffies(jiffies + HZ));

         /*

          * Keep the controller around but remove all namespaces if we don't have

          * any working I/O queue.

          */

         if (dev->online_queues < 2) {

                   dev_warn(dev->ctrl.device, "IO queues not created\n");

                   nvme_kill_queues(&dev->ctrl);

                   nvme_remove_namespaces(&dev->ctrl);

         } else {

                   nvme_start_queues(&dev->ctrl);

                   //  初始化io请求队列对应blk_mq_tag_set

                   nvme_dev_add(dev);

         }

         if (!nvme_change_ctrl_state(&dev->ctrl, NVME_CTRL_LIVE)) {

                   dev_warn(dev->ctrl.device, "failed to mark controller live\n");

                   goto out;

         }

         if (dev->online_queues > 1)

                   nvme_queue_scan(&dev->ctrl); //运行nvme_scan_work创建nvme_ns

         return;
 out:

         nvme_remove_dead_ctrl(dev, result);

}

static int nvme_dev_add(struct nvme_dev *dev)

{
         if (!dev->ctrl.tagset) {

                   //针对nvme盘的请求处理函数接口，请求最终通过该接口和盘进行数据交互

                   dev->tagset.ops = &nvme_mq_ops;

                   dev->tagset.nr_hw_queues = dev->online_queues - 1;

                   dev->tagset.timeout = NVME_IO_TIMEOUT;

                   dev->tagset.numa_node = dev_to_node(dev->dev);

                   dev->tagset.queue_depth =

                                     min_t(int, dev->q_depth, BLK_MQ_MAX_DEPTH) - 1;

                   dev->tagset.cmd_size = nvme_cmd_size(dev);

                   dev->tagset.flags = BLK_MQ_F_SHOULD_MERGE;

                   dev->tagset.driver_data = dev;
                   //不要被这个接口名误导，其实不是申请tag_set，只是进一步申请tag_set中的tags，并初始化，主要是建立硬件队列和CPU的映射，申请request空间。

                   if (blk_mq_alloc_tag_set(&dev->tagset))

                            return 0;

                   dev->ctrl.tagset = &dev->tagset;

         } else {

                   blk_mq_update_nr_hw_queues(&dev->tagset, dev->online_queues - 1);

                   /* Free previously allocated queues that are no longer usable */

                   nvme_free_queues(dev, dev->online_queues);

         }

         return 0;

}
//请求队列相关操作接口

static struct blk_mq_ops nvme_mq_ops = {

         .queue_rq         = nvme_queue_rq, //请求处理接口，该接口获取请求队列中的请求和物理盘进行数据交互

         .complete         = nvme_complete_rq,

         .init_hctx = nvme_init_hctx,

         .init_request    = nvme_init_request,

         .map_queues  = nvme_pci_map_queues,

         .timeout  = nvme_timeout,

         .poll           = nvme_poll,

};
```

（免费订阅,永久学习）学习地址: [Dpdk/网络协议栈/vpp/OvS/DDos/NFV/虚拟化/高性能专家-学习视频教程-腾讯课堂](https://link.zhihu.com/?target=https%3A//ke.qq.com/course/5066203%3FflowToken%3D1043717%23term_id%3D-1)

更多DPDK相关学习资料有需要的可以自行报名学习,免费订阅,永久学习,或点击这里加[qun](https://link.zhihu.com/?target=https%3A//qm.qq.com/cgi-bin/qm/qr%3Fk%3DLgkDbnTyj6YB3JyvjFrCDKpe8SPkDIkO%26authKey%3DaPe4EghzEjEQ2vA08u5On4rkMm9jzgKm5qjkOlz2AhEbBPW2m0PpdyV9ReMp6Ud8%26noverify%3D0%26group_code%3D793599096)免费
领取,关注我持续更新哦! !

## core.c相关代码：

```text
static void nvme_scan_work(struct work_struct *work)

{

         struct nvme_ctrl *ctrl =

                   container_of(work, struct nvme_ctrl, scan_work);

         struct nvme_id_ctrl *id;

         unsigned nn;

         if (ctrl->state != NVME_CTRL_LIVE)

                   return;

         if (nvme_identify_ctrl(ctrl, &id))

                   return;

         nn = le32_to_cpu(id->nn);

         //以下两个接口最终都会调用nvme_alloc_ns创建nvme_ns

         if (ctrl->vs >= NVME_VS(1, 1, 0) &&

             !(ctrl->quirks & NVME_QUIRK_IDENTIFY_CNS)) {

                   if (!nvme_scan_ns_list(ctrl, nn))

                            goto done;

         }
```



nvme_scan_ns_sequential(ctrl, nn);

## done:

```text
         mutex_lock(&ctrl->namespaces_mutex);

         list_sort(NULL, &ctrl->namespaces, ns_cmp);

         mutex_unlock(&ctrl->namespaces_mutex);

         kfree(id);

}
static void nvme_alloc_ns(struct nvme_ctrl *ctrl, unsigned nsid)

{

         struct nvme_ns *ns;

         struct gendisk *disk;

         struct nvme_id_ns *id;

         char disk_name[DISK_NAME_LEN];

         int node = dev_to_node(ctrl->dev);

         ns = kzalloc_node(sizeof(*ns), GFP_KERNEL, node);

         if (!ns)

                   return;

         ns->instance = ida_simple_get(&ctrl->ns_ida, 1, 0, GFP_KERNEL);

         if (ns->instance < 0)

                   goto out_free_ns;

         //通过tagset创建逻辑块设备请求队列，并建立请求队列和物理队列映射关系

         ns->queue = blk_mq_init_queue(ctrl->tagset);

         if (IS_ERR(ns->queue))

                   goto out_release_instance;

         queue_flag_set_unlocked(QUEUE_FLAG_NONROT, ns->queue);

         ns->queue->queuedata = ns;

         ns->ctrl = ctrl;

         kref_init(&ns->kref);

         ns->ns_id = nsid;

         ns->lba_shift = 9; /* set to a default value for 512 until disk is validated */

         blk_queue_logical_block_size(ns->queue, 1 << ns->lba_shift);

         nvme_set_queue_limits(ctrl, ns->queue);

         sprintf(disk_name, "nvme%dn%d", ctrl->instance, ns->instance);

         if (nvme_revalidate_ns(ns, &id))

                   goto out_free_queue;

         if (nvme_nvm_ns_supported(ns, id)) {

                   if (nvme_nvm_register(ns, disk_name, node,

                                                                 &nvme_ns_attr_group)) {

                            dev_warn(ctrl->dev, "%s: LightNVM init failure\n",

                                                                           __func__);

                            goto out_free_id;

                   }

         } else {

                   disk = alloc_disk_node(0, node);

                   if (!disk)

                            goto out_free_id;

                   //设置盘控制操作接口，通过该操作可以直接向盘发送读写命令，不经过io块设备管理系统

                   disk->fops = &nvme_fops;

                   disk->private_data = ns;

                   disk->queue = ns->queue;

                   disk->flags = GENHD_FL_EXT_DEVT;

                   memcpy(disk->disk_name, disk_name, DISK_NAME_LEN);

                   ns->disk = disk;

                   __nvme_revalidate_disk(disk, id);

         }

         mutex_lock(&ctrl->namespaces_mutex);

         list_add_tail(&ns->list, &ctrl->namespaces);

         mutex_unlock(&ctrl->namespaces_mutex);

         kref_get(&ctrl->kref);

         kfree(id);

         if (ns->ndev)

                   return;

         device_add_disk(ctrl->device, ns->disk);

         if (sysfs_create_group(&disk_to_dev(ns->disk)->kobj,

                                               &nvme_ns_attr_group))

                   pr_warn("%s: failed to create sysfs group for identification\n",

                            ns->disk->disk_name);

         return;

 out_free_id:

         kfree(id);

 out_free_queue:

         blk_cleanup_queue(ns->queue);

 out_release_instance:

         ida_simple_remove(&ctrl->ns_ida, ns->instance);

 out_free_ns:

         kfree(ns);

}
```

以上代码包含了nvme驱动最核心的实现部分，下面总结概括下nvme驱动架构组成

创建硬件相关描述tag_set（nvme盘硬件队列个数、深度、申请request空间，建立硬件队列、软件队列CPU核间的对应关系），该结构也包含request处理函数。

创建通用磁盘描述nvme_ns,该结构包含gendisk

创建io请求队列request_queue，用tag_set初始化队列，将nvme_ns的request_queue指针指向创建的io请求队列，初始化gendisk。

nvme盘读写数据的流程也可以分三个阶段：

> 1、发送将内存数据组建成请求request
> 2、将request提交到请求队列request_queue
> 3、调用tag_set中的queue_rq函数处理请求（请求下发到nvme盘）

nvme io命令prp参数使用算法(PAGESIZE不是host页大小，是盘设置的页大小)：

> 1、只使用prp1 entry传输数据：datalen < PAGESIZE - prp1偏移
> 2、使用prp1 entry和prp2 entry传数据: (PAGESIZE - prp1偏移) < datalen < ((PAGESIZE - prp1偏移) + PAGESIZE))
> 3、使用prp1 entry和prp2 list传数据:datalen > ((PAGESIZE - prp1偏移) + PAGESIZE))

如果我们要针对一个特定的盘开发驱动，只要实现以上三个部分即可。例如通过内存模拟一个虚拟磁盘对应驱动

原文链接：[https://blog.csdn.net/luky_zhou](
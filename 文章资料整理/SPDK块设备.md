# SPDK块设备

SPDK视角每个App由多个子系统(subsystem)构成，同时每个子系统又包含多个模块(module)，子系统和模块的注入都是可插拔的，通过相关的宏定义声明集成到SPDK组件容器里(其中子系统的注入可通过声明SPDK_SUBSYSTEM_REGISTER，块设备模块的注入可通过声明SPDK_BDEV_MODULE_REGISTER)

目前SPDK已支持的子系统包括：accel、bdev、iscsi、nbd、nvmf、scheduler、scsi、sock、vhost、vmd，其中块设备子系统需要依赖accel、sock和vmd

在块设备子系统内部，每个模块都代表了一种块设备类型，目前已支持的块设备类型有：aio、compress、crypto、delay、error、ftl、gpt、iscsi、lvol、malloc、null、nvme、opal、ocf、passthru、pmem、raid、rbd、split、uring、virtio_blk、virtio_scsi、bdev_zoned_block，这里主要看的是NVMe块设备。

子系统及模块的初始化是在App启动阶段进行的(代码参考spdk_app_start及bootstrap_fn)，需要初始化的子系统和模块可通过json形式进行声明，比如这里想要引入NVMe块设备

```text
{"subsystems": [{"subsystem": "bdev", 
   "config": [{
      "method": "bdev_nvme_attach_controller",
      "params": {
         "name": "Nvme0",
         "trtype": "pcie",
         "traddr": "0000:0a:00.0"
       }
   }]
}]}
```

App启动加载完成之后，便可以对NVMe块设备进行访问(基于[NVMe用户态驱动](https://zhuanlan.zhihu.com/p/460820772))

## 1.IO设备注册

SPDK的整个IO链路中，大部分组件都是以IO设备的方式对外提供服务的，为了便于对IO设备进行访问，SPDK提供了spdk_get_io_channel函数来建立与设备的通信管道，同时在管道内部维护了与设备相关的上下文信息(可通过spdk_io_channel_get_ctx获取)，通过该上下文对象可辅助完成相关的IO动作。为此我们需要先把功能组件注册成IO设备(通过spdk_io_device_register)

块设备子系统在初始化过程中会将spdk_bdev_mgr注册成IO设备，以便通过其IO Channel来获取对应的spdk_bdev_mgmt_channel实例(作为上下文对象提供thread_local粒度的对象池功能，用于缓存spdk_bdev_io)

同时，NVMe bdev模块还会将nvme_bdev_ctrlrs注册成IO设备，通过其IO Channel可获取对应的nvme_poll_group实例，其内部维护了一个poller来轮询指定分组绑定的qpair集合。

除此之外，其他IO设备则是在驱动绑定的时候进行注册的(代码可参考connect_attach_cb回调函数)

![img](https://pic4.zhimg.com/80/v2-82f425e178b391daeb566ccbb1b11b6b_720w.webp)

相关IO设备的说明如下：

1. nvme_ctrlr
   nvme_ctrlr主要用于描述设备控制器信息，其会注册一个poller函数(bdev_nvme_poll_adminq)来对adminq做周期性轮询。
   除此之外，通过其所提供的IO Channel还可获取每个NS对应的qpair实例，并将其加入指定的轮询分组(通过spdk_nvme_poll_group_add)
2. nvme_bdev
   nvme_bdev是在对NS进行populate过程中创建的(代码参考nvme_ctrlr_populate_namespace)，通过其所提供的IO Channel可获取每个NS对应的nvme_io_path信息，进而定位具体的IO QPair
3. spdk_bdev
   spdk_bdev也是在此期间构建的，并注入到spdk_bdev_mgr对应的集合中(代码参考spdk_bdev_register)

## 2.块设备使用

块设备的使用主要遵循以下3个操作步骤

1. 获取块设备描述符
   对此，SPDK提供了spdk_bdev_open_ext功能函数来对目标块设备进行open，并返回其对应的设备描述符spdk_bdev_desc
2. 建立通信管道
   拿到设备描述符之后，可通过spdk_bdev_get_io_channel建立与目标块设备的通信管道，其会对图片中的每个IO设备形成访问，来创建用于IO通信的qpair实例，并通过指定的poller对其进行轮询。
3. 通过管道对块数据进行读写
   对此，SPDK提供了spdk_bdev_write和spdk_bdev_read函数以便于向目标块设备提交对应的BIO请求。





 原文链接：https://zhuanlan.zhihu.com/p/461512223 原文作者：爱分享的码农
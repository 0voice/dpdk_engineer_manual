# DPDK kni模块分析

## 1.相关背景

dpdk缺少相应的协议栈，这提升了dpdk的入门门槛。如果我们采用dpdk来开发代码，当网卡被用户态驱动绑定之后，我们不仅需要处理业务报文，还需要对arp，ping，tcp握手等消息进行处理，对于笔者目前的一个业务场景（缺少协议栈，控制面和用户名共用一个网卡不同ip，数据面需要提升收包性能，因此采用了dpdk。为什么控制面和用户名不用两张网卡呢？产品经理决定的~）来说就比较头疼。幸运的是dpdk提供了kni模块，来解决相应的问题。这篇文章就是介绍dpdk的kni模块的实现机制。建议先阅读下dpdk官方提供的例子，对kni的大致使用方法有个了解。

## 2.什么是kni模块？

kni模块是dpdk提供的用于用户态和内核协议栈通信的一个机制，具体来说当我们启动了kni模块之后，从内核的角度来看，就多了一张网卡。可以类比tap/tun口，不同的是这个网卡可以收到dpdk用户态的数据包，发送给内核，也可以接收内核的数据发送给用户态。这样对于非业务报文，我们就可以直接扔给内核由内核处理。

## 3.kni的实现机制

### 3.1内核模块的加载

kni需要内核模块的支持，因此要想使用kni模块，首先要编译并加载rte_kni.ko，加载该模块的时候，共有三个参数，分别是lo_mode，kthread_mode和carrier。具体来说

lo_mode可配置为lo_mode_none，lo_mode_fifo，和lo_mode_fifo_skb，默认为lo_mode_none。
kthread_mode可配置为single和multiple，默认为single。
carrier可配置为off和on，默认为off。

加载了rte_kni.ko模块，就调用了module_init(kni_init)，大致的调用函数如下：

kni_init->
        misc_register,内核函数，内核使用misc_register函数注册一个混杂设备。注册成功后，linux内核为自动为该设备创建设备节点，在/dev/下会产生相应的节点。注册设备的相关函数由misc_register入参（kni_misc）决定。
        kni_net_config_lo_mode，配置lo_mode，使函数指针kni_net_rx_func指向不同的函数，默认为kni_net_rx_normal。

init完成之后，就完成了在内核里注册成功了一个kni设备，相关的函数会在用户态调用相关的open，ioctl调用。

关于相应的内核函数，熟悉linux驱动的朋友可能比较熟悉，不熟悉的建议百度~笔者也是一知半解。这里可以重点关注下这个数据结构：

```c++
static const struct file_operations kni_fops = {
    .owner = THIS_MODULE,
    .open = kni_open,
    .release = kni_release,
    .unlocked_ioctl = (void *)kni_ioctl,
    .compat_ioctl = (void *)kni_compat_ioctl,
};

static struct miscdevice kni_misc = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = KNI_DEVICE,
    .fops = &kni_fops,
};
```

misc_register就是misc_register函数的入参，这个数据结构决定了后续用户态调用open和ioctl的时候，内核调用的函数。

### 3.2用户态相关逻辑

参考dpdk官方给的例子，用户态的初始化函数为rte_kni_init。该函数调用open函数，如上面所述，内核调用的函数为之前指定的kni_open函数。打开相应的设备申请相关ring的内存。到这里，使用ifconfig应该能够看到kni网络设备了，你可以对这个设备设置ip等。

随后需要调用rte_kni_alloc函数。在这个函数里会看到调用ioctl，同样，内核调用的函数为kni_ioctl，然后进入kni_ioctl_create函数。对kni设备进行配置，将KNI的用户态和内核态使用同一片物理地址，从而做到零拷贝。该函数还会调用alloc_netdev()，

kni_ioctl->

     kni_ioctl_create:对kni设备进行配置->
    
          alloc_netdev，内核函数，给网络设备分配net_device_ops结构，指定了网络设备的诸多动作对应的函数。这里可以重点关注一下网络设备的发包函数kni_net_tx
    
          kni_run_thread，内核会启动一个线程用来处理kni设备的收发（根据kthread_mode来决定启动单个线程处理，所有kni设备的收发还是每个kni设备对应一个线程。本篇博客默认的场景是启动单个线程处理所有kni设备的收发动作）。

### 3.3用户态往kni设备发送数据

用户态对应的函数是rte_kni_tx_burst。当用户态需要向内核发送数据时，可以调用该函数向kni设备发送报文。首先将报文的虚拟地址转换成物理地址，然后调用kni_fifo_put函数将报文放入kni设备的收包队列中。随后调用kni_free_mbufs释放掉内核处理完的mbuf报文。在内核侧，启动的线程会遍历所有kni设备，执行接收动作。相应的接收函数为kni_net_rx_normal（具体是哪个函数取决于加载rte_kni.ko模块时的入参，这里是默认情况下对应的函数）。
1.检查当前free队列中还有无空闲位置，如果没有，则后面处理完的报文无法放入free队列，故直接返回，不处理这个报文
2.调用kni_fifo_get，取出设备的接收队列上的报文。
3.对于接收到的报文，将物理地址转换为虚拟地址，将数据复制到申请的skb中。然后调用netif_rx_ni，将报文送到内核协议栈。
4.对处理完成的报文，放入free队列，后面可以调用kni_free_mbufs释放相应的内存。
关于netif_rx_ni函数，这个函数是将数据送入协议栈ip层，主要把数据包链接到input_pkt_queue队列，并启动一次软中断函数，内核协议栈收到相应的报文后可以进行处理。

### 3.4用户态收kni设备发过来的数据

用户态使用rte_kni_rx_burst函数进行数据接收。

内核需要往外发送数据时，按照内核的流程处理，最终会调用到net_device_ops->ndo_start_xmit。对于KNI驱动来说，即上述所说的kni_net_tx函数。
kni_net_tx：从kni设备的alloc_q队列取出一个mbuff内存，然后将内核中的skb结构体中的数据拷贝到mbuff中，随后放入kni设备的tx_q队列中，最后释放sbk结构体的内存。对于用户态来说，会调用rte_kni_rx_burst函数进行数据接收，这个函数从kni设备的发送队列中取出报文数据。然后申请mbuff内存，放入设备的alloc_q队列。

 

## 4.应用场景

这里说下开头所述的业务场景是如何实现的：

首先分析下这个场景的痛点所在，两个不同网段的ip，只有一个网卡，这个网卡需要被dpdk绑定，没有协议栈。

OK，kni完美契合。我们在生成的kni设备上配置两个ip，然后对于物理网卡收上来的包，除了我们需要处理的业务报文，其余的报文都可以发送到这个kni设备上由内核处理。站在外界的角度来看，这个网卡就是拥有了两个ip，因为无论是arp还是ping，只要发到了这个物理网卡上，这两个ip都是可达的。







原文链接：https://blog.csdn.net/the_dog_tail_grass/article/details/112709196

原文作者：the_dog_tail_grass
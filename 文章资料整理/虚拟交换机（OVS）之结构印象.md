# 虚拟交换机（OVS）之结构印象

当拿到OVS这么大一个工程的时候，如何理解他的组织、实现的功能、数据结构的创新，对于这个从0到1的过程，该如何一步步深入呢？

首先，我觉得直接看架构也好，看文件目录也好，都是比较容易理解全局的办法。
那就先看看文件的组织：

![img](https://images2017.cnblogs.com/blog/614525/201801/614525-20180124220256006-776164795.png)

这些显示的是文件夹的目录，从目录中可以看出有window相关的，也有xenserver相关的，说明OVS不光支持Linux，还支持别的平台。
然后浏览一下其他的目录，根据之前的了解，有datapath,include,lib,ofproto,ovn,ovsdb,vswitchd，vetp这几个重要的目录，其他的如m4是跟编译相关的。除了目录之外，就是一些配置安装说明文件。但是还是要吐槽一下，OVS的功能划分在文件组织上非常混沌，干脆是一坨直接丢在一起，比如lib目录，跟DPDK的目录组织差不少。DPDK的目录可以借机出镜一下：

![img](https://images2017.cnblogs.com/blog/614525/201801/614525-20180124220309475-255230227.png)

所以，大致梳理一下的话，根据之前的了解

- datapath实现的是一个内核快速匹配转发模块。
- lib目录应该实现的是很多算法、结构之类的东西，如hash，log等。
- ofproto目录应该实现的是一个中间层，这也是重点要分析的地方，因为现在还看不出作用来。
- ovn是一个虚拟网络的平台，应该不是ovs的必备组件。最后再看。
- ovsdb是ovs的数据库，这个目录实现的是server，client的一些东西，还有必要的接口。
- vswitchd是交换机实现的目录，然鹅，里面的文件不多，网桥的实现。也算是核心了。
- vtep 是VxLAN隧道终结点设备，VxLAN是一种UDP隧道，多见于数据中心网络实现overlay的网络虚拟化。

从目录上看，顺带查找一些资料，基本能获得以上这些信息。另外此处先诞生一个疑问：

> OVS是能够用DPDK进行加速的，绕开内核，要修改的部分还不少，那么他们在目录的哪里呢？实际上就是在lib这个大杂烩中-_-||

然后再从架构上看一下的话，就如下面这张图：

![img](https://images2017.cnblogs.com/blog/614525/201801/614525-20180124220322553-486039362.jpg)

核心的组成部分主要有3个，vswitchd，ovsdb-server,datapath。从整体上了解了这些后，

1. 就可以从原理上探究每一部分的大致组成以及实现过程。
2. 配置OVS，发送和接收一个数据报文，追踪其流程，观察处理过程。
3. 回顾总结OVS的设计。

既然如此，就开始从第一部分出发吧，先探究这三个部分的主要内容。

## 1. datapath

第一个选datapath入手，是因为datapath是内核模块，关联性更少一些。
既然是内核模块，就先找到模块的初始化入口，在datapath.c中的`dp_init()`，这是个`__init`函数，__init告诉编译器这个函数只用于初始化。

```c
		static int __init dp_init(void)
{
	int err;
 
	BUILD_BUG_ON(sizeof(struct ovs_skb_cb) > FIELD_SIZEOF(struct sk_buff, cb));
 
	pr_info("Open vSwitch switching datapath %s\n", VERSION);
 
	err = compat_init();
	if (err)
		goto error;
 
	err = action_fifos_init(); /* 初始化延迟操作的队列 */
	if (err)
		goto error_compat_exit;
 
	err = ovs_internal_dev_rtnl_link_register(); /* internal设备操作集及netlink操作集的注册 */
	if (err)
		goto error_action_fifos_exit;
 
	err = ovs_flow_init(); /* 流表初始化 */
	if (err)
		goto error_unreg_rtnl_link;
 
	err = ovs_vport_init(); /* vport子系统初始化 */
	if (err)
		goto error_flow_exit;
 
	err = register_pernet_device(&ovs_net_ops); /* 注册namespace相关的ovs初始化 */
	if (err)
		goto error_vport_exit;
 
	err = register_netdevice_notifier(&ovs_dp_device_notifier); /* 注册设备通知 */
	if (err)
		goto error_netns_exit;
 
	err = ovs_netdev_init(); /* 注册netdev类型的设备操作集 */
	if (err)
		goto error_unreg_notifier;
 
	err = dp_register_genl(); /* 注册通用的netlink，共注册了四种*/
	if (err < 0)
		goto error_unreg_netdev;
 
	return 0;
```

从大致过程上说，在初始化完成后，就等待配置，如流表下发，添加端口等，然后就是等待报文匹配。
除此之外的`datapath.c`文件中主要就是注册的四种netlink的操作实现。

## 2. vswitchd

vswitchd是虚拟交换机的守护进程，主要的实现在vswitchd目录下，`bridge.c`。来看一下这个用户态进程的启动，vswitchd的的入口是在`ovs-vswitchd.c`的main()函数。

前面的解析参数和DPDK的初始化部分就不再说了，然后创建守护进程：

```c
daemonize_start(true);
```

之后创建了unixctl服务器，并注册命令：

```c
retval = unixctl_server_create(unixctl_path, &unixctl);
    if (retval) {
        exit(EXIT_FAILURE);
    }
    unixctl_command_register("exit", "", 0, 0, ovs_vswitchd_exit, &exiting);
```

之后初始化网桥，`bridge_init(remote);`，主要工作就是

- 创建和数据库的连接
- 注册各种协议（如stp,bond,lacp）的命令和回调函数。

再之后，就是启动网桥和网卡接收，循环等待退出

```c
while (!exiting) {
        memory_run();
        if (memory_should_report()) {
            struct simap usage;
 
            simap_init(&usage);
            bridge_get_memory_usage(&usage);
            memory_report(&usage);
            simap_destroy(&usage);
        }
        bridge_run();
        unixctl_server_run(unixctl);
        netdev_run();
 
        memory_wait();
        bridge_wait();
        unixctl_server_wait(unixctl);
        netdev_wait();
```

这里有个`memory_run()`的，是监控内存的使用的功能，赞一下这个东西，对于故障的监测蛮有用。然后是网桥，unixctl服务器，netdev运行。

这样子初始化过后，用户态的vswitchd就运行起来了。
但是这里需要分析一下软件在实现上的组织，因为还有一个ofproto的层存在，那么他和vswitchd有啥关系呢？

> ofproto库真正实现了交换机逻辑。除此之外，还有两个重要的库，一个是netdev,一个是dpif。前者是对设备的抽象，后者则实现了流表的操作。

对于这几个重要的库的作用和实现，等到追踪报文和配置流程的时候再仔细分析。

## 3. ovsdb-server

这一部分来说说数据库，南向接口把数据配置到数据库，然后数据库通过socket与vswitchd通信，把配置信息发给vswitchd。

同样，ovsdb-server的入口是在`ovsdb-server.c`中的main()函数。

```c
server_config.remotes = &remotes;
    server_config.config_tmpfile = config_tmpfile;
 
    save_config__(config_tmpfile, &remotes, &db_filenames);
 
    daemonize_start(false);
 
    /* Load the saved config. */
    load_config(config_tmpfile, &remotes, &db_filenames);
```

读取配置文件，加载配置信息到数据库中。然后创建ovsdb-server,并打开。

```c
jsonrpc = ovsdb_jsonrpc_server_create();
 
shash_init(&all_dbs);
server_config.all_dbs = &all_dbs;
server_config.jsonrpc = jsonrpc;
 
perf_counters_init();
 
SSET_FOR_EACH (db_filename, &db_filenames) {
    error = open_db(&server_config, db_filename);
    if (error) {
        ovs_fatal(0, "%s", error);
    }
}
```

之后就创建了unixctl服务器，注册了多个命令

```c
retval = unixctl_server_create(unixctl_path, &unixctl);
    if (retval) {
        exit(EXIT_FAILURE);
    }
```

那么这里注册unixctl服务器是让谁连接呢？vswitchd以及ovsdb。
最后又到了main死循环中，`main_loop()`。
在其中也是和vswitchd差不多，启动unixctl服务器，启动ovs-db server。主要的流程就这些咯。

至此，就把OVS的三大部分的主要组成说完了，这也完成了OVS的第一步的总体印象的分析。后续的第二篇会进一步跟踪配置和数据包的处理流程，详细分析代码的逻辑。期待下一篇吧。





转载自：  作者 [AISEED](https://home.cnblogs.com/u/yhp-smarthome/)   本文链接 https://www.cnblogs.com/yhp-smarthome/p/8343771.html
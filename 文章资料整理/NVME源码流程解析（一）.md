# NVME源码流程解析（一）

## 源码目录

源码在drivers/nvme下面，有两个文件夹，其中targets 文件夹用于将nvme 设备作为

磁盘导出供外部使用，host文件夹实现将nvme 设备供本系统使用。若配置NVME target 还

需要工具nvmetcli工具。

## ／drivers/nvme/host/Makefile

模块编译的五个配置参数：

> NVME_CORE:nvme的核心基础
> BLK_DEV_NVME:用于将ssd链接到pcie上
> CONFIG_NVME_FABRICS:支持FC协议
> CONFIG_NVME_RDMA:使得 NVMe over Fabric 可以通过 RDMA 传输
> CONFIG_NVME_FC:使得NVMe over Fabric可以在FC 传输

## ／drivers/nvme/host/pci.c

nvme驱动的注册和注销——整体函数流程如下图所所示：

![img](https://pic3.zhimg.com/80/v2-d57a731ddc2c0494c6cbbb3b71c68f4e_720w.webp)

其中nvme_init函数最终会调用到nvme_probe，其构造如下：

![img](https://pic1.zhimg.com/80/v2-b01478c2ff103da62a70a6ed6bc81044_720w.webp)

图片可以放大来看！主要记录了关键函数之间的调用关系！

### 驱动的注册和注销：

```text
module_init(nvme_init);
module_exit(nvme_exit);
```

### 调用 pci_register_driver()注册时，其参数为 struct pci_driver.

```text
static struct pci_driver nvme_driver = {
	.name		= "nvme",
	.id_table	= nvme_id_table,
	.probe		= nvme_probe,
	.remove		= nvme_remove,
	.shutdown	= nvme_shutdown,
#ifdef CONFIG_PM_SLEEP
	.driver		= {
		.pm	= &nvme_dev_pm_ops,
	},
#endif
	.sriov_configure = pci_sriov_configure_simple,
	.err_handler	= &nvme_err_handler,
};
```

###  /drivers/pci/pci-driver.c

```text
/**
 * __pci_register_driver - register a new pci driver
 * @drv: the driver structure to register
 * @owner: owner module of drv
 * @mod_name: module name string
 *
 * Adds the driver structure to the list of registered drivers.
 * Returns a negative value on error, otherwise 0.
 * If no error occurred, the driver remains registered even if
 * no device was claimed during registration.
 */
int __pci_register_driver(struct pci_driver *drv, struct module *owner,
			  const char *mod_name)
{
	/* initialize common driver fields */
	drv->driver.name = drv->name;
	drv->driver.bus = &pci_bus_type;    
                              //pci_bus_type是个结构体————struct bus_type pci_bus_type
	drv->driver.owner = owner;
	drv->driver.mod_name = mod_name;
	drv->driver.groups = drv->groups;
 
	spin_lock_init(&drv->dynids.lock);
	INIT_LIST_HEAD(&drv->dynids.list);               //初始化设备中的节点
 
	/* register with core */
	return driver_register(&drv->driver);
}
```

### 接下来看看driver_register(&drv->driver)函数 —— /drivers/base/driver.c

```text
/**
 * driver_register - register driver with bus
 * @drv: driver to register
 *
 * We pass off most of the work to the bus_add_driver() call,
 * since most of the things we have to do deal with the bus
 * structures.
 */
int driver_register(struct device_driver *drv)
{
	int ret;
	struct device_driver *other;
 
	if (!drv->bus->p) {
		pr_err("Driver '%s' was unable to register with bus_type '%s' because the bus was not initialized.\n",
			   drv->name, drv->bus->name);
		return -EINVAL;
	}
 
	if ((drv->bus->probe && drv->probe) ||
	    (drv->bus->remove && drv->remove) ||
	    (drv->bus->shutdown && drv->shutdown))
		printk(KERN_WARNING "Driver '%s' needs updating - please use "
			"bus_type methods\n", drv->name);
 
	other = driver_find(drv->name, drv->bus);          //寻找是否已注册同名驱动
	if (other) {
		printk(KERN_ERR "Error: Driver '%s' is already registered, "
			"aborting...\n", drv->name);
		return -EBUSY;
	}
 
	ret = bus_add_driver(drv);             //真正将驱动设备添加到总线中的地方 
	if (ret)
		return ret;
	ret = driver_add_groups(drv, drv->groups);         //通过sysfs_create_groups函数创建NVME驱动的属性组
	if (ret) {
		bus_remove_driver(drv);
		return ret;
	}
	kobject_uevent(&drv->p->kobj, KOBJ_ADD);          //通知用户驱动加载成功
 
	return ret;
}
```

### 真正加载驱动的函数是bus_add_driver(drv) —— /drivers/base/bus.c

```cpp
/**
 * bus_add_driver - Add a driver to the bus.
 * @drv: driver.
 */
int bus_add_driver(struct device_driver *drv)
{
	struct bus_type *bus;
	struct driver_private *priv;
	int error = 0;
 
	bus = bus_get(drv->bus);                  //获取总线
	if (!bus)
		return -EINVAL;
 
	pr_debug("bus: '%s': add driver %s\n", bus->name, drv->name);
 
	priv = kzalloc(sizeof(*priv), GFP_KERNEL);         //分配空间
	if (!priv) {
		error = -ENOMEM;
		goto out_put_bus;
	}
	klist_init(&priv->klist_devices, NULL, NULL);          //初始化列表
	priv->driver = drv;
	drv->p = priv;
	priv->kobj.kset = bus->p->drivers_kset;
	error = kobject_init_and_add(&priv->kobj, &driver_ktype, NULL,
				     "%s", drv->name);
                                    //初始化一个kobject结构体，并加入到kobject架构中
	if (error)
		goto out_unregister;
 
	klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);
                //将priv->knode_bus添加到总线的subsys_private->klist_drivers链表中
 
	if (drv->bus->p->drivers_autoprobe) {
		error = driver_attach(drv);
            //调用bus_for_each_dev负责在总线上遍历设备，并将设备传递给函数，会调用__driver_attach进行设备和驱动的匹配
 
		if (error)
			goto out_unregister;
	}
	module_add_driver(drv->owner, drv);
 
／＊
 
    该函数主要实现是sysfs_create_link 在sysfs文件系统中创建相关文件。
    sysfs_create_link(&drv->p->kobj, &mk->kobj, "module");
    sysfs_create_link(mk->drivers_dir, &drv->p->kobj, driver_name);
 
        第一个 sysfs_create_link 调用在／sys/bus/pci/drivers/nvme 中创建 module 指向
／sys/module/nvme,第二个 sysfs_create_link 在目录／sys/module/nvme/drivers 中创建 pci:nvme 链接，指向／sys/bus/pci/drivers/nvme 驱动。
 
＊／
 
	error = driver_create_file(drv, &driver_attr_uevent);
    //过 sysfs_create_file 函数，sysfs_create_file(&drv->p->kobj,&attr->attr),在/sys/bus/pci/drivers/nvme中创建驱动的属性文件
 
	if (error) {
		printk(KERN_ERR "%s: uevent attr (%s) failed\n",
			__func__, drv->name);
	}
	error = driver_add_groups(drv, bus->drv_groups);
    //通过sysfs_create_groups函数创建总线中驱动的属性组
	if (error) {
		/* How the hell do we get out of this pickle? Give up */
		printk(KERN_ERR "%s: driver_create_groups(%s) failed\n",
			__func__, drv->name);
	}
 
	if (!drv->suppress_bind_attrs) {
		error = add_bind_files(drv);       //创建绑定的属性文件
		if (error) {
			/* Ditto */
			printk(KERN_ERR "%s: add_bind_files(%s) failed\n",
				__func__, drv->name);
		}
	}
 
	return 0;
 
out_unregister:
	kobject_put(&priv->kobj);
	/* drv->p is freed in driver_release()  */
	drv->p = NULL;
out_put_bus:
	bus_put(bus);
	return error;
}
```

其中最重要的函数是driver_attach(drv)，其会负责进行设备和驱动的匹配，该函数留到下一小节来讲！

原文链接：[https://blog.csdn.net/weixin_38](
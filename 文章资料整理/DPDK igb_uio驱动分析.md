# DPDK igb_uio驱动分析

本文整理下之前的学习笔记，基于DPDK17.11版本源码分析。主要分析一下igb_uio驱动源码。

# 总线-设备-驱动

首先简单介绍一下kernel中的**总线-设备-驱动模型**，以pci总线为例，**pci总线上有两个表**，一个**用于保存系统中的pci设备**，一个**用于保存pci设备对应的驱动**。**每当加载pci设备驱动时，就会遍历pci总线上的pci设备进行匹配，每当插入pci设备到系统中时，热插拔机制就会自动遍历pci总线上的pci设备驱动进行匹配，如果匹配成功则使用此驱动初始化设备。**

**注册pci总线**
可以调用**bus_register**注册总线。比如下面的pci总线，平台总线和usb总线等。

```c++
//注册pci总线
struct bus_type pci_bus_type = {
    .name       = "pci",
    .match      = pci_bus_match,
    .uevent     = pci_uevent,
    .probe      = pci_device_probe,
    .remove     = pci_device_remove,
    .shutdown   = pci_device_shutdown,
    .dev_groups = pci_dev_groups,
    .bus_groups = pci_bus_groups,
    .drv_groups = pci_drv_groups,
    .pm     = PCI_PM_OPS_PTR,
};
bus_register(&pci_bus_type);
 
//注册平台总线
struct bus_type platform_bus_type = {
    .name       = "platform",
    .dev_groups = platform_dev_groups,
    .match      = platform_match,
    .uevent     = platform_uevent,
    .pm     = &platform_dev_pm_ops,
};
bus_register(&platform_bus_type);
 
//注册usb总线
struct bus_type usb_bus_type = {
    .name =     "usb",
    .match =    usb_device_match,
    .uevent =   usb_uevent,
};
bus_register(&usb_bus_type);
 
//注册virtio总线
static struct bus_type virtio_bus = {
    .name  = "virtio",
    .match = virtio_dev_match,
    .dev_groups = virtio_dev_groups,
    .uevent = virtio_uevent,
    .probe = virtio_dev_probe,
    .remove = virtio_dev_remove,
};
bus_register(&virtio_bus)
```

注册总线后，会在 **/sys/bus** 下生成总线目录，比如 pci 总线会生成目录 /sys/bus/pci

```rust
/**
 * bus_register - register a driver-core subsystem
 * @bus: bus to register
 *
 * Once we have that, we register the bus with the kobject
 * infrastructure, then register the children subsystems it has:
 * the devices and drivers that belong to the subsystem.
 */
int bus_register(struct bus_type *bus)
    struct subsys_private *priv;
    struct lock_class_key *key = &bus->lock_key;
 
    priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL);
    priv->bus = bus;
    bus->p = priv;
    kobject_set_name(&priv->subsys.kobj, "%s", bus->name);
    priv->subsys.kobj.kset = bus_kset;
    priv->subsys.kobj.ktype = &bus_ktype;
    kset_register(&priv->subsys);
    
    //此值为1加载驱动时会自动探测设备进行匹配
    priv->drivers_autoprobe = 1;
    
    bus_create_file(bus, &bus_attr_uevent);
    
    //在总线目录下，生成 devices 子目录，下面再包含具体pci设备子目录
    priv->devices_kset = kset_create_and_add("devices", NULL,
                         &priv->subsys.kobj);
    //在总线目录下，生成 drivers 子目录，下面再包含具体驱动子目录
    priv->drivers_kset = kset_create_and_add("drivers", NULL,
                         &priv->subsys.kobj);
    //此链表用于保存加载的pci设备驱动
    klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
    //此链表用于保存扫描到的pci设备
    klist_init(&priv->klist_drivers, NULL, NULL);
    
    //在sys文件系统创建 drivers_probe 和 drivers_autoprobe 文件
    add_probe_files(bus);
        bus_create_file(bus, &bus_attr_drivers_probe);
        bus_create_file(bus, &bus_attr_drivers_autoprobe);
    
    bus_add_groups(bus, bus->bus_groups);
```

注册总线后，会生成文件**/sys/bus/pci/drivers_autoprobe**，写此文件时在kernel中会调用如下函数，如果为1，表示 bus 支持自动探测 device，则加载驱动时，自动遍历所有pci设备进行匹配

```rust
store_drivers_autoprobe
static ssize_t store_drivers_autoprobe(struct bus_type *bus,
                       const char *buf, size_t count)
{
    if (buf[0] == '0')
        bus->p->drivers_autoprobe = 0;
    else
        bus->p->drivers_autoprobe = 1;
    return count;
}
```

**注册驱动到pci总线**

结构体struct pci_driver表示一个pci设备驱动，其中id_table和dynids用来保存此驱动支持的设备id等信息，如果有匹配的设备，则调用probe函数。

```
struct pci_driver {
    struct list_head node;
    const char *name;
    //静态table，用来保存驱动支持的id
    const struct pci_device_id *id_table;   /* must be non-NULL for probe to be called */
    int  (*probe)  (struct pci_dev *dev, const struct pci_device_id *id);   /* New device inserted */
    void (*remove) (struct pci_dev *dev);   /* Device removed (NULL if not a hot-plug capable driver) */
    int  (*suspend) (struct pci_dev *dev, pm_message_t state);  /* Device suspended */
    int  (*suspend_late) (struct pci_dev *dev, pm_message_t state);
    int  (*resume_early) (struct pci_dev *dev);
    int  (*resume) (struct pci_dev *dev);                   /* Device woken up */
    void (*shutdown) (struct pci_dev *dev);
    int (*sriov_configure) (struct pci_dev *dev, int num_vfs); /* PF pdev */
    const struct pci_error_handlers *err_handler;
    struct device_driver    driver;
    //动态table，通过写文件 new_id 动态添加id
    struct pci_dynids dynids;
};
```

调用函数pci_register_driver注册pci设备驱动。

```objectivec
static struct pci_driver igbuio_pci_driver = {
    .name = "igb_uio",
    .id_table = NULL,  //DPDK 用到的 igb_uio, vfio-pci等驱动的id_table默认为空
    .probe = igbuio_pci_probe,
    .remove = igbuio_pci_remove,
};
pci_register_driver(&igbuio_pci_driver);
 
 
static const struct pci_device_id igb_pci_tbl[] = {
    { PCI_VDEVICE(INTEL, E1000_DEV_ID_I354_BACKPLANE_1GBPS) },
    { PCI_VDEVICE(INTEL, E1000_DEV_ID_I354_SGMII) },
    ...
}
 
static struct pci_driver igb_driver = {
    .name     = igb_driver_name,
    .id_table = igb_pci_tbl,  //正常的kernel驱动都有一个静态的id_table
    .probe    = igb_probe,
    .remove   = igb_remove,
#ifdef CONFIG_PM
    .driver.pm = &igb_pm_ops,
#endif
    .shutdown = igb_shutdown,
    .sriov_configure = igb_pci_sriov_configure,
    .err_handler = &igb_err_handler
};
pci_register_driver(&igb_driver);
```

注册驱动后，会在/sys/bus/pci/drivers目录下创建以驱动名字命名的目录，并在此目录下创建new_id, bind和unbind等sys文件，可以通过这些文件动态修改驱动信息。

```rust
/*
 * pci_register_driver must be a macro so that KBUILD_MODNAME can be expanded
 */
#define pci_register_driver(driver)     \
    __pci_register_driver(driver, THIS_MODULE, KBUILD_MODNAME)
 
int __pci_register_driver(struct pci_driver *drv, struct module *owner,
              const char *mod_name)
{
    /* initialize common driver fields */
    drv->driver.name = drv->name;
    //bus固定为 pci_bus_type
    drv->driver.bus = &pci_bus_type;
    drv->driver.owner = owner;
    drv->driver.mod_name = mod_name;
 
    spin_lock_init(&drv->dynids.lock);
    INIT_LIST_HEAD(&drv->dynids.list);
 
    /* register with core */
    driver_register(&drv->driver);
        bus_add_driver(drv);
            struct bus_type *bus;
            struct driver_private *priv;
            
            bus = bus_get(drv->bus);
            priv = kzalloc(sizeof(*priv), GFP_KERNEL);
            klist_init(&priv->klist_devices, NULL, NULL);
            priv->driver = drv;
            drv->p = priv;
            priv->kobj.kset = bus->p->drivers_kset;
            kobject_init_and_add(&priv->kobj, &driver_ktype, NULL, "%s", drv->name);
 
            //将驱动添加到pci总线
            klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);
 
            //如果pci总线支持自动探测设备，则在加载驱动时就遍历所有pci设备进行匹配
            if (drv->bus->p->drivers_autoprobe) {
                driver_attach(drv);
                    //遍历所有的pci设备，和drv进行匹配
                    bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
                        //设备和驱动进行匹配
                        driver_match_device(drv, dev)
                        //如果匹配成功，并且设备还没有加载其他驱动，则使用当前驱动drv
                        if (!dev->driver)
                            driver_probe_device(drv, dev);
            }
 
            module_add_driver(drv->owner, drv);
 
            driver_create_file(drv, &driver_attr_uevent);
 
            //bus->drv_groups 为 pci_drv_groups，
            //在sys文件系统创建 new_id 和 remove_id 文件
            driver_add_groups(drv, bus->drv_groups);
            
            //在sys文件系统创建 bind 和 unbind 文件，用来将驱动绑定和解绑定设备
            if (!drv->suppress_bind_attrs) {
                add_bind_files(drv);
                    driver_create_file(drv, &driver_attr_unbind);
                    driver_create_file(drv, &driver_attr_bind);
            }
}
```

向new_id写入"0x0806 0x1521"信息(0x0806表示vendor id，0x1521为device id)时，会调用kernel中的store_new_id，解析相关字段后，保存到动态链表dynids，然后遍历当前所有的pci设备进行匹配。

```cpp
//定义struct driver_attribute driver_attr_new_id
static DRIVER_ATTR(new_id, S_IWUSR, NULL, store_new_id);
//定义 //struct driver_attribute driver_attr_remove_id
static DRIVER_ATTR(remove_id, S_IWUSR, NULL, store_remove_id);
 
//定义 struct attribute_group pci_drv_groups
static struct attribute *pci_drv_attrs[] = {
    &driver_attr_new_id.attr,
    &driver_attr_remove_id.attr,
    NULL,
};
ATTRIBUTE_GROUPS(pci_drv);
 
static ssize_t store_new_id(struct device_driver *driver, const char *buf,size_t count)
    fields = sscanf(buf, "%x %x %x %x %x %x %lx",
            &vendor, &device, &subvendor, &subdevice,
            &class, &class_mask, &driver_data);
    if (fields < 2)
        return -EINVAL;
        
    pci_add_dynid(pdrv, vendor, device, subvendor, subdevice, class, class_mask, driver_data);  
        struct pci_dynid *dynid;
 
        dynid = kzalloc(sizeof(*dynid), GFP_KERNEL);
        dynid->id.vendor = vendor;
        dynid->id.device = device;
        dynid->id.subvendor = subvendor;
        dynid->id.subdevice = subdevice;
        dynid->id.class = class;
        dynid->id.class_mask = class_mask;
        dynid->id.driver_data = driver_data;
 
        spin_lock(&drv->dynids.lock);
        list_add_tail(&dynid->node, &drv->dynids.list);
        spin_unlock(&drv->dynids.lock);
 
        //设置new id时，也会自动匹配设备
        return driver_attach(&drv->driver);
```

向bind文件写入网卡的pci地址时，会调用kernel中的bind_store，将此网卡绑定到此驱动。

向unbind文件写入网卡的pci地址时，会调用kernel中的unbind_store，将此网卡和此驱动解绑。

```cpp
//定义 struct driver_attribute driver_attr_bind，写文件时，调用 bind_store
static DRIVER_ATTR_WO(bind);
//定义 struct driver_attribute driver_attr_unbind，写文件时，调用 unbind_store
static DRIVER_ATTR_WO(unbind);
 
/*
 * Manually attach a device to a driver.
 * Note: the driver must want to bind to the device,
 * it is not possible to override the driver's id table.
 */
static ssize_t bind_store(struct device_driver *drv, const char *buf, size_t count)
    dev = bus_find_device_by_name(bus, NULL, buf);
    if (dev && dev->driver == NULL && driver_match_device(drv, dev)) {
        if (dev->parent)    /* Needed for USB */
            device_lock(dev->parent);
        device_lock(dev);
        err = driver_probe_device(drv, dev);
        device_unlock(dev);
        if (dev->parent)
            device_unlock(dev->parent);
 
        if (err > 0) {
            /* success */
            err = count;
        } else if (err == 0) {
            /* driver didn't accept device */
            err = -ENODEV;
        }
    }
    
/* Manually detach a device from its associated driver. */
static ssize_t unbind_store(struct device_driver *drv, const char *buf, size_t count)
{
    struct bus_type *bus = bus_get(drv->bus);
    struct device *dev;
    int err = -ENODEV;
 
    dev = bus_find_device_by_name(bus, NULL, buf);
    if (dev && dev->driver == drv) {
        if (dev->parent)    /* Needed for USB */
            device_lock(dev->parent);
        device_release_driver(dev);
        if (dev->parent)
            device_unlock(dev->parent);
        err = count;
    }
    put_device(dev);
    bus_put(bus);
    return err;
}
```

**发现pci设备**
系统启动时会扫描所有的pci设备，以他们的**pci地址为名字**创建目录，并在此目录下创建相关的sys文件。并且会遍历所有的pci设备驱动进行匹配。

```
pci_scan_root_bus
    pci_scan_child_bus(b);
        pci_scan_slot
            pci_scan_single_device
                pci_scan_device
                pci_device_add
                    device_add(&dev->dev);
                        bus_add_device(dev);
                            //bus->dev_groups为pci_dev_groups，
                            //会在 /sys/bus/pci/devices/'pci address'/ 目录下创建 vendor, device等目录
                            device_add_groups(dev, bus->dev_groups);
                            //将设备添加到pci总线链表
                            klist_add_tail(&dev->p->knode_bus, &bus->p->klist_devices);
 
    pci_bus_add_devices
        pci_bus_add_device
            pci_create_sysfs_dev_files(dev);
                //如果pci配置空间大于 PCI_CFG_SPACE_SIZE(256字节),则创建 /sys/bus/pci/devices/0000:81:00.0/config文件，
                //大小为 4096 字节
                if (pdev->cfg_size > PCI_CFG_SPACE_SIZE)
                    retval = sysfs_create_bin_file(&pdev->dev.kobj, &pcie_config_attr);
                else //否则config文件大小为 256 字节
                    retval = sysfs_create_bin_file(&pdev->dev.kobj, &pci_config_attr);
 
                //创建 resource 文件，用户态可以使用mmap映射 resource0 实现对网卡寄存器的操作
                pci_create_resource_files(pdev);
                    //创建 /sys/bus/pci/devices/0000:81:00.0/resource0 等文件
                    /* Expose the PCI resources from this device as files */
                    for (i = 0; i < PCI_ROM_RESOURCE; i++) {
                        /* skip empty resources */
                        if (!pci_resource_len(pdev, i))
                            continue;
 
                        retval = pci_create_attr(pdev, i, 0);
                            struct bin_attribute *res_attr;
                            res_attr = kzalloc(sizeof(*res_attr) + name_len, GFP_ATOMIC);
                            sysfs_bin_attr_init(res_attr);
                            if (write_combine) {
                                pdev->res_attr_wc[num] = res_attr;
                                sprintf(res_attr_name, "resource%d_wc", num);
                                res_attr->mmap = pci_mmap_resource_wc;
                            } else {
                                pdev->res_attr[num] = res_attr;
                                sprintf(res_attr_name, "resource%d", num);
                                res_attr->mmap = pci_mmap_resource_uc;
                            }
                            if (pci_resource_flags(pdev, num) & IORESOURCE_IO) {
                                res_attr->read = pci_read_resource_io;
                                res_attr->write = pci_write_resource_io;
                            }
                            res_attr->attr.name = res_attr_name;
                            res_attr->attr.mode = S_IRUSR | S_IWUSR;
                            res_attr->size = pci_resource_len(pdev, num);
                            res_attr->private = &pdev->resource[num];
                            //创建 kernel 文件
                            sysfs_create_bin_file(&pdev->dev.kobj, res_attr);
 
                        /* for prefetchable resources, create a WC mappable file */
                        if (!retval && pdev->resource[i].flags & IORESOURCE_PREFETCH)
                            retval = pci_create_attr(pdev, i, 1);
                    }
 
            //尝试匹配驱动
            device_attach(&dev->dev);
                //遍历所有driver，查看是否有匹配此设备的driver
                bus_for_each_drv(dev->bus, NULL, dev, __device_attach);
                    //判断驱动和设备是否匹配
                    driver_match_device
                        //pci_bus_match
                        drv->bus->match
                            pci_match_device(pci_drv, pci_dev);
                    //如果有匹配的，则调用驱动的probe函数
                    driver_probe_device
                        really_probe(dev, drv);
                            //pci_device_probe
                            dev->bus->probe
                                __pci_device_probe
                                    pci_call_probe
                                        local_pci_probe
                                            pci_drv->probe(pci_dev, ddi->id);
```

向设备的driver_override文件写入驱动名字，表示此设备只能绑定到此驱动。

```cpp
static ssize_t driver_override_store(struct device *dev,
                     struct device_attribute *attr,
                     const char *buf, size_t count)
    struct pci_dev *pdev = to_pci_dev(dev);
    driver_override = kstrndup(buf, count, GFP_KERNEL);
    pdev->driver_override = driver_override;
```

**如何匹配？**
前面多次提到设备和驱动进行匹配，究竟如何匹配呢？

先看一下用来表示一个pci设备的结构体pci_dev，其中如下几个成员变量表示此pci设备的类型，一般vendor和device就足够，vendor表示此设备是哪个厂商的，device表示此设备的类型。

```
struct pci_dev {
    ...
    unsigned short  vendor;
    unsigned short  device;
    unsigned short  subsystem_vendor;
    unsigned short  subsystem_device;
    unsigned int    class;      /* 3 bytes: (base,sub,prog-if) */
    ...
}
```

再看一下用来表示设备驱动的pci_driver，其中id_table和dynids用来保存此驱动支持的设备类型，前者是静态值，后者可以通过驱动目录下的new_id动态添加。设备类型使用pci_device_id结构体来表示，其成员变量也是vendor,device等信息，和pci_dev中的信息是一样的，所以可以使用这几个字段进行匹配。

```rust
struct pci_device_id {
    __u32 vendor, device;       /* Vendor and device ID or PCI_ANY_ID*/
    __u32 subvendor, subdevice; /* Subsystem ID's or PCI_ANY_ID */
    __u32 class, class_mask;    /* (class,subclass,prog-if) triplet */
    kernel_ulong_t driver_data; /* Data private to the driver */
};
 
struct pci_driver {
    struct pci_device_id *id_table
    struct pci_dynids dynids;
    ...
}
```

最终使用函数pci_match_device进行驱动和设备的匹配。

```rust
static const struct pci_device_id pci_device_id_any = {
    .vendor = PCI_ANY_ID,
    .device = PCI_ANY_ID,
    .subvendor = PCI_ANY_ID,
    .subdevice = PCI_ANY_ID,
};
 
static const struct pci_device_id *pci_match_device(struct pci_driver *drv, struct pci_dev *dev)
    //如果设备设置了 driver_override，则只能绑定到driver_override指定的驱动上。
    //如果不是此驱动直接返回NULL
    /* When driver_override is set, only bind to the matching driver */
    if (dev->driver_override && strcmp(dev->driver_override, drv->name))
        return NULL;
 
    //首先查找驱动的动态链表和设备进行匹配
    /* Look at the dynamic ids first, before the static ones */
    spin_lock(&drv->dynids.lock);
    list_for_each_entry(dynid, &drv->dynids.list, node) {
        if (pci_match_one_device(&dynid->id, dev)) {
            found_id = &dynid->id;
            break;
        }
    }
    spin_unlock(&drv->dynids.lock);
 
    //如果没匹配到，则查找驱动的静态table
    if (!found_id)
        found_id = pci_match_id(drv->id_table, dev);
            while (ids->vendor || ids->subvendor || ids->class_mask) {
                if (pci_match_one_device(ids, dev))
                    return ids;
                ids++;
            }
 
    //如果仍然没匹配到，但是指定了驱动，则强制认为匹配成功，返回 pci_device_id_any
    /* driver_override will always match, send a dummy id */
    if (!found_id && dev->driver_override)
        found_id = &pci_device_id_any;
 
    return found_id;
 
//具体的匹配规则
static inline const struct pci_device_id *
pci_match_one_device(const struct pci_device_id *id, const struct pci_dev *dev)
{
    if ((id->vendor == PCI_ANY_ID || id->vendor == dev->vendor) &&
        (id->device == PCI_ANY_ID || id->device == dev->device) &&
        (id->subvendor == PCI_ANY_ID || id->subvendor == dev->subsystem_vendor) &&
        (id->subdevice == PCI_ANY_ID || id->subdevice == dev->subsystem_device) &&
        !((id->class ^ dev->class) & id->class_mask))
        return id;
    return NULL;
}
```

# 绑定到 igb_uio 驱动

网卡如何绑定到igb_uio驱动呢？这里拿DPDK提供的脚步文件dpdk-devbind.py中的函数bind_one进行分析。

```
def bind_one(dev_id, driver, force):
    '''Bind the device given by "dev_id" to the driver "driver". If the device
    is already bound to a different driver, it will be unbound first'''
    dev = devices[dev_id]
    saved_driver = None  # used to rollback any unbind in case of failure
 
    //如果网卡已经绑定到某个驱动，则判断是否是要绑定的驱动，如果是则返回，
    //如果不是，则解绑之前的驱动。unbind_one只要向驱动的unbind写入此网卡的pci地址即可解绑。
    # unbind any existing drivers we don't want
    if has_driver(dev_id):
        if dev["Driver_str"] == driver:
            print("%s already bound to driver %s, skipping\n"
                  % (dev_id, driver))
            return
        else:
            saved_driver = dev["Driver_str"]
            unbind_one(dev_id, force)
            dev["Driver_str"] = ""  # clear driver string
 
    //绑定方法根据kernel版本有不同的绑定方法。
    //对于kernel版本大于等于3.15的，首先将驱动名字写入到网卡的文件 driver_override来指定此驱动。
    //而小于3.15的，需要将网卡的vendor和device id写入驱动的new_id文件。
    //为什么大于等于3.15的不使用new_id呢？这是因为高版本的new_id不只是将设备类型添加到驱动的
    //动态链表，也会遍历所有的设备将此类型的设备全部绑定到此驱动。如果你只想绑定一个网卡，
    //结果把同类型的网卡都绑定了，岂不是很尴尬。
    # For kernels >= 3.15 driver_override can be used to specify the driver
    # for a device rather than relying on the driver to provide a positive
    # match of the device.  The existing process of looking up
    # the vendor and device ID, adding them to the driver new_id,
    # will erroneously bind other devices too which has the additional burden
    # of unbinding those devices
    if driver in dpdk_drivers:
        filename = "/sys/bus/pci/devices/%s/driver_override" % dev_id
        if os.path.exists(filename):
            try:
                f = open(filename, "w")
            except:
                print("Error: bind failed for %s - Cannot open %s"
                      % (dev_id, filename))
                return
            try:
                f.write("%s" % driver)
                f.close()
            except:
                print("Error: bind failed for %s - Cannot write driver %s to "
                      "PCI ID " % (dev_id, driver))
                return
        # For kernels < 3.15 use new_id to add PCI id's to the driver
        else:
            filename = "/sys/bus/pci/drivers/%s/new_id" % driver
            try:
                f = open(filename, "w")
            except:
                print("Error: bind failed for %s - Cannot open %s"
                      % (dev_id, filename))
                return
            try:
                # Convert Device and Vendor Id to int to write to new_id
                f.write("%04x %04x" % (int(dev["Vendor"],16),
                        int(dev["Device"], 16)))
                f.close()
            except:
                print("Error: bind failed for %s - Cannot write new PCI ID to "
                      "driver %s" % (dev_id, driver))
                return
 
    //第二步是将网卡的pci地址写入驱动的文件 /sys/bus/pci/drivers/%s/bind，这样就能将
    //网卡和驱动绑定到一起。
    # do the bind by writing to /sys
    filename = "/sys/bus/pci/drivers/%s/bind" % driver
    try:
        f = open(filename, "a")
    except:
        print("Error: bind failed for %s - Cannot open %s"
              % (dev_id, filename))
        if saved_driver is not None:  # restore any previous driver
            bind_one(dev_id, saved_driver, force)
        return
    try:
        f.write(dev_id)
        f.close()
    except:
        # for some reason, closing dev_id after adding a new PCI ID to new_id
        # results in IOError. however, if the device was successfully bound,
        # we don't care for any errors and can safely ignore IOError
        tmp = get_pci_device_details(dev_id, True)
        if "Driver_str" in tmp and tmp["Driver_str"] == driver:
            return
        print("Error: bind failed for %s - Cannot bind to driver %s"
              % (dev_id, driver))
        if saved_driver is not None:  # restore any previous driver
            bind_one(dev_id, saved_driver, force)
        return
 
    //对于kernel版本大于等于3.15的，还要将文件 driver_override 清空，以便绑定到其他驱动。
    # For kernels > 3.15 driver_override is used to bind a device to a driver.
    # Before unbinding it, overwrite driver_override with empty string so that
    # the device can be bound to any other driver
    filename = "/sys/bus/pci/devices/%s/driver_override" % dev_id
    if os.path.exists(filename):
        try:
            f = open(filename, "w")
        except:
            print("Error: unbind failed for %s - Cannot open %s"
                  % (dev_id, filename))
            sys.exit(1)
        try:
            f.write("\00")
            f.close()
        except:
            print("Error: unbind failed for %s - Cannot open %s"
                  % (dev_id, filename))
            sys.exit(1)
```

igb_uio驱动的id_table为空，则在加载此驱动时，是不会匹配到任何设备的。

```objectivec
static struct pci_driver igbuio_pci_driver = {
    .name = "igb_uio",
    .id_table = NULL,  //DPDK 用到的 igb_uio, vfio-pci等驱动的id_table默认为空
    .probe = igbuio_pci_probe,
    .remove = igbuio_pci_remove,
};
```

经过上面的分析，有三种方法可以将网卡绑定到驱动igb_uio

```groovy
a. 如果kernel版本大于等于3.15，先向网卡的文件 /sys/bus/pci/devices/'pci address'/driver_override 写入驱动名字igb_uio，再向驱动igb_uio的文件 /sys/bus/pci/drivers/igb_uio/bind写入网卡的pci地址即可。
b. 如果kernel版本大于等于3.15，向驱动igb_uio的文件 /sys/bus/pci/drivers/igb_uio/new_id写入网卡的vendor和device id，则会自动将所有此类型并且没有绑定到任何驱动的网卡绑定到igb_uio。
c. 如果kernel版本小于3.15，先向驱动igb_uio的文件 /sys/bus/pci/drivers/igb_uio/new_id写入网卡的vendor和device id，再向驱动igb_uio的文件 /sys/bus/pci/drivers/igb_uio/bind写入网卡的pci地址即可。注意低版本的kernel，在向new_id写入值时，只会将设备类型添加到此驱动的动态链表，而不会自动探测设备。
```

**igb_uio probe**
经过前面的分析网卡绑定到了igb_uio驱动后，会调用驱动的probe函数igbuio_pci_probe，主要做了如下几个事情：
a. 调用pci_enable_device使能pci设备
b. 设置DMA mask
c. 填充struct uio_info信息，注册uio设备
d. 注册中断处理函数

```
static int
igbuio_pci_probe(struct pci_dev *dev, const struct pci_device_id *id)
    struct rte_uio_pci_dev *udev;
    dma_addr_t map_dma_addr;
    void *map_addr;
 
    udev = kzalloc(sizeof(struct rte_uio_pci_dev), GFP_KERNEL);
    
    //使能pci设备
    /*
     * enable device: ask low-level code to enable I/O and
     * memory
     */
    pci_enable_device(dev);
 
    /* enable bus mastering on the device */
    pci_set_master(dev);
    
    //将设备的memory类型BAR信息保存到 struct uio_info->mem中，
    //将设备的io类型BAR信息保存到 struct uio_info->port中
    /* remap IO memory */
    igbuio_setup_bars(dev, &udev->info);
    
    /* set 64-bit DMA mask */
    pci_set_dma_mask(dev,  DMA_BIT_MASK(64));
    pci_set_consistent_dma_mask(dev, DMA_BIT_MASK(64));
 
    //填充 struct uio_info 其他字段
    /* fill uio infos */
    udev->info.name = "igb_uio";
    udev->info.version = "0.1";
    udev->info.irqcontrol = igbuio_pci_irqcontrol;
    udev->info.open = igbuio_pci_open;
    udev->info.release = igbuio_pci_release;
    udev->info.priv = udev;
    udev->pdev = dev;
    
    //创建 /sys/bus/pci/devices/'pci address'/max_vf 文件，
    //写此文件用来生成 VF，这说明即使网卡绑定到igb_uio口，仍然可以
    //生成 VF。
    sysfs_create_group(&dev->dev.kobj, &dev_attr_grp);
    
    //注册uio，会生成 /dev/uiox 字符设备文件，
    //同时生成目录 /sys/bus/pci/devices/'pci address'/uio/uiox
    /* register uio driver */
    uio_register_device(&dev->dev, &udev->info);
 
    //保存 struct rte_uio_pci_dev 到 dev->driver_data
    pci_set_drvdata(dev, udev);
        dev_set_drvdata(&pdev->dev, data);
            dev->driver_data = data;
```

宏uio_register_device用来注册uio设备。

```rust
/* use a define to avoid include chaining to get THIS_MODULE */
#define uio_register_device(parent, info) \
    __uio_register_device(THIS_MODULE, parent, info)
 
int __uio_register_device(struct module *owner,
              struct device *parent,
              struct uio_info *info)
    //根据 uio_info 生成 uio_device
    struct uio_device *idev;
    idev = devm_kzalloc(parent, sizeof(*idev), GFP_KERNEL);
 
    idev->owner = owner;
    idev->info = info;
    init_waitqueue_head(&idev->wait);
    atomic_set(&idev->event, 0);
 
    //分配最小未使用的id，保存到 idev->minor
    uio_get_minor(idev);
    //创建字符设备 /dev/uiox
    idev->dev = device_create(&uio_class, parent, MKDEV(uio_major, idev->minor), idev, "uio%d", idev->minor);
    
    //在 /sys/class/uio/uiox/下创建maps目录，maps目录下根据 struct uio_info->mem和port信息
    //分别生成 mapx 和 portx 等目录，这些目录下又存放对应类型的信息，比如起始地址，name，offset和size。
    //用户态可以通过mmap mapx下的文件来操作网卡寄存器。
    //但是DPDK没有使用此方法，而是直接mmap /sys/bus/pci/devices/'pci address'/resource0 文件实现。
    uio_dev_add_attributes(idev);
    info->uio_dev = idev;
    
    //注册中断。但是在新版本的DPDK中，注册uio时没有分配info->irq来注册中断，
    //而是在用户态 open /dev/uiox 时，在函数 igbuio_pci_open 中注册中断。
    if (info->irq && (info->irq != UIO_IRQ_CUSTOM)) {
        devm_request_irq(idev->dev, info->irq, uio_interrupt, info->irq_flags, info->name, idev);
    }
```

简单总结一下，igb_uio是DPDK使用网卡的一个通用驱动，不只intel网卡可以用，其他厂商的网卡也可以用(有一个例外，mellanox的网卡不用绑定到igb_uio就能被使用DPDK)，因为它只使能了pci设备，注册uio，和注册中断处理函数，这些工作是不区分网卡类型的。
加载igb_uio时，不会自动探测pci设备，而是需要写sys文件将设备绑定到igb_uio。

igb_uio依赖uio驱动，注册uio设备后，会生成/dev/uiox，和网卡一一对应，用户态可以poll /dev/uiox监听中断是否到来。
同时uio设备还会将网卡的BAR地址通过sys文件系统暴露出去，用户态可以mmap sys文件后操作网卡寄存器。但是DPDK没有采用这种方式，而是直接mmap网卡自身暴露出去的sys文件 /sys/bus/pci/devices/'pci address'/resource0。


作者：[分享放大价值](https://www.jianshu.com/u/d7eb9e00077c)  转自：https://www.jianshu.com/p/e90405e712ed
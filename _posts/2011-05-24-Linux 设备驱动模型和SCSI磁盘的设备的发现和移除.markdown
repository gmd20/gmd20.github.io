```text
“The Linux Kernel Driver Model”
==============================
 
  由 bus  、driver 、device 、class 等部分组成，是linux为各种各样的总线和设备搞的一个统一模型。实现了支持现代计算机的设备热拔插模型（"plug and play", power management, and hot plug），满足英特尔和微软倡导的ACPI规范的各种各样的设备应该都可以通过这个模型表现出来。 这个模型的定义了一系列的回调函数，比如说新设备发现的回调、总线关闭回调等。
 
 
总线（ bus） 下面有很多属于这个总线的设备，以及这个总线上的驱动(driver). 但总线发现一个新设备之后，就会遍历所有这个总线上的驱动列表，看看那个驱动支持这个设备的，然后把 驱动和设备关联起来。 




linux 内核的sysfs文件系统 
========================

表示出来了各个bus和device driver等之间的关系，而且提供各个接口让驱动或者设备显示自己特有的功能到这个文件里面。
BUS_ATTR 
DRIVER_ATTR
DEVICE_ATTR 
等宏等价于定义
static bus_attribute
struct driver_attribute
这些属性又可以用来在sysfs的目录里面创建自己的自定义文件：
int bus_create_file(struct bus_type *, struct bus_attribute *);
void bus_remove_file(strstruct device_attribute 
int device_create_file(struct device *device, struct device_attribute * entry);
void device_remove_file(struct device * dev, struct device_attribute * attr);
 uct bus_type *, struct bus_attribute *);


结构例子
        /sys/bus/pci/
        |-- devices
        |   |-- 00:00.0 -> ../../../root/pci0/00:00.0
        |   |-- 00:01.0 -> ../../../root/pci0/00:01.0
        |   `-- 00:02.0 -> ../../../root/pci0/00:02.0
        `-- drivers



bus 总线
=======

http://lxr.linux.no/#linux+v2.6.39/include/linux/device.h#L50
  struct bus_type {
          const char              *name;
          struct bus_attribute    *bus_attrs;
          struct device_attribute *dev_attrs;
          struct driver_attribute *drv_attrs;
  
          int (*match)(struct device *dev, struct device_driver *drv);  ///这个回调让bus可以判断驱动是不是完全支持某种device id的设备， 比如某个id的设备一定和某些驱动搭配才能发回最佳性能等等。
          int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
          int (*probe)(struct device *dev);
          int (*remove)(struct device *dev);
          void (*shutdown)(struct device *dev);
  
          int (*suspend)(struct device *dev, pm_message_t state);
          int (*resume)(struct device *dev);
  
          const struct dev_pm_ops *pm;
  
          struct subsys_private *p;
  };

可以使用bus_register 函数向系统注册一个新的总线类型。
int bus_register(struct bus_type * bus);
 
 

内核提供两个帮助函数可以遍历一个总线上的所有驱动和设备。
int bus_for_each_dev(struct bus_type * bus, struct device * start, void * data,
                     int (*fn)(struct device *, void *));

int bus_for_each_drv(struct bus_type * bus, struct device_driver * start, 
                    void * data, int (*fn)(struct device_driver *, void *));
 
 
 
驱动 Device Drivers
==================
http://lxr.linux.no/#linux+v2.6.39/include/linux/device.h#L122
 struct device_driver {
         const char              *name;
         struct bus_type         *bus;                //////////////什么类型的总线的驱动
 
         struct module           *owner;
         const char              *mod_name;      /* used for built-in modules */
 
         bool suppress_bind_attrs;       /* disables bind/unbind via sysfs */
 
         const struct of_device_id       *of_match_table;
 
         int (*probe) (struct device *dev);
         int (*remove) (struct device *dev);
         void (*shutdown) (struct device *dev);
         int (*suspend) (struct device *dev, pm_message_t state);
         int (*resume) (struct device *dev);
         const struct attribute_group **groups;
 
         const struct dev_pm_ops *pm;
 
         struct driver_private *p;
 };


向系统注册一个驱动类型。
http://lxr.linux.no/#linux+v2.6.39/drivers/base/driver.c#L222
int driver_register(struct device_driver * drv);
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

        BUG_ON(!drv->bus->p);

        if ((drv->bus->probe && drv->probe) ||
            (drv->bus->remove && drv->remove) ||
            (drv->bus->shutdown && drv->shutdown))
                printk(KERN_WARNING "Driver '%s' needs updating - please use "
                        "bus_type methods\n", drv->name);

        other = driver_find(drv->name, drv->bus);
        if (other) {
                put_driver(other);
                printk(KERN_ERR "Error: Driver '%s' is already registered, "
                        "aborting...\n", drv->name);
                return -EBUSY;
        }

        ret = bus_add_driver(drv);
        if (ret)
                return ret;
        ret = driver_add_groups(drv, drv->groups);
        if (ret)
                bus_remove_driver(drv);
        return ret;
}
EXPORT_SYMBOL_GPL(driver_register); 

一般驱动注册的时候还需要初始化自己的总线相关的一些特殊的配置的，比如 pci_driver_register这些函数应都是包含自己特殊一些代码在里面的。


对应的有void driver_unregister(struct device_driver *drv)


遍历所有绑定到这个驱动上面的设备的帮助函数
int driver_for_each_dev(struct device_driver * drv, void * data, 
                       int (*callback)(struct device * dev, void * data));

 
 
 
 bus发现一个设备之后，会调用自己所有注册的driver的probe回调函数，来检查是不是某个驱动所支持的设备，以把设备和驱动关联起来。 内核文档的说明：
 ------------------------------
int     (*probe)        (struct device * dev);

The probe() entry is called in task context, with the bus's rwsem locked
 and the driver partially bound to the device.  Drivers commonly use
 container_of() to convert "dev" to a bus-specific type, both in probe()
 and other routines.  That type often provides device resource data, such
 as pci_dev.resource[] or platform_device.resources, which is used in
 addition to dev->platform_data to initialize the driver.
 
 This callback holds the driver-specific logic to bind the driver to a
 given device.  That includes verifying that the device is present, that
 it's a version the driver can handle, that driver data structures can
 be allocated and initialized, and that any hardware can be initialized.
 Drivers often store a pointer to their state with dev_set_drvdata().
 When the driver has successfully bound itself to that device, then probe()
 returns zero and the driver model code will finish its part of binding
 the driver to that device.
 
 A driver's probe() may return a negative errno value to indicate that
 the driver did not bind to this device, in which case it should have
 released all resources it allocated.
 
         int     (*remove)       (struct device * dev);
 
 remove is called to unbind a driver from a device. This may be
 called if a device is physically removed from the system, if the
 driver module is being unloaded, during a reboot sequence, or
 in other cases.
 
 It is up to the driver to determine if the device is present or
 not. It should free any resources allocated specifically for the
 device; i.e. anything in the device's driver_data field. 
 
 If the device is still present, it should quiesce the device and place
 it into a supported low-power state.
 -----------------------------------
  
 
 
 
 
 
 
 
 
设备 device
===========
struct device {
        struct device           *parent;

        struct device_private   *p;

        struct kobject kobj;
        const char              *init_name; /* initial name of the device */
        struct device_type      *type;

        struct mutex            mutex;  /* mutex to synchronize calls to
                                         * its driver.
                                         */

        struct bus_type *bus;           /* type of bus device is on */    ///////////////什么总线类型的设备
        struct device_driver *driver;   /* which driver has allocated this  /////////////////////设备关联的驱动
                                           device */
        void            *platform_data; /* Platform specific data, device
                                           core doesn't touch it */
        struct dev_pm_info      power;
        struct dev_power_domain *pwr_domain;

#ifdef CONFIG_NUMA
        int             numa_node;      /* NUMA node this device is close to */
#endif
        u64             *dma_mask;      /* dma mask (if dma'able device) */
        u64             coherent_dma_mask;/* Like dma_mask, but for
                                             alloc_coherent mappings as
                                             not all hardware supports
                                             64 bit addresses for consistent
                                             allocations such descriptors. */

        struct device_dma_parameters *dma_parms;

        struct list_head        dma_pools;      /* dma pools (if dma'ble) */

        struct dma_coherent_mem *dma_mem; /* internal for coherent mem
                                             override */
        /* arch specific additions */
        struct dev_archdata     archdata;

        struct device_node      *of_node; /* associated device tree node */

        dev_t                   devt;   /* dev_t, creates the sysfs "dev" */

        spinlock_t              devres_lock;
        struct list_head        devres_head;

        struct klist_node       knode_class;
        struct class            *class;
        const struct attribute_group **groups;  /* optional groups */

        void    (*release)(struct device *dev);
};


  
  
  
  
  当总线发现一个新设备之后就会调用
  int device_register(struct device * dev);
  函数来往系统里面注册这个设备。
  
总线会初始化设备的

    - parent
    - name
    - bus_id
    - bus
    
等内容  

设备的引用计数入为0就会被从核心系统里面移除
这两个函数用来改变计数值的
struct device * get_device(struct device * dev);
void put_device(struct device * dev);
驱动可以使用这两个函数已锁定某个设备
void lock_device(struct device * dev);
void unlock_device(struct device * dev);


  
  
  
  
  
  
  
驱动和设备的绑定Driver Binding
=============================


Bus结构包含系统属于这个总线的所有设别的列表。

device_register
当新设备被创建时，就变量整个bus的驱动列表，以找到支持这个设备的驱动。


int match(struct device * dev, struct device_driver * drv);
如果找到匹配就设置device结构的 driver项，并且调用dirve的probe回调函数，驱动就可以检查是不是自己支持的硬件设备了，确保硬件是在工作状态等检查。

驱动结构也维护着一个自己驱动所支持的设备的列表。

如果设备的引用计数为0被移除的时候，驱动remove回调函数就会被调用。如果驱动卸载，它所有的设备也会遍历调用 remove回调。



内核对应源码 http://lxr.linux.no/#linux+v2.6.39/drivers/base/dd.c


 /**
 * device_bind_driver - bind a driver to one device.
 * @dev: device.
 *
 * Allow manual attachment of a driver to a device.
 * Caller must have already set @dev->driver.
 *
 * Note that this does not modify the bus reference count
 * nor take the bus's rwsem. Please verify those are accounted
 * for before calling this. (It is ok to call with no other effort
 * from a driver's probe() method.)
 *
 * This function must be called with the device lock held.
 */
int device_bind_driver(struct device *dev)
{
        int ret;

        ret = driver_sysfs_add(dev);
        if (!ret)
                driver_bound(dev);
        return ret;
}
EXPORT_SYMBOL_GPL(device_bind_driver);


static int really_probe(struct device *dev, struct device_driver *drv)
{
        int ret = 0;

        atomic_inc(&probe_count);
        pr_debug("bus: '%s': %s: probing driver %s with device %s\n",
                 drv->bus->name, __func__, drv->name, dev_name(dev));
        WARN_ON(!list_empty(&dev->devres_head));

        dev->driver = drv;
        if (driver_sysfs_add(dev)) {
                printk(KERN_ERR "%s: driver_sysfs_add(%s) failed\n",
                        __func__, dev_name(dev));
                goto probe_failed;
        }

        if (dev->bus->probe) {
                ret = dev->bus->probe(dev);
                if (ret)
                        goto probe_failed;
        } else if (drv->probe) {
                ret = drv->probe(dev);        ///调用驱动probe回调函数，让驱动可以判断是不是自己的备和设备是否可以工作状态
                if (ret)
                        goto probe_failed;
        }

        driver_bound(dev);
。。。。。。。。。。。。。。。
}




/**
 * driver_probe_device - attempt to bind device & driver together
 * @drv: driver to bind a device to
 * @dev: device to try to bind to the driver
 *
 * This function returns -ENODEV if the device is not registered,
 * 1 if the device is bound successfully and 0 otherwise.
 *
 * This function must be called with @dev lock held.  When called for a
 * USB interface, @dev->parent lock must be held as well.
 */
int driver_probe_device(struct device_driver *drv, struct device *dev)
{
        int ret = 0;

        if (!device_is_registered(dev))
                return -ENODEV;

        pr_debug("bus: '%s': %s: matched device %s with driver %s\n",
                 drv->bus->name, __func__, dev_name(dev), drv->name);

        pm_runtime_get_noresume(dev);
        pm_runtime_barrier(dev);
        ret = really_probe(dev, drv); >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
        pm_runtime_put_sync(dev);

        return ret;
}


static int __device_attach(struct device_driver *drv, void *data)
{
        struct device *dev = data;

        if (!driver_match_device(drv, dev))
                return 0;

        return driver_probe_device(drv, dev); >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
}

/**
 * device_attach - try to attach device to a driver.
 * @dev: device.
 *
 * Walk the list of drivers that the bus has and call
 * driver_probe_device() for each pair. If a compatible
 * pair is found, break out and return.
 *
 * Returns 1 if the device was bound to a driver;
 * 0 if no matching driver was found;
 * -ENODEV if the device is not registered.
 *
 * When called for a USB interface, @dev->parent lock must be held.
 */
int device_attach(struct device *dev)
{
        int ret = 0;

        device_lock(dev);
        if (dev->driver) {
                ret = device_bind_driver(dev);
                if (ret == 0)
                        ret = 1;
                else {
                        dev->driver = NULL;
                        ret = 0;
                }
        } else {
                pm_runtime_get_noresume(dev);
                ret = bus_for_each_drv(dev->bus, NULL, dev, __device_attach);  ///遍历总线上的所有驱动，看看是那个驱动支持这个设备
                pm_runtime_put_sync(dev);
        }
        device_unlock(dev);
        return ret;
}
EXPORT_SYMBOL_GPL(device_attach);

static int __driver_attach(struct device *dev, void *data)
{
        struct device_driver *drv = data;

        /*
         * Lock device and try to bind to it. We drop the error
         * here and always return 0, because we need to keep trying
         * to bind to devices and some drivers will return an error
         * simply if it didn't support the device.
         *
         * driver_probe_device() will spit a warning if there
         * is an error.
         */

        if (!driver_match_device(drv, dev))
                return 0;

        if (dev->parent)        /* Needed for USB */
                device_lock(dev->parent);
        device_lock(dev);
        if (!dev->driver)
                driver_probe_device(drv, dev);    //////////////////////////////////
        device_unlock(dev);
        if (dev->parent)
                device_unlock(dev->parent);

        return 0;
}

/**
 * driver_attach - try to bind driver to devices.
 * @drv: driver.
 *
 * Walk the list of devices that the bus has on it and try to
 * match the driver with each one.  If driver_probe_device()
 * returns 0 and the @dev->driver is set, we've found a
 * compatible pair.
 */
int driver_attach(struct device_driver *drv)
{
        return bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);  ////遍历整个总线上的设备，为每个设备调用__driver_attach函数
}
EXPORT_SYMBOL_GPL(driver_attach);




      http://lxr.linux.no/#linux+v2.6.39/drivers/base/bus.c#L493
/**
 * bus_probe_device - probe drivers for a new device
 * @dev: device to probe
 *
 * - Automatically probe for a driver if the bus allows it.
 */
void bus_probe_device(struct device *dev)
{
        struct bus_type *bus = dev->bus;
        int ret;

        if (bus && bus->p->drivers_autoprobe) {
                ret = device_attach(dev);           ///////////bus部分的代码
                WARN_ON(ret < 0);
        }
}


 
          http://lxr.linux.no/#linux+v2.6.39/drivers/base/core.c  部分代码
  /**
  * device_add - add device to device hierarchy.
  * @dev: device.
  *
  * This is part 2 of device_register(), though may be called
  * separately _iff_ device_initialize() has been called separately.
  *
  * This adds @dev to the kobject hierarchy via kobject_add(), adds it
  * to the global and sibling lists for the device, then
  * adds it to the other relevant subsystems of the driver model.
  *
  * NOTE: _Never_ directly free @dev after calling this function, even
  * if it returned an error! Always use put_device() to give up your
  * reference instead.
  */
 int device_add(struct device *dev)
 {
         struct device *parent = NULL;
         struct class_interface *class_intf;
         int error = -EINVAL;
 
         dev = get_device(dev);
 ......
 
         error = device_add_class_symlinks(dev);
         if (error)
                 goto SymlinkError;
         error = device_add_attrs(dev);
         if (error)
                 goto AttrsError;
         error = bus_add_device(dev);        >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>.
         if (error)
                 goto BusError;
         error = dpm_sysfs_add(dev);
         if (error)
                 goto DPMError;
         device_pm_add(dev);
 
         /* Notify clients of device addition.  This call must come
          * after dpm_sysf_add() and before kobject_uevent().
          */
         if (dev->bus)
                 blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
                                              BUS_NOTIFY_ADD_DEVICE, dev);
 
         kobject_uevent(&dev->kobj, KOBJ_ADD);
         bus_probe_device(dev);
 
 
 
 
/**
 * device_register - register a device with the system.
 * @dev: pointer to the device structure
 *
 * This happens in two clean steps - initialize the device
 * and add it to the system. The two steps can be called
 * separately, but this is the easiest and most common.
 * I.e. you should only call the two helpers separately if
 * have a clearly defined need to use and refcount the device
 * before it is added to the hierarchy.
 *
 * NOTE: _Never_ directly free @dev after calling this function, even
 * if it returned an error! Always use put_device() to give up the
 * reference initialized in this function instead.
 */
int device_register(struct device *dev)
{
        device_initialize(dev);
        return device_add(dev);  >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
}
 
 
 
 
 
 整个添加设备的调用树应该是这样子的
 device_register
    device_add
       bus_add_device         
           bus_probe_device
              device_attach
                 bus_for_each_drv(dev->bus, NULL, dev, __device_attach)
                    driver_probe_device
                       really_probe
                           driver->probe  注册的回调函数
                           
                           
====================
看看scsi的新设备发现
====================
   

http://lxr.linux.no/#linux+v2.6.39/drivers/scsi/scsi_scan.c




int scsi_add_device(struct Scsi_Host *host, uint channel,
                    uint target, uint lun)
{
        struct scsi_device *sdev =
                __scsi_add_device(host, channel, target, lun, NULL);  >>>>>>>>>>>>>>>>>>
        if (IS_ERR(sdev))
                return PTR_ERR(sdev);

        scsi_device_put(sdev);
        return 0;
}



struct scsi_device *__scsi_add_device(struct Scsi_Host *shost, uint channel,
                                      uint id, uint lun, void *hostdata)
{
        struct scsi_device *sdev = ERR_PTR(-ENODEV); ///////////////////////
        struct device *parent = &shost->shost_gendev;
        struct scsi_target *starget;

        if (strncmp(scsi_scan_type, "none", 4) == 0)
                return ERR_PTR(-ENODEV);

        starget = scsi_alloc_target(parent, channel, id);
        if (!starget)
                return ERR_PTR(-ENOMEM);
        scsi_autopm_get_target(starget);

        mutex_lock(&shost->scan_mutex);
        if (!shost->async_scan)
                scsi_complete_async_scans();

        if (scsi_host_scan_allowed(shost) && scsi_autopm_get_host(shost) == 0) {
                scsi_probe_and_add_lun(starget, lun, NULL, &sdev, 1, hostdata);  >>>>>>>>>>>>>>>>>>>>>>
                scsi_autopm_put_host(shost);
        }
        mutex_unlock(&shost->scan_mutex);
        scsi_autopm_put_target(starget);
        scsi_target_reap(starget);
        put_device(&starget->dev);

        return sdev;
}
EXPORT_SYMBOL(__scsi_add_device);







/**
 * scsi_probe_and_add_lun - probe a LUN, if a LUN is found add it
 * @starget:    pointer to target device structure
 * @lun:        LUN of target device
 * @bflagsp:    store bflags here if not NULL
 * @sdevp:      probe the LUN corresponding to this scsi_device
 * @rescan:     if nonzero skip some code only needed on first scan
 * @hostdata:   passed to scsi_alloc_sdev()
 *
 * Description:
 *     Call scsi_probe_lun, if a LUN with an attached device is found,
 *     allocate and set it up by calling scsi_add_lun.
 *
 * Return:
 *     SCSI_SCAN_NO_RESPONSE: could not allocate or setup a scsi_device
 *     SCSI_SCAN_TARGET_PRESENT: target responded, but no device is

 *         attached at the LUN
 *     SCSI_SCAN_LUN_PRESENT: a new scsi_device was allocated and initialized
 **/
static int scsi_probe_and_add_lun(struct scsi_target *starget,
                                  uint lun, int *bflagsp,
                                  struct scsi_device **sdevp, int rescan,
                                  void *hostdata)
{
        struct scsi_device *sdev;
        unsigned char *result;
        int bflags, res = SCSI_SCAN_NO_RESPONSE, result_len = 256;
        struct Scsi_Host *shost = dev_to_shost(starget->dev.parent);

        /*
         * The rescan flag is used as an optimization, the first scan of a
         * host adapter calls into here with rescan == 0.
         */
        sdev = scsi_device_lookup_by_target(starget, lun);  
        if (sdev) {
                if (rescan || !scsi_device_created(sdev)) {
                        SCSI_LOG_SCAN_BUS(3, printk(KERN_INFO
                                "scsi scan: device exists on %s\n",
                                dev_name(&sdev->sdev_gendev)));
                        if (sdevp)
                                *sdevp = sdev;
                        else
                                scsi_device_put(sdev);

                        if (bflagsp)
                                *bflagsp = scsi_get_device_flags(sdev,
                                                                 sdev->vendor,
                                                                 sdev->model);
                        return SCSI_SCAN_LUN_PRESENT;
                }
                scsi_device_put(sdev);
        } else
                sdev = scsi_alloc_sdev(starget, lun, hostdata);  ////////////新建一个scsi_device
        if (!sdev)
                goto out;

        result = kmalloc(result_len, GFP_ATOMIC |
                        ((shost->unchecked_isa_dma) ? __GFP_DMA : 0));
        if (!result)
                goto out_free_sdev;

        if (scsi_probe_lun(sdev, result, result_len, &bflags))  ////往设备上发送inquery命令来检测指定的device是不是可以正常工作的
                goto out_free_result;

。。。。。。。。。。。。。。。。。。。。。。。。。。。。。

        res = scsi_add_lun(sdev, result, &bflags, shost->async_scan);  ///////如果前面检查过这个设备确实可以使用的话，就调用这个完整的初始化scsi-device设备

。。。。。。。。。。。。。。
}









/**
 * scsi_alloc_sdev - allocate and setup a scsi_Device
 * @starget: which target to allocate a &scsi_device for
 * @lun: which lun
 * @hostdata: usually NULL and set by ->slave_alloc instead
 *
 * Description:
 *     Allocate, initialize for io, and return a pointer to a scsi_Device.
 *     Stores the @shost, @channel, @id, and @lun in the scsi_Device, and
 *     adds scsi_Device to the appropriate list.
 *
 * Return value:
 *     scsi_Device pointer, or NULL on failure.
 **/
static struct scsi_device *scsi_alloc_sdev(struct scsi_target *starget,
                                           unsigned int lun, void *hostdata)
{
        struct scsi_device *sdev;
        int display_failure_msg = 1, ret;
        struct Scsi_Host *shost = dev_to_shost(starget->dev.parent);
        extern void scsi_evt_thread(struct work_struct *work);
        extern void scsi_requeue_run_queue(struct work_struct *work);

        sdev = kzalloc(sizeof(*sdev) + shost->transportt->device_size,
                       GFP_ATOMIC);   /////////////////////新分配一个内存空间来放这个scsi_device结构，注意一般不通的bus实现都是在device结构的外面又加上自己的信息，就形成了现在这个scsi_device结构了
        if (!sdev)
                goto out;

        sdev->vendor = scsi_null_device_strs;
        sdev->model = scsi_null_device_strs;
        sdev->rev = scsi_null_device_strs;
        sdev->host = shost;
        sdev->queue_ramp_up_period = SCSI_DEFAULT_RAMP_UP_PERIOD;
.......................

         * Assume that the device will have handshaking problems,
         * and then fix this field later if it turns out it
         * doesn't
         */
        sdev->borken = 1;               /////这时设备还没有检测过是不是可以使用的状态的，后面需要检测的

        sdev->request_queue = scsi_alloc_queue(sdev);
        if (!sdev->request_queue) {
                /* release fn is set up in scsi_sysfs_device_initialise, so
                 * have to free and put manually here */
                put_device(&starget->dev);
                kfree(sdev);
                goto out;
        }

        sdev->request_queue->queuedata = sdev;
        scsi_adjust_queue_depth(sdev, 0, sdev->host->cmd_per_lun);

        scsi_sysfs_device_initialize(sdev);     ///////////sysyfs文件的初始化。///////////

        if (shost->hostt->slave_alloc) {
                ret = shost->hostt->slave_alloc(sdev);    /////////////调用scsi底层驱动的slave_alloc回调函数，让底层驱动可以scan在下一步作scan检测磁盘状态前可以做些额外初始化，如果底层驱动需要的话

.............
}



void scsi_sysfs_device_initialize(struct scsi_device *sdev)
{
        unsigned long flags;
        struct Scsi_Host *shost = sdev->host;
        struct scsi_target  *starget = sdev->sdev_target;

        device_initialize(&sdev->sdev_gendev);
        sdev->sdev_gendev.bus = &scsi_bus_type;        /////创建的时候就初始化 device结构的bus类型了，
        sdev->sdev_gendev.type = &scsi_dev_type;
        dev_set_name(&sdev->sdev_gendev, "%d:%d:%d:%d",
                     sdev->host->host_no, sdev->channel, sdev->id, sdev->lun);

        device_initialize(&sdev->sdev_dev);
        sdev->sdev_dev.parent = get_device(&sdev->sdev_gendev);
        sdev->sdev_dev.class = &sdev_class;
        dev_set_name(&sdev->sdev_dev, "%d:%d:%d:%d",
                     sdev->host->host_no, sdev->channel, sdev->id, sdev->lun);
        sdev->scsi_level = starget->scsi_level;
        transport_setup_device(&sdev->sdev_gendev);
        spin_lock_irqsave(shost->host_lock, flags);
        list_add_tail(&sdev->same_target_siblings, &starget->devices);
        list_add_tail(&sdev->siblings, &shost->__devices);
        spin_unlock_irqrestore(shost->host_lock, flags);
}









/**
 * scsi_probe_lun - probe a single LUN using a SCSI INQUIRY
 * @sdev:       scsi_device to probe
 * @inq_result: area to store the INQUIRY result
 * @result_len: len of inq_result
 * @bflags:     store any bflags found here
 *
 * Description:
 *     Probe the lun associated with @req using a standard SCSI INQUIRY;
 *
 *     If the INQUIRY is successful, zero is returned and the
 *     INQUIRY data is in @inq_result; the scsi_level and INQUIRY length
 *     are copied to the scsi_device any flags value is stored in *@bflags.
 **/
static int scsi_probe_lun(struct scsi_device *sdev, unsigned char *inq_result,
                          int result_len, int *bflags)
{
        unsigned char scsi_cmd[MAX_COMMAND_SIZE];
        int first_inquiry_len, try_inquiry_len, next_inquiry_len;
        int response_len = 0;
        int pass, count, result;
        struct scsi_sense_hdr sshdr;

        *bflags = 0;

        /* Perform up to 3 passes.  The first pass uses a conservative
         * transfer length of 36 unless sdev->inquiry_len specifies a
         * different value. */
        first_inquiry_len = sdev->inquiry_len ? sdev->inquiry_len : 36;
        try_inquiry_len = first_inquiry_len;
        pass = 1;

 next_pass:
        SCSI_LOG_SCAN_BUS(3, sdev_printk(KERN_INFO, sdev,
                                "scsi scan: INQUIRY pass %d length %d\n",
                                pass, try_inquiry_len));

        /* Each pass gets up to three chances to ignore Unit Attention */
        for (count = 0; count < 3; ++count) {           /////////尝试3次 SCSI INQUIRY 命令
                int resid;

                memset(scsi_cmd, 0, 6);
                scsi_cmd[0] = INQUIRY;
                scsi_cmd[4] = (unsigned char) try_inquiry_len;

                memset(inq_result, 0, try_inquiry_len);

                result = scsi_execute_req(sdev,  scsi_cmd, DMA_FROM_DEVICE,
                                          inq_result, try_inquiry_len, &sshdr,
                                          HZ / 2 + HZ * scsi_inq_timeout, 3,
                                          &resid);  ////把探测指令插到scsi的队列里面去，就会调用scsi底层驱动queuecommand回调来真正的执行命令了。

                SCSI_LOG_SCAN_BUS(3, printk(KERN_INFO "scsi scan: INQUIRY %s "
                                "with code 0x%x\n",
                                result ? "failed" : "successful", result));

                if (result) {
...............
 
}







/**
 * scsi_add_lun - allocate and fully initialze a scsi_device
 * @sdev:       holds information to be stored in the new scsi_device
 * @inq_result: holds the result of a previous INQUIRY to the LUN
 * @bflags:     black/white list flag
 * @async:      1 if this device is being scanned asynchronously
 *
 * Description:
 *     Initialize the scsi_device @sdev.  Optionally set fields based
 *     on values in *@bflags.
 *
 * Return:
 *     SCSI_SCAN_NO_RESPONSE: could not allocate or setup a scsi_device
 *     SCSI_SCAN_LUN_PRESENT: a new scsi_device was allocated and initialized
 **/
static int scsi_add_lun(struct scsi_device *sdev, unsigned char *inq_result,
                int *bflags, int async)
{
        int ret;

 
        /* set the device running here so that slave configure
         * may do I/O */
        ret = scsi_device_set_state(sdev, SDEV_RUNNING);  //////////改变设备状态

。。。。。。。。。。。。
        if (sdev->host->hostt->slave_configure) {
                ret = sdev->host->hostt->slave_configure(sdev);
                if (ret) {
                        /*
                         * if LLDD reports slave not present, don't clutter
                         * console with alloc failure messages
                         */
                        if (ret != -ENXIO) {
                                sdev_printk(KERN_ERR, sdev,
                                        "failed to configure device\n");
                        }
                        return SCSI_SCAN_NO_RESPONSE;
                }
        }

        sdev->max_queue_depth = sdev->queue_depth;

        /*
         * Ok, the device is now all set up, we can
         * register it and tell the rest of the kernel
         * about it.
         */
        if (!async && scsi_sysfs_add_sdev(sdev) != 0)       ////////////////////调用这个函数把先发现的scsi设备注册到系统的驱动模型里面去。这个完成之后scsi设备的sysfs文件，bus和驱动关联都会被配置好了
                return SCSI_SCAN_NO_RESPONSE;

        return SCSI_SCAN_LUN_PRESENT;
}







/**
 * scsi_sysfs_add_sdev - add scsi device to sysfs
 * @sdev:       scsi_device to add
 *
 * Return value:
 *      0 on Success / non-zero on Failure
 **/
int scsi_sysfs_add_sdev(struct scsi_device *sdev)
{
        int error, i;
        struct request_queue *rq = sdev->request_queue;
        struct scsi_target *starget = sdev->sdev_target;

        error = scsi_device_set_state(sdev, SDEV_RUNNING);
        if (error)
                return error;

        error = scsi_target_add(starget);
        if (error)
                return error;

        transport_configure_device(&starget->dev);

        device_enable_async_suspend(&sdev->sdev_gendev);
        scsi_autopm_get_target(starget);
        pm_runtime_set_active(&sdev->sdev_gendev);
        pm_runtime_forbid(&sdev->sdev_gendev);
        pm_runtime_enable(&sdev->sdev_gendev);
        scsi_autopm_put_target(starget);

        /* The following call will keep sdev active indefinitely, until
         * its driver does a corresponding scsi_autopm_pm_device().  Only
         * drivers supporting autosuspend will do this.
         */
        scsi_autopm_get_device(sdev);

        error = device_add(&sdev->sdev_gendev); /////////往系统中注册设备，这个进去就会把设备和驱动还有相关的总线关联起来了。
        if (error) {
                sdev_printk(KERN_INFO, sdev,
                                "failed to add device: %d\n", error);
                return error;
        }
        device_enable_async_suspend(&sdev->sdev_dev);
        error = device_add(&sdev->sdev_dev);   
        if (error) {
                sdev_printk(KERN_INFO, sdev,
                                "failed to add class device: %d\n", error);
                device_del(&sdev->sdev_gendev);
                return error;
        }
        transport_add_device(&sdev->sdev_gendev);
        sdev->is_visible = 1;

        /* create queue files, which may be writable, depending on the host */
        if (sdev->host->hostt->change_queue_depth) {
                error = device_create_file(&sdev->sdev_gendev,
                                           &sdev_attr_queue_depth_rw);
                error = device_create_file(&sdev->sdev_gendev,
                                           &sdev_attr_queue_ramp_up_period);
        }
        else
                error = device_create_file(&sdev->sdev_gendev, &dev_attr_queue_depth); ///////自己添加sysfs自定义文件
        if (error)
                return error;

        if (sdev->host->hostt->change_queue_type)
                error = device_create_file(&sdev->sdev_gendev, &sdev_attr_queue_type_rw);
        else
                error = device_create_file(&sdev->sdev_gendev, &dev_attr_queue_type);
        if (error)
                return error;

        error = bsg_register_queue(rq, &sdev->sdev_gendev, NULL, NULL);

        if (error)
                /* we're treating error on bsg register as non-fatal,
                 * so pretend nothing went wrong */
                sdev_printk(KERN_INFO, sdev,
                            "Failed to register bsg queue, errno=%d\n", error);

        /* add additional host specific attributes */
        if (sdev->host->hostt->sdev_attrs) {
                for (i = 0; sdev->host->hostt->sdev_attrs[i]; i++) {
                        error = device_create_file(&sdev->sdev_gendev,
                                        sdev->host->hostt->sdev_attrs[i]);
                        if (error)
                                return error;
                }
        }

        return error;
}






struct bus_type scsi_bus_type = {
        .name           = "scsi",
        .match          = scsi_bus_match,    >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
        .uevent         = scsi_bus_uevent,
#ifdef CONFIG_PM
        .pm             = &scsi_bus_pm_ops,
#endif
};
EXPORT_SYMBOL_GPL(scsi_bus_type);

int scsi_sysfs_register(void)
{
        int error;

        error = bus_register(&scsi_bus_type);
        if (!error) {
                error = class_register(&sdev_class);
                if (error)
                        bus_unregister(&scsi_bus_type);
        }

        return error;
}







 根据scsi磁盘设备驱动的注册，发现新设备之后将调到 sd驱动sd_probe 函数去






static struct scsi_driver sd_template = {
        .owner                  = THIS_MODULE,
        .gendrv = {
                .name           = "sd",
                .probe          = sd_probe,  >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
                .remove         = sd_remove,
                .suspend        = sd_suspend,
                .resume         = sd_resume,
                .shutdown       = sd_shutdown,
        },
        .rescan                 = sd_rescan,
        .done                   = sd_done,
};




/**
 *      init_sd - entry point for this driver (both when built in or when
 *      a module).
 *
 *      Note: this function registers this driver with the scsi mid-level.
 **/
static int __init init_sd(void)
{
        int majors = 0, i, err;

        SCSI_LOG_HLQUEUE(3, printk("init_sd: sd driver entry point\n"));

        for (i = 0; i < SD_MAJORS; i++)
                if (register_blkdev(sd_major(i), "sd") == 0)
                        majors++;

        if (!majors)
                return -ENODEV;

        err = class_register(&sd_disk_class);
        if (err)
                goto err_out;

        err = scsi_register_driver(&sd_template.gendrv);
        if (err)



://lxr.linux.no/#linux+v2.6.39/drivers/scsi/scsi_sysfs.c#L1009

int scsi_register_driver(struct device_driver *drv)
{
        drv->bus = &scsi_bus_type;

        return driver_register(drv);
}




static int sd_probe(struct device *dev)


            sdkp->driver = &sd_template;
            sdkp->disk = gd;




   

   
   
新设备的发现：
echo "- - -" >  /sys/class/scsi_host/host1/scan (横杆表示所有的0～255的所有设备)到 sys 文件系统的host文件夹的scan文件里面去（看一看scsi_scan_host_selected 在scsi_sysfs.c文件里实现），这个就会触发调用scsi_scan_host_selected函数，然后就会触发底层驱动扫描指定的设备有没有是不是可以工作， 然后添加设备到系统。

scsi驱动应该在适当的时候，自己调用scsi_remove_device 来把设备的引用计数减少，以把设备从系统中删除。 比如在驱动卸载的时候，就会把他自己所有的设备从系统里面移除。
   
   
内核文档
http://lxr.linux.no/#linux+v2.6.39/Documentation/scsi/scsi_mid_low_api.txt
的“Hotplug initialization model‘ 也有个流程介绍，可以看一下那里。

参考了内核文档：
 http://lxr.linux.no/#linux+v2.6.39/Documentation/driver-model/
                           
```

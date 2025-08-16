# 第8章：设备驱动模型

## 本章内容提要

Linux 设备驱动模型是内核中最复杂但也最优雅的子系统之一。从早期简单的字符设备和块设备抽象，演化到今天统一的设备模型，这一路径反映了 Linux 内核对硬件多样性和热插拔需求的不断适应。本章深入剖析设备驱动模型的核心机制，从 kobject/kset 的对象层次，到总线-设备-驱动的三角关系，再到中断处理的精妙设计。通过学习本章，读者将掌握如何编写高效、可靠的设备驱动，理解内核如何管理成千上万的硬件设备，以及现代 Linux 如何实现设备的自动发现和配置。

## 8.1 设备模型基础架构

### 8.1.1 kobject：内核对象的基石

Linux 2.6 引入的 kobject（kernel object）是整个设备模型的基础。每个 kobject 代表内核中的一个对象，通过引用计数管理生命周期，通过 sysfs 向用户空间导出信息。

```c
struct kobject {
    const char              *name;          // 对象名称
    struct list_head        entry;          // 链入 kset 的链表节点
    struct kobject          *parent;        // 父对象指针
    struct kset             *kset;          // 所属的 kset
    struct kobj_type        *ktype;         // 对象类型和操作
    struct kernfs_node      *sd;            // sysfs 目录项
    struct kref             kref;           // 引用计数
    unsigned int state_initialized:1;       // 初始化标志
    unsigned int state_in_sysfs:1;         // 在 sysfs 中的标志
    unsigned int state_add_uevent_sent:1;  // uevent 发送标志
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;        // 抑制 uevent
};
```

kobject 的关键特性：

1. **引用计数管理**：通过 kref 实现自动内存管理
2. **层次结构**：通过 parent 指针形成树形结构
3. **sysfs 表示**：每个 kobject 对应 /sys 下的一个目录
4. **事件通知**：通过 uevent 机制通知用户空间

### 8.1.2 kset：对象的集合

kset 是 kobject 的集合，提供了对同类对象的统一管理：

```c
struct kset {
    struct list_head list;           // kobject 链表头
    spinlock_t list_lock;           // 保护链表的自旋锁
    struct kobject kobj;            // 内嵌的 kobject
    const struct kset_uevent_ops *uevent_ops;  // uevent 操作集
};
```

kset 的主要功能：
- 将相关的 kobject 组织在一起
- 提供热插拔事件的过滤和环境变量设置
- 在 sysfs 中表现为包含多个子目录的目录

### 8.1.3 ktype：对象类型和操作

ktype 定义了 kobject 的类型特定操作：

```c
struct kobj_type {
    void (*release)(struct kobject *kobj);     // 释放函数
    const struct sysfs_ops *sysfs_ops;        // sysfs 操作
    struct attribute **default_attrs;          // 默认属性
    const struct attribute_group **default_groups;  // 默认属性组
    const struct kobj_ns_type_operations *child_ns_type;
    const void *(*namespace)(struct kobject *kobj);
    void (*get_ownership)(struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

## 8.2 总线、设备、驱动模型

### 8.2.1 总线（Bus）抽象

总线是连接设备和驱动的桥梁，Linux 将物理总线（如 PCI、USB）和虚拟总线（如 platform）都抽象为 bus_type：

```c
struct bus_type {
    const char *name;                      // 总线名称
    const char *dev_name;                  // 设备名称前缀
    struct device *dev_root;               // 根设备
    const struct attribute_group **bus_groups;     // 总线属性组
    const struct attribute_group **dev_groups;     // 设备属性组
    const struct attribute_group **drv_groups;     // 驱动属性组
    
    int (*match)(struct device *dev, struct device_driver *drv);  // 匹配函数
    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
    int (*probe)(struct device *dev);      // 探测设备
    void (*sync_state)(struct device *dev);
    int (*remove)(struct device *dev);     // 移除设备
    void (*shutdown)(struct device *dev);  // 关闭设备
    
    int (*online)(struct device *dev);
    int (*offline)(struct device *dev);
    
    int (*suspend)(struct device *dev, pm_message_t state);  // 电源管理
    int (*resume)(struct device *dev);
    
    const struct dev_pm_ops *pm;           // 电源管理操作集
    const struct iommu_ops *iommu_ops;     // IOMMU 操作集
    
    struct subsys_private *p;              // 私有数据
    struct lock_class_key lock_key;
    
    bool need_parent_lock;                 // 是否需要父设备锁
};
```

总线的核心职责：
1. **设备枚举**：发现连接到总线上的设备
2. **驱动匹配**：为设备找到合适的驱动程序
3. **电源管理**：协调设备的电源状态转换
4. **热插拔处理**：处理设备的动态添加和移除

### 8.2.2 设备（Device）表示

每个硬件设备在内核中用 struct device 表示：

```c
struct device {
    struct kobject kobj;                   // 内嵌的 kobject
    struct device *parent;                 // 父设备
    
    struct device_private *p;              // 私有数据
    
    const char *init_name;                 // 初始名称
    const struct device_type *type;        // 设备类型
    
    struct bus_type *bus;                  // 所属总线
    struct device_driver *driver;          // 绑定的驱动
    void *platform_data;                   // 平台特定数据
    void *driver_data;                     // 驱动私有数据
    
    struct mutex mutex;                    // 设备互斥锁
    
    struct dev_links_info links;           // 设备链接信息
    struct dev_pm_info power;              // 电源管理信息
    struct dev_pm_domain *pm_domain;       // PM 域
    
    struct em_perf_domain *em_pd;          // 能效模型
    
    struct irq_domain *msi_domain;         // MSI 中断域
    struct list_head msi_list;             // MSI 描述符链表
    
    const struct dma_map_ops *dma_ops;     // DMA 操作集
    u64 *dma_mask;                         // DMA 掩码
    u64 coherent_dma_mask;                 // 一致性 DMA 掩码
    u64 bus_dma_limit;                     // 总线 DMA 限制
    
    struct device_node *of_node;           // 设备树节点
    struct fwnode_handle *fwnode;          // 固件节点
    
    dev_t devt;                            // 设备号
    u32 id;                                // 设备 ID
    
    spinlock_t devres_lock;                // 资源管理锁
    struct list_head devres_head;          // 资源链表
    
    struct class *class;                   // 设备类
    const struct attribute_group **groups; // 属性组
    
    void (*release)(struct device *dev);   // 释放函数
    struct iommu_group *iommu_group;       // IOMMU 组
    struct dev_iommu *iommu;              // IOMMU 数据
    
    bool offline_disabled:1;               // 禁止离线
    bool offline:1;                        // 离线状态
    bool of_node_reused:1;                // 设备树节点复用
    bool state_synced:1;                  // 状态已同步
    bool can_match:1;                     // 可以匹配驱动
};
```

### 8.2.3 驱动（Driver）实现

设备驱动用 struct device_driver 表示：

```c
struct device_driver {
    const char *name;                      // 驱动名称
    struct bus_type *bus;                  // 所属总线
    
    struct module *owner;                  // 所属模块
    const char *mod_name;                  // 模块名称
    
    bool suppress_bind_attrs;              // 抑制绑定属性
    enum probe_type probe_type;            // 探测类型
    
    const struct of_device_id *of_match_table;     // 设备树匹配表
    const struct acpi_device_id *acpi_match_table; // ACPI 匹配表
    
    int (*probe)(struct device *dev);      // 探测函数
    void (*sync_state)(struct device *dev);
    int (*remove)(struct device *dev);     // 移除函数
    void (*shutdown)(struct device *dev);  // 关闭函数
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);
    const struct attribute_group **groups; // 属性组
    const struct attribute_group **dev_groups;
    
    const struct dev_pm_ops *pm;          // 电源管理操作
    void (*coredump)(struct device *dev);  // 核心转储
    
    struct driver_private *p;              // 私有数据
};
```

### 8.2.4 设备与驱动的绑定过程

设备与驱动的绑定是设备模型的核心过程：

```
1. 设备注册 (device_register)
   ├── device_initialize()：初始化设备结构
   └── device_add()：添加到系统
       ├── kobject_add()：添加到 kobject 层次
       ├── device_create_file()：创建 sysfs 属性
       ├── bus_add_device()：添加到总线
       └── bus_probe_device()：尝试绑定驱动
           └── device_initial_probe()
               └── __device_attach()
                   ├── bus->match()：调用总线匹配函数
                   └── driver_probe_device()：探测设备
                       └── really_probe()
                           ├── driver->probe()：调用驱动探测函数
                           └── driver_bound()：标记绑定成功

2. 驱动注册 (driver_register)
   ├── bus_add_driver()：添加到总线
   ├── driver_attach()：尝试绑定设备
   │   └── bus_for_each_dev()：遍历总线上的设备
   │       └── __driver_attach()：尝试绑定每个设备
   └── module_add_driver()：关联到模块
```

## 8.3 平台设备与设备树

### 8.3.1 平台设备架构

平台设备（platform device）是 Linux 为不依附于传统总线的设备提供的抽象：

```c
struct platform_device {
    const char *name;                      // 设备名称
    int id;                                // 设备 ID
    bool id_auto;                          // 自动分配 ID
    struct device dev;                     // 内嵌的 device
    u64 platform_dma_mask;                // DMA 掩码
    struct device_dma_parameters dma_parms;
    u32 num_resources;                     // 资源数量
    struct resource *resource;             // 资源数组
    
    const struct platform_device_id *id_entry;  // ID 表项
    char *driver_override;                 // 驱动覆盖
    
    struct mfd_cell *mfd_cell;            // MFD 单元
    
    struct pdev_archdata archdata;        // 架构特定数据
};

struct platform_driver {
    int (*probe)(struct platform_device *);    // 探测函数
    int (*remove)(struct platform_device *);   // 移除函数
    void (*shutdown)(struct platform_device *); // 关闭函数
    int (*suspend)(struct platform_device *, pm_message_t state);
    int (*resume)(struct platform_device *);
    struct device_driver driver;               // 内嵌的 driver
    const struct platform_device_id *id_table; // ID 匹配表
    bool prevent_deferred_probe;               // 阻止延迟探测
};
```

资源描述：
```c
struct resource {
    resource_size_t start;                 // 起始地址
    resource_size_t end;                   // 结束地址
    const char *name;                      // 资源名称
    unsigned long flags;                   // 资源标志
    unsigned long desc;                    // 描述符
    struct resource *parent, *sibling, *child;  // 层次结构
};

// 资源类型标志
#define IORESOURCE_IO      0x00000100     // I/O 端口资源
#define IORESOURCE_MEM     0x00000200     // 内存资源
#define IORESOURCE_REG     0x00000300     // 寄存器资源
#define IORESOURCE_IRQ     0x00000400     // 中断资源
#define IORESOURCE_DMA     0x00000800     // DMA 资源
#define IORESOURCE_BUS     0x00001000     // 总线资源
```

### 8.3.2 设备树（Device Tree）

设备树是描述硬件的数据结构，从 PowerPC 引入，现已成为 ARM/RISC-V 等架构的标准：

```dts
/ {
    compatible = "vendor,board";
    #address-cells = <2>;
    #size-cells = <2>;
    
    cpus {
        #address-cells = <1>;
        #size-cells = <0>;
        
        cpu@0 {
            device_type = "cpu";
            compatible = "arm,cortex-a72";
            reg = <0x0>;
            enable-method = "psci";
        };
    };
    
    memory@80000000 {
        device_type = "memory";
        reg = <0x0 0x80000000 0x0 0x80000000>;
    };
    
    soc {
        compatible = "simple-bus";
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;
        
        uart0: serial@fe650000 {
            compatible = "vendor,uart";
            reg = <0x0 0xfe650000 0x0 0x100>;
            interrupts = <GIC_SPI 117 IRQ_TYPE_LEVEL_HIGH>;
            clocks = <&cru SCLK_UART0>, <&cru PCLK_UART0>;
            clock-names = "baudclk", "apb_pclk";
            status = "okay";
        };
        
        i2c0: i2c@fe700000 {
            compatible = "vendor,i2c";
            reg = <0x0 0xfe700000 0x0 0x1000>;
            interrupts = <GIC_SPI 36 IRQ_TYPE_LEVEL_HIGH>;
            #address-cells = <1>;
            #size-cells = <0>;
            clock-frequency = <400000>;
            
            rtc@51 {
                compatible = "nxp,pcf8563";
                reg = <0x51>;
                interrupt-parent = <&gpio0>;
                interrupts = <RK_PD3 IRQ_TYPE_EDGE_FALLING>;
            };
        };
    };
};
```

设备树解析和匹配：
```c
// OF (Open Firmware) 匹配表
static const struct of_device_id my_driver_of_match[] = {
    { .compatible = "vendor,device-v1", .data = &device_v1_data },
    { .compatible = "vendor,device-v2", .data = &device_v2_data },
    { }
};
MODULE_DEVICE_TABLE(of, my_driver_of_match);

// 从设备树获取资源
static int my_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    struct resource *res;
    void __iomem *base;
    int irq;
    
    // 获取内存资源
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(base))
        return PTR_ERR(base);
    
    // 获取中断资源
    irq = platform_get_irq(pdev, 0);
    if (irq < 0)
        return irq;
    
    // 解析自定义属性
    u32 value;
    if (of_property_read_u32(np, "custom-property", &value))
        value = DEFAULT_VALUE;
    
    return 0;
}
```

## 8.4 字符设备驱动

### 8.4.1 字符设备基础

字符设备是 Linux 中最基本的设备类型之一，以字节流方式访问，不支持随机访问。典型的字符设备包括终端、串口、键盘、鼠标等。

字符设备核心数据结构：
```c
struct cdev {
    struct kobject kobj;                   // 内嵌的 kobject
    struct module *owner;                  // 所属模块
    const struct file_operations *ops;     // 文件操作集
    struct list_head list;                 // 设备链表
    dev_t dev;                             // 设备号
    unsigned int count;                    // 次设备号数量
};

// 文件操作集
struct file_operations {
    struct module *owner;
    loff_t (*llseek)(struct file *, loff_t, int);
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
    int (*iopoll)(struct kiocb *kiocb, struct io_comp_batch *, unsigned int flags);
    int (*iterate)(struct file *, struct dir_context *);
    int (*iterate_shared)(struct file *, struct dir_context *);
    __poll_t (*poll)(struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    long (*compat_ioctl)(struct file *, unsigned int, unsigned long);
    int (*mmap)(struct file *, struct vm_area_struct *);
    unsigned long mmap_supported_flags;
    int (*open)(struct inode *, struct file *);
    int (*flush)(struct file *, fl_owner_t id);
    int (*release)(struct inode *, struct file *);
    int (*fsync)(struct file *, loff_t, loff_t, int datasync);
    int (*fasync)(int, struct file *, int);
    int (*lock)(struct file *, int, struct file_lock *);
    ssize_t (*sendpage)(struct file *, struct page *, int, size_t, loff_t *, int);
    unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long,
                                       unsigned long, unsigned long);
    int (*check_flags)(int);
    int (*flock)(struct file *, int, struct file_lock *);
    ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, 
                            size_t, unsigned int);
    ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *,
                          size_t, unsigned int);
    int (*setlease)(struct file *, long, struct file_lock **, void **);
    long (*fallocate)(struct file *file, int mode, loff_t offset, loff_t len);
    void (*show_fdinfo)(struct seq_file *m, struct file *f);
    ssize_t (*copy_file_range)(struct file *, loff_t, struct file *, loff_t,
                               size_t, unsigned int);
    loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
                               struct file *file_out, loff_t pos_out,
                               loff_t len, unsigned int remap_flags);
    int (*fadvise)(struct file *, loff_t, loff_t, int);
};
```

### 8.4.2 设备号管理

Linux 使用主设备号和次设备号来标识设备：

```c
// 设备号宏
#define MINORBITS    20                          // 次设备号位数
#define MINORMASK    ((1U << MINORBITS) - 1)     // 次设备号掩码

#define MAJOR(dev)   ((unsigned int) ((dev) >> MINORBITS))  // 获取主设备号
#define MINOR(dev)   ((unsigned int) ((dev) & MINORMASK))   // 获取次设备号
#define MKDEV(ma,mi) (((ma) << MINORBITS) | (mi))           // 合成设备号

// 分配设备号
int register_chrdev_region(dev_t from, unsigned count, const char *name);  // 静态分配
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count,   // 动态分配
                        const char *name);
void unregister_chrdev_region(dev_t from, unsigned count);                 // 释放
```

### 8.4.3 字符设备驱动实例

完整的字符设备驱动示例：

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/slab.h>

#define DEVICE_NAME "mychardev"
#define CLASS_NAME "mychar"
#define BUFFER_SIZE 1024

struct mychar_dev {
    struct cdev cdev;              // 字符设备结构
    struct device *device;         // 设备结构
    struct mutex mutex;            // 互斥锁
    char *buffer;                  // 数据缓冲区
    size_t size;                   // 当前数据大小
};

static dev_t dev_number;          // 设备号
static struct class *dev_class;   // 设备类
static struct mychar_dev *mydev;  // 设备实例

// 打开设备
static int mychar_open(struct inode *inode, struct file *filp)
{
    struct mychar_dev *dev;
    
    dev = container_of(inode->i_cdev, struct mychar_dev, cdev);
    filp->private_data = dev;
    
    return 0;
}

// 读取设备
static ssize_t mychar_read(struct file *filp, char __user *buf,
                           size_t count, loff_t *f_pos)
{
    struct mychar_dev *dev = filp->private_data;
    ssize_t retval = 0;
    
    mutex_lock(&dev->mutex);
    
    if (*f_pos >= dev->size)
        goto out;
    
    if (*f_pos + count > dev->size)
        count = dev->size - *f_pos;
    
    if (copy_to_user(buf, dev->buffer + *f_pos, count)) {
        retval = -EFAULT;
        goto out;
    }
    
    *f_pos += count;
    retval = count;
    
out:
    mutex_unlock(&dev->mutex);
    return retval;
}

// 写入设备
static ssize_t mychar_write(struct file *filp, const char __user *buf,
                            size_t count, loff_t *f_pos)
{
    struct mychar_dev *dev = filp->private_data;
    ssize_t retval = -ENOMEM;
    
    mutex_lock(&dev->mutex);
    
    if (*f_pos >= BUFFER_SIZE)
        goto out;
    
    if (*f_pos + count > BUFFER_SIZE)
        count = BUFFER_SIZE - *f_pos;
    
    if (copy_from_user(dev->buffer + *f_pos, buf, count)) {
        retval = -EFAULT;
        goto out;
    }
    
    *f_pos += count;
    if (*f_pos > dev->size)
        dev->size = *f_pos;
    retval = count;
    
out:
    mutex_unlock(&dev->mutex);
    return retval;
}

// ioctl 操作
static long mychar_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    struct mychar_dev *dev = filp->private_data;
    long retval = 0;
    
    // ioctl 命令定义
    #define MYCHAR_IOC_MAGIC 'k'
    #define MYCHAR_IOCRESET  _IO(MYCHAR_IOC_MAGIC, 0)
    #define MYCHAR_IOCGSIZE  _IOR(MYCHAR_IOC_MAGIC, 1, int)
    #define MYCHAR_IOCSSIZE  _IOW(MYCHAR_IOC_MAGIC, 2, int)
    
    switch (cmd) {
    case MYCHAR_IOCRESET:
        mutex_lock(&dev->mutex);
        dev->size = 0;
        memset(dev->buffer, 0, BUFFER_SIZE);
        mutex_unlock(&dev->mutex);
        break;
        
    case MYCHAR_IOCGSIZE:
        if (put_user(dev->size, (int __user *)arg))
            retval = -EFAULT;
        break;
        
    case MYCHAR_IOCSSIZE:
        if (get_user(dev->size, (int __user *)arg))
            retval = -EFAULT;
        break;
        
    default:
        retval = -EINVAL;
    }
    
    return retval;
}

// 文件操作集
static struct file_operations mychar_fops = {
    .owner = THIS_MODULE,
    .open = mychar_open,
    .read = mychar_read,
    .write = mychar_write,
    .unlocked_ioctl = mychar_ioctl,
};

// 模块初始化
static int __init mychar_init(void)
{
    int ret;
    
    // 分配设备号
    ret = alloc_chrdev_region(&dev_number, 0, 1, DEVICE_NAME);
    if (ret < 0) {
        pr_err("Failed to allocate device number\n");
        return ret;
    }
    
    // 创建设备类
    dev_class = class_create(THIS_MODULE, CLASS_NAME);
    if (IS_ERR(dev_class)) {
        unregister_chrdev_region(dev_number, 1);
        return PTR_ERR(dev_class);
    }
    
    // 分配设备结构
    mydev = kzalloc(sizeof(*mydev), GFP_KERNEL);
    if (!mydev) {
        ret = -ENOMEM;
        goto fail_alloc;
    }
    
    // 分配缓冲区
    mydev->buffer = kzalloc(BUFFER_SIZE, GFP_KERNEL);
    if (!mydev->buffer) {
        ret = -ENOMEM;
        goto fail_buffer;
    }
    
    // 初始化字符设备
    cdev_init(&mydev->cdev, &mychar_fops);
    mydev->cdev.owner = THIS_MODULE;
    mutex_init(&mydev->mutex);
    
    // 添加字符设备
    ret = cdev_add(&mydev->cdev, dev_number, 1);
    if (ret) {
        pr_err("Failed to add cdev\n");
        goto fail_cdev;
    }
    
    // 创建设备节点
    mydev->device = device_create(dev_class, NULL, dev_number, NULL, DEVICE_NAME);
    if (IS_ERR(mydev->device)) {
        ret = PTR_ERR(mydev->device);
        goto fail_device;
    }
    
    pr_info("mychar driver initialized\n");
    return 0;
    
fail_device:
    cdev_del(&mydev->cdev);
fail_cdev:
    kfree(mydev->buffer);
fail_buffer:
    kfree(mydev);
fail_alloc:
    class_destroy(dev_class);
    unregister_chrdev_region(dev_number, 1);
    return ret;
}

// 模块退出
static void __exit mychar_exit(void)
{
    device_destroy(dev_class, dev_number);
    cdev_del(&mydev->cdev);
    kfree(mydev->buffer);
    kfree(mydev);
    class_destroy(dev_class);
    unregister_chrdev_region(dev_number, 1);
    pr_info("mychar driver exited\n");
}

module_init(mychar_init);
module_exit(mychar_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Linux Kernel Developer");
MODULE_DESCRIPTION("A simple character device driver");
```

## 8.5 块设备驱动

### 8.5.1 块设备架构

块设备与字符设备的主要区别在于：块设备支持随机访问，以固定大小的块为单位进行数据传输，并且具有请求队列和 I/O 调度机制。

块设备核心数据结构：
```c
struct gendisk {
    int major;                              // 主设备号
    int first_minor;                       // 第一个次设备号
    int minors;                             // 次设备号数量
    
    char disk_name[DISK_NAME_LEN];        // 磁盘名称（如 sda）
    
    unsigned short events;                  // 支持的事件
    unsigned short event_flags;            // 事件标志
    
    struct xarray part_tbl;                // 分区表
    struct block_device *part0;            // 整个磁盘的 block_device
    
    const struct block_device_operations *fops;  // 块设备操作集
    struct request_queue *queue;           // 请求队列
    void *private_data;                    // 私有数据
    
    int flags;                             // 磁盘标志
    unsigned long state;                   // 磁盘状态
    struct mutex open_mutex;               // 打开互斥锁
    unsigned open_partitions;              // 打开的分区数
    
    struct backing_dev_info *bdi;         // 后备设备信息
    struct kobject *slave_dir;            // 从属目录
    
    struct timer_rand_state *random;      // 随机数状态
    atomic_t sync_io;                      // 同步 I/O 计数
    struct disk_events *ev;               // 磁盘事件
    
    struct kobject integrity_kobj;        // 完整性 kobject
    struct cdrom_device_info *cdi;        // CD-ROM 信息
    
    int node_id;                           // NUMA 节点 ID
    struct badblocks *bb;                  // 坏块信息
    struct lockdep_map lockdep_map;       // 锁依赖映射
    u64 diskseq;                          // 磁盘序列号
};

// 块设备操作集
struct block_device_operations {
    int (*open)(struct block_device *, fmode_t);       // 打开设备
    void (*release)(struct gendisk *, fmode_t);        // 释放设备
    int (*rw_page)(struct block_device *, sector_t, struct page *, unsigned int);
    int (*ioctl)(struct block_device *, fmode_t, unsigned, unsigned long);
    int (*compat_ioctl)(struct block_device *, fmode_t, unsigned, unsigned long);
    unsigned int (*check_events)(struct gendisk *disk, unsigned int clearing);
    void (*unlock_native_capacity)(struct gendisk *);
    int (*revalidate_disk)(struct gendisk *);
    int (*getgeo)(struct block_device *, struct hd_geometry *);  // 获取几何信息
    void (*swap_slot_free_notify)(struct block_device *, unsigned long);
    int (*report_zones)(struct gendisk *, sector_t sector,
                       unsigned int nr_zones, report_zones_cb cb, void *data);
    char *(*devnode)(struct gendisk *disk, umode_t *mode);
    struct module *owner;
    const struct pr_ops *pr_ops;           // 持久预留操作
    int (*alternative_gpt_sector)(struct gendisk *disk, sector_t *sector);
};
```

### 8.5.2 请求队列和 bio

块设备的 I/O 请求通过 bio（Block I/O）结构描述：

```c
struct bio {
    struct bio          *bi_next;          // 请求队列中的下一个 bio
    struct gendisk      *bi_disk;          // 目标磁盘
    unsigned int        bi_opf;            // 操作标志
    unsigned short      bi_flags;          // 状态标志
    unsigned short      bi_ioprio;         // I/O 优先级
    unsigned short      bi_write_hint;     // 写入提示
    blk_status_t        bi_status;         // 完成状态
    u8                  bi_partno;         // 分区号
    atomic_t            __bi_remaining;    // 剩余计数
    
    struct bvec_iter    bi_iter;           // 当前迭代器
    
    bio_end_io_t        *bi_end_io;        // 完成回调函数
    void                *bi_private;       // 私有数据
    
    struct blkcg_gq     *bi_blkg;          // cgroup 信息
    struct bio_issue    bi_issue;          // 发出时间戳
    
    union {
        struct bio_integrity_payload *bi_integrity;  // 完整性载荷
    };
    
    unsigned short      bi_vcnt;           // bio_vec 数量
    unsigned short      bi_max_vecs;       // 最大 bio_vec 数量
    
    atomic_t            __bi_cnt;          // 引用计数
    
    struct bio_vec      *bi_io_vec;        // I/O 向量数组
    
    struct bio_set      *bi_pool;          // 来源的 bio 池
    
    struct bio_vec      bi_inline_vecs[];  // 内联向量数组
};

// bio 向量：描述内存页面
struct bio_vec {
    struct page     *bv_page;              // 页面指针
    unsigned int    bv_len;                // 长度
    unsigned int    bv_offset;             // 页内偏移
};

// 请求结构
struct request {
    struct request_queue *q;               // 所属队列
    struct blk_mq_ctx *mq_ctx;            // 多队列上下文
    struct blk_mq_hw_ctx *mq_hctx;        // 硬件队列上下文
    
    unsigned int cmd_flags;                // 命令标志
    req_flags_t rq_flags;                 // 请求标志
    
    int tag;                               // 请求标签
    int internal_tag;                      // 内部标签
    
    unsigned int __data_len;              // 数据长度
    sector_t __sector;                     // 起始扇区
    
    struct bio *bio;                       // bio 链表头
    struct bio *biotail;                   // bio 链表尾
    
    struct list_head queuelist;           // 队列链表节点
    
    union {
        struct hlist_node hash;            // 哈希链表节点
        struct list_head ipi_list;        // IPI 链表
    };
    
    union {
        struct rb_node rb_node;            // 红黑树节点
        struct bio_vec special_vec;       // 特殊向量
        void *completion_data;             // 完成数据
        int error_count;                   // 错误计数
    };
    
    union {
        struct {
            struct io_cq *icq;            // I/O 上下文队列
            void *priv[2];                // 私有数据
        } elv;
        
        struct {
            unsigned int seq;              // 序列号
            struct list_head list;        // 链表
            rq_end_io_fn *saved_end_io;   // 保存的完成函数
        } flush;
    };
    
    union {
        struct __call_single_data csd;    // 单次调用数据
        u64 fifo_time;                    // FIFO 时间
    };
    
    rq_end_io_fn *end_io;                 // 完成回调函数
    void *end_io_data;                    // 完成回调数据
};
```

### 8.5.3 块设备驱动实例

简单的内存块设备驱动示例：

```c
#include <linux/module.h>
#include <linux/blkdev.h>
#include <linux/hdreg.h>

#define MEMBLK_DISK_NAME    "memblk"
#define MEMBLK_MINORS       16
#define MEMBLK_SECTOR_SIZE  512
#define MEMBLK_NSECTORS     (16 * 1024 * 2)  // 16MB

struct memblk_device {
    int major;
    struct gendisk *disk;
    struct request_queue *queue;
    spinlock_t lock;
    u8 *data;                              // 数据存储区
    size_t size;                           // 设备大小
};

static struct memblk_device *memblk_dev;

// 处理 I/O 请求
static blk_status_t memblk_queue_rq(struct blk_mq_hw_ctx *hctx,
                                    const struct blk_mq_queue_data *bd)
{
    struct request *rq = bd->rq;
    struct memblk_device *dev = rq->q->queuedata;
    struct bio_vec bvec;
    struct req_iterator iter;
    loff_t pos = blk_rq_pos(rq) << 9;     // 扇区转字节
    loff_t len = blk_rq_bytes(rq);
    void *buffer;
    
    blk_mq_start_request(rq);
    
    if (pos + len > dev->size) {
        pr_err("Request beyond device capacity\n");
        blk_mq_end_request(rq, BLK_STS_IOERR);
        return BLK_STS_IOERR;
    }
    
    // 遍历请求中的所有 bio 向量
    rq_for_each_segment(bvec, rq, iter) {
        buffer = page_address(bvec.bv_page) + bvec.bv_offset;
        
        if (rq_data_dir(rq) == WRITE) {
            // 写操作：从 buffer 复制到设备
            memcpy(dev->data + pos, buffer, bvec.bv_len);
        } else {
            // 读操作：从设备复制到 buffer
            memcpy(buffer, dev->data + pos, bvec.bv_len);
        }
        
        pos += bvec.bv_len;
    }
    
    blk_mq_end_request(rq, BLK_STS_OK);
    return BLK_STS_OK;
}

// 块设备操作集
static int memblk_open(struct block_device *bdev, fmode_t mode)
{
    return 0;
}

static void memblk_release(struct gendisk *disk, fmode_t mode)
{
}

static int memblk_getgeo(struct block_device *bdev, struct hd_geometry *geo)
{
    struct memblk_device *dev = bdev->bd_disk->private_data;
    long size = dev->size / MEMBLK_SECTOR_SIZE;
    
    geo->cylinders = (size & ~0x3f) >> 6;
    geo->heads = 4;
    geo->sectors = 16;
    geo->start = 0;
    
    return 0;
}

static const struct block_device_operations memblk_ops = {
    .owner = THIS_MODULE,
    .open = memblk_open,
    .release = memblk_release,
    .getgeo = memblk_getgeo,
};

// 多队列操作集
static const struct blk_mq_ops memblk_mq_ops = {
    .queue_rq = memblk_queue_rq,
};

// 模块初始化
static int __init memblk_init(void)
{
    int ret;
    
    // 分配设备结构
    memblk_dev = kzalloc(sizeof(*memblk_dev), GFP_KERNEL);
    if (!memblk_dev)
        return -ENOMEM;
    
    // 分配数据存储区
    memblk_dev->size = MEMBLK_NSECTORS * MEMBLK_SECTOR_SIZE;
    memblk_dev->data = vzalloc(memblk_dev->size);
    if (!memblk_dev->data) {
        ret = -ENOMEM;
        goto fail_data;
    }
    
    // 注册块设备主设备号
    memblk_dev->major = register_blkdev(0, MEMBLK_DISK_NAME);
    if (memblk_dev->major < 0) {
        ret = memblk_dev->major;
        goto fail_register;
    }
    
    // 分配 gendisk
    memblk_dev->disk = alloc_disk(MEMBLK_MINORS);
    if (!memblk_dev->disk) {
        ret = -ENOMEM;
        goto fail_disk;
    }
    
    // 初始化请求队列
    memblk_dev->disk->queue = blk_mq_init_sq_queue(&memblk_dev->tag_set,
                                                   &memblk_mq_ops,
                                                   128,
                                                   BLK_MQ_F_SHOULD_MERGE);
    if (IS_ERR(memblk_dev->disk->queue)) {
        ret = PTR_ERR(memblk_dev->disk->queue);
        goto fail_queue;
    }
    
    memblk_dev->queue = memblk_dev->disk->queue;
    memblk_dev->queue->queuedata = memblk_dev;
    
    // 设置 gendisk 属性
    memblk_dev->disk->major = memblk_dev->major;
    memblk_dev->disk->first_minor = 0;
    memblk_dev->disk->fops = &memblk_ops;
    memblk_dev->disk->private_data = memblk_dev;
    snprintf(memblk_dev->disk->disk_name, 32, MEMBLK_DISK_NAME);
    set_capacity(memblk_dev->disk, MEMBLK_NSECTORS);
    
    // 添加磁盘
    add_disk(memblk_dev->disk);
    
    pr_info("memblk: initialized, size=%zu bytes\n", memblk_dev->size);
    return 0;
    
fail_queue:
    put_disk(memblk_dev->disk);
fail_disk:
    unregister_blkdev(memblk_dev->major, MEMBLK_DISK_NAME);
fail_register:
    vfree(memblk_dev->data);
fail_data:
    kfree(memblk_dev);
    return ret;
}

// 模块退出
static void __exit memblk_exit(void)
{
    del_gendisk(memblk_dev->disk);
    blk_cleanup_queue(memblk_dev->queue);
    put_disk(memblk_dev->disk);
    unregister_blkdev(memblk_dev->major, MEMBLK_DISK_NAME);
    vfree(memblk_dev->data);
    kfree(memblk_dev);
    pr_info("memblk: exited\n");
}

module_init(memblk_init);
module_exit(memblk_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Linux Kernel Developer");
MODULE_DESCRIPTION("Simple memory block device driver");
```

---
title:       自制操作系统 - 硬盘驱动
subtitle:    "硬盘驱动"
description: "PC机最多支持4块IDE硬盘，因此操作系统首先需要识别硬盘，识别硬盘利用IDENTIFY机制。识别硬盘后需要对硬盘进行读写，默认是同步的，因为磁盘IO效率问题，可以通过中断机制实现异步IO。最后还需要对硬盘进行分区，以及通过虚拟设备来进行硬件的读写管理。"
excerpt:     "PC机最多支持4块IDE硬盘，因此操作系统首先需要识别硬盘，识别硬盘利用IDENTIFY机制。识别硬盘后需要对硬盘进行读写，默认是同步的，因为磁盘IO效率问题，可以通过中断机制实现异步IO。最后还需要对硬盘进行分区，以及通过虚拟设备来进行硬件的读写管理。"
date:        2024-02-21T09:35:44+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/6c/9b/autumn_fall_forest_person_iphone-21693.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-disk-driver"
categories:  [ "操作系统" ]
---

## 一、磁盘同步PIO
PIO(Programmed Input/Output) 编程输入输出，PIO 模式使用了大量的 CPU 资源，因为磁盘和 CPU 之间传输的每个字节的数据都必须通过 CPU 的 IO 端口总线(而不是内存)传送。在某些 CPU 上，PIO 模式仍然可以实现每秒 16 MB 的实际传输速度，但是机器上的其他进程将得不到任何 CPU 时间片。


### 1.1、磁盘同步PIO实现
在CPU从硬盘加载bootloader和内核时，我们使用汇编语言实现过硬盘的读写就是通过同步PIO，这里我们通过C语言在内核中来实现。 
```cpp
#define SECTOR_SIZE 512 // 扇区大小
#define IDE_CTRL_NR 2 // 控制器数量，固定为 2
#define IDE_DISK_NR 2 // 每个控制器可挂磁盘数量，固定为 2

typedef struct ide_disk_t  // IDE 磁盘
{
    char name[8];            // 磁盘名称
    struct ide_ctrl_t *ctrl; // 控制器指针
    u8 selector;             // 磁盘选择
    bool master;             // 主盘
} ide_disk_t;

typedef struct ide_ctrl_t   // IDE 控制器
{
    char name[8];                  // 控制器名称
    lock_t lock;                   // 控制器锁
    u16 iobase;                    // IO 寄存器基址
    ide_disk_t disks[IDE_DISK_NR]; // 磁盘
    ide_disk_t *active;            // 当前选择的磁盘
} ide_ctrl_t;
```
IDE（Integrated Drive Electronics） 是一种老式硬盘接口协议，也叫 ATA。传统 PC 架构中，一个系统默认最多支持 两个 IDE 控制器（Primary 和 Secondary），每个控制器可以挂接 最多 2 块磁盘（一块主盘 master，一块从盘 slave）。接下来就是控制器的初始化和磁盘的读写，我们这里不在贴这部分代码，其中的原理通过汇编实现磁盘读写时已经展示过。

同步PIO在读写硬盘过程中，发起读写请求后，需要原地等待磁盘数据准备完毕，因此性能比较低。

## 二、磁盘异步PIO
同步状态监测会消耗大量CPU资源，所以可以使用异步方式等待硬盘驱动器。发完读写命令后进程可以进入阻塞态，当驱动器完成一个扇区的操作 (读/写) 时，会发送中断，可以在中断中恢复进程到就绪态，继续执行。
```cpp
int ide_pio_read(ide_disk_t *disk, void *buf, u8 count, idx_t lba)
{
    assert(count > 0);
    assert(!get_interrupt_state()); // 异步方式，调用该函数时不许中断
    ide_ctrl_t *ctrl = disk->ctrl;
    lock_acquire(&ctrl->lock);
    ide_select_drive(disk);     // 选择磁盘
    ide_busy_wait(ctrl, IDE_SR_DRDY);   // 等待就绪
    ide_select_sector(disk, lba, count);   // 选择扇区
    outb(ctrl->iobase + IDE_COMMAND, IDE_CMD_READ);  // 发送读命令

    for (size_t i = 0; i < count; i++)
    {
        task_t *task = running_task();
        if (task->state == TASK_RUNNING) // 系统初始化时，不能使用异步方式
        {
            ctrl->waiter = task;         // 阻塞自己等待中断的到来，等待磁盘准备数据
            task_block(task, NULL, TASK_BLOCKED);
        }
        ide_busy_wait(ctrl, IDE_SR_DRQ);

        u32 offset = ((u32)buf + i * SECTOR_SIZE);
        ide_pio_read_sector(disk, (u16 *)offset);
    }
    lock_release(&ctrl->lock);
    return 0;
}

void ide_handler(int vector)
{
    send_eoi(vector); // 向中断控制器发送中断处理结束信号
    ide_ctrl_t *ctrl = &controllers[vector - IRQ_HARDDISK - 0x20];  // 得到中断向量对应的控制器
    u8 state = inb(ctrl->iobase + IDE_STATUS);  // 读取常规状态寄存器，表示中断处理结束
    LOGK("harddisk interrupt vector %d state 0x%x\n", vector, state);
    if (ctrl->waiter)
    {
        // 如果有进程阻塞，则取消阻塞
        task_unblock(ctrl->waiter);
        ctrl->waiter = NULL;
    }
}
```
如上所示，在读取硬盘时，发起读请求后当前进程主动进入阻塞态。当数据准备完成时，就会产生中断，中断函数为ide_handler。我们这里实现的是没读写一个扇区就产生一次中断，当然还可以实现读写多块，读写多个扇区产生一次中断，这比较复杂我们的系统不做实现。

注意这里在异步读写硬盘时不允许中断，因为有可能发起读请求后，数据很快准备完成了，就产生了中断进行中断处理，但此时trl->waiter为NULL，因此本次中断没起作用。然后我们的读硬盘继续向下执行自动主动阻塞，但是不会再有中断到来了，因此可能永远阻塞在这里。

## 三、识别硬盘
PC最多支持四个IDE硬盘，因此我们需要识别哪些挂了硬盘。目前所有的 BIOS 都标准化了 IDENTIFY 命令的使用，以检测所有类型的 ATA 总线设备的存在 PATA, PATAPI, SATAPI, SATA。
```cpp

static u32 ide_identify(ide_disk_t *disk, u16 *buf)
{
    LOGK("identifing disk %s...\n", disk->name);
    lock_acquire(&disk->ctrl->lock);   // 加锁
    ide_select_drive(disk);        // 选择磁盘
    outb(disk->ctrl->iobase + IDE_COMMAND, IDE_CMD_IDENTIFY);  // 向磁盘发生识别命令0xEC，此时磁盘回准备512字节的磁盘信息数据
    ide_busy_wait(disk->ctrl, IDE_SR_NULL);
    ide_params_t *params = (ide_params_t *)buf;    // 磁盘信息的数据结构，我们一般只关注total lba
    ide_pio_read_sector(disk, buf);       // 读取一个扇区到buf中
    LOGK("disk %s total lba %d\n", disk->name, params->total_lba);
    u32 ret = EOF;
    if (params->total_lba == 0)
    {
        goto rollback;
    }
    ide_swap_pairs(params->serial, sizeof(params->serial));    // 以下为一些打印信息
    LOGK("disk %s serial number %s\n", disk->name, params->serial);
    ide_swap_pairs(params->firmware, sizeof(params->firmware));
    LOGK("disk %s firmware version %s\n", disk->name, params->firmware);
    ide_swap_pairs(params->model, sizeof(params->model));
    LOGK("disk %s model number %s\n", disk->name, params->model);
    disk->total_lba = params->total_lba;
    disk->cylinders = params->cylinders;
    disk->heads = params->heads;
    disk->sectors = params->sectors;
    ret = 0;

rollback:
    lock_release(&disk->ctrl->lock);
    return ret;
}
```
如上为识别硬盘的代码，通过IDENTIFY命令，使用IDENTIFY命令也是向指定发生指令。就会获取到是否有硬盘以及硬盘的大小等信息。BIOS通过IDENTIFY来识别硬盘。这里会分别去识别4块硬盘是否存在，如果totle_lba为0，则认为硬盘不存在。效果如下：
![图片加载失败](/post_images/os/{{< filename >}}/3-01.png)

## 四、硬盘分区
为了实现多个操作系统共享硬盘资源，硬盘可以在逻辑上分为 4 个主分区。每个分区之间的扇区号是邻接的。分区表由 4 个表项组成，每个表项由 16 字节组成，对应一个分区的信息，存放有分区的大小和起止的柱面号、磁道号和扇区号。磁盘分区存储在主引导扇区的64个字节中，即 0 柱面 0 头第 1 个扇区的 0x1BE ~ 0x1FD 处。
![图片加载失败](/post_images/os/{{< filename >}}/4-01.png)
如图为每个主分区的存储信息，主要关注分区类型字节、分区起始位置和分区扇区数即可

### 4.1、扩展分区
扩展分区是一种可以多加 4 个分区的方式。如果分区表中的 SystemID 字段的值位 0x5，表示该分区为扩展分区。

可以将扩展分区的所有扇区组合起来认为是一个新的磁盘，然后再对其进行分区，这种逻辑有点套娃，所以如果磁盘空间足够大，理论上可以分出无数个分区。
![图片加载失败](/post_images/os/{{< filename >}}/4.1-01.png)

### 4.2、创建分区
可以通过 fdisk 命令对磁盘进行分区，这样就可以对我们的master.img进行分区操作了。以下是分区相关的几条命令：
```cmd
sfdisk -d /dev/... > master.sfdisk  # 然后可以将分好区的分区信息备份
sfdisk /dev/... < master.sfdisk     # 有分区信息可以直接对磁盘进行分区
sudo losetup /dev/loop0 --partscan master.img  # 可以将磁盘挂载到系统
sudo losetup -d /dev/loop0   # 取消挂载
```
做完分区后，就可以在我们的操作系统中来读取分区了。
```cpp
#define IDE_PART_NR 4 // 每个磁盘分区数量，只支持主分区，总共 4 个
typedef struct part_entry_t   // 这是主引导扇区内的16个字节信息
{
    u8 bootable;             // 引导标志
    u8 start_head;           // 分区起始磁头号
    u8 start_sector : 6;     // 分区起始扇区号
    u16 start_cylinder : 10; // 分区起始柱面号
    u8 system;               // 分区类型字节
    u8 end_head;             // 分区的结束磁头号
    u8 end_sector : 6;       // 分区结束扇区号
    u16 end_cylinder : 10;   // 分区结束柱面号
    u32 start;               // 分区起始物理扇区号 LBA
    u32 count;               // 分区占用的扇区数
} _packed part_entry_t;

typedef struct boot_sector_t   // 主引导扇区结构
{
    u8 code[446];    // 446个字节的代码
    part_entry_t entry[4];  // 4个分区信息
    u16 signature;   // 2个字节的校验位 0x55aa
} _packed boot_sector_t;

typedef struct ide_part_t    // 我们的系统使用的分区信息
{
    char name[8];            // 分区名称
    struct ide_disk_t *disk; // 磁盘指针
    u32 system;              // 分区类型
    u32 start;               // 分区起始物理扇区号 LBA
    u32 count;               // 分区占用的扇区数
} ide_part_t;

// 分区文件系统， 以下是一些常用的分区类型
// 参考 https://www.win.tue.nl/~aeb/partitions/partition_types-1.html
typedef enum PART_FS
{
    PART_FS_FAT12 = 1,    // FAT12
    PART_FS_EXTENDED = 5, // 扩展分区
    PART_FS_MINIX = 0x80, // minux
    PART_FS_LINUX = 0x83, // linux
} PART_FS;

// 读分区
int ide_pio_part_read(ide_part_t *part, void *buf, u8 count, idx_t lba)
{
    return ide_pio_read(part->disk, buf, count, part->start + lba);
}

// 写分区
int ide_pio_part_write(ide_part_t *part, void *buf, u8 count, idx_t lba)
{
    return ide_pio_write(part->disk, buf, count, part->start + lba);
}

// 分区初始化
static void ide_part_init(ide_disk_t *disk, u16 *buf)
{
    if (!disk->total_lba) // total_lba为0 ，说明磁盘不可用
        return;
    ide_pio_read(disk, buf, 1, 0);  // 读取主引导扇区
    boot_sector_t *boot = (boot_sector_t *)buf;   // 初始化主引导扇区

    for (size_t i = 0; i < IDE_PART_NR; i++)  // 分别读取主引导扇区中的4个主分区信息
    {
        part_entry_t *entry = &boot->entry[i];  
        ide_part_t *part = &disk->parts[i];
        if (!entry->count)      // 如果分区的扇区数量为0，直接跳过
            continue;

        sprintf(part->name, "%s%d", disk->name, i + 1);  // 以下打印分区信息，并设置分区的结构体

        LOGK("part %s \n", part->name);
        LOGK("    bootable %d\n", entry->bootable);
        LOGK("    start %d\n", entry->start);
        LOGK("    count %d\n", entry->count);
        LOGK("    system 0x%x\n", entry->system);

        part->disk = disk;
        part->count = entry->count;
        part->system = entry->system;
        part->start = entry->start;

        if (entry->system == PART_FS_EXTENDED)   // 如果是扩展分区直接报错，我们的系统不支持
        {
            LOGK("Unsupported extended partition!!!\n");

            boot_sector_t *eboot = (boot_sector_t *)(buf + SECTOR_SIZE); // 读取一下扩展分区的内容
            ide_pio_read(disk, (void *)eboot, 1, entry->start);

            for (size_t j = 0; j < IDE_PART_NR; j++)
            {
                part_entry_t *eentry = &eboot->entry[j];
                if (!eentry->count)
                    continue;
                LOGK("part %d extend %d\n", i, j);
                LOGK("    bootable %d\n", eentry->bootable);
                LOGK("    start %d\n", eentry->start);
                LOGK("    count %d\n", eentry->count);
                LOGK("    system 0x%x\n", eentry->system);
            }
        }
    }
}
```
在做完硬盘识别后既可以进行分区的初始化。在分区初始化中，我们就可以读取到分区信息。其中第一个分区开始的位置是2048扇区，也就是从1M的位置开始进行分区。这1M的保留空间用来被用于重要的系统结构和兼容性目的，可以存储额外的引导程序，这里不做过多介绍。

## 五、虚拟设备
虚拟设备是对硬件设备进行一层抽象，使得读写更加的统一，方便以后的操作。
```cpp
#define NAMELEN 16   // 设备数量，最多64个
enum device_type_t  // 设备类型
{
    DEV_NULL,  // 空设备
    DEV_CHAR,  // 字符设备 以char为单位进行读写，如键盘、控制台
    DEV_BLOCK, // 块设备 以块为单位进行读写，如磁盘以扇区为单位读写
};

enum device_subtype_t  // 设备子类型
{
    DEV_CONSOLE = 1, // 控制台
    DEV_KEYBOARD,    // 键盘
};

typedef struct device_t
{
    char name[NAMELEN]; // 设备名
    int type;           // 设备类型
    int subtype;        // 设备子类型
    dev_t dev;          // 设备号
    dev_t parent;       // 父设备号
    void *ptr;          // 设备指针
    
    int (*ioctl)(void *dev, int cmd, void *args, int flags);  // 设备控制
    int (*read)(void *dev, void *buf, size_t count, idx_t idx, int flags);  // 读设备
    int (*write)(void *dev, void *buf, size_t count, idx_t idx, int flags);  // 写设备
} device_t;

dev_t device_install(    // 安装设备
    int type, int subtype,
    void *ptr, char *name, dev_t parent,
    void *ioctl, void *read, void *write);

device_t *device_find(int type, idx_t idx);   // 根据子类型查找设备
device_t *device_get(dev_t dev);          // 根据设备号查找设备
int device_ioctl(dev_t dev, int cmd, void *args, int flags);   // 控制设备
int device_read(dev_t dev, void *buf, size_t count, idx_t idx, int flags);  // 读设备
int device_write(dev_t dev, void *buf, size_t count, idx_t idx, int flags);  // 写设备
```
首先需要对设备进行初始化，即将64个设备结构数组全设置为空。安装设备从数组中获取一个空设备进行结构体的赋值。获取空设备时从1开始，数组0号就放空设备。因为控制台和键盘都属于字符设备，因此控制台的操作需要同步进行调整，例如console_write函数。这样就可以把控制台和键盘进行设备管理：
```cpp
void console_init()
{
    console_clear();
    device_install(
        DEV_CHAR, DEV_CONSOLE,
        NULL, "console", 0,
        NULL, NULL, console_write);
}

static u32 sys_test()
{
    char ch;
    device_t *device;

    device = device_find(DEV_KEYBOARD, 0);
    assert(device);
    device_read(device->dev, &ch, 1, 0, 0);

    device = device_find(DEV_CONSOLE, 0);
    assert(device);
    device_write(device->dev, &ch, 1, 0, 0);
    return 255;
}
```
如上所示，我们在初始化控制台时，就可以把控制台安装到虚拟设备中。同样的键盘初始化也安装到虚拟设备，在0号系统调用就可以测试通过虚拟设备进行读写。


## 六、块设备请求
块设备 (如硬盘，软盘) 的读写以扇区(512B) 为单位，操作比较耗时，需要寻道，寻道时需要旋转磁头臂。所以需要一种策略来完成磁盘的访问。这个策略就是电梯算法，我们在下一篇文章再详细说明。
```cpp
// 执行块设备请求
static void do_request(request_t *req)
{
    switch (req->type)
    {
    case REQ_READ:
        device_read(req->dev, req->buf, req->count, req->idx, req->flags);
        break;
    case REQ_WRITE:
        device_write(req->dev, req->buf, req->count, req->idx, req->flags);
        break;
    default:
        panic("req type %d unknown!!!");
        break;
    }
}

// 块设备请求
void device_request(dev_t dev, void *buf, u8 count, idx_t idx, int flags, u32 type)
{
    device_t *device = device_get(dev);
    assert(device->type = DEV_BLOCK); // 是块设备
    idx_t offset = idx + device_ioctl(device->dev, DEV_CMD_SECTOR_START, 0, 0); // 获取到磁盘扇区的位置

    if (device->parent)  // 有父设备，例如分区的话就找到分区的磁盘，对磁盘操作
    {
        device = device_get(device->parent);
    }

    request_t *req = kmalloc(sizeof(request_t));  // 申请内存存放请求参数
    req->dev = dev;
    req->buf = buf;
    req->count = count;
    req->idx = offset;
    req->flags = flags;
    req->type = type;
    req->task = NULL;

    // 判断列表是否为空
    bool empty = list_empty(&device->request_list);

    // 将请求压入链表
    list_push(&device->request_list, &req->node);

    // 如果列表不为空，则阻塞，因为已经有请求在处理了，等待处理完成； 这块是压入前判断的是否为空
    if (!empty)
    {
        req->task = running_task();
        task_block(req->task, NULL, TASK_BLOCKED);
    }

    do_request(req);  // 执行请求

    list_remove(&req->node);
    kfree(req);

    if (!list_empty(&device->request_list))
    {
        // 先来先服务
        request_t *nextreq = element_entry(request_t, node, device->request_list.tail.prev);
        assert(nextreq->task->magic == ONIX_MAGIC);
        task_unblock(nextreq->task);
    }
}

static u32 sys_test()
{
    char ch;
    device_t *device;

    void *buf = (void *)alloc_kpage(1);
    device = device_find(DEV_IDE_PART, 0);  // 查找第一个磁盘分区
    assert(device);
    memset(buf, running_task()->pid, 512); // 设置buf为进程id
    device_request(device->dev, buf, 1, running_task()->pid, 0, REQ_WRITE);  // 将进程id写入磁盘
    free_kpage((u32)buf, 1);

    return 255;
}
```
如上为块设备请求的实现，磁盘就属于块设备。这里我们把磁盘和磁盘分区作为子设备类型安装到虚拟设备， 这里在磁盘初始化时进程完成。我们依然在0号系统调用进行测试，然后开启三个现成来调用test。

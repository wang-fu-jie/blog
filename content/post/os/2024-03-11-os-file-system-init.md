---
title:      自制操作系统 - 文件系统格式化与虚拟磁盘
subtitle:    "文件系统格式化与虚拟磁盘"
description: "前边文章中，我们一直是依赖开发机进行文件系统的格式化，本文我们自行实现文件系统格式化，通过系统调用来实现，这样简单一些，因为已经有了写磁盘的系统调用。另外本文还将实现虚拟磁盘和标准输入输出。虚拟磁盘是将一块内存当做磁盘进行读写，这样设备文件就可以直接写入虚拟磁盘。"
excerpt:     "前边文章中，我们一直是依赖开发机进行文件系统的格式化，本文我们自行实现文件系统格式化，通过系统调用来实现，这样简单一些，因为已经有了写磁盘的系统调用。另外本文还将实现虚拟磁盘和标准输入输出。虚拟磁盘是将一块内存当做磁盘进行读写，这样设备文件就可以直接写入虚拟磁盘。"
date:        2024-03-11T14:15:58+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/d8/a5/brown_bear_grizzly_canada_mammal_animal_brown_bear_wild-943220.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-file-system-init"
categories:  [ "操作系统" ]
---

## 一、文件系统格式化
在 文件系统简介 里，我们知道 minux 系统总共有一下几个组成部分：
* 文件块：用于存储文件内容
* inode 块：用于存储 inode，inode 中是一个文件或目录的信息
* 块位图：用于表示哪些文件块被占用
* inode 位图：用于表示哪些 inode 被占用
* 超级块：用于描述以上四部分的位置，超级块位于第 1 块
* 引导块：其中有主引导扇区，是第 0 块

在对一个文件系统进行初始化，需要把以上信息写入磁盘。我们这里通过一个系统调用来实现文件系统的格式化。如下为代码实现：
```cpp
int sys_mkfs(char *devname, int icount)  // mkfs系统调用的内核实现， icount参数是需要格式化的inode数量
{
    inode_t *inode = NULL;
    int ret = EOF;
    inode = namei(devname);    // 获取设备文件的inode
    if (!inode)
        goto rollback;
    if (!ISBLK(inode->desc->mode))   // 确保dev必须是块设备
        goto rollback;
    dev_t dev = inode->desc->zone[0];   // 获取到设备号
    assert(dev);
    ret = devmkfs(dev, icount);   // 进行文件系统的格式化

rollback:
    iput(inode);
    return ret;
}

int devmkfs(dev_t dev, u32 icount)     // 文件系统格式化函数，这个函数后续还会复用。因此单独封装
{
    super_block_t *sb = NULL;
    buffer_t *buf = NULL;
    int ret = EOF;
    int total_block = device_ioctl(dev, DEV_CMD_SECTOR_COUNT, NULL, 0) / BLOCK_SECS; // 获取设备块的总数量
    assert(total_block);
    assert(icount < total_block);
    if (!icount)   // 如果需要格式化的inode数量为0， 就赋予总块数除以3。 这个数是随便给的没有特殊意义。
    {
        icount = total_block / 3;
    }

    sb = get_free_super();   // 申请一个空的超级块，并进行初始化
    sb->dev = dev;
    sb->count = 1;
    buf = bread(dev, 1);  // 读取超级块
    sb->buf = buf;
    buf->dirty = true;

    super_desc_t *desc = (super_desc_t *)buf->data;   // 初始化超级块
    sb->desc = desc;                                  // 得到超级块描述符
    int inode_blocks = div_round_up(icount * sizeof(inode_desc_t), BLOCK_SIZE);  // 计算inode块占用的块数量
    desc->inodes = icount;
    desc->zones = total_block;
    desc->imap_blocks = div_round_up(icount, BLOCK_BITS);    // 计算imap占用的块数量
    int zcount = total_block - desc->imap_blocks - inode_blocks - 2;   // 剩余的为文件块的数量
    desc->zmap_blocks = div_round_up(zcount, BLOCK_BITS);    // 计算zmap占用的块数量
    desc->firstdatazone = 2 + desc->imap_blocks + desc->zmap_blocks + inode_blocks;  // 计算第一个文件块的索引
    desc->log_zone_size = 0;  // 每逻辑块数据块数为0 ，也就是2的0次方，即1个zone位一块
    desc->max_size = BLOCK_SIZE * TOTAL_BLOCK;   // 单个文件的最大大小
    desc->magic = MINIX1_MAGIC;   // 文件系统魔数

    memset(sb->imaps, 0, sizeof(sb->imaps));  // 清空位图
    memset(sb->zmaps, 0, sizeof(sb->zmaps));
    int idx = 2;
    for (int i = 0; i < sb->desc->imap_blocks; i++)    // 把位图读出并全部置为0
        if ((sb->imaps[i] = bread(dev, idx)))
        {
            memset(sb->imaps[i]->data, 0, BLOCK_SIZE);
            sb->imaps[i]->dirty = true;
            idx++;
        }
        else
            break;
    for (int i = 0; i < sb->desc->zmap_blocks; i++)
        if ((sb->zmaps[i] = bread(dev, idx)))
        {
            memset(sb->zmaps[i]->data, 0, BLOCK_SIZE);
            sb->zmaps[i]->dirty = true;
            idx++;
        }
        else
            break;

    // 初始化位图
    idx = balloc(dev);   // 块位图的第一块被占用，这是根目录
    idx = ialloc(dev);   // inode位图的第0块和第1块被占用
    idx = ialloc(dev);

    int counts[] = {    // 位图尾部置位， 尾部一些没有用到的位，置为1
        icount + 1,
        zcount,
    };

    buffer_t *maps[] = {
        sb->imaps[sb->desc->imap_blocks - 1],
        sb->zmaps[sb->desc->zmap_blocks - 1],
    };
    for (size_t i = 0; i < 2; i++)   // 这里是尾部置为的操作
    {
        int count = counts[i];
        buffer_t *map = maps[i];
        map->dirty = true;
        int offset = count % (BLOCK_BITS);
        int begin = (offset / 8);
        char *ptr = (char *)map->data + begin;
        memset(ptr + 1, 0xFF, BLOCK_SIZE - begin - 1);
        int bits = 0x80;
        char data = 0;
        int remain = 8 - offset % 8;
        while (remain--)
        {
            data |= bits;
            bits >>= 1;
        }
        ptr[0] = data;
    }

    // 创建根目录 
    task_t *task = running_task();    // 获取当前进程
    inode_t *iroot = new_inode(dev, 1);   // 申请根节点的inode
    sb->iroot = iroot;
    iroot->desc->mode = (0777 & ~task->umask) | IFDIR;
    iroot->desc->size = sizeof(dentry_t) * 2; // 当前目录和父目录两个目录项
    iroot->desc->nlinks = 2;                  // 一个是 '.' 一个是 name

    buf = bread(dev, bmap(iroot, 0, true));   // 读取根目录的第0块。
    buf->dirty = true;
    dentry_t *entry = (dentry_t *)buf->data;
    memset(entry, 0, BLOCK_SIZE);
    strcpy(entry->name, ".");   // 写入 . 代表当前目录
    entry->nr = iroot->nr;

    entry++;
    strcpy(entry->name, "..");   // 写入 .. 当前父目录，因为是根目录，父目录也是当前目录
    entry->nr = iroot->nr;
    brelse(buf);
    ret = 0;
rollback:
    put_super(sb);
    return ret;
}
```
完成文件系统格式化的系统调用，就可以对从盘上的分区进行格式化了。这块可以测试验证，格式化后盘内数据就丢失了，挂载后可以重新写入新数据。


## 二、虚拟磁盘
虚拟磁盘是用内存模拟磁盘，以创建临时的文件系统，前边我们设备文件直接写入到了磁盘，这样是不对的。因为不同的机器设备可能是变化的， 因此需要在启动时区检测设备和初始化设备。 因此 /dev 应该使用虚拟磁盘。系统掉电之后这些设备文件也不会写入磁盘。我们这里再看一下内存布局：
![图片加载失败](/post_images/os/{{< filename >}}/2-01.png)
如图，我们使用 0xC00000 ~ 0x1000000 这 4M 的内存位置来做虚拟磁盘。

### 2.1、虚拟磁盘实现
我们给虚拟磁盘预留了4MB的空间，这次创建4个虚拟磁盘，每个大小1MB。代码如下：
```cpp
#define RAMDISK_NR 4   // 创建四个虚拟磁盘
typedef struct ramdisk_t
{
    u8 *start; // 内存开始位置
    u32 size;  // 占用内存大小
} ramdisk_t;
static ramdisk_t ramdisks[RAMDISK_NR];  // 虚拟磁盘表

int ramdisk_ioctl(ramdisk_t *disk, int cmd, void *args, int flags)  // 虚拟磁盘设备控制，只支持两个命令
{
    switch (cmd)
    {
    case DEV_CMD_SECTOR_START:  // 获取虚拟磁盘扇区的开始位置，固定为0
        return 0;
    case DEV_CMD_SECTOR_COUNT:   // 返回虚拟磁盘的大小
        return disk->size / SECTOR_SIZE;
    default:
        panic("device command %d can't recognize!!!", cmd);
        break;
    }
}

int ramdisk_read(ramdisk_t *disk, void *buf, u8 count, idx_t lba)  // 虚拟磁盘读
{
    void *addr = disk->start + lba * SECTOR_SIZE;   // 获取读取的偏移位置
    u32 len = count * SECTOR_SIZE;          // 获取读取的长度
    assert(((u32)addr + len) < (KERNEL_RAMDISK_MEM + KERNEL_MEMORY_SIZE));
    memcpy(buf, addr, len);  // 虚拟磁盘直接进行内存拷贝
    return count;
}

int ramdisk_write(ramdisk_t *disk, void *buf, u8 count, idx_t lba)   // 虚拟磁盘写
{
    void *addr = disk->start + lba * SECTOR_SIZE;
    u32 len = count * SECTOR_SIZE;
    assert(((u32)addr + len) < (KERNEL_RAMDISK_MEM + KERNEL_MEMORY_SIZE));
    memcpy(addr, buf, len);
    return count;
}

void ramdisk_init()    // 虚拟磁盘初始化
{
    LOGK("ramdisk init...\n");
    u32 size = KERNEL_RAMDISK_SIZE / RAMDISK_NR;  // 大小为1MB
    char name[32];
    for (size_t i = 0; i < RAMDISK_NR; i++)
    {
        ramdisk_t *ramdisk = &ramdisks[i];
        ramdisk->start = (u8 *)(KERNEL_RAMDISK_MEM + size * i);   // 设置虚拟磁盘的内存开始位置
        ramdisk->size = size;
        sprintf(name, "md%c", i + 'a');
        device_install(DEV_BLOCK, DEV_RAMDISK, ramdisk, name, 0,     // 安装虚拟磁盘设备
                       ramdisk_ioctl, ramdisk_read, ramdisk_write);
    }
}
```
完成虚拟磁盘的初始化，接下来需要修改设备文件初始化的代码。我们之前把设备文件直接写入了磁盘，这里需要改到虚拟磁盘上。
```cpp
void dev_init()
{
    mkdir("/dev", 0755);    // 创建/dev文件夹，这个文件夹在磁盘上
    device_t *device = NULL;

    // 第一个虚拟磁盘作为 /dev 文件系统
    device = device_find(DEV_RAMDISK, 0);
    assert(device);
    devmkfs(device->dev, 0);   // 对虚拟磁盘进行格式化

    super_block_t *sb = read_super(device->dev);   // 挂载虚拟磁盘到/dev目录
    sb->iroot = iget(device->dev, 1);
    sb->imount = namei("/dev");
    sb->imount->mount = device->dev;
    ......
}
```
接下来就可以创建设备文件了。这样设备文件就会直接写在内存中。


## 三、标准输入输出
前边我们直接使用了整形数字0、1、2代表了stdin、stdour、stderr。 其实标准输入输出应该是file_t结构。
```cpp
typedef enum std_fd_t
{
    STDIN_FILENO,   // 标准输入输出文件描述符
    STDOUT_FILENO,
    STDERR_FILENO,
} std_fd_t;
```
如上定义了标准输入输出的文件描述符，分别为0,1,2。接下来在创建进程的函数中，需要定义标准输入输出文件结构。如下：
```cpp
extern file_t file_table[];   // 声明系统文件表，
static task_t *task_create(target_t target, const char *name, u32 priority, u32 uid)
{
    task_t *task = get_free_task();
    ......
    task->umask = 0022; // 对应 0755
    task->files[STDIN_FILENO] = &file_table[STDIN_FILENO];
    task->files[STDOUT_FILENO] = &file_table[STDOUT_FILENO];
    task->files[STDERR_FILENO] = &file_table[STDERR_FILENO];
    task->files[STDIN_FILENO]->count++;
    task->files[STDOUT_FILENO]->count++;
    task->files[STDERR_FILENO]->count++;
    task->magic = ONIX_MAGIC;
    return task;
}
```
在创建进程时设置进程文件表的前三个元素分别为标准输入、标准输出、标准错误输出。在文件表进行初始化时，从3号索引元素开始，避免覆盖标准输入输出。同时需要调用文件读写的系统调用实现，对块设备读写和字符设备字符统一接口：
```cpp
int sys_read(fd_t fd, char *buf, int count)  // 文件读系统调用
{
    task_t *task = running_task();  // 获取要进行读写的文件结构
    file_t *file = task->files[fd];
    int len = 0;
    if ((file->flags & O_ACCMODE) == O_WRONLY)
        return EOF;

    inode_t *inode = file->inode;
    if (ISCHR(inode->desc->mode))  // 如果是字符设备，就获取设备号，通过设备号调用字符设备的读接口
    {
        assert(inode->desc->zone[0]);
        len = device_read(inode->desc->zone[0], buf, count, 0, 0);
        return len;
    }
    else if (ISBLK(inode->desc->mode))     // 如果是块设备，就获取块设备的设备号。并进行文件块的读
    {
        assert(inode->desc->zone[0]);
        device_t *device = device_get(inode->desc->zone[0]);
        assert(file->offset % BLOCK_SIZE == 0);
        assert(count % BLOCK_SIZE == 0);
        len = device_read(inode->desc->zone[0], buf, count / BLOCK_SIZE, file->offset / BLOCK_SIZE, 0);
        return len;
    }
    else
    {
        len = inode_read(inode, buf, count, file->offset);
    }
    if (len != EOF)
    {
        file->offset += len;
    }
    return len;
}
```
如上为文件读系统调用的封装，写操作时类似的，我们这里不再展示代码实现。最后，还需要创建标准输入输出的设备文件。在dev_init中添加如下代码段：
```cpp
void dev_init()
{
    link("/dev/console", "/dev/stdout");
    link("/dev/console", "/dev/stderr");
    link("/dev/keyboard", "/dev/stdin");

    file_t *file;
    inode_t *inode;
    file = &file_table[STDIN_FILENO];
    inode = namei("/dev/stdin");
    file->inode = inode;
    file->mode = inode->desc->mode;
    file->flags = O_RDONLY;
    file->offset = 0;

    file = &file_table[STDOUT_FILENO];
    inode = namei("/dev/stdout");
    file->inode = inode;
    file->mode = inode->desc->mode;
    file->flags = O_WRONLY;
    file->offset = 0;

    file = &file_table[STDERR_FILENO];
    inode = namei("/dev/stderr");
    file->inode = inode;
    file->mode = inode->desc->mode;
    file->flags = O_WRONLY;
    file->offset = 0;
}
```
如上所示，就设置了标准输入输出的设备文件，通过读写设备文件就可以实现对标准输入输出的读写。 例如读取键盘输入，就可以读文件/dev/stdin。例如使用cat /dev/stdin。 会打开标准输入的文件描述符，并使用read系统调用去读取 0 号文件描述符。sys_read就会判断是字符设备，进行去进行键盘读取。cat 命令再把读取内容输出到屏幕即标准输出。
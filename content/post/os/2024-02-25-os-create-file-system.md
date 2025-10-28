---
title:       自制操作系统 - 创建文件系统
subtitle:    "创建文件系统"
description: "为简单起见，我们使用 minix 第一版文件系统。我们优先使用现成的fdisk，mkfs命令等虚拟磁盘进行分区和格式化。在本文中介绍块、inode、超级块、目录、根超级块等概念。并对文件系统进行代码实现。"
excerpt:     "为简单起见，我们使用 minix 第一版文件系统。我们优先使用现成的fdisk，mkfs命令等虚拟磁盘进行分区和格式化。在本文中介绍块、inode、超级块、目录、根超级块等概念。并对文件系统进行代码实现。"
date:        2024-02-25T11:18:04+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/37/46/tricycle_chain_sidewalk_wheel-101.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-create-file-system"
categories:  [ "操作系统" ]
---

## 一、文件系统创建
为简单起见，我们使用 minix 第一版文件系统。首先我们创建两个虚拟磁盘，master.img 和 slave.img。每个磁盘创建一个主分区。创建文件系统指令如下：
```shell
sudo mkfs.minix -1 -n 14 /dev/loop0p1
```
这里我们是依赖已有操作系统的fdisk、 mkfs、mount、umount、losetup等命令进行磁盘的分区、格式化与挂载。


## 二、文件系统简介
一般常见的文件系统分为两种类型：文件分配表 (File Allocation Table FAT) 和 索引表。 minux 文件系统使用索引表结构。

## 2.1、块的概念
文件系统把硬件分成了许多块。一块是两个扇区。如下所示：
![图片加载失败](/post_images/os/{{< filename >}}/2.1-01.png)
文件就存储在这些块内，文件系统有很多文件，因此需要记录每个文件存在于哪些块中。当一个文件使用多个块时，就需要对这些块进行管理。因此产生了两种管理块的方案，第一个是文件分配表，它的逻辑是每一块最后有一个指针指向下一块。第二个是索引表，使用一个inode结构指示下一个块的记录的位置。

## 2.2、inode
inode用于记录一个文件存在于哪些块中，它的结构如下：
```cpp
typedef struct inode_desc_t
{
    u16 mode;    // 文件类型和属性(rwx 位)
    u16 uid;     // 用户id（文件拥有者标识符）
    u32 size;    // 文件大小（字节数）
    u32 mtime;   // 修改时间戳 这个时间戳应该用 UTC 时间，不然有瑕疵
    u8 gid;      // 组id(文件拥有者所在的组)
    u8 nlinks;   // 链接数（多少个文件目录项指向该i 节点）
    u16 zone[9]; // 直接 (0-6)、间接(7)或双重间接 (8) 逻辑块号， 记录文件数据所在的磁盘块的编号
} inode_desc_t;
```
文件有大有小，对于很小的文件，即小于1kb的文件，一个块用来存储就足够了。但是对于很大的文件，需要用到多个块来存储。它使用块的存储如下：
![图片加载失败](/post_images/os/{{< filename >}}/2.2-01.png)
如上所示，当一个文件大小超过7个块时，前7个是直接存储的块，第8个块是1级间接块，一块是1024字节，块的索引是u16，因此一级间接块可以扩充512个块，如果仍然不够，就继续使用二级间接块。磁盘被划分为若干个固定大小的块（block），比如 1KB、2KB，zone 里存的就是这些块的编号。

inode本身也存在于文件系统中，因此需要一些块来存储inode信息。因此文件系统需要分为两部分，一部分存储文件，一部分存储inode。因此需要用到一个超级块，来记录一个文件系统有多少inode块，多少文件块。

### 2.3、超级块
超级块用来记录一个文件系统，有多少个inode和逻辑块，它的结构如下：
```cpp
typedef struct super_desc_t
{
    u16 inodes;        // 节点数
    u16 zones;         // 逻辑块数
    u16 imap_blocks;   // inode 节点位图所占用的数据块数
    u16 zmap_blocks;   // 逻辑块位图所占用的数据块数
    u16 firstdatazone; // 第一个数据逻辑块号
    u16 log_zone_size; // log2(每逻辑块数据块数)
    u32 max_size;      // 文件最大长度
    u16 magic;         // 文件系统魔数
} super_desc_t;
```
如上，超级块中使用了两个位图，分别记录inode和逻辑块哪些被占用了。超级块位于文件系统的第一块，因为第0块是主引导块。
![图片加载失败](/post_images/os/{{< filename >}}/2.3-01.png)

### 2.4、目录
如果inode 文件 是目录，那么文件块的内容就是下面这种数据结构。
```cpp
// 文件目录项结构
typedef struct dentry_t
{
    u16 nr;        // i 节点
    char name[14]; // 文件名
} dentry_t;
```
当一个分区被格式化为文件系统后，第一个inode就是根目录，根目录的 inode 号是固定的 1。当文件系统挂载时，会直接读取 inode 号 1，作为起点遍历整个目录树

## 三、文件系统的实现
我们这里使用代码来实现一下文件系统的读写，代码如下：
```cpp
void super_init()
{
    device_t *device = device_find(DEV_IDE_PART, 0);  // 找到第一个主分区(这是分区的主引导扇区，并不是磁盘的主引导扇区)
    buffer_t *boot = bread(device->dev, 0);           // 读取主引导块，这是boot都是0，因为这不是硬盘的主引导扇区
    buffer_t *super = bread(device->dev, 1);          // 读取超级块
    super_desc_t *sb = (super_desc_t *)super->data;   // 超级块的data就是超级块的描述符 （这里是buffer的结构体）
    buffer_t *imap = bread(device->dev, 2);   // 读取inode 位图
    buffer_t *zmap = bread(device->dev, 2 + sb->imap_blocks);  // 读取块位图

    // 读取第一个 inode 块。也就是根目录
    buffer_t *buf1 = bread(device->dev, 2 + sb->imap_blocks + sb->zmap_blocks);
    inode_desc_t *inode = (inode_desc_t *)buf1->data;
    buffer_t *buf2 = bread(device->dev, inode->zone[0]);  // 读取第一个inode下的文件块，即根目录下的信息。

    dentry_t *dir = (dentry_t *)buf2->data;   // 这里是根目录下的文件信息
    inode_desc_t *helloi = NULL;
    while (dir->nr)
    {
        LOGK("inode %04d, name %s\n", dir->nr, dir->name);
        if (!strcmp(dir->name, "hello.txt"))   // 查找hello.txt 文件
        {
            helloi = &((inode_desc_t *) buf1->data)[dir->nr - 1];  // dir->nr就是文件节点的编号 获取该文件的inode,并修改名字写会磁盘
            strcpy(dir->name, "world.txt");
            buf2->dirty = true;
            bwrite(buf2);
        }
        dir++;
    }

    buffer_t *buf3 = bread(device->dev, helloi->zone[0]);  // 修改文件的内容
    LOGK("content %s", buf3->data);

    strcpy(buf3->data, "This is modified content!!!\n");
    buf3->dirty = true;
    bwrite(buf3);

    helloi->size = strlen(buf3->data);  // 更新文件的长度等信息
    buf1->dirty = true;
    bwrite(buf1);
}
```
接下来我们来调试一下这段代码，结果如下：
![图片加载失败](/post_images/os/{{< filename >}}/3-01.png)
如上为超级块的信息，inode为5056个，也就是最多5056个文件。第一个数据块在163块。inode位图占了一块，块位图占了两块。


## 四、根超级块
我们先说一下超级块和根超级块，每个分区如果被格式化成某种文件系统，那么它就会在分区里写入一个 超级块，每个分区都有自己的超级块。根超级块是内存里的概念，当操作系统挂载一个文件系统时，会把该分区的超级块从磁盘读出来，加载到内存中，形成一个内核的 super_block 数据结构。例如分区 1 上有 minix 的超级块 → 内存里变成 / 的根超级块。

创建超级块表，以及读取根超级块；这里使用已经创建好的文件系统，后续再自行实现文件系统的格式化等操作。我们这里首先添加两个结构，分为是inode和超级块在内存中的结构，上文中的结构是在磁盘上的结构：
```cpp
typedef struct inode_t   // inode在内存中的结构
{
    inode_desc_t *desc;   // inode 描述符
    struct buffer_t *buf; // inode 描述符对应 buffer
    dev_t dev;            // 设备号
    idx_t nr;             // i 节点号
    u32 count;            // 引用计数
    time_t atime;         // 访问时间
    time_t ctime;         // 创建时间
    list_node_t node;     // 链表结点
    dev_t mount;          // 安装设备
} inode_t;

typedef struct super_block_t   // 超级块在内存中的结构
{
    super_desc_t *desc;              // 超级块描述符
    struct buffer_t *buf;            // 超级块描述符 buffer
    struct buffer_t *imaps[IMAP_NR]; // inode 位图缓冲
    struct buffer_t *zmaps[ZMAP_NR]; // 块位图缓冲
    dev_t dev;                       // 设备号
    list_t inode_list;               // 使用中 inode 链表
    inode_t *iroot;                  // 根目录 inode
    inode_t *imount;                 // 安装到的 inode
} super_block_t;
```
inode在内存中的结构这里我们暂时使用不到。 接下来我们看实现的代码：
```cpp
#define SUPER_NR 16

static super_block_t super_table[SUPER_NR]; // 超级块表, 最多支持挂载16个文件系统。
static super_block_t *root;                 // 根文件系统超级块

static super_block_t *get_free_super()    // 从超级块表中查找一个空闲块
{
    for (size_t i = 0; i < SUPER_NR; i++)
    {
        super_block_t *sb = &super_table[i];
        if (sb->dev == EOF)
        {
            return sb;
        }
    }
    panic("no more super block!!!");
}

super_block_t *get_super(dev_t dev)     // 获得设备 dev 的超级块
{
    for (size_t i = 0; i < SUPER_NR; i++)
    {
        super_block_t *sb = &super_table[i];
        if (sb->dev == dev)
        {
            return sb;
        }
    }
    return NULL;
}

super_block_t *read_super(dev_t dev)  // 读设备 dev 的超级块
{
    super_block_t *sb = get_super(dev);
    if (sb)
    {
        return sb;
    }

    LOGK("Reading super block of device %d\n", dev);

    sb = get_free_super();   // 获得空闲超级块
    buffer_t *buf = bread(dev, 1);   // 读取超级块

    sb->buf = buf;
    sb->desc = (super_desc_t *)buf->data;
    sb->dev = dev;
    assert(sb->desc->magic == MINIX1_MAGIC);
    memset(sb->imaps, 0, sizeof(sb->imaps));
    memset(sb->zmaps, 0, sizeof(sb->zmaps));
    // 读取 inode 位图
    int idx = 2; // 块位图从第 2 块开始，第 0 块 引导块，第 1 块 超级块
    for (int i = 0; i < sb->desc->imap_blocks; i++)
    {
        assert(i < IMAP_NR);
        if ((sb->imaps[i] = bread(dev, idx)))
            idx++;
        else
            break;
    }

    for (int i = 0; i < sb->desc->zmap_blocks; i++)    // 读取超级块
    {
        assert(i < ZMAP_NR);
        if ((sb->zmaps[i] = bread(dev, idx)))
            idx++;
        else
            break;
    }

    return sb;
}

static void mount_root()     // 挂载根文件系统
{
    LOGK("Mount root file system...\n");
    device_t *device = device_find(DEV_IDE_PART, 0);   // 假设主硬盘第一个分区是根文件系统
    assert(device);
    root = read_super(device->dev);   // 读取文件系统超级块
}

void super_init()
{
    for (size_t i = 0; i < SUPER_NR; i++)
    {
        super_block_t *sb = &super_table[i];
        sb->dev = EOF;
        sb->desc = NULL;
        sb->buf = NULL;
        sb->iroot = NULL;
        sb->imount = NULL;
        list_init(&sb->inode_list);
    }

    mount_root();
}
```
超级块表初始化时，所有块的dev都初始化为EOF。再查找空闲块是只要判断dev是EOF那就是空闲块。mount_root用来获取主硬盘的第一个分区作文根文件系统，
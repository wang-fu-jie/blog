---
title:       自制操作系统 - 文件系统相关操作(一)
subtitle:    "文件系统相关操作(一)"
description: "创建完文件系统后，应该实现对文件系统的操作。本节将实现文件系统的位图管理、inode管理以及文件状态的管理，并实现umask系统调用来设置进程的默认权限。"
excerpt:     "创建完文件系统后，应该实现对文件系统的操作。本节将实现文件系统的位图管理、inode管理以及文件状态的管理，并实现umask系统调用来设置进程的默认权限。"
date:        2024-02-28T15:19:47+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/ad/d6/off_road_buggy_motorcycle_jump_bike_motorcycles_speed_motocross_motor-774199.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-file-system-operation"
categories:  [ "操作系统" ]
---

## 一、文件系统位图操作
文件系统位图用于控制inode和文件块的占用情况，标记哪些块被占用了。涉及如下四个函数：
```cpp
idx_t balloc(dev_t dev)    // 分配一个文件块
{
    super_block_t *sb = get_super(dev);  // 获取设备的超级块
    buffer_t *buf = NULL;
    idx_t bit = EOF;  
    bitmap_t map;

    for (size_t i = 0; i < ZMAP_NR; i++)     
    {
        buf = sb->zmaps[i];  // 循环逐个获取文件位图占用的块
        bitmap_make(&map, buf->data, BLOCK_SIZE, i * BLOCK_BITS + sb->desc->firstdatazone - 1); // 将整个缓冲区作为位图
        bit = bitmap_scan(&map, 1);  // 从位图中扫描一位
        if (bit != EOF)
        {
            assert(bit < sb->desc->zones);
            buf->dirty = true;    // 如果扫描成功，则 标记缓冲区脏，中止查找
            break;
        }
    }
    bwrite(buf); // todo 调试期间强同步
    return bit;
}

void bfree(dev_t dev, idx_t idx)    // 释放一个文件块， idx是被释放块的索引
{
    super_block_t *sb = get_super(dev);   // 获取设备的超级块
    buffer_t *buf;
    bitmap_t map;
    for (size_t i = 0; i < ZMAP_NR; i++)
    {
        if (idx > BLOCK_BITS * (i + 1))   // 跳过开始的块
        {
            continue;
        }
        buf = sb->zmaps[i];
        bitmap_make(&map, buf->data, BLOCK_SIZE, BLOCK_BITS * i + sb->desc->firstdatazone - 1);  // 将整个缓冲区作为位图
        assert(bitmap_test(&map, idx));    // 将 idx 对应的位图置位 0
        bitmap_set(&map, idx, 0);
        buf->dirty = true;    // 标记缓冲区脏
        break;
    }
    bwrite(buf); // todo 调试期间强同步
}

idx_t ialloc(dev_t dev)    // 分配一个文件系统 inode
void ifree(dev_t dev, idx_t idx)    // 释放一个文件系统 inode
```
如上所示，四个函数分别用于申请和释放文件块和inode。这四个函数比较简单，通过读取超级块在内存中修改位图来实现文件块或者inode的申请和释放。这里展示的代码只贴了其中一对儿函数的实现，另外一堆逻辑基本是类似的。

## 二、文件系统 inode
在磁盘中，一个inode描述符占32字节，因此一个inode块(1kb)可以存储32个inode。
```cpp
#define INODE_NR 64  // 内核同时能“打开”或“缓存”的 inode 数量是64个
static inode_t inode_table[INODE_NR];  // 定义一个inode表, 这里的inode是内存中的inode结构，而不是磁盘上的inode描述符

void inode_init()    // 初始化inode, 所有inode的设备初始化为EOF
{
    for (size_t i = 0; i < INODE_NR; i++)
    {
        inode_t *inode = &inode_table[i];
        inode->dev = EOF;
    }
}

static inode_t *get_free_inode()    // 从inode表中，申请一个空闲 inode
{
    for (size_t i = 0; i < INODE_NR; i++)
    {
        inode_t *inode = &inode_table[i];
        if (inode->dev == EOF)
        {
            return inode;
        }
    }
    panic("no more inode!!!");
}

static void put_free_inode(inode_t *inode)  // 释放一个 inode，这里省略了断言，即释放的inode引用计数需要为0，且不能释放根inode
{
    inode->dev = EOF;
}

inode_t *get_root_inode()   // 获取根 inode
{
    return inode_table;
}

static inline idx_t inode_block(super_block_t *sb, idx_t nr)  // 计算 inode nr 对应的块号， nr是inode的索引
{
    return 2 + sb->desc->imap_blocks + sb->desc->zmap_blocks + (nr - 1) / BLOCK_INODES;    // inode 编号 从 1 开始，即1号inode是根
}

static inode_t *find_inode(dev_t dev, idx_t nr)   // 从已有 inode 中查找编号为 nr 的 inode
{
    super_block_t *sb = get_super(dev);  // 获取超级块
    list_t *list = &sb->inode_list;

    for (list_node_t *node = list->head.next; node != &list->tail; node = node->next)  // 遍历inode链表
    {
        inode_t *inode = element_entry(inode_t, node, node);
        if (inode->nr == nr)
        {
            return inode;
        }
    }
    return NULL;
}

inode_t *iget(dev_t dev, idx_t nr)   // 获得设备 dev 的 nr inode
{
    inode_t *inode = find_inode(dev, nr);
    if (inode)
    {
        inode->count++;
        inode->atime = time();
        return inode;
    }

    super_block_t *sb = get_super(dev);   // 如果已有inode没找到，则新申请一个inode
    inode = get_free_inode();
    inode->dev = dev;
    inode->nr = nr;
    inode->count = 1;
    list_push(&sb->inode_list, &inode->node);    // 加入超级块 inode 链表

    idx_t block = inode_block(sb, inode->nr);
    buffer_t *buf = bread(inode->dev, block);
    inode->buf = buf;

    // 将缓冲视为一个 inode 描述符数组，获取对应的指针；
    inode->desc = &((inode_desc_t *)buf->data)[(inode->nr - 1) % BLOCK_INODES]; 
    inode->ctime = inode->desc->mtime;
    inode->atime = time();
    return inode;
}

void iput(inode_t *inode)   // 释放 inode
{
    if (!inode)
        return;
    inode->count--;   // 引用计数减1
    if (inode->count)
    {
        return;
    }
    brelse(inode->buf);   // 释放 inode 对应的缓冲
    list_remove(&inode->node);   // 从超级块链表中移除
    put_free_inode(inode);   // 释放 inode 内存
}
```
如上所示，为inode的管理，在内存中使用一个数组，最大管理64个inode。 这是还有一个重点是根inode, 它永远inode表的第一个元素。因此在挂载根文件系统时应该初始化根文件目录。
```cpp
// 挂载根文件系统
static void mount_root()
{
    LOGK("Mount root file system...\n");
    device_t *device = device_find(DEV_IDE_PART, 0);  // 假设主硬盘第一个分区是根文件系统
    assert(device);
    root = read_super(device->dev);   // 读根文件系统超级块
    // 初始化根目录 inode
    root->iroot = iget(device->dev, 1);  // 获得根目录 inode
    root->imount = iget(device->dev, 1); // 根目录挂载 inode
}
```
这里的iroot和imount分别是进程所见的根目录（/）和件系统被挂载到的目录 inode。一般情况下，iroot 就是系统的全局根 inode。正常情况下：所有进程的 / 都指向系统根 inode。日常使用一般也不会使用chroot来修改进程所见的根目录。

最后还有一个很重要的函数，获取 inode 指定块的索引值，我们还是结合代码来分析：
```cpp
// 获取 inode 第 block 块的索引值, 如果不存在 且 create 为 true，则创建
idx_t bmap(inode_t *inode, idx_t block, bool create)
{
    assert(block >= 0 && block < TOTAL_BLOCK);   // block是逻辑块号， 确保 block 合法
    u16 index = block;   // 数组索引
    u16 *array = inode->desc->zone;   // inode的描述符对应的zone。zone决定了inde使用了哪些文件块
    buffer_t *buf = inode->buf;   // 缓冲区
    buf->count += 1;    // 用于下面的 brelse，传入参数 inode 的 buf 不应该释放
    int level = 0;    // 当前处理级别
    int divider = 1;   // 当前子级别块数量
    if (block < DIRECT_BLOCK)   // 直接块, 如果小于7块，那就是使用直接块。
    {
        goto reckon;
    }

    block -= DIRECT_BLOCK;  // 如果大于7， 那就是间接块
    if (block < INDIRECT1_BLOCK)   // 判断是否是1级间接块
    {
        index = DIRECT_BLOCK;
        level = 1;
        divider = 1;
        goto reckon;
    }
    block -= INDIRECT1_BLOCK;   // 否则就是二级间接块
    index = DIRECT_BLOCK + 1;
    level = 2;
    divider = BLOCK_INDEXES;

reckon:
    for (; level >= 0; level--)  
    {
        if (!array[index] && create)   // 如果不存在 且 create 则申请一块文件块
        {
            array[index] = balloc(inode->dev);  // 分配一个物理块
            buf->dirty = true;
        }
        brelse(buf);

        if (level == 0 || !array[index])  // 如果 level == 0 或者 索引不存在，直接返回
        {
            return array[index];
        }

        buf = bread(inode->dev, array[index]);   // level 不为 0，处理下一级索引
        index = block / divider;    // 计算在下一级块中的索引
        block = block % divider;    // 余数留给更下一级
        divider /= BLOCK_INDEXES;   // 每向下一级，divider缩小
        array = (u16 *)buf->data;   // array改为指向下一级索引块的数据区
    }
}
```
如上所示，我们假设申请第三个逻辑块，这里属于直接块。这里会直接申请一个块并返回。并给zone[3]这里赋值为申请到的逻辑块号。这里提一点，zone是u16的数字，因此逻辑块最多65536个，也就是一个文件系统最大支持到64MB。间接块的逻辑可自行调试。

## 三、文件系统的状态
文件系统的状态指的是文件的访问权限，文件的属主，已经文件的类型等信息。也就是在linux上使用 ls -l 输出的内容。例如 rwx 表示文件权限，d代表目录，这里我们不做过多的介绍。minux 的文件信息存储在 inode.mode 字段中，总共有 16 位，其中：
* 高 4 位用于表示文件类型
* 中 3 位用于表示特殊标志
* 低 9 位用于表示文件权限

这里需要做一些宏定义，用来表示文件类型，权限等内容。如：
```cpp
// 文件类型
#define IFMT 00170000 // 文件类型（8 进制表示）
#define IFREG 0100000 // 常规文件
#define IFBLK 0060000 // 块特殊（设备）文件，如磁盘 dev/fd0
#define IFDIR 0040000 // 目录文件
#define IFCHR 0020000 // 字符设备文件
#define IFIFO 0010000 // FIFO 特殊文件
#define IFSYM 0120000 // 符号连接

typedef struct stat_t
{
    dev_t dev;    // 含有文件的设备号
    idx_t nr;     // 文件 i 节点号
    u16 mode;     // 文件类型和属性
    u8 nlinks;    // 指定文件的连接数
    u16 uid;      // 文件的用户(标识)号
    u8 gid;       // 文件的组号
    dev_t rdev;   // 设备号(如果文件是特殊的字符文件或块文件)
    size_t size;  // 文件大小（字节数）（如果文件是常规文件）
    time_t atime; // 上次（最后）访问时间
    time_t mtime; // 最后修改时间
    time_t ctime; // 最后节点修改时间
} stat_t;
```
在这里高4为表示文件类型，这里罗列一些文件类型的宏定义，权限的宏定义就不再展开说明。stat_t结构是用来描述文件状态，包含文件的属性等信息。


## 四、umask系统调用
umask用来设置系统的默认文件权限，这里需要通过系统调用来实现。在创建任务时设置默认 task->umask = 0022。 如下为uamsk的函数实现：
```cpp
mode_t sys_umask(mode_t mask)
{
    task_t *task = running_task();
    mode_t old = task->umask;
    task->umask = mask & 0777;
    return old;
}
```
当用户态程序执行umask()时，就会修改当前进程的默认umask。

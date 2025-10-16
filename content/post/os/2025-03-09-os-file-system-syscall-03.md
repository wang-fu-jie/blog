---
title:       自制操作系统 - 文件系统相关系统调用(三)
subtitle:    "文件系统相关系统调用(三)"
description: ""
excerpt:     ""
date:        2024-03-09T17:45:39+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/e2/e3/tropical_palm_trees_woman_happy_happiness_palm_tree_vacation-595403.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-file-system-syscall-03"
categories:  [ "操作系统" ]
---

## 一、获取文件状态
文件状态指的是文件的属性、大小、最近访问时间、最后修改时间、所属用户、 所属用户组等信息，包含两个函数，一个是通过文件名获取文件状态，一个是通过文件描述符获取文件状态。当ls命令加上 -l 参数时，就可以获取文件状态并打印。
```cpp
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

int sys_stat(char *filename, stat_t *statbuf)
{
    inode_t *inode = namei(filename);   // 获取文件状态
    if (!inode)
    {
        return EOF;
    }
    copy_stat(inode, statbuf);  // 从内核中 拷贝inode状态到用户态。
    iput(inode);
    return 0;
}
```
完成文件状态调用时，修改ls内建命令的实现，就可以使用-l参数来获取文件的详细信息了。

## 二、创建设备文件
mknod 是用来创建设备文件的。在linux系统上，可以在/dev目录下看到许多的设备文件，这些都是通过mknod系统调用实现的。
```cpp
int sys_mknod(char *filename, int mode, int dev)  // 传入文件名，权限和设备
{
    char *next = NULL;
    inode_t *dir = NULL;
    buffer_t *buf = NULL;
    inode_t *inode = NULL;
    int ret = EOF;
    dir = named(filename, &next);  // 获取父目录的inode
    // ...... 权限等信息的判断

    char *name = next;
    dentry_t *entry;
    buf = find_entry(&dir, name, &next, &entry);
    if (buf) // 目录项存在
        goto rollback;

    buf = add_entry(dir, name, &entry);  // 如果不存在添加目录项
    buf->dirty = true;
    entry->nr = ialloc(dir->dev);
    inode = new_inode(dir->dev, entry->nr);  // 申请一个inode
    inode->desc->mode = mode;
    if (ISBLK(mode) || ISCHR(mode))  // 如果是块设备或字符设备，就把设备直接写入inode的zone[0]
        inode->desc->zone[0] = dev;
    ret = 0;

rollback:
    brelse(buf);
    iput(inode);
    iput(dir);
    return ret;
}
```
在完成mknod系统调用后，就可以进行设备的初始化了。如下：
```cpp
void dev_init()
{
    mkdir("/dev", 0755);  // 创建 /dev 目录
    device_t *device = NULL;
    device = device_find(DEV_CONSOLE, 0);   // 查找控制台和键盘设备，并创建文件
    mknod("/dev/console", IFCHR | 0200, device->dev);
    device = device_find(DEV_KEYBOARD, 0);
    mknod("/dev/keyboard", IFCHR | 0400, device->dev);

    char name[32];
    for (size_t i = 0; true; i++)  // 初始化磁盘设备
    {
        device = device_find(DEV_IDE_DISK, i);
        if (!device)
            break;
        sprintf(name, "/dev/%s", device->name);
        mknod(name, IFBLK | 0600, device->dev);
    }

    for (size_t i = 0; true; i++)
    {
        device = device_find(DEV_IDE_PART, i);  // 初始化分区设备
        if (!device)
        {
            break;
        }
        sprintf(name, "/dev/%s", device->name);
        mknod(name, IFBLK | 0600, device->dev);
    }
}
```
在完成设备初始化后，就会在/dev目录下看到设备文件。目前我们是直接把这些设备文件直接写到了磁盘，这些设备是无法使用，暂时先这样。

## 三、设备挂载与卸载
设备挂载与卸载使用到的系统调用时 mount 和 umount。
```cpp
int sys_mount(char *devname, char *dirname, int flags)  // 设备挂载
{
    LOGK("mount %s to %s\n", devname, dirname);   
    inode_t *devinode = NULL;
    inode_t *dirinode = NULL;
    super_block_t *sb = NULL;
    devinode = namei(devname);
    if (!devinode)
        goto rollback;
    if (!ISBLK(devinode->desc->mode))
        goto rollback;

    dev_t dev = devinode->desc->zone[0];
    dirinode = namei(dirname);
    if (!dirinode)
        goto rollback;
    if (!ISDIR(dirinode->desc->mode))
        goto rollback;
    if (dirinode->count != 1 || dirinode->mount)
        goto rollback;

    sb = read_super(dev);
    if (sb->imount)
        goto rollback;
    sb->iroot = iget(dev, 1);
    sb->imount = dirinode;
    dirinode->mount = dev;
    iput(devinode);
    return 0;
rollback:
    put_super(sb);
    iput(devinode);
    iput(dirinode);
    return EOF;
}
```
卸载和挂载是相反的操作，这里不再提供卸载的代码展示。

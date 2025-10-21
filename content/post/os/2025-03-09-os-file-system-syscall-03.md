---
title:       自制操作系统 - 文件系统相关系统调用(三)
subtitle:    "文件系统相关系统调用(三)"
description: "完成了一系列文件系统的读写操作，最后一个需要实现的系统调用时文件状态的获取，和文件系统的挂载卸载。本文重点进行文件系统的挂载，涉及mount和umount两个系统调用。"
excerpt:     "完成了一系列文件系统的读写操作，最后一个需要实现的系统调用时文件状态的获取，和文件系统的挂载卸载。本文重点进行文件系统的挂载，涉及mount和umount两个系统调用"
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
    copy_stat(inode, statbuf);  // 从内核中 拷贝inode的状态到用户态。
    iput(inode);
    return 0;
}
```
完成文件状态调用时，修改ls内建命令的实现，就可以使用-l参数来获取文件的详细信息了。通过文件描述符获取文件状态基本是一样的，通过文件描述符获取到文件结构，再获取到inode，最后拷贝文件状态到内核态。

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
    devinode = namei(devname);   // 查找设备文件，如果不存在或者不是块设备就结束
    if (!devinode)
        goto rollback;
    if (!ISBLK(devinode->desc->mode))
        goto rollback;

    dev_t dev = devinode->desc->zone[0];  // 找到设备号
    dirinode = namei(dirname);   // 查找被挂载点， 如果不存在或者不是目录就结束
    if (!dirinode)
        goto rollback;
    if (!ISDIR(dirinode->desc->mode))
        goto rollback;
    if (dirinode->count != 1 || dirinode->mount)  // 如果目录的引用不是1 或者 该目录已经被挂载了就结束。
        goto rollback;

    sb = read_super(dev);  // 读取设备的超级块
    if (sb->imount)
        goto rollback;
    sb->iroot = iget(dev, 1);   // 将根目录设置为这个设备的第一个inode
    sb->imount = dirinode;      // 这个设备的imount设置为被挂载点的inode
    dirinode->mount = dev;      // 目录的mount属性设置为设备号
    iput(devinode);             // 释放设备的inode
    return 0;
rollback:
    put_super(sb);
    iput(devinode);
    iput(dirinode);
    return EOF;
}
```
卸载和挂载是相反的操作，这里不再提供卸载的代码展示。这里还有两个特别的点，一个是如果一个目录被挂载了，那在进行iget时这个目录inode就需要变成被挂载设备的根目录。
```cpp
static inode_t *fit_inode(inode_t *inode)
{
    if (!inode->mount)
        return inode;

    super_block_t *sb = get_super(inode->mount);
    assert(sb);
    iput(inode);
    inode = sb->iroot;
    inode->count++;
    return inode;
}
``` 
如上，在iget中如果找到了inode，就需要调用下fit_node判断inode是不是被挂载的，如果是被挂载，那就是设置这个目录的inode为被挂载设备的根目录。

第二个点是通过 .. 访问父目录。这里修改的是find_entry函数，通过父目录的nr是否为1进行判断。当在文件的根目录需要解析 .. 文件夹时，执行如下代码：
```cpp
static buffer_t *find_entry(inode_t **dir, const char *name, char **next, dentry_t **result)
{
    // 保证 dir 是目录
    assert(ISDIR((*dir)->desc->mode));

    if (match_name(name, "..", next) && (*dir)->nr == 1)
    {
        super_block_t *sb = get_super((*dir)->dev);  // 获取设备的超级块
        inode_t *inode = *dir;      // 保存旧的
        (*dir) = sb->imount;        // 把当前目录切换到挂载点目录
        (*dir)->count++;            // 新 inode 引用计数+1
        iput(inode);                // 释放旧 inode
    }
    ......
}
```
所以当访问 .. 时：不再返回子分区的根目录本身，而是返回到主分区中的挂载点目录 /mnt。举个例子，当位于  /mnt/etc 时，如果执行了 cd .. 。 那就应该回到挂载点 /mnt。这段逻辑就是为了解决「在挂载的子文件系统根目录访问 .. 时，要正确跳回挂载点目录」的问题。

实现了mount的系统调用后，就可以给从盘进行挂载操作了。需要提醒一点的是，主盘的挂载前边我们在代码mount_root中写死了, mount_root设置了主盘的超级块的iroot为主盘的第一个inode, 设置了mount为主盘的设备号。 这里注意系统初始化是mount_root是第一个被挂载的盘，因此它使用的inode就是申请的系统表的第0个元素的位置。在创建进程时设置了进程的iroot和ipwd为系统inode表的第0个元素， 并且设置了名字为 / 。

通过mount系统调用可以进行从盘的挂载，通过挂载将从盘超级块的inode设置为1号inode, 然后把超级块的imount设置为挂载点的inode。设置目录的属性为被挂载的设备号。然后释放设备文件的inode。这样当我们访问被挂载点目录查找被挂载点的inode时，就会判断这个inode是一个挂载点，将这个inode进行释放，并将设备的跟目录inode返回。这样就是在该设备中进行读写了。注意这里被挂载点的inode被释放只是在内存中被释放了，暂时无法访问，但是仍然存在于它的原磁盘上，当解除挂载时，这个目录就恢复正常访问状态了。

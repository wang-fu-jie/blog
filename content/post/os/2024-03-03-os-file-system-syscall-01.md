---
title:       自制操作系统 - 文件系统相关系统调用(一)
subtitle:    "文件系统相关系统调用(一)"
description: "用户态对文件的操作需要通过系统调用来实现，本文中我们将会实现目录相关的系统调用和文件的初始化，包括创建目录、删除目录，硬链接。在文件初始化中创建系统文件表和进程文件表。"
excerpt:     "用户态对文件的操作需要通过系统调用来实现，本文中我们将会实现目录相关的系统调用和文件的初始化，包括创建目录、删除目录，硬链接。在文件初始化中创建系统文件表和进程文件表。"
date:        2024-03-03T20:20:45+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/85/17/surfer_surfing_surfboard_sports_outdoor_waves_wave_surf-633096.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-file-system-syscall-01"
categories:  [ "操作系统" ]
---

## 一、创建目录和删除目录
系统调用的框架在前面我们已经搭建过了，这里可以直接实现系统调用。这里只展示系统调用进入内核态的代码：
```cpp
int sys_mkdir(char *pathname, int mode)    // 创建目录
{
    char *next = NULL;
    buffer_t *ebuf = NULL;
    inode_t *dir = named(pathname, &next);  // 获取父目录，和待创建的目录名称
    if (!dir)                 // 父目录不存在
        goto rollback;
    if (!*next)                // 目录名为空
        goto rollback;
    if (!permission(dir, P_WRITE))  // 父目录无写权限
        goto rollback;

    char *name = next;
    dentry_t *entry;
    ebuf = find_entry(&dir, name, &next, &entry); // 查找待创建目录是否已存在
    if (ebuf)             // 目录项已存在
        goto rollback;

    ebuf = add_entry(dir, name, &entry);  // 添加目录项
    ebuf->dirty = true;
    entry->nr = ialloc(dir->dev);   // 分配一个文件系统 inode
    task_t *task = running_task();   // 获取当前进程
    inode_t *inode = iget(dir->dev, entry->nr);  // 获取inode
    inode->buf->dirty = true;
    inode->desc->gid = task->gid;  // 设置inode的用户id和组id
    inode->desc->uid = task->uid;
    inode->desc->mode = (mode & 0777 & ~task->umask) | IFDIR;  // 设置inde权限
    inode->desc->size = sizeof(dentry_t) * 2; // 当前目录和父目录两个目录项
    inode->desc->mtime = time();              // 时间戳
    inode->desc->nlinks = 2;                  // 一个是 '.' 一个是 name

    dir->buf->dirty = true;     // 父目录链接数加 1
    dir->desc->nlinks++; // ..
    buffer_t *zbuf = bread(inode->dev, bmap(inode, 0, true));     // 写入 inode 目录中的默认目录项
    zbuf->dirty = true;
    entry = (dentry_t *)zbuf->data;
    strcpy(entry->name, ".");   // . 代表当前目录
    entry->nr = inode->nr;
    entry++;
    strcpy(entry->name, "..");  // .. 代表父目录
    entry->nr = dir->nr;
    iput(inode);
    iput(dir);
    brelse(ebuf);
    brelse(zbuf);
    return 0;

rollback:
    brelse(ebuf);
    iput(dir);
    return EOF;
}

static bool is_empty(inode_t *inode)  // 判断目录是否为空
{
    int entries = inode->desc->size / sizeof(dentry_t);  // 获取当前有多少个目录项
    if (entries < 2 || !inode->desc->zone[0])   // 如果小于2或者文件块没有数据则报错，因为至少需要有 . 和 .. 两个目录项
    {
        LOGK("bad directory on dev %d\n", inode->dev);
        return false;
    }
    idx_t i = 0;
    idx_t block = 0;
    buffer_t *buf = NULL;
    dentry_t *entry;
    int count = 0;
    for (; i < entries; i++, entry++)     // 遍历目录项，判断目录项是否有效
    {
        if (!buf || (u32)entry >= (u32)buf->data + BLOCK_SIZE)
        {
            brelse(buf);
            block = bmap(inode, i / BLOCK_DENTRIES, false); // 根据目录项索引计算对应的磁盘块号
            buf = bread(inode->dev, block);  // 读取磁盘块到缓冲区
            entry = (dentry_t *)buf->data;   // 重置 entry 指针指向新缓冲区的起始位置
        }
        if (entry->nr)  // // 如果目录项有效
            count++;
    };
    brelse(buf);
    if (count < 2)  // 如果数量小于2则报错
    {
        LOGK("bad directory on dev %d\n", inode->dev);
        return false;
    }
    return count == 2;  // 如果等于2则说明是空目录，否则不是空目录
}

int sys_rmdir(char *pathname)   // 删除目录项
{
    char *next = NULL;
    buffer_t *ebuf = NULL;
    inode_t *dir = named(pathname, &next);
    inode_t *inode = NULL;
    int ret = EOF;

    // 判断父目录不存在、目录名为空、父目录无写权限、 目录项不存在 判断inode是否存在，inode是否是目录，inode是否等于传参进来的目录inode
    // ，是否有删除权限，目录是否为空。 这里省略这些判断逻辑，如果不通过则跳转到rollback
  
    inode_truncate(inode);      // 截断inode，即清空inode对应的文件块内容
    ifree(inode->dev, inode->nr);  // 释放inode节点
    inode->desc->nlinks = 0;
    inode->buf->dirty = true;
    inode->nr = 0;
    dir->desc->nlinks--;
    dir->ctime = dir->atime = dir->desc->mtime = time();
    dir->buf->dirty = true;
    assert(dir->desc->nlinks > 0);
    entry->nr = 0;
    ebuf->dirty = true;
    ret = 0;

rollback:
    iput(inode);
    iput(dir);
    brelse(ebuf);
    return ret;
}
```
如上为删除目录和创建目录，以及判断目录是否为空的实现，通过这两个系统调用，就可以在用户态进行目录的增删操作了。

## 二、系统调用link和unlink
link系统调用是硬链接的实现。unlink是解除硬链接。硬链接是只允许链接文件，不允许链接目录。因为我们希望目录是树形结构，最次也是有向无环图。如果目录出现了循环，在目录查找时就可能会进入死循环。
```cpp
int sys_link(char *oldname, char *newname)
{
    inode_t *inode = namei(oldname);      // 获取旧文件的inode
    buf = add_entry(dir, name, &entry);   // 添加新的目录项
    entry->nr = inode->nr;                // 新的目录项和旧文件指向同一个inode
    buf->dirty = true;
    inode->desc->nlinks++;                // 旧inode引用加1
    inode->ctime = time();
    inode->buf->dirty = true;
    ret = 0;
}

int sys_unlink(char *filename)
{
    inode_t *dir = named(filename, &next);   // 获取文件父目录的inode
    char *name = next;   // 获取文件名
    dentry_t *entry;
    buf = find_entry(&dir, name, &next, &entry);  // 查找目录项
    inode = iget(dir->dev, entry->nr);       // 获取文件的inode
    entry->nr = 0;
    buf->dirty = true;
    inode->desc->nlinks--;
    inode->buf->dirty = true;
    if (inode->desc->nlinks == 0)
    {
        inode_truncate(inode);        // 清空inode的内容
        ifree(inode->dev, inode->nr);  // 释放inode节点。
    }
    ret = 0;
}
```
为了节省篇幅，这两个函数省略了对异常情况的判断，只保留了核心逻辑。这里对比看到硬链接和创建目录的区别，创建目录是添加目录项后，申请了一个新的inode指向这个目录。但是硬链接添加目录项后，将旧的inode直接指向了这个目录项。


## 三、打开inode
打开文件，返回 inode，用于系统调用 open。打开文件有多种方式，只读、只写、读写、追加等等。
```cpp
inode_t *inode_open(char *pathname, int flag, int mode) // flag是以什么模式打开，mode是如果文件需要被创建（O_CREAT）时，指定的权限位
{
    inode_t *dir = NULL;
    inode_t *inode = NULL;
    buffer_t *buf = NULL;
    dentry_t *entry = NULL;
    char *next = NULL;
    dir = named(pathname, &next);   // 获取父目录的inode
    char *name = next;   // 要打开的文件名
    buf = find_entry(&dir, name, &next, &entry);  // 查找文件
    if (buf)
    {
        inode = iget(dir->dev, entry->nr);  // 如果文件存在，直接获取文件的inode
        goto makeup;
    }

    if (!(flag & O_CREAT))  // 如果文件不存在，且没有创建标记，直接结束
        goto rollback;
    if (!permission(dir, P_WRITE))   // 判断是否有父目录的写权限。
        goto rollback;
    buf = add_entry(dir, name, &entry);  // 文件不存在则创建
    entry->nr = ialloc(dir->dev);
    inode = iget(dir->dev, entry->nr);  // 获取到新创建的inode

    task_t *task = running_task();
    mode &= (0777 & ~task->umask);
    mode |= IFREG;
    inode->desc->uid = task->uid;
    inode->desc->gid = task->gid;
    inode->desc->mode = mode;
    inode->desc->mtime = time();
    inode->desc->size = 0;
    inode->desc->nlinks = 1;
    inode->buf->dirty = true;

makeup:
    if (ISDIR(inode->desc->mode) || !permission(inode, flag & O_ACCMODE))
        goto rollback; // 如果目标 inode 是目录，则不能用 open 打开（除非用 opendir），所以失败
    inode->atime = time();
    if (flag & O_TRUNC)  // 如果 O_TRUNC 标志存在，对文件进行截断，清空其内容
        inode_truncate(inode);
    brelse(buf);
    iput(dir);
    return inode;  // 返回打开的 inode
```
这个函数我们这里也进行了精简，删掉了部分用于异常判断的代码，保留了核心逻辑。这个函数的核心意义就是：根据路径名，找到（或创建）对应的文件，在内存中获取该文件的 inode，并做好访问权限、截断、时间等管理，为后续读写做准备。


## 四、文件初始化
文件初始化涉及三个内容，分别是定义文件数据结构、创建全局文件表、创建进程文件表。
```cpp
typedef struct file_t   // 文件结构
{
    inode_t *inode; // 文件 inode
    u32 count;      // 引用计数
    off_t offset;   // 文件偏移
    int flags;      // 文件标记
    int mode;       // 文件模式
} file_t;

#define FILE_NR 128   // 系统全局最多打开128个文件
file_t file_table[FILE_NR];  // 系统文件表

file_t *get_file()     // 从文件表中获取一个空文件
{
    for (size_t i = 3; i < FILE_NR; i++)  // 0、1、2是标准输入输出，因此从3开始
    {
        file_t *file = &file_table[i];
        if (!file->count)
        {
            file->count++;  // 文件的引用计数+1
            return file;
        }
    }
    panic("Exceed max open files!!!"); // 如果没找到，就报错
}

void put_file(file_t *file)  // 释放一个文件
{
    assert(file->count > 0);
    file->count--;   // 文件引用计数减去1
    if (!file->count)  // 如果文件的没有引用了，就释放文件的inode
    {
        iput(file->inode);
    }
}

void file_init()   // 系统文件表初始化为空
{
    for (size_t i = 0; i < FILE_NR; i++)
    {
        file_t *file = &file_table[i];
        file->mode = 0;
        file->count = 0;
        file->flags = 0;
        file->offset = 0;
        file->inode = NULL;
    }
}
```
以上是系统文件表，写死了整个操作系统最多打开128个文件。接下来还需要实现进程文件表，每个进程最多打开16个文件。首先给进程结构体添加一个元素，即进程文件表：
```cpp
#define TASK_FILE_NR 16 // 进程文件数量
typedef struct task_t
{
    ....
    struct inode_t *ipwd;               // 进程当前目录 inode program work directory
    struct inode_t *iroot;              // 进程根目录 inode
    u16 umask;                          // 进程用户权限
    struct file_t *files[TASK_FILE_NR]; // 进程文件表
    ...
} task_t;

fd_t task_get_fd(task_t *task)  // 申请一个文件描述符
{
    fd_t i;
    for (i = 3; i < TASK_FILE_NR; i++)
    {
        if (!task->files[i])
            break;
    }
    if (i == TASK_FILE_NR)
    {
        panic("Exceed task max open files.");
    }
    return i;
}

void task_put_fd(task_t *task, fd_t fd)  // 释放文件描述符。
{
    if (fd < 3)  // 如果文件描述符小于3， 说明是属于标准输入输出。
        return;
    assert(fd < TASK_FILE_NR);
    task->files[fd] = NULL;
}
```
如上所示，系统文件表是每个进程都会有一个，它是指针数组，每个元素指向一个 file_t，下标（索引）就是文件描述符 fd。每个进程的文件描述符是独立的，因此多个进程可以打开同一个文件，内核为这个文件分配了一个系统文件表项，然后每个进程都在自己的文件表引用这同一个系统文件表项了。

文件初始化完后就可以实现open close等系统调用了，我们在下一篇文章中来实现。
---
title:      自制操作系统 - 文件系统相关系统调用(二)
subtitle:    "文件系统相关系统调用(二)"
description: "上文中做了文件的初始化，接下来就可以实现文件相关的系统调用了。本文中将实现文件的打开、关闭、文件读、文件写、创建普通文件、设置文件偏移。以及设置当前工作目录，修改当前工作目录等系统调用。"
excerpt:     "上文中做了文件的初始化，接下来就可以实现文件相关的系统调用了。本文中将实现文件的打开、关闭、文件读、文件写、创建普通文件、设置文件偏移。以及设置当前工作目录，修改当前工作目录等系统调用。"
date:        2024-03-05T19:39:34+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/7e/cf/tree_koala_sleep_branch_nap-42541.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-file-system-syscall-02"
categories:  [ "操作系统" ]
---

## 一、文件打开与关闭open close
本节将完成以下三个系统调用，分别是打开文件，创建普通文件和关闭文件。实现如下：
```cpp
fd_t sys_open(char *filename, int flags, int mode)  // 打开文件
{
    inode_t *inode = inode_open(filename, flags, mode);  // 打开文件inode
    if (!inode)
        return EOF;
    task_t *task = running_task();   // 获取当前进程
    fd_t fd = task_get_fd(task);     // 从进程文件表中获取一个文件描述符
    file_t *file = get_file();       // 从系统文件表中获取一个文件结构
    assert(task->files[fd] == NULL);
    task->files[fd] = file;          // 设置该文件描述符为系统分配的文件结构
    file->inode = inode;
    file->flags = flags;
    file->count = 1;
    file->mode = inode->desc->mode;
    file->offset = 0;
    if (flags & O_APPEND)      // 如果以追加的方式打开文件，设置文件偏移为文件的当前大小
    {
        file->offset = file->inode->desc->size;
    }
    return fd;
}

int sys_creat(char *filename, int mode)  // 创建文件
{
    return sys_open(filename, O_CREAT | O_TRUNC, mode);  // 直接调用打开文件，并增加O_CREAT和O_TRUNC标识
}

void sys_close(fd_t fd)  // 关闭文件
{
    assert(fd < TASK_FILE_NR);
    task_t *task = running_task();  // 获取当前进程
    file_t *file = task->files[fd];  // 根据文件描述符获取文件结构
    if (!file)
        return;

    assert(file->inode);
    put_file(file);     // 从系统文件表中释放文件
    task_put_fd(task, fd);   // 释放文件描述符
}
```
每个进程的文件描述符是独立的，进程文件表的下标就是文件描述符，文件描述符一般从3开始，因为0、1、2是标准输入输出。

## 二、文件读写系统调用
实现了文件的打开和关闭，接下来就可以实现对文件的读写操作了。涉及两个系统调用read 和 write。
```cpp
int sys_read(fd_t fd, char *buf, int count)   // 读文件
{
    if (fd == stdin)   // 如果文件描述符是标准输入，就从键盘中读取
    {
        device_t *device = device_find(DEV_KEYBOARD, 0);  // 查找键盘设备
        return device_read(device->dev, buf, count, 0, 0);  // 进行设备读
    }

    task_t *task = running_task();   // 不是标准输入则进行文件读
    file_t *file = task->files[fd]; 
    if ((file->flags & O_ACCMODE) == O_WRONLY) // 进行权限判断，如果没有读权限，则返回EOF
        return EOF;

    inode_t *inode = file->inode;    // 获取文件inode
    int len = inode_read(inode, buf, count, file->offset);  // 读取指定长度到buf中
    if (len != EOF)
    {
        file->offset += len;  // 设置文件偏移为读取后的位置
    }
    return len;   // 返回读取的文件长度
}

int sys_write(unsigned int fd, char *buf, int count)
{
    if (fd == stdout || fd == stderr)   // 如果文件描述符是标准输入或标准错误输出，则写入到控制台
    {
        device_t *device = device_find(DEV_CONSOLE, 0);
        return device_write(device->dev, buf, count, 0, 0);
    }
    task_t *task = running_task();
    file_t *file = task->files[fd];
    if ((file->flags & O_ACCMODE) == O_RDONLY)
        return EOF;

    inode_t *inode = file->inode;
    int len = inode_write(inode, buf, count, file->offset);
    if (len != EOF)
    {
        file->offset += len;
    }

    return len;
}
```
如上所示，即实现了文件的读和文件写操作。


## 三、设置文件偏移 lseek
设置文件偏移有三种方式，分别是直接设置偏移、当前位置偏移和结束位置偏移。实现如下：
```cpp
typedef enum whence_t
{
    SEEK_SET = 1, // 直接设置偏移
    SEEK_CUR,     // 当前位置偏移
    SEEK_END      // 结束位置偏移
} whence_t;

int sys_lseek(fd_t fd, off_t offset, whence_t whence)  
{
    task_t *task = running_task();
    file_t *file = task->files[fd];   // 获取文件结构
    switch (whence)   // 如果直接设置偏移
    {
    case SEEK_SET:
        assert(offset >= 0);
        file->offset = offset;  // 把文件的偏移设置为参数offset即可
        break;
    case SEEK_CUR:              // 当前位置偏移， 把文件的偏移增加offset
        assert(file->offset + offset >= 0);
        file->offset += offset;
        break;
    case SEEK_END:              // 结束位置偏移，把文件的偏移设置为，文件结尾加 offset
        assert(file->inode->desc->size + offset >= 0);
        file->offset = file->inode->desc->size + offset;
        break;
    default:
        panic("whence not defined !!!");
        break;
    }
    return file->offset;
}
```

## 四、获取和切换工作目录
工作目录的系统调用有三个，分别是获取当前路径、切换当前路径和切换根目录，实现如下：
```cpp
char *sys_getcwd(char *buf, size_t size)  // 获取当前路径
{
    task_t *task = running_task();
    strncpy(buf, task->pwd, size);  // 直接把当前进程的的pwd属性拷贝到buf
    return buf;
}

int sys_chdir(char *pathname)  // 修改当前路径
{
    task_t *task = running_task();
    inode_t *inode = namei(pathname);

    abspath(task->pwd, pathname);  // 计算 当前路径 pwd 和新路径 pathname, 存入 pwd
    iput(task->ipwd);             // 释放当前的inode
    task->ipwd = inode;           // 把新路径的inode设置为当前inode
    return 0;
}

int sys_chroot(char *pathname)
{
    task_t *task = running_task();
    inode_t *inode = namei(pathname);
    iput(task->iroot);
    task->iroot = inode;
    return 0;
}

```
这里一样简化了代码，删掉了异常处理，只保留了核心逻辑。切换根目录的作用是提升安全性，根目录以外的位置进程是无法访问的。因为进程中增加了根目录和当前目录属性，因此在创建进程、fork进程和进程退出时一样需要增加对这两个属性的处理。
```cpp
// task_create 增加如下代码段
task->iroot = task->ipwd = get_root_inode();  // 设置当前目录为根目录
task->iroot->count += 2;

task->pwd = (void *)alloc_kpage(1);
strcpy(task->pwd, "/");


// fork需要增加如下代码段
// 拷贝 pwd
child->pwd = (char *)alloc_kpage(1);
strncpy(child->pwd, task->pwd, PAGE_SIZE);  // 直接拷贝父进程的pwd
// 工作目录引用加一
task->ipwd->count++;
task->iroot->count++;

// 文件引用加一
for (size_t i = 0; i < TASK_FILE_NR; i++)
{
    file_t *file = child->files[i];
    if (file)
        file->count++;
}
```
同样当进程退出时，需要释放这一页内存，减少文件和目录引用。这里为了存放pwd字符串申请一页内存其实有点浪费，但是这样处理比较简单。
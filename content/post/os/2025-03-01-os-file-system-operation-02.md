---
title:       自制操作系统 - 文件系统相关操作(二)
subtitle:    "文件系统相关操作(二)"
description: "本节继续实现对文件系统的操作，包括对目录的操作如添加目录项，获取目录下的文件、获取文件的innode和文件父目录的inode。并且通过inode实现对文件的读写和清空操作。"
excerpt:     "本节继续实现对文件系统的操作，包括对目录的操作如添加目录项，获取目录下的文件、获取文件的innode和文件父目录的inode。并且通过inode实现对文件的读写和清空操作。"
date:        2024-03-01T09:55:05+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/1e/bf/vintage_bike_flowers_bicycle_retro_vintage_bicycle-665333.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-file-system-operation-02"
categories:  [ "操作系统" ]
---

## 一、文件系统目录操作
文件系统目录的操作主要涉及以下几个功能，判断文件名是否相等、获取 dir 目录下的 文件名和在 dir 目录中添加目录项。以下为这个三个功能的代码实现：
```cpp
static bool match_name(const char *name, const char *entry_name, char **next)  // 判断文件名是否相等
{
    char *lhs = (char *)name;  
    char *rhs = (char *)entry_name;
    while (*lhs == *rhs && *lhs != EOS && *rhs != EOS) // 逐个字符进行判断，直到字符不相等或字符结束
    {
        lhs++;
        rhs++;
    }
    if (*rhs)   //  如果entry_name还有剩余字符，说明不匹配，直接返回false
        return false;
    if (*lhs && !IS_SEPARATOR(*lhs)) // 如果 name 还有字符但不是路径分隔符，说明不匹配
        return false;
    if (IS_SEPARATOR(*lhs)) // 如果name还有字符且是路径分割符。
        lhs++;              // 跳过路径分割符
    *next = lhs;            // 返回剩余部分
    return true;
}

// 获取 dir 目录下的 name 目录 所在的 dentry_t 和 buffer_t
static buffer_t *find_entry(inode_t **dir, const char *name, char **next, dentry_t **result)
{
    u32 entries = (*dir)->desc->size / sizeof(dentry_t);   // dir 目录最多子目录数量
    idx_t i = 0;                // 当前处理的目录项索引
    idx_t block = 0;            // 当前数据块号
    buffer_t *buf = NULL;       // 当前数据块的缓冲区
    dentry_t *entry = NULL;    // 当前目录项指针
    idx_t nr = EOF;

    for (; i < entries; i++, entry++)
    {
        if (!buf || (u32)entry >= (u32)buf->data + BLOCK_SIZE) // 数据块边界检查
        {
            brelse(buf);
            block = bmap((*dir), i / BLOCK_DENTRIES, false);  // 计算需要读取的块号：目录项索引 / 每块目录项数
            buf = bread((*dir)->dev, block);  // 读取数据块到缓冲区
            entry = (dentry_t *)buf->data;
        }
        if (match_name(name, entry->name, next))
        {
            *result = entry;   // 返回找到的目录项
            return buf;   // 返回包含目录项的缓冲区（引用计数+1）
        }
    }
    brelse(buf);
    return NULL;  // 返回NULL说明没找到目录
}

static buffer_t *add_entry(inode_t *dir, const char *name, dentry_t **result)  // 在 dir 目录中添加 name 目录项
{
    char *next = NULL;
    buffer_t *buf = find_entry(&dir, name, &next, result);  // 判断目录中是否已有指定目录项，有则直接返回
    if (buf)
    {
        return buf;
    }
    for (size_t i = 0; i < NAME_LEN && name[i]; i++)   // name 中不能有路径分隔符
    {
        assert(!IS_SEPARATOR(name[i]));
    }

    idx_t i = 0;           // 目录项索引
    idx_t block = 0;       // 数据块号
    dentry_t *entry;       // 目录项指针

    for (; true; i++, entry++)  // 主循环 寻找空闲目录项位置
    {
        if (!buf || (u32)entry >= (u32)buf->data + BLOCK_SIZE)
        {
            brelse(buf);
            block = bmap(dir, i / BLOCK_DENTRIES, true);   // // true表示需要分配新块
            buf = bread(dir->dev, block);     // 读取块
            entry = (dentry_t *)buf->data;
        }
        if (i * sizeof(dentry_t) >= dir->desc->size)
        {
            entry->nr = 0;                  // 标记为空闲目录项
            dir->desc->size = (i + 1) * sizeof(dentry_t);  // 扩展目录大小
            dir->buf->dirty = true;   // // 标记目录inode为脏
        }
        if (entry->nr)     // 目录项已被使用，继续查找
            continue;
        strncpy(entry->name, name, NAME_LEN);  // 找到空闲位置，创建新目录项。复制名称
        buf->dirty = true;
        dir->desc->mtime = time();
        dir->buf->dirty = true;
        *result = entry;
        return buf;
    };
}
```
通过以上三个操作目录的函数就可以进行目录项的查找和创建了，通过add_entry不仅可以创建目录，还可以创建硬链接。只需要把新创建的目录项的nr设置为何原来的nr相等即可，这样就指向了同一个inode。

## 二、文件系统namei
文件系统的namei设置两个功能，分别是获取 pathname 对应的父目录 inode 和  获取 pathname 对应的 inode。 代码实现如下：
```cpp
inode_t *named(char *pathname, char **next)  // 获取 pathname 对应的父目录 inode
{
    inode_t *inode = NULL;
    task_t *task = running_task();  // 获取当前运行的进程
    char *left = pathname;
    if (IS_SEPARATOR(left[0]))  // 判断第一个字符是否是路径分隔符。
    {
        inode = task->iroot;  // 如果是以根目录起始位置，获取进程的根目录。进程根目录默认与文件系统根目录一直，但是可以通过chroot来修改。
        left++;
    }
    else if (left[0])   // 如果不是绝对路径，获取到进程的当前工作目录
        inode = task->ipwd; 
    else                 // 否则返回空
        return NULL;   
    inode->count++;  // inode的引用计数+1。防止innode被释放
    *next = left;  // 设置剩余路径
    if (!*left)   // 如果路径已解析完（如："/" 或 ""） 直接返回当前inode
    {
        return inode;
    }
    char *right = strrsep(left); // 从后往前找最后一个分隔符
    if (!right || right < left)  // 没找到分隔符或位置错误 ，返回当前inode
    {
        return inode;
    }
    right++;    // 指向最后一个分量（文件名部分）
    *next = left;   // 重置next为完整剩余路径
    dentry_t *entry = NULL;
    buffer_t *buf = NULL;
    while (true)
    {
        brelse(buf);
        buf = find_entry(&inode, left, next, &entry);   // 在当前目录中查找下一个路径分量
        if (!buf)
            goto failure;  // 查找失败

        dev_t dev = inode->dev;
        iput(inode);    // 释放当前目录inode
        inode = iget(dev, entry->nr);  // 获取下一级目录/文件的inode
        if (!ISDIR(inode->desc->mode) || !permission(inode, P_EXEC))  // 检查是否是目录且有执行权限
            goto failure;
        if (right == *next)  // 已解析到最后一个分量之前
            goto success;   // 成功找到父目录
        left = *next;
    }

success:
    brelse(buf);
    return inode; // 返回的inode仍有有效引用

failure:
    brelse(buf);
    iput(inode);  // 减少引用，平衡前面引用计数的增加
    return NULL;
}

inode_t *namei(char *pathname)    // 获取 pathname 对应的 inode
{
    char *next = NULL;
    inode_t *dir = named(pathname, &next);  // 找到父目录的inode
    if (!dir)
        return NULL;
    if (!(*next))
        return dir;

    char *name = next;
    dentry_t *entry = NULL;
    buffer_t *buf = find_entry(&dir, name, &next, &entry);
    if (!buf)
    {
        iput(dir);
        return NULL;
    }
    inode_t *inode = iget(dir->dev, entry->nr);
    iput(dir);
    brelse(buf);
    return inode;
}
```
假设对于路径 /home/user/file.txt：
初始：left = "home/user/file.txt", right = "file.txt"
* 第一轮：在根目录查找"home"，进入/home目录
* 第二轮：在/home目录查找"user"，进入/home/user目录
* 检查：*next = "file.txt" == right，停止循环
* 返回：/home/user目录的inode（file.txt的父目录）


## 三、inode读写与截断
读写inode就是实现对文件的读写操作，截断inode是清空文件内容，释放inode的所有文件块，但是这个文件还是存在的。涉及两个函数实现如下：
```cpp
int inode_read(inode_t *inode, char *buf, u32 len, off_t offset)  // 从 inode 的 offset 处，读 len 个字节到 buf
{
    if (offset >= inode->desc->size)  // 如果偏移量超过文件大小，返回 EOF
    {
        return EOF;
    }
    u32 begin = offset;  // 开始读取的位置
    u32 left = MIN(len, inode->desc->size - offset);  // 剩余字节数
    while (left)
    {
        idx_t nr = bmap(inode, offset / BLOCK_SIZE, false);   // 找到对应的文件偏移，所在文件块
        buffer_t *bf = bread(inode->dev, nr);   // 读取文件块缓冲
        u32 start = offset % BLOCK_SIZE;   // 文件块中的偏移量
        u32 chars = MIN(BLOCK_SIZE - start, left);  // 本次需要读取的字节数
        offset += chars;  // 更新 偏移量 和 剩余字节数
        left -= chars;
        char *ptr = bf->data + start;   // 文件块中的指针
        memcpy(buf, ptr, chars);        // 拷贝内容
        buf += chars;          // 更新缓存位置
        brelse(bf);         // 释放文件块缓冲
    }
    inode->atime = time();         // 更新访问时间
    return offset - begin;         // 返回读取数量
}

int inode_write(inode_t *inode, char *buf, u32 len, off_t offset)  // 从 inode 的 offset 处，将 buf 的 len 个字节写入磁盘
{
    assert(ISFILE(inode->desc->mode));     // 不允许目录写入目录文件，修改目录有其他的专用方法
    u32 begin = offset;      // 开始的位置
    u32 left = len;          // 剩余数量

    while (left)
    {
        idx_t nr = bmap(inode, offset / BLOCK_SIZE, true);  // 找到文件块，若不存在则创建
        buffer_t *bf = bread(inode->dev, nr);    // 将读入文件块
        bf->dirty = true;
        u32 start = offset % BLOCK_SIZE;      // 块中的偏移量
        char *ptr = bf->data + start;     // 文件块中的指针
        u32 chars = MIN(BLOCK_SIZE - start, left);   // 读取的数量
        offset += chars;     // 更新偏移量
        left -= chars;      // 更新剩余字节数
        if (offset > inode->desc->size)    // 如果偏移量大于文件大小，则更新
        { 
            inode->desc->size = offset;
            inode->buf->dirty = true;
        }
        memcpy(ptr, buf, chars);    // 拷贝内容
        buf += chars;        // 更新缓存偏移
        brelse(bf);          // 释放文件块
    }
    inode->desc->mtime = inode->atime = time();    // 更新修改时间
    bwrite(inode->buf);   // 实时刷盘
    return offset - begin;    // 返回写入大小
}
```
截断inode文件的代码这里不再展示，逻辑就是找到inode截断后，逐级释放直接块和间接块。这里测试这些功能时，路径可以使用 .. 和 . 两个特殊目录项。这里有些读者可能比较好奇，我们的系统中并未实现这两个特殊的目录项。这是因为在测试时目录是在makefike中通过已有操作系统的mkdir命令创建的，会自动创建这两个目录项。


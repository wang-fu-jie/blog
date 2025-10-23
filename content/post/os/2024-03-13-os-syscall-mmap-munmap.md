---
title:      自制操作系统 - 映射内存系统调用mmap&munmap
subtitle:    "映射内存系统调用mmap&munmap"
description: "mmap 是 Linux/Unix 系统中非常重要的一个系统调用，它的作用是将文件或设备映射到进程的虚拟内存空间中。用来实现虚拟内存的映射，实现这个系统调用需要调整内存部署，用户映射内存开始位置调整到128M，栈顶放到256M的位置。"
excerpt:     "mmap 是 Linux/Unix 系统中非常重要的一个系统调用，它的作用是将文件或设备映射到进程的虚拟内存空间中。用来实现虚拟内存的映射，实现这个系统调用需要调整内存部署，用户映射内存开始位置调整到128M，栈顶放到256M的位置。"
date:        2024-03-13T15:39:18+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/ff/a6/yellowstone_national_park_wyoming_landscape_sunrise_sunset_mountains_snow_winter-551109.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-syscall-mmap-munmap"
categories:  [ "操作系统" ]
---

## 一、内存布局
我们前面的内存布局是用户栈顶在128M的位置。在进行内存映射需要修改下内存布局。将文件映射的地址改到128M的位置，用户栈顶改到256M的位置。如下所示：
![图片加载失败](/post_images/os/{{< filename >}}/1-01.png)
前面在进入用户态时，我们设置了用户内存虚拟位图只有一页，因此操作系统最大支持128M。这里我们修改了内存布局，最大可以使用256M内存。因为用户内存映射从128M开始，用户依然最大可以使用128M的内存。但是内核只使用了16M。中间的用户程序段（.text、.data、.bss）和用户堆（heap）是不需要位图管理的。程序段：固定大小，内核加载时映射，不需 bitmap。堆：顺序增长，通过 brk/sbrk 分配物理页，也不需 bitmap。

## 二、系统调用 mmap&munmap
mmap 是 Linux/Unix 系统中非常重要的一个系统调用，它的作用是将文件或设备映射到进程的虚拟内存空间中。通过 mmap，进程可以像访问普通内存一样直接访问文件内容，而不必显式地使用 read()、write() 等系统调用。这段映射关系可以实现：
* 高效文件读写（文件直接映射到内存）
* 共享内存通信（多个进程映射同一个文件或匿名页）
* 内存分配（堆外内存）（匿名映射）
* 动态库加载（如 ELF 可执行文件加载）

### 2.1、mmap的具体实现。
我们修改了内存布局，因此首先需要修改下内存布局的相关宏定义：
```cpp
#define USER_EXEC_ADDR KERNEL_MEMORY_SIZE   // 用户程序地址, 内核占用了16M, 因此用户程序地址是16M开始的位置。
#define USER_MMAP_ADDR 0x8000000           // 用户映射内存开始位置， 128M
#define USER_MMAP_SIZE 0x8000000           // 用户映射内存大小, 也是128M大小
#define USER_STACK_TOP 0x10000000          // 用户栈顶地址 256M
#define USER_STACK_SIZE 0x200000           // 用户栈最大 2M
#define USER_STACK_BOTTOM (USER_STACK_TOP - USER_STACK_SIZE)   // 用户栈底地址

typedef struct page_entry_t
{
    ....
    u8 shared : 1;   // 共享内存页，与 CPU 无关
    u8 privat : 1;   // 私有内存页，与 CPU 无关
    u8 flag : 1;     // 该安排的都安排了，送给操作系统吧
    u32 index : 20;  // 页索引
} _packed page_entry_t;

enum mmap_type_t
{
    PROT_NONE = 0,   // 不允许访问，访问会触发异常
    PROT_READ = 1,   // 可读
    PROT_WRITE = 2,  // 可写
    PROT_EXEC = 4,   // 可执行

    MAP_SHARED = 1,   // 映射区对所有进程共享，对映射区的写会回写到文件
    MAP_PRIVATE = 2,   // 私有映射，对映射区的写不会影响原文件（写时拷贝
    MAP_FIXED = 0x10,  // 指定固定地址进行映射，如果地址冲突会覆盖原有映射
};
```
这里用户映射区虚拟地址范围是128M ~ 256M。 但是用户栈的范围是 254M ~ 256M。这2M的虚拟内存地址看似有重叠，这里我们会在代码实现中进行判断保证内存安全 ，理解即可。page_entry_t是页表结构，CPU是留了3位没有使用，这里我们的操作系统使用其中两位，分别用于表示共享内存页和私有内存页。枚举 mmap_type_t 定义了内存映射（mmap）相关的 保护权限标志和映射类型标志。

接下来需要修改用户的虚拟位图位置， 用户的虚拟位图然后使用1页，因此用户依然是最大使用128M的虚拟内存。只是需要调整位图的开始位置
```cpp
void task_to_user_mode(target_t target)
{
    task_t *task = running_task();

    // 创建用户进程虚拟内存位图
    task->vmap = kmalloc(sizeof(bitmap_t));
    void *buf = (void *)alloc_kpage(1);
    bitmap_init(task->vmap, buf, USER_MMAP_SIZE / PAGE_SIZE / 8, USER_MMAP_ADDR / PAGE_SIZE);
    ......
}
```
前面我们提到 中间的用户程序段（.text、.data、.bss）和用户堆（heap）是不需要位图管理的。这块内存在使用的时候直接使用物理页，通过brk系统调用来分配。因此这里在 将 vaddr 映射物理内存 和 去掉 vaddr 对应的物理内存映射。 即对应 link_page 和 unlink_page这两个函数时，不需要对用户进程的虚拟位图进行修改了。如果进行用户内存映射，会在mmap中修改位图，再调用link_page进行映射。
接下来就可以实现mmap和munmap了。
```cpp
// addr 是希望映射到的虚拟地址，如果为0，表示让系统自己选择， length是映射的长度。 prot是权限标志、flags是映射类型、fd是文件描述符， offset是文件偏移量。
void *sys_mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset)
{
    ASSERT_PAGE((u32)addr);
    u32 count = div_round_up(length, PAGE_SIZE);  // 计算需要映射多少页
    u32 vaddr = (u32)addr;                        // 虚拟地址就是我们希望映射的地址

    task_t *task = running_task();        // 获取当前进程
    if (!vaddr)                           // 如果虚拟地址为0， 就去位图中查找连续的页
    {
        vaddr = scan_page(task->vmap, count);  // 返回操作系统自动分配的虚拟地址。
    }

    assert(vaddr >= USER_MMAP_ADDR && vaddr < USER_STACK_BOTTOM);   // 保证虚拟地址大于用户内存且小于栈底

    for (size_t i = 0; i < count; i++)   // 进行虚拟内存映射
    {
        u32 page = vaddr + PAGE_SIZE * i;
        link_page(page);
        bitmap_set(task->vmap, IDX(page), true);  // 修改虚拟内存位图
        page_entry_t *entry = get_entry(page, false);  // 获取到页表项目
        entry->user = true;    // 设置属性为所有人
        entry->write = false;   // 设置不可写，然后根据传入的参数来判断是否可写
        if (prot & PROT_WRITE)
        {
            entry->write = true;
        }
        if (flags & MAP_SHARED)
        {
            entry->shared = true;
        }
        if (flags & MAP_PRIVATE)
        {
            entry->privat = true;
        }
        flush_tlb(page);
    }

    if (fd != EOF)   // 如果传入的文件描述符有效，就把文件读入内存
    {
        lseek(fd, offset, SEEK_SET);
        read(fd, (char *)vaddr, length);
    }

    return (void *)vaddr;
}

int sys_munmap(void *addr, size_t length)   // 解除内存映射
{
    task_t *task = running_task();
    u32 vaddr = (u32)addr;
    assert(vaddr >= USER_MMAP_ADDR && vaddr < USER_STACK_BOTTOM);
    ASSERT_PAGE(vaddr);
    u32 count = div_round_up(length, PAGE_SIZE);
    for (size_t i = 0; i < count; i++)
    {
        u32 page = vaddr + PAGE_SIZE * i;
        unlink_page(page);
        assert(bitmap_test(task->vmap, IDX(page)));
        bitmap_set(task->vmap, IDX(page), false);
    }
    return 0;
}
```
到此就实现了用户内存映射。

## 三、内存映射测试
我们在shell中创建一个测试命令来进行测试，代码如下：
```cpp
void builtin_test(int argc, char *argv[])
{
    u32 status;
    int *counter = (int *)mmap(0, sizeof(int), PROT_WRITE, MAP_SHARED, EOF, 0);  // MAP_SHARED 可改为0进行测试
    pid_t pid = fork();

    if (pid)
    {
        while (true)
        {
            (*counter)++;
            sleep(300);
        }
    }
    else
    {
        while (true)
        {
            (*counter)++;
            printf("counter %d\n", *counter);
            sleep(100);
        }
    }
}
```
如上为测试函数。mmap当传入flag为0时，说明不是共享内存区，即父进程写内存时，子进程打印是没有变化的。当传入flag为MAP_SHARED时，说明是共享内存区，父进程写，子进程打印也会跟着变化。也就是子进程和父进程使用了同一块内存。

最后我们来梳理一下用户内存映射的过程
1. test命令申请内存，申请4字节，即1个int类型的大小，申请的地址为 0。因为不是文件映射，因此fd传入EOF,
2. mmap返回映射后的虚拟地址，并赋值给变量 counter
3. 父进程fork一个子进程。子进程会拷贝父进程的页目录。为子分配新的页目录/页表结构（PDE/PTE 复制），但物理页不复制。

这里我们分两种情况讨论，如果是共享内存，那会执行如下流程：
1. 拷贝页目录的函数 copy_pde 做判断。如果一块内存是共享内存，那这块物理页不会被设置为只读
2. 因此这块虚拟内存，父进程在修改这个页，子进程读取的内容也在同步修改。

如果不是共享内存，那会执行如下流程：
1. 对于非共享页，内核把父、子对应的 PTE 都置为只读， 因为先置为只读后进行拷贝
2. 父进程第一次执行 (*counter)++， 执行写操作，发现这一页只读了，触发写保护异常，为父进程重新分配物理页。父进程继续写。
3. 子进程继续访问原来的只读页，因此一直打印原来的值。

可以看到，子进程被fork出来时，如果父进程的内存是共享的，那子进程就会映射到同一个物理页，如果不是共享的，子进程当使用时候，就会触发写时赋值，申请一个新的物理页，这里重点在于copy_pde这个函数的实现。
```cpp
page_entry_t *copy_pde()
{
    task_t *task = running_task();
    page_entry_t *pde = (page_entry_t *)alloc_kpage(1);
    memcpy(pde, (void *)task->pde, PAGE_SIZE);

    // 将最后一个页表指向页目录自己，方便修改
    page_entry_t *entry = &pde[1023];
    entry_init(entry, IDX(pde));
    page_entry_t *dentry;

    for (size_t didx = (sizeof(KERNEL_PAGE_TABLE) / 4); didx < 1023; didx++)
    {
        dentry = &pde[didx];
        if (!dentry->present)
            continue;
        page_entry_t *pte = (page_entry_t *)(PDE_MASK | (didx << 12));
        for (size_t tidx = 0; tidx < 1024; tidx++)
        {
            entry = &pte[tidx];
            if (!entry->present)
                continue;
            assert(memory_map[entry->index] > 0);  // 对应物理内存引用大于 0
            // 若不是共享内存，则置为只读
            if (!entry->shared)
            {
                entry->write = false;
            }
            // 对应物理页引用加 1
            memory_map[entry->index]++;
            assert(memory_map[entry->index] < 255);
        }
        u32 paddr = copy_page(pte);
        dentry->index = IDX(paddr);
    }
    set_cr3(task->pde);
    return pde;
}
```
如上所示，我们如果申请了一块共享内存，那会映射一页，并标记这一页是共享内存。在复制时，这一页标记为可写，因此子进程访问时直接访问。但是如果不是共享内存，子进程访问时就会触发缺页异常，因此就会重新映射一块物理内存。父进程在执行时会恢复父进程自身的页目录。

## 四、疑问
如果申请的内存是私有的，子进程和父进程都修改了counter值。那么子进程和父进程都会触发写保护异常并申请新页进行映射。那此时原来的页引用就会变为0，这里也没有任何释放机制。这个问题咨询了原作者，答复如下：单CPU页错误禁止中断，保证只有一个会触发写时复制，第二个进程就不需要了，直接写。另外，如果是多 CPU 的情况，应该对页面加锁，保证写时复制只发生一次！

这里看了下代码，假设子进程进行了写时复制，当父进程也触发页错误时发现自己是物理页的唯一引用，就把页改成了可写状态。来源于代码page_fault。
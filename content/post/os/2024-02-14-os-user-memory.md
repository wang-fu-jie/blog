---
title:       自制操作系统 - 内核堆内存管理与用户内存映射
subtitle:    "内核对内存管理与用户内存映射"
description: "之前实现的内存分配以页为单位一次分配的太大，因此本文将实现内核堆内存的管理，可申请指定大小的内存。在内核内存中，我们只使用了前8M，在进入用户态后需要进行用户内存映射，用户内存映射原理和内核一致，区别是每次使用内存可以申请内存映射，将虚拟地址映射为一个物理地址，使用完毕后可释放映射。"
excerpt:     "之前实现的内存分配以页为单位一次分配的太大，因此本文将实现内核堆内存的管理，可申请指定大小的内存。在内核内存中，我们只使用了前8M，在进入用户态后需要进行用户内存映射，用户内存映射原理和内核一致，区别是每次使用内存可以申请内存映射，将虚拟地址映射为一个物理地址，使用完毕后可释放映射。"
date:        2024-02-14T21:02:51+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/c9/da/hipster_wander_photographer_photo_woman_header_website_wild-868964.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-user-memory"
categories:  [ "操作系统" ]
---

## 一、内核堆内存映射
前面我们已经实现了分配回收内核的内存，但是分配的内存是以页为单位的，一次分配4k。如果需要小于4k的内存空间就需要进行堆内存映射。首先需要两个结构：
```cpp
#define DESC_COUNT 7
typedef list_node_t block_t; // 内存块

typedef struct arena_descriptor_t    // 内存块儿描述符
{
    u32 total_block;  // 一页内存分成了多少块
    u32 block_size;   // 块大小
    list_t free_list; // 空闲列表
} arena_descriptor_t;

typedef struct arena_t   // 一页或多页内存
{
    arena_descriptor_t *desc; // 该 arena 的描述符
    u32 count;                // 当前剩余多少块 或 页数
    u32 large;                // 表示是不是超过 1024 字节
    u32 magic;                // 魔数
} arena_t;

void *kmalloc(size_t size);
void kfree(void *ptr);
```
第一个结构如上所示：内存块儿描述符，它描述了这块内存的大小。 arena_t是一页或多页内存被分成了许多块。 DESC_COUNT是描述符的数量一共有7个。接下来我们结合实现代码来具体理解：
```cpp
extern u32 free_pages;
static arena_descriptor_t descriptors[DESC_COUNT];    // 定义内存块描述符数组

void arena_init()   // arena 初始化
{
    u32 block_size = 16;   // 申请16字节以内的内存，都分配16字节
    for (size_t i = 0; i < DESC_COUNT; i++)
    {
        arena_descriptor_t *desc = &descriptors[i];
        desc->block_size = block_size;
        desc->total_block = (PAGE_SIZE - sizeof(arena_t)) / block_size;  // 每页开头的位置存储存储一个 arena 数据结构
        list_init(&desc->free_list);
        block_size <<= 1; // block *= 2;
    }
}

static void *get_arena_block(arena_t *arena, u32 idx)   // 获得 arena 第 idx 块内存指针
{
    assert(arena->desc->total_block > idx);
    void *addr = (void *)(arena + 1);  // 获取内存块开始的位置，每页开头存储的arena结构，要跳过去
    u32 gap = idx * arena->desc->block_size;  // 获取第idx块前面所有块的总大小
    return addr + gap;
}

static arena_t *get_block_arena(block_t *block)
{
    return (arena_t *)((u32)block & 0xfffff000);
}
```
首先看arena_init函数，内存块最小为16字节，如果小于1申请的内存小于16字节，那就分配一个12字节的内存块。初始化内存块描述符数组，一共7个，下一个内存块大小是上一个的两倍, 最大的内存块是1024字节即1k。并且初始化空闲块链表。

get_arena_block函数获取arena 第 idx 块内存指针。get_block_arena函数用于获取当前内存块对应的arena结构。注意这里每页内存只能被划分为固定大小的内存块。

### 1.1、申请内存
通过arena对内存做好内存块的划分，就可以申请内存了，申请内存的实现如下：
```cpp
void *kmalloc(size_t size)
{
    arena_descriptor_t *desc = NULL;
    arena_t *arena;
    block_t *block;
    char *addr;
    if (size > 1024)     // 如果申请的大小大于1024字节，就直接分配整页内存。
    {
        u32 asize = size + sizeof(arena_t);   // 开始的位置依然要存储arena结构，因此实际申请size应该加上arena
        u32 count = div_round_up(asize, PAGE_SIZE); // 计算申请这么多内存需要多少页，向上取整
        arena = (arena_t *)alloc_kpage(count);  // 申请内存 
        memset(arena, 0, count * PAGE_SIZE);   // 申请到的内存全部置为0
        arena->large = true;              // 设置arena数据结构
        arena->count = count;             // 这里表示剩余多少页
        arena->desc = NULL;
        arena->magic = ONIX_MAGIC;
        addr = (char *)((u32)arena + sizeof(arena_t));  // 计算去除arena后的内存地址并返回
        return addr;
    }

    for (size_t i = 0; i < DESC_COUNT; i++)   // 当申请内存小于等于1024字节时
    {
        desc = &descriptors[i];
        if (desc->block_size >= size)   // 从数组中找到刚好大于等于size的那个描述符
            break;
    }

    if (list_empty(&desc->free_list))     // 如果内存块空链表中已经没有了就新申请一页内存
    {
        arena = (arena_t *)alloc_kpage(1);  // 申请一页内存并置为0
        memset(arena, 0, PAGE_SIZE);
        arena->desc = desc;           // 设置该内存页的arena结果
        arena->large = false;
        arena->count = desc->total_block;  // count就是该页被分为了多少块，因为是刚申请下来的
        arena->magic = ONIX_MAGIC;
        for (size_t i = 0; i < desc->total_block; i++)   // 新申请的快全部加入空闲链表
        {
            block = get_arena_block(arena, i);
            list_push(&arena->desc->free_list, block);
        }
    }
    block = list_pop(&desc->free_list);  // 从空闲列表返回一个内存块
    arena = get_block_arena(block);      // 获取到改内存块的 arena 结构
    assert(arena->magic == ONIX_MAGIC && !arena->large);
    arena->count--;      // arena 的块数量减1
    return block;
}
```
以上就是申请内存的实现，假设需要申请58字节的内存，实际会申请出来一个64字节的内存，这样会造成一些内存的浪费但是方便了内存管理。

### 1.2、释放内存
申请了内存，用完必须进行内存释放，否则就会造成内存泄漏。
```cpp
void kfree(void *ptr)
{
    block_t *block = (block_t *)ptr;
    arena_t *arena = get_block_arena(block);   // 获取到块对应的arena
    assert(arena->large == 1 || arena->large == 0);
    if (arena->large)         // 如果超过了1024字节，字节释放内存页
    {
        free_kpage((u32)arena, arena->count);
        return;
    }
    list_push(&arena->desc->free_list, block);  // 如果小于1024，把该内存块放回空闲链表即可
    arena->count++;

    if (arena->count == arena->desc->total_block)  // 如果该页所有块都被释放了，那就把整页内存也给释放
    {
        for (size_t i = 0; i < arena->desc->total_block; i++)
        {
            block = get_arena_block(arena, i);
            list_remove(block);
        }
        free_kpage((u32)arena, 1);
    }
}
```
这样我们实现了内核堆内存的申请和释放。这里存在一个问题点：我们将内存块直接放入链表，如果申请了内存块之后写入的数据大于了内存块的大小，这样就会破坏双向链表的指针。

### 1.3、申请内存测试
我们完成了内核堆内存的管理，对内存的申请和释放进行下测试。创建一个测试进程来申请内存。
```cpp
void test_thread()
{
    set_interrupt_state(true);
    u32 counter = 0;
    while (true)
    {
        void *ptr = kmalloc(1200);
        LOGK("kmalloc 0x%p....\n", ptr);
        kfree(ptr);
        ptr = kmalloc(1024);
        LOGK("kmalloc 0x%p....\n", ptr);
        kfree(ptr);
        ptr = kmalloc(54);
        LOGK("kmalloc 0x%p....\n", ptr);
        kfree(ptr);
        sleep(5000);
    }
}
```
如上所示，我们申请了不同大小的内存，并打印申请后的内存地址。


## 二、用户内存映射
前边文章中我们做了内核内存映射，指定了8M以前的位置给内存使用，8M开始往后的位置给用户使用。但是用户每个进程的可用物理内存区域都是不一样的，因此用户内存映射比内核内存映射更复杂。但是映射的原理基本是一致的。

在内核内存映射中，我们将也目录放到了 0x1000 的位置，然后在 0x2000、0x3000防止了两个页表，共映射了8M的内存空间。在用户内存映射，需要进行适当的修改。首先定义了几个宏定义，指定了页目录的存放位置，用户栈顶地址和内核使用的内存大小。
```cpp
#define KERNEL_MEMORY_SIZE 0x800000  // 内核占用的内存大小 8M, 以后可能还会修改
#define USER_STACK_TOP 0x8000000     // 用户栈顶地址 128M
#define KERNEL_PAGE_DIR 0x1000       // 内核页目录索引
#define PDE_MASK 0xFFC00000
static u32 KERNEL_PAGE_TABLE[] = {   // 内核页表索引
    0x2000,
    0x3000,
};
```
这里把用户栈顶地址定义在了128M的位置，是因为我们自制的操作系统最多能使用128M的内存空间。这样是为了实现上更方便。在实现方便，修改了get_pte函数。因为我们要映射用户的内存，0x2000和0x3000映射了前8M内核内存，因此用户内存就需要往后找，此时有可能页表是不存在的，因此需要做一些处理，。如下：
```cpp
static page_entry_t *get_pte(u32 vaddr, bool create)  // 获取虚拟地址 vaddr 对应的页表
{
    page_entry_t *pde = get_pde();   // 找到页目录
    u32 idx = DIDX(vaddr);           // 找到页目录的索引，即页表的入口
    page_entry_t *entry = &pde[idx];   // 找到页表的入口位置
    assert(create || (!create && entry->present));  // 判断要么页表存在，要么不存在但create为true
    page_entry_t *table = (page_entry_t *)(PDE_MASK | (idx << 12));  // 找到可以修改的页表

    if (!entry->present)  // 如果页表入口不存在
    {
        LOGK("Get and create page table entry for 0x%p\n", vaddr);
        u32 page = get_page();     // 获取一页物理内存
        entry_init(entry, IDX(page));   // 初始化并链接到entry上
        memset(table, 0, PAGE_SIZE);
    }
    return table;    // 返回该虚拟地址对应的页表
}
```
这里的create参数就代表是否存在。如果不存在就创建。

### 2.1、映射物理内存
以下两盒函数实现将虚拟地址映射为物理内存和解除虚拟地址映射的物理内存。
```cpp
void link_page(u32 vaddr)
{
    ASSERT_PAGE(vaddr);   // 判断虚拟地址为页开始的位置，即最后三位为0
    page_entry_t *pte = get_pte(vaddr, true);  // 获取vaddr对应的页表
    page_entry_t *entry = &pte[TIDX(vaddr)];  // 获取vaddr页框的入口
    task_t *task = running_task();            // 获取当前的进程
    bitmap_t *map = task->vmap;               // 当前进程的虚拟位图
    u32 index = IDX(vaddr);                   // 获取vaddr对应的位图索引，标记这一页是否被占用

    if (entry->present)       // 如果页面已存在，说明该虚拟地址已经被映射了。则直接返回
    {
        assert(bitmap_test(map, index));  // 判断位图中这一位是否为1
        return;
    }
    assert(!bitmap_test(map, index));  
    bitmap_set(map, index, true);   // 将位图中这一位置为1
    u32 paddr = get_page();         //找到一页物理内存
    entry_init(entry, IDX(paddr));   // 将物理内存初始化为页表入口
    flush_tlb(vaddr);                // 刷新快表
    LOGK("LINK from 0x%p to 0x%p\n", vaddr, paddr);
}


void unlink_page(u32 vaddr)   // 去掉 vaddr 对应的物理内存映射
{
    ASSERT_PAGE(vaddr);
    page_entry_t *pte = get_pte(vaddr, true);
    page_entry_t *entry = &pte[TIDX(vaddr)];
    task_t *task = running_task();
    bitmap_t *map = task->vmap;
    u32 index = IDX(vaddr);

    if (!entry->present)
    {
        assert(!bitmap_test(map, index));
        return;
    }
    assert(entry->present && bitmap_test(map, index));
    entry->present = false;
    bitmap_set(map, index, false);
    u32 paddr = PAGE(entry->index);
    DEBUGK("UNLINK from 0x%p to 0x%p\n", vaddr, paddr);
    if (memory_map[entry->index] == 1)   // 判断物理内存是否被多次引用
    {
        put_page(paddr);
    }
    flush_tlb(vaddr);
}
```
unlink_page 和 link_page刚好是相反的操作，这里不在逐行注释。

## 三、进入用户态修改
在做完用户内存映射后，一些其他代码需要做对应的修改。先看task的修改。在进入用户态的函数中不能再使用内核的虚拟位图了。因此需要申请一块内存来作为位图，并需要申请一页内存来存储位图。一页的内存位图最多能表示128M的内存空间，这也是前边我们提到的为了简便我们的操作系统最大只能使用128M内存。
```cpp
void task_to_user_mode(target_t target)
{
    task_t *task = running_task();

    task->vmap = kmalloc(sizeof(bitmap_t)); // todo kfree
    void *buf = (void *)alloc_kpage(1);     // todo free_kpage
    bitmap_init(task->vmap, buf, PAGE_SIZE, KERNEL_MEMORY_SIZE / PAGE_SIZE);
    ......
}
```

## 四、用户内存映射测试
用户内存映射的测试主要是测试unlink_page 和 link_page这两个函数，这两个函数需要再用户态测试，但是这两个函数又需要再内核态调用。因此测试需要通过系统调用来实现，我们这里修改test系统调用来实现测试。即在 sys_test 中调用这两个函数。在user_init_thread中直接调用test系统调用即可完成测试。
![图片加载失败](/post_images/os/{{< filename >}}/4-01.png)
如图所示：在执行完link_page后，先申请了一页内存作为页表，然后申请了一页物理内存作为页框映射到了 0x1600000 的虚拟内存地址。注意：这里unlink_page只释放了映射，但是没有释放页表，因为必要性不太大。

最后理一下 0x1600000 是如何映射到物理地址的。
* 虚拟地址转换为二进制 0x01600000 = 0b 00000001 01100000 00000000 00000000
* 页目录索引 PDI = 0x05（第 6 项），也就是二进制的前 10 位, 这里是在get_pte函数中设置的
* 页表索引 PTI = 0x200（第 513 项）也就是二进制的中 10 位
* 页内偏移 = 0x000
因此通过这样的映射，就被映射到了 0x8001000的物理内存位置。

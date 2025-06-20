---
title:       自制操作系统 - 内核内存映射
subtitle:    "内核内存映射"
description: "内核使用前边8M的内存，本文将对内核使用的内存进行映射。在已经启用内存分页后通过页目录最后一项指向自身来修改页目录和页表。并通过位图来管理内核的1M~8M的内存位置。"
excerpt:     "内核使用前边8M的内存，本文将对内核使用的内存进行映射。在已经启用内存分页后通过页目录最后一项指向自身来修改页目录和页表。并通过位图来管理内核的1M~8M的内存位置。"
date:        2024-01-29T20:14:33+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/90/b8/sea_ocean_water_man_swim-78244.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-kernel-memory-map"
categories:  [ "操作系统" ]
---

## 一、修改页目录和页表
在启用内存分页后，如何能修改页目录和页表呢？一般来说会将最后一个页表指向页目录自身，不过这样会浪费最后 4M 的线性地址空间，只能用来管理页表。映射完成后的页面分布就是这样的：
![图片加载失败](/post_images/os/{{< filename >}}/1-01.png)
如图：cr3寄存器存储的是页目录的地址，页目录的最后一项改为它自身的地址（注意这是0x1007, 7是属性，索引只取前20位，因此最后一项指向的是0x1000）。这样在页目录最后一项找页表找到它自身，页目录这个位置又会被当做页表，继续找页表的最后一项又会把同样的这个位置认为是页框。因此就可以通过固定的虚拟地址修改页目录和页表
```cpp
page_entry_t *entry = &pde[1023];            // 将最后一个页表指向页目录自己
entry_init(entry, IDX(KERNEL_PAGE_DIR));
static page_entry_t *get_pde()
{
    return (page_entry_t *)(0xfffff000);
}
static page_entry_t *get_pte(u32 vaddr)
{
    return (page_entry_t *)(0xffc00000 | (DIDX(vaddr) << 12));
}
```
访问虚拟地址 0xFFFFF000 会通过 PDE[1023] 找到页目录自身,访问虚拟地址 0xFFC00xxx 会通过 PDE[1023] 和 PTE[0] 找到页目录（因为 PTI=0)。这里可以通过虚拟地址映射线性地址的方式自行推算。

### 1.1、刷新快表
修改完页目录和页表后需要刷新快表，快表是页目录和页表的缓存，刷新快表有两种方法，第一种是重新给cr3寄存器赋值会刷新所有页目录和页表，性能比较低。第二种是invlpg指令，它可以刷新某个虚拟地址特定的页表。

## 二、内核内存映射
内核使用前边8M的位置，因此需要将前 8M 的内存映射到自已原先的位置。内核页目录放到 0x1000的位置，页表共两个放到0x2000, 0x3000这两页。
```cpp
#define KERNEL_PAGE_DIR 0x1000   // 内核页目录索引
static u32 KERNEL_PAGE_TABLE[] = {   // // 内核页表索引
    0x2000,
    0x3000,
};
```
初始化内存映射的代码逻辑和上一篇文章还是类似的，区别是将最后一个页表指向了页目录自身，第0页未做映射为造成空指针访问，缺页异常，便于排错。
![图片加载失败](/post_images/os/{{< filename >}}/2-01.png)
如图所示，最开始的8M被映射到了前8M的位置(除了第0个页的4k未做映射)。0xfffff000被映射到了0x1000用于页目录的修改，0xffc00000后的两个页映射到了0x2000用于页表的修改。


## 三、内存映射测试
通过以下代码进行内存映射的测试，代码如下：
```cpp
void memory_test()
{
    BMB;
    // 将 20 M 0x1400000 内存映射到 64M 0x4000000 的位置
    // 我们还需要一个页表，0x900000
    u32 vaddr = 0x4000000; // 线性地址几乎可以是任意的
    u32 paddr = 0x1400000; // 物理地址必须要确定存在
    u32 table = 0x900000;  // 页表也必须是物理地址
    page_entry_t *pde = get_pde();
    page_entry_t *dentry = &pde[DIDX(vaddr)];
    entry_init(dentry, IDX(table));
    page_entry_t *pte = get_pte(vaddr);
    page_entry_t *tentry = &pte[TIDX(vaddr)];
    entry_init(tentry, IDX(paddr));
    BMB;
    char *ptr = (char *)(0x4000000);
    ptr[0] = 'a';
    BMB;
    entry_init(tentry, IDX(0x1500000));
    flush_tlb(vaddr);
    BMB;
    ptr[2] = 'b';
    BMB;
}
```
如上所示，在线性地址64M的位置分别写入了a和b，但是实际写入的物理地址在14M和15M的位置。


## 四、数据结构位图
为了节约内存的使用，我们可以使用位图来标记一些二值 (0/1) 的信息；
```cpp
typedef struct bitmap_t
{
    u8 *bits;   // 位图缓冲区
    u32 length; // 位图缓冲区长度
    u32 offset; // 位图开始的偏移
} bitmap_t;

// 初始化位图, 全部置为0
void bitmap_init(bitmap_t *map, char *bits, u32 length, u32 offset);
// 构造位图
void bitmap_make(bitmap_t *map, char *bits, u32 length, u32 offset);
// 测试位图的某一位是否为 1
bool bitmap_test(bitmap_t *map, u32 index);
// 设置位图某位的值
void bitmap_set(bitmap_t *map, u32 index, bool value);
// 从位图中得到连续的 count 位，找到连续多个值为0的位置
int bitmap_scan(bitmap_t *map, u32 count);
```
位图的函数原形如上所示，实现比较简单，不再贴位图实现的代码。


## 五、内核虚拟内存管理
我们这里使用位图来管理内核使用的内存。通过位图标记了内存哪些页被使用了，哪些页未被使用。前边我们知道内核使用前8M。内核内存布局如下所示：
![图片加载失败](/post_images/os/{{< filename >}}/5-01.png)
如上，也就是从start_page开始的位置属于内核可用的内存区域。实现如下：
```cpp
#define KERNEL_MAP_BITS 0x4000   // 内核位图存放的位置
#define KERNEL_MEMORY_SIZE (0x100000 * sizeof(KERNEL_PAGE_TABLE))  // 对1M往后的位置做管理，1M以后属于内核程序
bitmap_t kernel_map;
// 初始化内核虚拟内存位图，需要 8 位对齐
u32 length = (IDX(KERNEL_MEMORY_SIZE) - IDX(MEMORY_BASE)) / 8;
bitmap_init(&kernel_map, (u8 *)KERNEL_MAP_BITS, length, IDX(MEMORY_BASE));
bitmap_scan(&kernel_map, memory_map_pages);   // 物理内存数组占用的两页置为1

// 分配 count 个连续的内核页
u32 alloc_kpage(u32 count)
{
    assert(count > 0);
    u32 vaddr = scan_page(&kernel_map, count);
    LOGK("ALLOC kernel pages 0x%p count %d\n", vaddr, count);
    return vaddr;
}

// 释放 count 个连续的内核页
void free_kpage(u32 vaddr, u32 count)
{
    ASSERT_PAGE(vaddr);
    assert(count > 0);
    reset_page(&kernel_map, vaddr, count);
    LOGK("FREE  kernel pages 0x%p count %d\n", vaddr, count);
}
```
代码如上：对内核1~8M的内存进行管理，前边两页是物理内存数组占用的直接置1。通过两个函数alloc_kpage 和 free_kpage进行内核页的申请和释放。scan_page和reset_page是从位图中找到连续的页和重置这些页为可用状态。
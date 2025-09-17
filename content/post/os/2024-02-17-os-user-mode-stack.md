---
title:       自制操作系统 - 进程用户态栈
subtitle:    "进程用户态栈"
description: "在进程进入用户态后，栈应该修改到128M的位置。因为栈是向下增长的，且我们的操作系统最大支持128M内存。用户态栈最大占用2M的物理内存空间，即 0x7e00000 ~ 0x80000000 的空间范围。用户访问这块内存空间就报缺页异常。"
excerpt:     "在进程进入用户态后，栈应该修改到128M的位置。因为栈是向下增长的，且我们的操作系统最大支持128M内存。用户态栈最大占用2M的物理内存空间，即 0x7e00000 ~ 0x80000000 的空间范围。用户访问这块内存空间就报缺页异常。"
date:        2024-02-17T16:05:57+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/5c/7c/elephant_animals_asia_large_bright_close_the_environment_survey-1366104.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-user-mode-stack"
categories:  [ "操作系统" ]
---

## 一、进程用户态栈
在进程进入用户态后，栈应该修改到128M的位置。因为栈是向下增长的，且我们的操作系统最大支持128M内存。用户态栈最大占用2M的物理内存空间，即 0x7e00000 ~ 0x80000000 的空间范围。用户访问这块内存空间就报缺页异常。

这里我们正好利用缺页异常这个机制，在触发缺页异常后再映射用户栈这2M内存。因为我们不知道用户程序会用到多大的栈，有可能使用一页就足够了。

## 二、用户态栈实现
用户态栈实现涉及到内存的处理和缺页异常的实现。

### 2.1、定义用户栈使用的内存
首先需要使用宏定义来定义用户态栈的大小和位置。当触发缺页异常时，CPU会把访问造成缺页的地址放到cr2寄存器中，因此还需要实现从cr2寄存器中获取到缺页的地址。
```cpp
#define USER_STACK_TOP 0x8000000     // 用户栈顶地址 128M
#define USER_STACK_SIZE 0x200000     // 用户栈最大 2M
#define USER_STACK_BOTTOM (USER_STACK_TOP - USER_STACK_SIZE)  // 用户栈底地址 128M - 2M
 
u32 get_cr2()  // 得到 cr2 寄存器
{
    asm volatile("movl %cr2, %eax\n");    // 直接将 mov eax, cr2，返回值在 eax 中
}

page_entry_t *copy_pde()    // 拷贝当前页目录
{
    task_t *task = running_task();
    page_entry_t *pde = (page_entry_t *)alloc_kpage(1); // todo free
    memcpy(pde, (void *)task->pde, PAGE_SIZE);
    page_entry_t *entry = &pde[1023];      // 将最后一个页表指向页目录自己，方便修改
    entry_init(entry, IDX(pde));
    return pde;
}

typedef struct page_error_code_t
{
    u8 present : 1;
    u8 write : 1;
    u8 user : 1;
    u8 reserved0 : 1;
    u8 fetch : 1;
    u8 protection : 1;
    u8 shadow : 1;
    u16 reserved1 : 8;
    u8 sgx : 1;
    u16 reserved2;
} _packed page_error_code_t;

void page_fault(
    u32 vector,
    u32 edi, u32 esi, u32 ebp, u32 esp,
    u32 ebx, u32 edx, u32 ecx, u32 eax,
    u32 gs, u32 fs, u32 es, u32 ds,
    u32 vector0, u32 error, u32 eip, u32 cs, u32 eflags)
{
    assert(vector == 0xe);
    u32 vaddr = get_cr2();    // 通过cr2寄存器获取触发缺页异常的虚拟地址
    LOGK("fault address 0x%p\n", vaddr);
    page_error_code_t *code = (page_error_code_t *)&error;
    task_t *task = running_task();
    assert(KERNEL_MEMORY_SIZE <= vaddr < USER_STACK_TOP);  // 判断虚拟地址位于用户内存空间，即8M以上的位置
    if (!code->present && (vaddr > USER_STACK_BOTTOM))   // 如果虚拟地址属于用户态栈，就申请进行内存映射
    {
        u32 page = PAGE(IDX(vaddr));
        link_page(page);
        return;
    }
    panic("page fault!!!");
}
```
copy_pde函数实现的是拷贝页表，用户进程中每个进程都是使用自己独立的页表，因此需要每个用户进程需要拷贝页表。copy_pde函数会获取当前进程的页目录，申请一页内核内存，把当前进程的页目录拷贝到新申请的这页内存中。 这里拷贝只拷贝了页目录，因此拷贝完后页目录中的内容是不变的， 也就是前8M依然映射到了前8M。

page_error_code_t结构是缺页异常号每位代表的信息。page_fault函数是用于处理缺页异常的。如果引发了缺页异常且异常地址位于用户态的栈空间范围内，就进行内存映射。

### 2.2、缺页异常处理
缺页异常的异常编号是 0xe。之前触发确实异常我们是用默认的中断处理函数来处理的。在内存代码中我们已经定义了page_fault函数是用于处理缺页异常。因此在初始化中断描述符应该加上：
```cpp
handler_table[0xe] = page_fault;
```

### 2.3、进程任务修改
在任务状态中，我们存在激活任务的函数。现在需要进程调整，因此每个进程都有自己独立的页表，因此在切换进程时要判断当前cr3寄存器的页目录和即将被激活的任务pde是否相等
```cpp
void task_activate(task_t *task)   // 激活任务
{
    assert(task->magic == ONIX_MAGIC);
    if (task->pde != get_cr3())
    {
        set_cr3(task->pde);
    }
    if (task->uid != KERNEL_USER)
    {
        tss.esp0 = (u32)task + PAGE_SIZE;
    }
}

void task_to_user_mode(target_t target)
{
    ......
    // 创建用户进程页表
    task->pde = (u32)copy_pde();
    set_cr3(task->pde);
    ......
    iframe->esp = USER_STACK_TOP;
}
```
另外在进入用户态时，创建一个用户态的页表，因为之前的页表是内核态的。并将用户态的栈顶置于128M的位置。

## 三、测试
在开启进程的代码中进行测试，创建一个递归函数。
```cpp
void test_recursion()
{
    char tmp[0x400];
    test_recursion();
}

static void user_init_thread()
{
    u32 counter = 0;

    char ch;
    while (true)
    {
        printf("task is in user mode %d\n", counter++);
        BMB;
        test_recursion();
        sleep(1000);
    }
}
```
栈被设置到了128M的位置，因此在进入用户态时就会触发缺页异常，异常处理函数就会去做内存映射。因为我们是递归，因此栈顶会不断地递减，当栈使用完了2M的内存空间时，就会触发panic。
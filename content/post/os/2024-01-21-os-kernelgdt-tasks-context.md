---
title:       自制操作系统 - 内核GDT与任务上下文
subtitle:    "内核GDT与任务上下文"
description: "在内核加载器中实现了全局描述符，但是为了管理方便，需要把GDT写入到内核中，另外本文将介绍任务如何进行上下文切换的，从交替打印A和B来感受下进程切换的原理。"
excerpt:     "在内核加载器中实现了全局描述符，但是为了管理方便，需要把GDT写入到内核中，另外本文将介绍任务如何进行上下文切换的，从交替打印A和B来感受下进程切换的原理。"
date:        2024-01-21T14:26:25+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/28/53/waterfall_sunset_river_water_sky-140382.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-kernelgdt-tasks-context"
categories:  [ "操作系统" ]
---

## 一、内核全局描述符表
前面在内核加载器中实现了全局描述符，这里我们把全局描述符写到内核中去，这样管理起来更加的方便。首先定义GDT、段选择子、描述符指针结构体。然后定义全局描述符表初始化函数。
```cpp
descriptor_t gdt[GDT_SIZE]; // 内核全局描述符表, GDT_SIZE设置128
pointer_t gdt_ptr;          // 内核全局描述符表指针

// 初始化内核全局描述符表
void gdt_init()
{
    DEBUGK("init gdt!!!\n");
    asm volatile("sgdt gdt_ptr");  // 将内核加载器中实现的GDT保存到指针里面
    memcpy(&gdt, (void *)gdt_ptr.base, gdt_ptr.limit + 1);  // 将内核加载器的描述符表拷贝到内核的 gdt 里面
    gdt_ptr.base = (u32)&gdt;         
    gdt_ptr.limit = sizeof(gdt) - 1;
    asm volatile("lgdt gdt_ptr\n");  // 加载内核的GDT.
}
```
执行完唯一的变化是全局描述符由3个变成了128个。前边3个还是和原来内核加载器的gdt一样，后边新的125个size都是0。


## 二、任务及上下文
任务就是一个进程或线程，也可以理解为一个执行流。它需要有一个程序入口地址，这可以理解我们在高级语言编程中的线程入口函数。每个进程都有自己的堆栈和寄存器信息。因此进程切换时要保存寄存器信息。

### 2.1、ABI调用约定
ABI调用（Application Binary Interface）规定了函数调用约定，寄存器的使用规则。哪些寄存器需要保存等信息。Linux系统使用的是System V ABI。需要实现方保存的寄存器有ebx、esi、 edi、 ebp、 esp， 调用完成后这些寄存器的值不变。需要调用方保存的寄存器有eax, ecx, edx。例如eax需要存返回值，是有可能发生改变的。

### 2.1、任务内存分布
在80386CPU中，内存按照4K大小进行分页， 每一页开始的内存地址最低三位都是0。任务是存储在内存中的，这里给每个进程分配一页大小的内存。如图所示为一页内存中进程的信息：
<rawhtml>
<img src="/post_images/os/{{< filename >}}/2.1-01.png" alt="图片加载失败" width="300"/>
</rawhtml>
PCB是进程控制块的缩写。最高位置是栈底，栈向下增长。

### 2.2、进程上下切换实现
我们首先来定义进程相关的结构
```cpp
typedef u32 target_t();  // 定义进程的入口地址函数

typedef struct task_t
{
    u32 *stack; // 内核栈，每个任务自己的栈分配一页内存
} task_t;

typedef struct task_frame_t  // ABI约定要保存的寄存器，在进行切换前要保存。
{
    u32 edi;
    u32 esi;
    u32 ebx;
    u32 ebp;
    void (*eip)(void);
} task_frame_t;

void task_init();  // 任务初始化函数
```
接下来我们来实现一个在屏幕上打印A和B的进程，轮流调度这两个进程，实现轮流打印A和B的功能。代码实现：
```cpp
#define PAGE_SIZE 0x1000  // 定义一页的大小为4K

//定义进程的内存位置x1000，这个内存位置我们放了内核加载器，当然现在已经进入了内核，内核加载器已经没用了
task_t *a = (task_t *)0x1000;  
task_t *b = (task_t *)0x2000;

extern void task_switch(task_t *next);

task_t *running_task() // 获取当前运行的进程
{
    asm volatile(
        "movl %esp, %eax\n"   // 获取当前进程的esp，即栈顶
        "andl $0xfffff000, %eax\n");  // 获取当前进程内存页开始的位置，即得到当前进程内存所在页
}

void schedule()
{
    task_t *current = running_task();
    task_t *next = current == a ? b : a;  // 如果当前是a就换b，否则就换a。交替执行
    task_switch(next);
}

u32 thread_a()
{
    while (true)
    {
        printk("A");
        schedule();
    }
}

u32 thread_b()
{
    while (true)
    {
        printk("B");
        schedule();
    }
}

static void task_create(task_t *task, target_t target)
{
    u32 stack = (u32)task + PAGE_SIZE;  // 设置进程所在页的最高位为栈顶

    stack -= sizeof(task_frame_t);   // 内存的最高位存储task_frame_t结构
    task_frame_t *frame = (task_frame_t *)stack;
    frame->ebx = 0x11111111;   // 这里我们用不到这些个寄存器，随便存一些值
    frame->esi = 0x22222222;
    frame->edi = 0x33333333;
    frame->ebp = 0x44444444;
    frame->eip = (void *)target; // eip指向程序的入口位置

    task->stack = (u32 *)stack;  // 将task结构的栈指向这里
}

void task_init()
{
    task_create(a, thread_a);  // 创建两个进程
    task_create(b, thread_b);
    schedule();
}
```
从这个代码我们可以看到，创建两个任务，分配到内存0x1000和0x2000的位置。然后交替调度两个任务。这里重点是task_switch函数，这个函数用汇编实现，代码如下：
```asm
task_switch:
    push ebp   ; 保存栈帧
    mov ebp, esp

    push ebx   ; 保存ABI约定的三个寄存器，保存了当前任务的寄存器
    push esi
    push edi

    mov eax, esp;
    and eax, 0xfffff000 ; 获取当前的任务信息

    mov [eax], esp   ; 保存当前线程的栈顶

    mov eax, [ebp + 8] ; 参数传进来的next，即下一个进程的内存地址
    mov esp, [eax]     ; 恢复next线程的栈顶， 注意这里esp已经指向下一个任务的栈顶了

    pop edi   ; 恢复下一个任务的寄存器
    pop esi
    pop ebx
    pop ebp

    ret
```

## 三、调试进程切换
现在我们启动调试，看下0x1000处的内存，如下所示：
![图片加载失败](/post_images/os/{{< filename >}}/3-01.png)
如图可以看到，在0x1000这一页内存位置，最高位存储是我们代码中设置的寄存器值，低位是PCB信息，PCB的最低位也是保存的esp。和我们前边给的图一一对应。

这里我们重点关注一下 eip 寄存器，它被设置在栈底的位置，task_switch函数在弹出保存的栈之后，最后一个就是进程的函数入口地址，ret指令会弹出这个地址给eip寄存器，CPU就会继续去eip指向的地址取指令并运行，这样就实现了进程切换。

---
title:       自制操作系统 - 创建内核线程
subtitle:    "创建内核线程"
description: "在内核进行相关组件初始化后，就会进入内核线程，执行内核的相关任务。本文将创建内核线程，通过时间片和任务优先级进行线程的切换和调度"
excerpt:     "在内核进行相关组件初始化后，就会进入内核线程，执行内核的相关任务。本文将创建内核线程，通过时间片和任务优先级进行线程的切换和调度"
date:        2024-01-31T18:20:40+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/b6/8b/guitar_classical_guitar_acoustic_guitar_electric_guitar_musician_band_recording_studio_sound-732162.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-kernel-thread"
categories:  [ "操作系统" ]
---

## 一、内核线程说明
在前面章节任务上下文和中断上下文中，我们知道每个进程占用一页内存。并且通过时钟中断进行任务的切换。但是我们之前创建的任务是没有状态的，只是循环在屏幕打印A和B。这里我们将创建内核进程并分片时间片和优先级，通过时钟中断来完成任务调度。

在内核加载器中启用了保护模式后，我们将栈顶设置在了0x10000的位置，因此在创建内核任务前，内核栈位于0xf000这一页内存中。

## 二、创建内核线程
我们需要做一些宏定义，约束任务名的长度，以及是内核线程还是用户线程
```cpp
#define KERNEL_USER 0       // 内核用户
#define NORMAL_USER 1       // 普通用户
#define TASK_NAME_LEN 16    // 任务名的长度
typedef enum task_state_t   // 定义任务的几种状态
{
    TASK_INIT,     // 初始化
    TASK_RUNNING,  // 执行
    TASK_READY,    // 就绪
    TASK_BLOCKED,  // 阻塞
    TASK_SLEEPING, // 睡眠
    TASK_WAITING,  // 等待
    TASK_DIED,     // 死亡
} task_state_t;

typedef struct task_t   // 定义任务结构
{
    u32 *stack;              // 内核栈
    task_state_t state;      // 任务状态
    u32 priority;            // 任务优先级
    u32 ticks;               // 剩余时间片
    u32 jiffies;             // 上次执行时全局时间片
    char name[TASK_NAME_LEN]; // 任务名
    u32 uid;                 // 用户 id
    u32 pde;                 // 页目录物理地址
    struct bitmap_t *vmap;   // 进程虚拟内存位图
    u32 magic;               // 内核魔数，用于检测栈溢出
} task_t;
```
如上所示，定义了任务结构体，包含任务栈，任务状态、优先级、剩余时间片等信息。还定义了 task_frame_t 结构提，包含 edi、esi、ebx、ebp、void (*eip)(void)等五个寄存器的值。


### 2.1、创建任务
相比之前的实现，创建任务的参数增加了任务名称，任务优先级，用户id等参数，实现如下：
```cpp
static task_t *task_create(target_t target, const char *name, u32 priority, u32 uid)
{
    task_t *task = get_free_task();
    memset(task, 0, PAGE_SIZE);

    u32 stack = (u32)task + PAGE_SIZE;

    stack -= sizeof(task_frame_t);
    task_frame_t *frame = (task_frame_t *)stack;
    frame->ebx = 0x11111111;
    frame->esi = 0x22222222;
    frame->edi = 0x33333333;
    frame->ebp = 0x44444444;
    frame->eip = (void *)target;
    strcpy((char *)task->name, name);
    task->stack = (u32 *)stack;
    task->priority = priority;
    task->ticks = task->priority;
    task->jiffies = 0;
    task->state = TASK_READY;
    task->uid = uid;
    task->vmap = &kernel_map;
    task->pde = KERNEL_PAGE_DIR;
    task->magic = ONIX_MAGIC;
    return task;
}
```
如上所示为创建任务的实现，ebx等四个寄存器仍然先设置为固定值。这里注意存在一个魔数，魔数存在的目的是防止栈溢出。另外一个重要的函数是get_free_task。它从任务列表里获取一个空闲任务，

### 2.2、任务列表
如下为任务列表的实现，它最大支持64个任务。
```cpp
#define NR_TASKS 64
static task_t *task_table[NR_TASKS];  // 任务结构体数组
static task_t *get_free_task()        // 从 task_table 里获得一个空闲的任务
{
    for (size_t i = 0; i < NR_TASKS; i++)
    {
        if (task_table[i] == NULL)
        {
            task_table[i] = (task_t *)alloc_kpage(1); // todo free_kpage
            return task_table[i];
        }
    }
    panic("No more tasks");
}
```
任务列表最大支持64个，每创建一个任务就在任务数组中申请一个位置，并申请一页内存。当任务数组达到64个就会报错。

### 2.3、任务查找与调度
当多个任务同时执行时，CPU要根据时间片和优先级轮询对执行多个任务。当正在执行的任务时间片耗尽之后，需要去任务列表查找下一个需要调度的任务，并将当前任务转为就绪态放回任务列表中。任务查找的代码如下：
```cpp
static task_t *task_search(task_state_t state)  // 从任务数组中查找某种状态的任务，自己除外
{
    assert(!get_interrupt_state());
    task_t *task = NULL;
    task_t *current = running_task();  // 获取当前正在运行的任务

    for (size_t i = 0; i < NR_TASKS; i++)   // 遍历任务数组
    {
        task_t *ptr = task_table[i];
        if (ptr == NULL)   // 如果任务为空跳过
            continue;

        if (ptr->state != state)   // 如果任务不为当前查找的任务状态跳过
            continue;
        if (current == ptr)    // 如果任务为当前任务跳过
            continue;
        // 如果任务为NULL或者task的剩余时间片大于ptr的时间片，如果剩余时间片一样，就执行上次时间片较晚执行的那个，
        // 这里是循环从任务数组中找到剩余时间片最高的那个任务做为下一个任务。
        if (task == NULL || task->ticks < ptr->ticks || ptr->jiffies < task->jiffies)
            task = ptr;
    }
    return task;
}
```
找到下一个要运行的任务后，就可以进行任务调度了，进行任务的上下文切换。任务调度的代码我们就不贴了，他的核心是把当前任务的状态改为就绪态，把下一个任务的状态改为执行态。并执行任务切换函数。task_switch就是我们前边用汇编实现的任务上下文切换。


### 2.4、时钟中断处理
有了内核线程后，就需要设置时钟中断来进行任务调度的切换，我们这里看下时钟中断的处理。
```cpp
void clock_handler(int vector)
{
    assert(vector == 0x20);
    send_eoi(vector);
    stop_beep();
    jiffies++;  // 全局时间片+1
    // DEBUGK("clock jiffies %d ...\n", jiffies);
    task_t *task = running_task();   // 获取当前运行的任务
    assert(task->magic == ONIX_MAGIC);
    task->jiffies = jiffies;    // 设置任务的时间片为当前系统时间片
    task->ticks--;      // 当前任务的时间片减1
    if (!task->ticks)   // 如果当前任务的时间片已经为0了，就把当前任务的时间片设置为它的优先级
    {
        task->ticks = task->priority;
        schedule();
    }
}
```
这里每次时钟周期都要调用一下stop_beep()，原因如果有输出\a字符就会触发蜂鸣器，蜂鸣器的代码设置了5个时钟周期且蜂鸣结束后不会主动去停止蜂鸣，因此每次时钟周期都需要去判断一下蜂鸣器是否应该关闭，否则就会一直蜂鸣。


## 三、运行任务
前边我们知道在进入内核进程之前，内核栈位于0x9f00这个位置，但是这个进程是初始化内核操作的，它并没有设置魔数等，但是我们进程切换时要判断当前进程的魔数，因此需要先设置0x9f00这个进程的魔数，因此一旦进程内核进程后就再也不会回到这个进程。
```cpp
static void task_setup()
{
    task_t *task = running_task();
    task->magic = ONIX_MAGIC;
    task->ticks = 1;
    memset(task_table, 0, sizeof(task_table));
}
u32 thread_c()
{
    set_interrupt_state(true);
    while (true)
    {
        printk("C");
    }
}
void task_init()
{
    task_setup();
    task_create(thread_a, "a", 5, KERNEL_USER);
    task_create(thread_b, "b", 5, KERNEL_USER);
    task_create(thread_c, "c", 5, KERNEL_USER);
}
```
如上所示，通过任务创建函数创建了3个任务，当然这里省略了A和B任务的代码，和C是一样的。这里重点是task_setup，它设置了ox9f00这个位置的进程的魔数和时间片，并初始化了任务数组全为0.然后就进入了线程创建等待下一个时钟周期进行任务调度。这里可以看下任务的调度图：
![图片加载失败](/post_images/os/{{< filename >}}/3-01.png)

这里还有一个重点是在内核初始化中断时，中断是关闭的。在task_init()之后才打开中断，因此初始化进行不会被中断扰乱。且在内核的ABC三个进行都这是了开中断，原因是我们中断使用的是中断门，每次中断后都会自动关中断，因此线程中需要将中断打开才能接受下一个时钟中断。


## 四、说明
这里之所以称为线程，是因为A、B、C三个任务共用的同一个页目录和页表。这里还有有一个bug，在打印字符时会出现屏幕为空的现象。这是因为在打印时可能随时发生中断。如果在给缓冲区写入字符但是还未写入字符样式时发生了中断。此时pos就会变成奇数。这样下一个进程再往字符缓冲区输入内容时就无法正常显示了。总是这属于多线程共用静态变量争抢造成的问题。

解决这个问题的方式在将console_write函数作为临界区，在进入函数时关闭中断，函数执行结束再打开中断。等后续实现了锁之后就可以通过加锁机制进行处理了。
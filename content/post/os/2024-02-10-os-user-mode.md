---
title:       自制操作系统 - 进入用户模式
subtitle:    "进入用户模式"
description: "用户态和内核态的特权级不同，intel将特权级分为0~3，内核运行在0特权级，用户运行在3特权级，特权级的切换需要依赖任务状态段TSS完成。我们使用 TSS 的主要作用是利用其中的 ss0 和 esp0，使得用户态的程序可以转到内核态，不会使用TSS进行硬件任务切换。"
excerpt:     "用户态和内核态的特权级不同，intel将特权级分为0~3，内核运行在0特权级，用户运行在3特权级，特权级的切换需要依赖任务状态段TSS完成。我们使用 TSS 的主要作用是利用其中的 ss0 和 esp0，使得用户态的程序可以转到内核态，不会使用TSS进行硬件任务切换。"
date:        2024-02-10T20:49:05+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/f7/43/church_decoration_night_architecture_color_winter_church_street_burlington-1233900.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-user-mode"
categories:  [ "操作系统" ]
---

## 一、任务状态段
intel的CPU对任务分为了四个特权级，分别为level0~3。内核的特权级是level0，应用程序是level3，1和2特权级没有使用到。 任务状态段是用来进行任务的切换，即用户态到内核态的切换过程。任务被分为两个部分，执行空间和状态空间。

一个任务的执行需要有代码、数据和寄存器的值，寄存器的值就是状态。任务状态段就是用来保存寄存器的值。任务状态段 (TSS Task State Segment) 是 IA32 中一种二进制数据结构，保存了某个任务的信息，保护模式中 TSS 主要用于硬件任务切换，这种情况下，每个任务有自己独立的 TSS，对于软件的任务切换，也通常会有一两个任务状态段，使得任务从在中断时，能从 用户态(Ring3) 转到 内核态(Ring0)。

硬件任务切换由CPU硬件直接支持，通过专用指令（如x86的TSS任务状态段或ARM的PSTATE保存）自动完成寄存器状态的保存/恢复。软件任务切换完全由操作系统通过软件代码实现，手动保存/恢复寄存器状态（通常通过栈操作）。

硬件任务切换主要和任务门有关，使得任务切换时，CPU 可以自动保存任务上下文，但是这样干会很低效，所以现代操作系统一般不会用这种方法，而且为了更高的可移植性，也更应该使用软件的方式来做任务切换。

### 1.1、任务状态段结构
在硬件任务切换中，任务状态段保存了很多寄存器的信息，任务状态段的结构如下：
```cpp
typedef struct tss_t
{
    u32 backlink; // 前一个任务的链接，保存了前一个任状态段的段选择子
    u32 esp0;     // ring0 的栈顶地址
    u32 ss0;      // ring0 的栈段选择子
    u32 esp1;     // ring1 的栈顶地址
    u32 ss1;      // ring1 的栈段选择子
    u32 esp2;     // ring2 的栈顶地址
    u32 ss2;      // ring2 的栈段选择子
    u32 cr3;
    u32 eip;
    u32 flags;
    u32 eax;
    u32 ecx;
    u32 edx;
    u32 ebx;
    u32 esp;
    u32 ebp;
    u32 esi;
    u32 edi;
    u32 es;
    u32 cs;
    u32 ss;
    u32 ds;
    u32 fs;
    u32 gs;
    u32 ldtr;          // 局部描述符选择子
    u16 trace : 1;     // 如果置位，任务切换时将引发一个调试异常
    u16 reversed : 15; // 保留不用
    u16 iobase;        // I/O 位图基地址，16 位从 TSS 到 IO 权限位图的偏移
    u32 ssp;           // 任务影子栈指针
} _packed tss_t;
```
任务状态段用来低权级向高权级切换，因为没有 ring3 的栈顶地址和栈段选择子。

### 1.2、任务状态段描述符
任务状态段需要占用一块内存来保存。cpu如何知道它在哪呢，就需要用到任务状态段描述符，这个描述符是存在于全局描述符表中。，当 segment = 0 时，1001和1001表示TSS段。

任务状态段描述符，描述了当前正在执行的任务，B 表示busy 位， B 位为 0 时，表示任务不繁忙， B 位为 1 时，表示任务繁忙。CPU 提供了 TR(Task Register) 寄存器来存储该描述符，加载描述符到 TR 寄存器中的指令是 ltr。

我们使用 TSS 的主要作用是利用其中的 ss0 和 esp0，使得用户态的程序可以转到内核态。


### 1.3、TSS段代码实现
TSS段描述符需要定义在GDT中，如下：
```cpp
#define KERNEL_CODE_IDX 1    // 内核代码段
#define KERNEL_DATA_IDX 2    // 内核数据段
#define KERNEL_TSS_IDX 3     // 内核TSS段
#define USER_CODE_IDX 4      // 用户代码段
#define USER_DATA_IDX 5      // 用户数据段
#define KERNEL_CODE_SELECTOR (KERNEL_CODE_IDX << 3)
#define KERNEL_DATA_SELECTOR (KERNEL_DATA_IDX << 3)
#define KERNEL_TSS_SELECTOR (KERNEL_TSS_IDX << 3)        // 内核TSS段选择子，特权级为0
#define USER_CODE_SELECTOR (USER_CODE_IDX << 3 | 0b11)    // 用户段选择子的特权级为3
#define USER_DATA_SELECTOR (USER_DATA_IDX << 3 | 0b11)
```
在GDT初始化时，对用户态的描述符也进行了初始化，这里不再贴具体的代码。区别是用户态的DPL被设置为3，且目前用户态可以访问内核态的代码，这个后续会再改进。接下来是TSS的初始化：
```cpp
void tss_init()
{
    memset(&tss, 0, sizeof(tss));

    tss.ss0 = KERNEL_DATA_SELECTOR;   // 内核数据段的选择子
    tss.iobase = sizeof(tss);  // iobase的存在是为了用户态也可以直接做IO，目前我们用不到

    descriptor_t *desc = gdt + KERNEL_TSS_IDX;
    descriptor_init(desc, (u32)&tss, sizeof(tss) - 1);
    desc->segment = 0;     // 系统段
    desc->granularity = 0; // 字节
    desc->big = 0;         // 固定为 0
    desc->long_mode = 0;   // 固定为 0
    desc->present = 1;     // 在内存中
    desc->DPL = 0;         // 用于任务门或调用门
    desc->type = 0b1001;   // 32 位可用 tss  (1011表示 32位TSS忙)

    BMB;
    asm volatile(
        "ltr %%ax\n" ::"a"(KERNEL_TSS_SELECTOR));
}
```
接下来就可以通过bochs调试来观察GDT，初始化后的结果如下, 从GDT中可以看到TSS描述符的存在，并且被加载到了tr寄存器。
![图片加载失败](/post_images/os/{{< filename >}}/1.3-01.png)


## 二、进入用户态实现
用户态和内核态切换涉及两个关键点： 第一个是在任务状态段中，有两个变量ss0和esp0是特权级0的栈。还有一个关键中断门的处理过程，当发生中断时，如果中断处理程序运行在低特权级，那么栈切换就会发生：
* 内核特权级的 栈段选择子 和 栈顶指针 将会从当前的 TSS 段中获得，在内核栈中将会压入用户态的 栈段选择子 和 栈顶指针；
* 保存当前的状态 eflags, cs, eip 到内核栈
* 如果存在错误码的话，压入错误码

如果处理器运行在相同的特权级，那么相同特权级的中断代码将被执行：
* 保存当前的状态 eflags, cs, eip 到内核栈
* 如果存在错误码的话，压入错误码

### 2.1、进入用户态准备
从内核态进入用户态前，首先需要修改内核栈，修改用户栈，然后模拟一个中断，通过中断返回的方式来返回到用户态。有一点需要注意的是，在tss段初始化的已经已经被赋予了内核段的数据段选择子。

### 2.2、用户态代码实现
在内核态进入用户态，需要被中断来触发，因此需要定义中断帧，中断帧的内容在进入用户态时需要进行修改。定义如下：
```cpp
typedef struct intr_frame_t
{
    u32 vector;
    u32 edi;
    u32 esi;
    u32 ebp;
    // 虽然 pushad 把 esp 也压入，但 esp 是不断变化的，所以会被 popad 忽略
    u32 esp_dummy;
    u32 ebx;
    u32 edx;
    u32 ecx;
    u32 eax;
    u32 gs;
    u32 fs;
    u32 es;
    u32 ds;
    u32 vector0;
    u32 error;
    u32 eip;
    u32 cs;
    u32 eflags;
    u32 esp;
    u32 ss;
} intr_frame_t;
```
接下来就需要返回到用户态，返回用户态的函数如下：
```cpp
void task_to_user_mode(target_t target)  // target是进入用户态需执行的函数
{
    task_t *task = running_task();

    u32 addr = (u32)task + PAGE_SIZE;

    addr -= sizeof(intr_frame_t);
    intr_frame_t *iframe = (intr_frame_t *)(addr);

    iframe->vector = 0x20;    // 中断被模拟为时钟中断，
    iframe->edi = 1;
    iframe->esi = 2;
    iframe->ebp = 3;
    iframe->esp_dummy = 4;
    iframe->ebx = 5;
    iframe->edx = 6;
    iframe->ecx = 7;
    iframe->eax = 8;
    iframe->gs = 0;
    iframe->ds = USER_DATA_SELECTOR;   // 改为用户态的段选择子
    iframe->es = USER_DATA_SELECTOR;
    iframe->fs = USER_DATA_SELECTOR;
    iframe->ss = USER_DATA_SELECTOR;
    iframe->cs = USER_CODE_SELECTOR;
    iframe->error = ONIX_MAGIC;
    u32 stack3 = alloc_kpage(1); // todo replace to user stack， 这里是用内核内存申请了用户栈，因为此时我们还没有内存管理
    iframe->eip = (u32)target;
    iframe->eflags = (0 << 12 | 0b10 | 1 << 9);
    iframe->esp = stack3 + PAGE_SIZE;
    asm volatile(
        "movl %0, %%esp\n"
        "jmp interrupt_exit\n" ::"m"(iframe));
}
```


## 三、进入用户态
接下来就可以在线程初始化的函数中来调用进入用户态的函数了。如下：
```cpp
static void real_init_thread()
{
    u32 counter = 0;

    char ch;
    while (true)
    {
        BMB;
        // asm volatile("in $0x92, %ax\n");
        sleep(100);
        // LOGK("%c\n", ch);
        // printk("%c", ch);
    }
}

void init_thread()
{
    // set_interrupt_state(true);
    char temp[100]; // 为栈顶有足够的空间
    task_to_user_mode(real_init_thread);
}
```
现在就可以进行调试了。在进入了内核态后，再做IO就会触发一般性保护异常。并且进入用户态之后不能再使用内核态的打印函数。进入用户态之后可以通过系统调用回到内核态。我们这里梳理一下进入内核态的过程：
1. 内核初始化完成后，在main文件中打开中断，时间中断触发后调度到init进程
2. 执行init进程，init调用task_to_user_mode函数
3. task_to_user_mode函数中获取当前运行的进程，即init进程。并修改init进程的栈数据，如选择子修改为用户态选择子
4. task_to_user_mode函数将定义好的iframe赋值给esp，主动调用一次中断处理返回函数
5. CPU检测到特权级切换，IRET 指令从栈中恢复 ESP 和 SS
6. 这些被设置好栈数据和esp就会直接在中断返回函数中出栈。这样就返回到了read_init_thread函数进入了用户态

进入用户态后，cs的选择子是23，即cpu的rpl被设置为3，运行在特权级3上。此时再执行IO(int/out操作)操作就触发一般保护异常。在用户态时可以进行系统调用，系统调用时将特权级3转换为特权级0的。接下来是系统调用的过程：
1. 用户态执行sleep()函数，触发系统调用int 0x80
2. 中断处理函数保存用户态的寄存器，并将当前线程休眠，进行下一个线程的调度
3. 在线程调度执行task_activate函数，如果任务的id不是内核，就将tss.esp0设置为内核态的栈顶。

task_activate函数的实现如下，在schdule()中调用。每个任务都有自己独立的内核栈，如果是用户态进程系统调用，完成后会切换到这个位置。
```cpp
// 激活任务
void task_activate(task_t *task)
{
    assert(task->magic == ONIX_MAGIC);

    if (task->uid != KERNEL_USER)
    {
        tss.esp0 = (u32)task + PAGE_SIZE; // 获取用户进程的内核栈，因为只有用户进行做特权级切换会用到，内核进程时用不到的
    }
}
```
在调试过程中，你会发现当执行0x80系统调用时，CPU会自动检测当前的特权级属于用户态，进行特权级提升，栈自动从用户栈跳转到了内核栈，这是cpu硬件完成的，硬件自动 从 TSS（Task State Segment） 中加载 SS0 和 ESP0（内核栈指针），替换当前的用户栈 SS:ESP （SS0在TSS初始化时就已经指定了, ESP0在进程切换时进行赋值）。中断执行完就会通过 iret 回到用户态的栈。

## 四、说明
因为目前系统调用只有sleep，sleep之后就进入了idle线程，无法调试系统调用进入内核态执行完成后再返回用户态的过程。这里我们下一篇文章再进行调试。另外虽然目前已经进入了用户态，但是仍然可以操作内核的内存，因为目前还没有对用户态的内存进行管理。

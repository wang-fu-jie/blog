---
title:       自制操作系统 - CPU检测与FPU浮点运算
subtitle:    "CPU检测与FPU浮点运算"
description: "CPU检测指的是利用 cpuid 这个指令获取CPU的信息，如可以获取供应商字符串，验证FPU功能是否支持等。FPU也被成为x87， 后来被集成到CPU中，借助FPU可以实现浮点运算。"
excerpt:     "CPU检测指的是利用 cpuid 这个指令获取CPU的信息，如可以获取供应商字符串，验证FPU功能是否支持等。FPU也被成为x87， 后来被集成到CPU中，借助FPU可以实现浮点运算。"
date:        2024-04-03T19:12:49+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/4d/11/ruins_scary_mystery_fantasy_landscape_dark_virtual_set_digital_art-917183.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-cpuid-fpu"
categories:  [ "操作系统" ]
---

## 一、CPU检测
CPU检测指的是利用 cpuid 这个指令获取CPU的信息。比如供应商(Vendor) 字符串，和模型数，内部缓存大小等。在执行 CPUID 指令之前，首先应该检测处理器是否支持该指令，如果 EFLAGS 的 ID 位可修改，那么表示处理器支持 CPUID 指令。例如80386就不支持这个指令，但是奔腾处理器支持。

执行 CPUID 指令前，将参数 0 传入 EAX，不同的参数将返回不同的信息。当 EAX=0 时，CPUID 将返回供应商 ID 字符串到 EBX, EDX 和 ECX，将它们写入内存将得到长度 12 的字符串。这个字符串表示了供应商的 ID， EAX 中返回最大的输入数。  使用参数 EAX = 1 调用 CPUID，返回一个位图存储在 EDX 和 ECX 中 2。这个位图就标注CPU支持的功能等内容。注意不同品牌的 CPU 可能有不同的含义。

```cpp
bool cpu_check_cpuid()       // 检测是否支持 cpuid 指令
{
    bool ret;
    asm volatile(
        "pushfl \n" // 保存 eflags

        "pushfl \n"                   // 得到 eflags
        "xorl $0x00200000, (%%esp)\n" // 反转 ID 位
        "popfl\n"                     // 写入 eflags

        "pushfl\n"                  // 得到 eflags
        "popl %%eax\n"              // 写入 eax
        "xorl (%%esp), %%eax\n"     // 将写入的值与原值比较
        "andl $0x00200000, %%eax\n" // 得到 ID 位
        "shrl $21, %%eax\n"         // 右移 21 位，得到是否支持

        "popfl\n" // 恢复 eflags
        : "=a"(ret));
    return ret;
}

// 得到供应商 ID 字符串
void cpu_vendor_id(cpu_vendor_t *item)
{
    asm volatile(
        "cpuid \n"
        : "=a"(*((u32 *)item + 0)),
          "=b"(*((u32 *)item + 1)),
          "=d"(*((u32 *)item + 2)),
          "=c"(*((u32 *)item + 3))
        : "a"(0));
    item->info[12] = 0;
}

void cpu_version(cpu_version_t *item)
{
    asm volatile(
        "cpuid \n"
        : "=a"(*((u32 *)item + 0)),
          "=b"(*((u32 *)item + 1)),
          "=c"(*((u32 *)item + 2)),
          "=d"(*((u32 *)item + 3))
        : "a"(1));
}

```
以上为CPU的检测，可以在test系统调用中来进行测试。CPU检测的一大目的是，在操作系统运行时，可以先去检测CPU是否支持这个功能，例如当系统运行到一个非常古老的CPU上，就有可能存在功能不支持。例如检测是否支持FPU。


## 二、FPU浮点运算
x86 FPU最初是处理器的一个可选组件 x87，能够在硬件上执行浮点运算，但后来被集成到 CPU 中。在使用FPU时，可以先通过cpuid来检测是否支持FPU。如果发现存在 FPU，则应相应地设置控制寄存器。如果 FPU 不存在，也应该相应地设置寄存器。下面介绍是CR0寄存器和FPU相关的两个位：
* CR0.EM (bit 2) (EMulated) 如果设置了 EM 位，所有 FPU 和向量操作都将导致 #NM，因此它们可以在软件中模拟。清除 EM 位才能使用 FPU；
* CR0.TS (bit 3) 任务切换。FPU 状态被设计为延迟切换，以节省读写周期。如果设置，所有有意义的操作都会导致 #NM 异常，以便 OS 备份 FPU 状态。该位在硬件任务开关上自动设置，可以使用 CLTS 操作码清除。如果软件任务切换想要延迟存储 FPU 状态，可能需要手动设置这个位重新调度。


### 2.1、FPU浮点运算单元实现
在多个进程都需要使用FPU时，如果进程切换，那也应该支持保存和恢复FPU的寄存器状态。以下结构定义了FPU的状态：
```cpp
typedef struct fpu_t
{
    u16 control;
    u16 RESERVED;
    u16 status;
    u16 RESERVED;
    u16 tag;
    u16 RESERVED;
    u32 fip0;
    u32 fop0;
    u32 fdp0;
    u32 fdp1;
    u8 regs[80];
} _packed fpu_t;
```
在每个进程的结构中需要增加 fpu_t 元素，接下来是FPU的实现：
```cpp
task_t *last_fpu_task = NULL;  // 标记上次使用fpu的进程

void fpu_enable(task_t *task)     // 激活 FPU
{
    LOGK("fpu enable...\n");

    set_cr0(get_cr0() & ~(CR0_EM | CR0_TS));  // 设置CR0寄存器
    if (last_fpu_task == task)                // 如果使用的任务没有变化，则无需恢复浮点环境
        return;
    
    if (last_fpu_task && last_fpu_task->flags & TASK_FPU_ENABLED)   // 如果存在使用过浮点处理单元的进程，则保存浮点环境
    {
        assert(last_fpu_task->fpu);
        asm volatile("fnsave (%%eax) \n" ::"a"(last_fpu_task->fpu));
        last_fpu_task->flags &= ~TASK_FPU_ENABLED;
    }

    last_fpu_task = task;

    if (task->fpu)      // 如果 fpu 不为空，则恢复浮点环境
    {
        asm volatile("frstor (%%eax) \n" ::"a"(task->fpu));
    }
    else
    {
        asm volatile(       // 否则，初始化浮点环境
            "fnclex \n"
            "fninit \n");
        task->fpu = (fpu_t *)kmalloc(sizeof(fpu_t));
        task->flags |= (TASK_FPU_ENABLED | TASK_FPU_USED);
    }
}

void fpu_disable(task_t *task)   // 禁用 fpu， 设置CR0寄存器
{
    set_cr0(get_cr0() | (CR0_EM | CR0_TS));
}

void fpu_handler(int vector)   // fpu异常处理函数
{
    assert(vector == INTR_NM);
    task_t *task = running_task();
    assert(task->uid);
    fpu_enable(task);
}

void fpu_init()        // 初始化 FPU
{
    bool exist = fpu_check(); // 检查是否支持FPU   
    last_fpu_task = NULL;
    assert(exist);

    if (exist)
    {
        set_exception_handler(INTR_NM, fpu_handler);  // 设置异常处理函数，非常类似于中断
        set_cr0(get_cr0() | CR0_EM | CR0_TS | CR0_NE);   // 设置 CR0 寄存器
    }
    else
        LOGK("fpu not exists...\n");
}
```
以上为fpu的实现，这里省略了fpu检查的代码。保留了主要流程，在操作系统初始化是，进行FPU的初始化。设置INTR_NM异常处理函数，CPU 在执行需要使用 FPU/MMX/SSE/AVX 指令时，发现当前任务 FPU 不可用时，就触发和这个异常。以下为应用程序进行浮点运算的测试程序：
```cpp
void builtin_test(int argc, char *argv[])
{
    pid_t pid = fork();
    while (true)
    {
        double num = 1.0;
        num += 23.3;
        num -= 5.7;
        num *= 2.5;
        num /= 0.7;
        printf("test %d called\n", getpid());
        sleep(1000);
    }
}
```
* 应用程序执行，fork了一个子进程，此时子进程和父进程都会继续向下执行进入循环。
* 当其中一个进程获取CPU执行权，会进行浮点运算，此时该进程未进行FPU启用，会触发异常，进而去启用FPU。
* 当发生任务切换，保存原进程的FPU状态并禁用FPU， 现代 CPU 使用 fxsave / fxrstor 指令快速保存和恢复全部浮点上下文。
* 另一个进程触发异常，也进行FPU启用，并进行浮点运算。
* 再次进行FPU切换时，会继续触发异常，加载FPU，如此往复。

需注意的是：操作系统内核一般默认只允许用户态使用，内核自身很少动用 FPU。因为内核本身不需要进行浮点运算。

## 三、基础浮点运算
CPU提供了一系列的指令可以进行浮点运算，如 FABS 计算绝对值，FCOS计算三角函数cos，借助这些指令我们就可以通过内敛汇编来实现数学函数，如：
```cpp
double cos(double x)
{
    asm volatile(
        "fldl %0 \n"
        "fcos \n"
        "fstpl %0\n"
        : "+m"(x));
    return x;
}
```
如上，我们就实现了cos函数，当然FPU还支持更多的指令，我们这里不做一一介绍。
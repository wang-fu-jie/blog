---
title:       自制操作系统 - 系统调用
subtitle:    "系统调用"
description: "系统调用是用户进程与内核沟通的方式。可以将 CPU 从用户态转向内核态；系统调用通过中断门来实现。本文将实现系统调用的框架，并实现一个yield系统调用。yield的作用是进程主动交出执行权，去调度执行其他进程。"
excerpt:     "系统调用是用户进程与内核沟通的方式。可以将 CPU 从用户态转向内核态；系统调用通过中断门来实现。本文将实现系统调用的框架，并实现一个yield系统调用。yield的作用是进程主动交出执行权，去调度执行其他进程。"
date:        2024-02-02T09:36:49+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/81/09/bike_flower_field_meadow_landscape-16921.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-system-call"
categories:  [ "操作系统" ]
---

## 一、系统调用
系统调用是用户进程与内核沟通的方式。可以将 CPU 从用户态转向内核态；系统调用通过中断门来实现，为了与 linux 兼容，使用 0x80 号中断函数；linux 32位的系统调用使用的寄存器如下：
* eax 存储系统调用号
* ebx 存储第一个参数
* ecx 存储第二个参数
* edx 存储第三个参数
* esi 存储第四个参数
* edi 存储第五个参数
* ebp 存储第六个参数
* eax 存储返回值

一般情况下，使用前三个参数就足够了。系统调用的过程如下：
```s
mov eax, 系统调用号
mov ebx, 第一个参数
mov ecx, 第二个参数
mov edx, 第三个参数
int 0x80;
```

### 1.1、linux的系统调用
我们这里使用汇编语言测试下ubuntu上的系统调用，代码如下：
```s
[bits 32]
section .text
global main
main:

    ; write(stdout, message, len)
    mov eax, 4; write
    mov ebx, 1; stdout
    mov ecx, message; buffer
    mov edx, message.end - message
    int 0x80

    ; exit(0)
    mov eax, 1; exit
    mov ebx, 0; status
    int 0x80

section .data
message:
     db "hello world!!!", 10, 13, 0
.end:
```
如上所示，调用了write系统调用，ebx是标准输出，这个代码运行的结果是在屏幕上打印hello world!!!, 和C语言printf的效果一样。第二个是退出的系统调用


## 二、系统调用框架
系统调用时通过0x80中断实现的，因此首先需要添加0x80号中断的处理函数。实现如下：
```s
extern syscall_check
extern syscall_table
global syscall_handler
syscall_handler:
    xchg bx, bx
    push eax     ; 验证系统调用号
    call syscall_check
    add esp, 4

    push 0x20222202
    push 0x80
    push ds   ; 保存上文寄存器信息
    push es
    push fs
    push gs
    pusha

    push 0x80; 向中断处理函数传递参数中断向量 vector

    push edx; 第三个参数
    push ecx; 第二个参数
    push ebx; 第一个参数
    call [syscall_table + eax * 4]  ; 调用系统调用处理函数，syscall_table 中存储了系统调用处理函数的指针
    add esp, 12; 系统调用结束恢复栈
    mov dword [esp + 8 * 4], eax    ; 修改栈中 eax 寄存器，设置系统调用返回值
    jmp interrupt_exit   ; 跳转到中断返回
```
如上所示，在中断处理handle.asm文件中增加这个函数，处理系统调用的中断。系统调用的实现存放到syscall_table数组中。另外就是需要修改初始化中断描述符的代码，增加系统调用中断的初始化，中断描述符中的函数指向 syscall_handler。和其他中断不同的是，系统调用中断允许用户态访问，即IDT的 DPL属性为3。

接下来看下系统调用表的初始化，这里可以容纳64个系统调用。
```cpp
#define SYSCALL_SIZE 64

handler_t syscall_table[SYSCALL_SIZE];

void syscall_check(u32 nr)
{
    if (nr >= SYSCALL_SIZE)
    {
        panic("syscall nr error!!!");
    }
}

static void sys_default()
{
    panic("syscall not implemented!!!");
}

static u32 sys_test()
{
    LOGK("syscall test...\n");
    return 255;
}

void syscall_init()
{
    for (size_t i = 0; i < SYSCALL_SIZE; i++)
    {
        syscall_table[i] = sys_default;
    }

    syscall_table[0] = sys_test;
}
```
如上所示，初始化过程对syscall_table进行初始化，设置0号系统调用为测试系统调用，没有实现的系统调用就打印panic。在内核start.asm中就可以通过0x80中断来触发0号系统调用了。


## 三、yield系统调用
yield是一个简单的系统调用，它的作用是进程主动交出执行权，去调度执行其他进程。这里将yield作为1号系统调用。定义一个枚举来保存系统调用，如下：
```cpp
typedef enum syscall_t
{
    SYS_NR_TEST,
    SYS_NR_YIELD,
} syscall_t;

static _inline u32 _syscall0(u32 nr)
{
    u32 ret;
    asm volatile(
        "int $0x80\n"
        : "=a"(ret)
        : "a"(nr));
    return ret;
}

u32 test()
{
    return _syscall0(SYS_NR_TEST);
}
void yield()
{
    _syscall0(SYS_NR_YIELD);
}
```
如上，0号系统调用时TEST，1号系统调用时YIELD，接下来就需要去实现系统调用了，通过一个_syscall0函数定义无参系统调用，这样只给eax寄存传入系统调用好即可。

在进程增加一个task_yield函数，该函数只调用了一下进程调度函数schedle()。在系统调用初始化函数同时需要修改syscall_table。syscall_table的1号元素指向task_yield。

为了测试我们在线程的函数打印A，B, C三个任务后调用一次yield函数。yeild函数会触发1号系统调用，1号系统调用中断函数是task_yield， task_yield中进行了任务调度，这样每打印一次就会马上切换到另一个线程。三个字符就会循环打印，而不是原来的打印一堆A再切换到B。
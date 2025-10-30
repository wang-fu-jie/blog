---
title:       自制操作系统 - C 运行时环境
subtitle:    "C 运行时环境"
description: "前面执行的用户程序都是使用汇编语言编写的。我们写的汇编还主动调用了exit系统调用。但是正常C语言程序不会主动调用exit。因此从execve进入用户程序到退出还是一些细节需要补充。这就是C 运行时环境，它要求在main函数开始和结束后都需要做一些工作。另外用户程序参数应该存放在用户栈中，我们也在本节中进行用户参数和环境变量的拷贝。"
excerpt:     "前面执行的用户程序都是使用汇编语言编写的。我们写的汇编还主动调用了exit系统调用。但是正常C语言程序不会主动调用exit。因此从execve进入用户程序到退出还是一些细节需要补充。这就是C 运行时环境，它要求在main函数开始和结束后都需要做一些工作。另外用户程序参数应该存放在用户栈中，我们也在本节中进行用户参数和环境变量的拷贝。"
date:        2023-03-21T19:12:16+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/images/4e/74/f28aac7c77a73545a20851f09b75-1435821.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-c-runtime-env"
categories:  [ "操作系统" ]
---

## 一、C 运行时环境
前面执行的用户程序都是使用汇编语言编写的。我们写的汇编还主动调用了exit系统调用。但是正常C语言程序不会主动调用exit。因此从execve进入用户程序到退出还是一些细节需要补充。

ISO C 标准定义了 main 函数可以定义为没有参数的函数，或者带有两个参数 argc 和 argv 的函数，表示命令行参数的数量和字符指针：
```cpp
int main (int argc, char *argv[])
```
而对于 UNIX 系统，main 函数的定义有第三种方式：
```cpp
int main (int argc, char *argv[], char *envp[])
```
其中，第三个参数 envp 表示系统环境变量的字符指针数组，环境变量字符串是形如 NAME=value 这样以 = 连接的字符串，比如 SHELL=/bin/sh。

在执行 main 函数之前，libc C 函数库需要首先为程序做初始化，比如初始化堆内存，初始化标准输入输出文件，程序参数和环境变量等，在 main 函数结束之后，还需要做一些收尾的工作，比如执行 atexit 3 注册的函数，和调用 exit 系统调用，让程序退出。所以 ld 默认的入口地址在 _start，而不是 main。


### 1.1、C 运行时环境实现
在定义 C 运行时环境是，首先我们的操作系统需要有自己的标准库 libc。因此我们需要修改makefile，生成自己的标准库：
```make
$(BUILD)/lib/libc.o: $(BUILD)/lib/crt.o \    # 将内核实现的一些库封装为标准库
	$(BUILD)/lib/crt1.o \
	$(BUILD)/lib/string.o \
	$(BUILD)/lib/vsprintf.o \
	$(BUILD)/lib/stdlib.o \
	$(BUILD)/lib/syscall.o \
	$(BUILD)/lib/printf.o \
	$(BUILD)/lib/assert.o \
	ld -m elf_i386 -r $^ -o $@

$(BUILD)/builtin/%.out: $(BUILD)/builtin/%.o \   # 使用自己的标准库来编译链接用户程序
	$(BUILD)/lib/libc.o \
    ld -m elf_i386 -static $^ -o $@ -Ttext 0x1001000
```
这里有一点需要注意，之前的assert断言库的实现使用了内核的打印函数，用户程序是无法使用的。因此在lib中需要使用printf再次实现断言。重点是main函数执行前和执行后需要做的工作，这里在crt中实现：

```cpp
#include <onix/types.h>
#include <onix/syscall.h>
#include <onix/string.h>
int main(int argc, char **argv, char **envp);

weak void _init()   // libc 构造函数
{
}


weak void _fini()   // libc 析构函数
{
}

int __libc_start_main(
    int (*main)(int argc, char **argv, char **envp),
    int argc, char **argv,
    void (*_init)(),
    void (*_fini)(),
    void (*ldso)(), // 动态连接器
    void *stack_end)
{
    char **envp = argv + argc + 1;
    _init();
    int i = main(argc, argv, envp);
    _fini();
    exit(i);
}
```
如上所示，首先定义了构造函数和析构函数，这里没有定义函数体，只定义了框架，暂时用不到。__libc_start_main 为 libc 执行时的main函数，可以看到它调用了应用程序的main函数，并执行exit系统调用做了进程退出。程序开始的位置应该是_start。因此我们这里使用汇编定义了C 运行时启动文件的入口。如下：
```asm
[bits 32]

section .text
global _start

extern __libc_start_main
extern _init
extern _fini
extern main

_start:
    xor ebp, ebp; 清除栈底，表示程序开场
    pop esi; 栈顶参数为 argc
    mov ecx, esp; 其次为 argv

    and esp, -16; 栈对齐，SSE 需要 16 字节对齐
    push eax; 感觉没什么用
    push esp; 用户程序栈最大地址
    push edx; 动态链接器
    push _init; libc 构造函数
    push _fini; libc 析构函数
    push ecx; argv
    push esi; argc
    push main; 主函数

    call __libc_start_main

    ud2; 程序不可能走到这里，不然可能是其他什么地方有问题
```
_start 之所以是程序入口点，不是 C 标准规定的，而是 系统级 ABI（应用二进制接口, Application Binary Interface） 的约定。每个操作系统的可执行文件格式都有一个「入口点」（Entry Point）字段这个字段告诉内核：当加载可执行文件到内存后，从哪条指令开始执行，而这个地址，就是链接器根据 _start 符号来设置的。当然在链接时也可以人为改掉，入口不使用_start。

### 1.2、C 运行时环境测试
有了 C 运行时环境，用户程序就可以写C语言的代码了：
```cpp
#ifdef ONIX
#include <onix/types.h>
#include <onix/stdio.h>
#include <onix/syscall.h>
#include <onix/string.h>
#else
#include <stdio.h>
#include <string.h>
#endif

int main(int argc, char const *argv[], char const *envp[])
{
    for (size_t i = 0; i < argc; i++)
    {
        printf("%s\n", argv[i]);
    }

    for (size_t i = 0; 1; i++)
    {
        if (!envp[i])
            break;
        int len = strlen(envp[i]);
        if (len >= 1022)
            continue;
        printf("%s\n", envp[i]);
    }
    return 0;
}
```
如上用户程序，定义两种方式，如果没有定义ONIX。则使用开发机操作系统的标准库来执行，如果定义了，那说明视图我们自己的操作系统中的 libc 标准库链接的。这个定义是在 makefile 中写死的。

编译时，ld 会把 hello.o 和 libc.o 一起合并成一个 ELF 可执行文件。 hello.o 里定义了 main。libc.o（由 crt1.o 等组成）里定义了 _start，链接器会自动把 _start 当成入口点。


## 二、参数和环境变量
调用 execve 系统调用时，操作系统会把参数 argv 和环境变量 envp 拷贝到用户栈顶，然后跳转到程序入口地址。参数和环境变量的内存布局如下图所示：
<rawhtml>
<img src="/post_images/os/{{< filename >}}/2-01.png" alt="图片加载失败" width="300"/>
</rawhtml>
因此在 exexve 中就需要去处理环境变量，将参数和环境变量拷贝到用户栈顶。
```cpp
static int count_argv(char *argv[])       // 计算参数数量
{
    if (!argv)   // 如果没有参数，直接返回0
        return 0;
    int i = 0;
    while (argv[i])
        i++;
    return i;
}

static u32 copy_argv_envp(char *filename, char *argv[], char *envp[])
{
    int argc = count_argv(argv) + 1;    // 计算参数数量
    int envc = count_argv(envp);        // 计算环境变量数量
    u32 pages = alloc_kpage(4);         // 分配内核内存，用于临时存储参数
    u32 pages_end = pages + 4 * PAGE_SIZE;
    char *ktop = (char *)pages_end;       // 内核临时栈顶地址
    char *utop = (char *)USER_STACK_TOP;  // 用户栈顶地址
    char **argvk = (char **)alloc_kpage(1);   // 内核参数
    argvk[argc] = NULL;                       // 以 NULL 结尾
    char **envpk = argvk + argc + 1;          // 内核环境变量
    envpk[envc] = NULL;                       // 以 NULL 结尾

    int len = 0;
    for (int i = envc - 1; i >= 0; i--)     // 拷贝 envp
    {
        len = strlen(envp[i]) + 1;          // 计算长度
        ktop -= len;           // 得到拷贝地址
        utop -= len;
        memcpy(ktop, envp[i], len);   // 拷贝字符串到内核
        envpk[i] = utop;   // 数组中存储的是用户态栈地址
    }

    for (int i = argc - 1; i > 0; i--)   // 拷贝 argv
    {
        len = strlen(argv[i - 1]) + 1;    // 计算长度
        ktop -= len;           // 得到拷贝地址
        utop -= len;
        memcpy(ktop, argv[i - 1], len);  // 拷贝字符串到内核
        argvk[i] = utop;      // 数组中存储的是用户态栈地址
    }

    len = strlen(filename) + 1;    // 拷贝 argv[0]，程序路径
    ktop -= len;
    utop -= len;
    memcpy(ktop, filename, len);
    argvk[0] = utop;
    ktop -= (envc + 1) * 4;         // 将 envp 数组拷贝内核
    memcpy(ktop, envpk, (envc + 1) * 4);
    ktop -= (argc + 1) * 4;          // 将 argv 数组拷贝内核
    memcpy(ktop, argvk, (argc + 1) * 4);
    ktop -= 4;                  // 为 argc 赋值
    *(int *)ktop = argc;
    assert((u32)ktop > pages);
    len = (pages_end - (u32)ktop);          // 将参数和环境变量拷贝到用户栈
    utop = (char *)(USER_STACK_TOP - len);
    memcpy(utop, ktop, len);

    free_kpage((u32)argvk, 1);        // 释放内核内存
    free_kpage(pages, 4);
    return (u32)utop;
}
```
然后再 execve 系统调用的实现中，进行环境变量的处理，这样就实现了把参数和用户变量拷贝到了用户栈顶。然后在shell定义几个环境变量进行测试：
```cpp
static char *envp[] = {
    "HOME=/",
    "PATH=/bin",
    NULL,
};
```

## 三、内建命令优化
在shell中我们实现的命令如 echo、ls、 cat 等都是直接写死了再osh.c文件中，有了参数之后，这个命令的实现就可以单独存放一个文件，不再需要全部写到shell中了。如echo的实现：
```cpp
#ifdef ONIX
#include <onix/types.h>
#include <onix/stdio.h>
#include <onix/syscall.h>
#include <onix/string.h>
#else
#include <stdio.h>
#include <string.h>
#endif

int main(int argc, char const *argv[])
{
    for (size_t i = 1; i < argc; i++)
    {
        printf(argv[i]);
        if (i < argc - 1)
        {
            printf(" ");
        }
    }
    printf("\n");
    return 0;
}
```
如上echo单独实例，然后编译为echo.out文件，存放到bin目录下，这样shell的 execute函数就会走如下分支：
```cpp
static void execute(int argc, char *argv[])
{
    char *line = argv[0];
    
    stat_t statbuf;
    sprintf(buf, "/bin/%s.out", argv[0]);
    if (stat(buf, &statbuf) == EOF)
    {
        printf("osh: command not found: %s\n", argv[0]);
        return;
    }
    return builtin_exec(buf, argc - 1, &argv[1]);
}
```
首先会判断exec.out存在，然后去只执行这个文件。就是一个完整的执行用户程序的流程。

---
title:       自制操作系统 - gcc编译过程与汇编分析
subtitle:    "gcc编译过程与汇编分析"
description: "C代码编译为可执行程序需要经过预处理、编译、汇编、链接等过程，本文将介绍C语言编译成可执行程序的步骤细节。并介绍C语言中局部变量是如何存储的，以及是如何给函数进行传参。"
excerpt:     "C代码编译为可执行程序需要经过预处理、编译、汇编、链接等过程，本文将介绍C语言编译成可执行程序的步骤细节。并介绍C语言中局部变量是如何存储的，以及是如何给函数进行传参。"
date:        2024-01-15T17:12:32+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/20/5b/water_wafe_ocean_wave_wet-136845.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-gcc-compile"
categories:  [ "操作系统" ]
---

## 一、编写内核
在说编译之前，我们首先给操作系统的内核使用C语言写点代码，实现打印一串字符的功能。代码如下:
```cpp
// onix.h
#ifndef ONIX_H
#define ONIX_H
#define ONIX_MAGIC 20220205
void kernel_init(); //初始化内核
#endif

// main.c
#include <onix/onix.h>

int magic = ONIX_MAGIC;
char message[] = "hello onix!!!"; // .data
char buf[1024];                   // .bss

void kernel_init()
{
    char *video = (char *) 0xb8000; // 文本显示器的内存位置
    for (int i = 0; i < sizeof(message); i++)
    {
        video[i * 2] = message[i];
    }
}
```

## 二、编译过程
C语言编译为可执行文件需要5个步骤：
* 源代码(.h .c .cpp)    ——> 预处理生成源文件
* 源文件(.c)            ——> 编译生成汇编
* 汇编文件(.s .asm)     ——> 汇编生成目标文件
* 目标文件(.o .so)      ——> 链接生成可执行文件

### 2.1、预处理
预处理器是编译的第一步，它负责对源代码进行文本级别的处理，为后续的编译阶段做准备。预处理的做了以下几件事： 
* 头文件包含（#include）将指定头文件的内容插入到当前文件中。 
* 宏替换（#define）将代码中的宏标识符替换为定义的值或代码片段。
* 条件编译（#ifdef, #if, #else, #endif等）根据条件决定是否编译某段代码。
* 删除注释。将所有注释（//单行注释、/* */多行注释）替换为空格
* 处理特殊指令#pragma：编译器特定的功能（如优化、警告控制） 
* 处理续行符（\） 将反斜杠（\）后紧跟换行的代码行合并为一行

可以执行以下命令进行预处理，生成c文件
```
gcc -m32 -E main.c -I../include > test.c
```

### 2.1、编译
将源文件进行编译就会得到汇编文件。命令：
```
gcc -m32 -S test.c > test.s
```
生成的汇编文件比较复杂，这里不进行粘贴，通过汇编文件可以看到汇编中有magic，message, buf等变量和函数的定义。

### 2.3、汇编
汇编器对汇编文件进行汇编就会得到目标文件，命令如下：
```
as -32 test.s -o test.o
```
得到目标文件后，可以通过readelf来查看目标文件。汇编将汇编指令翻译为机器码生成目标文件的结构。处理前边提过的代码段和数据段之外，还包含符号表记录所有符号（函数名、全局变量名等）的信息。重定位表（Relocation Table）记录需要重定位的指令或数据地址。例如：当汇编代码中调用外部函数（如 call printf）时，printf 的地址在汇编阶段是未知的，汇编器会在重定位表中记录这条指令的位置，链接阶段填充正确地址。

### 2.4、链接
链接阶段是将多个目标文件（.o 或 .obj）和库文件（.a 静态库或 .so/.dll 动态库）合并，生成最终可执行文件或共享库。它进行符号解析、地址与内存分配、重定位等工作。
```
ld -m elf_i386 -static test.o -o test.out -e kernel_init
```
gcc集成了整个编译过程，可以一条命令完成编译
```
gcc --verbose -m32 main.c -I../include -o main.out -e kernel_init -nostartfiles
```

## 三、编译内核
到此就完成整个编译过程，当然这里生成的程序是不能执行运行的，因为在保护模式中，访问了只有操作系统才能访问的内存，会报段错误。因此我们需要将目标文件链接到操作系统。修改makefile。将生成的main.o 链接到 system.bin。最后还需要在内核中调用kernel_init函数
```asm
[bits 32]
extern kernel_init
global _start
_start:
    call kernel_init
    jmp $; 阻塞
```
需要说明的是，编译内核和编译应用程序有很大区别，例如在编译内核需要加上许多参数，例如不能引入C标准库和内置函数。这里可以参考makefile文件。

## 四、GCC汇编分析
我们拿helloworld的程序来编译成汇编文件。可以看到内容还是挺多的，其中包含：
* CFI: Call Frame Information / 调用栈帧信息 是一种 DWARF 的信息，用于调试，获得调用异常；使用 -fno-asynchronous-unwind-tables 可以禁用CFI
* PIC: Position Independent Code / 位置无关的代码， 使用 -fno-pic 参数禁用
* ident： GCC的版本信息， 使用-Qn参数去掉版本信息
* 栈对齐：将栈对齐到 16 字节，字节对齐的话，访问内存更加高效，使用更少的时钟周期；-mpreferred-stack-boundary=2参数可以禁用栈对齐

### 4.1、栈帧
在汇编中，可以看到每个函数都遵循这样的格式：
```asm
pushl %ebp
movl %esp, %ebp
...
leave
```
leave相当于前边两条指令的反操作，这个格式叫做栈帧。可以通过参数 -fomit-frame-pointer 去掉栈帧信息。最终得到的代码如下：
```asm
	.file	"hello.c" # 文件名

.text # 代码段
	.globl	message # 将 message 导出

.data # 数据段
	.align 4 # 对齐到四个字节

	.type	message, @object
	.size	message, 16
message:
	.string	"hello world!!!\n"

.globl	buf
	.bss # bss 段
	.align 32
	.type	buf, @object
	.size	buf, 1024
buf:
	.zero	1024

.text
	.globl	main
	.type	main, @function
main:

	pushl	$message # message 的地址压入栈中
	call	printf # 调用 printf 
	addl	$4, %esp # 恢复栈
	movl	$0, %eax # 函数返回值存储在 eax 寄存器中；
	ret # 函数调用返回
	.size	main, .-main
	.section	.note.GNU-stack,"",@progbits # 标记栈不可运行
```


### 4.2、堆栈与函数
栈是一个重要的数据结构，特点是后进先出。栈顶指针是在 ss:esp 寄存器中，栈底是在高地址，向下增长。push 入栈、pop 出栈、pusha 将 8 个基础寄存器压入栈中、popa 将 7 个基础寄存器弹出，与 pusha 相对，会忽略 esp。

函数调用使用 call 指令，call指令会将 call 返回的下一条指令地址压入栈中。ret指令将栈顶弹出到 eip。  这两个指令经常成对出现，但是其实这两个指令无关，可以只调用其中一个。

### 4.3、局部变量与参数传递
局部变量和参数传递都是通过栈来实现的。如下一个C代码示例：
```cpp
int add(int x, int y)
{
    int z = x + y;
    return z;
}

int main()
{
    int a = 5;
    int b = 3;
    int c = add(a, b);
    return 0;
}
```
将这串C语言的代码编译成汇编语言，查看汇编是怎么实现的
```asm
	.file	"params.c"
	.text
	.globl	add
	.type	add, @function
add:
	pushl	%ebp
	movl	%esp, %ebp

	subl	$4, %esp # 一个局部变量
	movl	8(%ebp), %edx # a
	movl	12(%ebp), %eax # b
	addl	%edx, %eax # eax += edx
	movl	%eax, -4(%ebp) # z = x + y;
	movl	-4(%ebp), %eax # eax = z;

	leave
	ret
	.size	add, .-add
	.globl	main
	.type	main, @function
main:
	pushl	%ebp
	movl	%esp, %ebp # 保存栈帧

	subl	$12, %esp # 保存 12 个字节，有三个局部变量
	movl	$5, -12(%ebp) # a
	movl	$3, -8(%ebp) # b

	pushl	-8(%ebp) # b
	pushl	-12(%ebp) # a
	call	add # 调用
	addl	$8, %esp # 恢复栈

	movl	%eax, -4(%ebp) # c = add(a, b);
	movl	$0, %eax # 返回值存储在 eax 寄存器中

	leave # 恢复栈帧
	ret # 函数返回
	.size	main, .-main
	.section	.note.GNU-stack,"",@progbits
```
这里先说下栈帧的作用是为了保存函数局部变量的信息，可以用于回溯调用函数；我们在调试C时查看函数调用栈就是栈帧实现的，如果删除了栈帧代码一样能运行，但是就获取不到函数的调用栈了。

在看下局部变量。函数首先会在栈顶分配空间存储局部变量，在进行参数传递时，参数按照入参顺序从右至左压入栈。我们在学习汇编时，使用寄存器给函数传参。C是使用栈对函数传参，这里因为在C中，存在例如printf这样的函数，参数可以非常多，但是寄存器只有6个这是远远不够的。
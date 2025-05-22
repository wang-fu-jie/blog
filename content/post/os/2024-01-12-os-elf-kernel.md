---
title:       自制操作系统 - ELF文件介绍&进入内核
subtitle:    "ELF文件介绍&进入内核"
description: "ELF文件时类Unix操作系统中的文件格式，包括可重定位文件、可执行文件、共享对象文件。内核代码编译后就是ELF文件，我们需要提取内核ELF文件的可加载段写入内存，然后让内核加载器跳转到这里执行。"
excerpt:     "ELF文件时类Unix操作系统中的文件格式，包括可重定位文件、可执行文件、共享对象文件。内核代码编译后就是ELF文件，我们需要提取内核ELF文件的可加载段写入内存，然后让内核加载器跳转到这里执行"
date:        2024-01-12T19:06:25+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/22/46/animal_deer_wildlife_tree_light-8999.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-elf-kernel"
categories:  [ "操作系统" ]
---

## 一、ELF文件简介
ELF（Executable and Linkable Format）是一种在类Unix操作系统（如Linux、FreeBSD等）中广泛使用的文件格式，主要用于可执行文件、目标代码、共享库和核心转储文件。ELF文件的有三种类型：
* 可重定位文件（Relocatable File, .o）由编译器生成，尚未链接。包含代码、数据及符号表，需通过链接器与其他目标文件合并生成可执行文件或共享库。
* 可执行文件（Executable File）可直接运行的程序（如/bin/ls）。包含代码段、数据段及程序入口点。
* 共享对象文件（Shared Object File, .so）动态链接库，在运行时由动态链接器（如ld-linux.so）加载到进程内存中。

我们重点关注可执行文件。

### 1.1、ELF文件结构
ELF文件由以下关键部分组成：

ELF头部（ELF Header）位于文件开头，包含文件的元数据：魔数（Magic Number）：0x7F 'E' 'L' 'F'，标识为ELF文件。文件类型（32位/64位）、字节序（大端/小端）。
目标架构（如x86、ARM）。入口点地址（Entry Point）：程序执行的起始地址。程序头表（Program Header Table）和节头表（Section Header Table）的位置及大小。

程序头表（Program Header Table）描述段（Segments），用于指导操作系统如何将文件加载到内存以执行。每个条目定义了一个段（如代码段、数据段）的属性：类型（如可加载PT_LOAD、动态链接信息PT_DYNAMIC）。在文件中的偏移量、内存中的虚拟地址、大小、权限（读/写/执行）等。

节头表（Section Header Table）描述节（Sections），用于链接和调试。每个条目定义了一个节（如.text、.data、.rodata）的属性：节名、类型（如代码SHT_PROGBITS、符号表SHT_SYMTAB）。地址、偏移量、大小等。

### 1.2、可执行文件
ELF文件有两种视图，链接视图（Linker View）以节（Sections）为单位，关注代码和数据的组织，用于编译和静态链接。执行视图（Execution View）以段（Segments）为单位，指导操作系统如何将文件映射到内存。段包含代码段（LOAD）：可读、可执行（包含.text和.rodata）。数据段（LOAD）：可读、可写（包含.data和.bss）.data 是已经初始化过的数据。 .bss是未初始化的数据。
```cpp
#include <stdio.h>

int main()
{
    printf("hello world!!!\n");
    return 0;
}
```
将这个代码编译成为 32 位的程序并通过 readelf 读取：
```shell
gcc -m32 -static hello.c -o hello
readelf -e hello

ELF 头：
  Magic：   7f 45 4c 46 01 01 01 03 00 00 00 00 00 00 00 00 
  类别:                              ELF32
  数据:                              2 补码，小端序 (little endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - GNU
  ABI 版本:                          0
  类型:                              EXEC (可执行文件)
  系统架构:                          Intel 80386
  版本:                              0x1
  入口点地址：               0x8049700
  程序头起点：          52 (bytes into file)
   ......

节头：
  ......
  [ 6] .text             PROGBITS        08049090 001090 0654fa 00  AX  0   0 16
  [ 8] .rodata           PROGBITS        080af000 067000 01bc5c 00   A  0   0 32
  [18] .data             PROGBITS        080e3040 09a040 000edc 00  WA  0   0 32
  [19] .bss              NOBITS          080e3f20 09af1c 003148 00  WA  0   0 32
  [20] .comment          PROGBITS        00000000 09af1c 00002b 01  MS  0   0  1
  ......
```
如上所示，我们使用readelf读取可执行文件的结果。这里只贴了少部分内存，可执行文件至少需要.text .data .bss就可以执行了。

## 二、进入内核
前边我们已经写好了boot.asm和loader.asm。 这俩个文件时的主要功能就是从 BIOS 加载内核，并且提供内核需要的参数。现在我们就开始进入内核了。我们先在内核代码中写一个简单的程序
```asm
## start.asm
[bits 32]

global _start
_start:
    mov byte [0xb8000], 'K'; 表示进入了内核
    jmp $; 阻塞
```
然后我们需要编译内核并将内核写入磁盘。我们这里写makefile文件，使用make来进行编译：
```makefile
BUILD:=../build
SRC:=.

ENTRYPOINT:=0x10000

$(BUILD)/kernel/%.o: $(SRC)/kernel/%.asm
	$(shell mkdir -p $(dir $@))
	nasm -f elf32 $< -o $@

$(BUILD)/kernel.bin: $(BUILD)/kernel/start.o
	$(shell mkdir -p $(dir $@))
	ld -m elf_i386 -static $^ -o $@ -Ttext $(ENTRYPOINT)

$(BUILD)/system.bin: $(BUILD)/kernel.bin
	objcopy -O binary $< $@

$(BUILD)/system.map: $(BUILD)/kernel.bin
	nm $< | sort > $@

$(BUILD)/master.img: $(BUILD)/boot/boot.bin \
	$(BUILD)/boot/loader.bin \
	$(BUILD)/system.bin \
	$(BUILD)/system.map \

	yes | bximage -q -hd=16 -func=create -sectsize=512 -imgmode=flat $@
	dd if=$(BUILD)/boot/boot.bin of=$@ bs=512 count=1 conv=notrunc
	dd if=$(BUILD)/boot/loader.bin of=$@ bs=512 count=4 seek=2 conv=notrunc
	dd if=$(BUILD)/system.bin of=$@ bs=512 count=200 seek=10 conv=notrunc
```
如上所示，我们将内核入口加载到0x10000的位置。并将内核写入到磁盘第10个扇区开始往后的200个扇区内，这么大空间是足够容纳内核代码了。这里使用objcopy仅提取编译生成ELF文件的ELF中可加载的段（代码和数据）。

接下来就可以修改内核加载器从磁盘读取内核了。
```asm
mov edi, 0x10000; 读取的目标内存
mov ecx, 10; 起始扇区
mov bl, 200; 扇区数量
call read_disk
jmp dword code_selector:0x10000  ; 跳转到内核代码的位置，开始执行内核代码
```
注意：我们将内核读取到0x10000的位置，是因为这里属于可用区域，并且取了个整数位置。
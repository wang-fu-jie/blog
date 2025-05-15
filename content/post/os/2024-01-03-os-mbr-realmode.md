---
title:       自制操作系统 - 主引导扇区与实模式
subtitle:    "主引导扇区与实模式"
description: "计算机开机后，会先执行BIOS，BIOS将主引导扇区的代码加载到内存0x7c00并跳转到这里执行。主引导扇区是磁盘的第一个扇区，有512字节。这时候CPU是运行在实模式下的，实模式是16位模式，可访问1MB内存。"
excerpt:     "计算机开机后，会先执行BIOS，BIOS将主引导扇区的代码加载到内存0x7c00并跳转到这里执行。主引导扇区是磁盘的第一个扇区，有512字节。这时候CPU是运行在实模式下的，实模式是16位模式，可访问1MB内存。"
date:        2024-01-03T09:48:15+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/63/8c/background_pretty_blue_bright_cloud_cloudy_color_colorful-1179137.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-mbr-realmode"
categories:  [ "操作系统" ]
---

## 一、计算机的开机过程
计算机在按下电源后，电源供应器（PSU）向主板和其他硬件提供稳定电压，主板芯片组向CPU发送复位信号，CPU从固定地址（如x86的0xFFFF0）开始执行指令，指向BIOS/UEFI固件。通过bochs可以观测，开机后cs和ip寄存器的值如下：
```asm
cs = f0000
ip = fff0
```
所以第一条执行的指令位置是 0xFFFF0。 这里对应的第一条指令是 JMP 0xF000:0xE05B，这里正是BIOS的主代码区域。

### 1.1、BIOS
BIOS（Basic Input/Output System，基本输入输出系统） 负责硬件的初始化和启动操作系统。BIOS提供了一系列的功能可以供系统使用，例如int 0x10就是BIOS提供的功能。BIOS运行后首先会进行硬件的初始化与自检。并读取主引导扇区MBR到内存的0x7c00位置，然后跳转到0x7C00位置执行代码。

## 二、主引导扇区
主引导扇区MBR是存储在 磁盘第一个扇区（LBA 0） 的 512 字节 数据结构。包含以下部分：
* 引导代码（Bootstrap Code，446 字节）：由 BIOS 执行，负责加载操作系统。
* 分区表（Partition Table，64 字节）：记录磁盘的分区信息（最多 4 个主分区）。
* 魔数（Magic Number，2 字节 0x55AA）：用于验证 MBR 是否有效。

引导代码的通常是 操作系统或 Bootloader的初始加载程序，即读取并执行内核加载器。

这里说明下，为什么不在主引导扇区直接写内核加载器。因为主引导扇区的代码部分只有446个字节，不足以容纳内核加载器的代码。

### 2.1、0x7C00
BIOS会将引导扇区读取到内存0x7C00的位置，并跳转到这个位置执行引导代码。这个位置是因为历史原因、硬件限制和软件兼容性 共同决定的。早期的 IBM PC 使用 Intel 8088 CPU（16位，实模式），最大寻址 1MB（0x00000–0xFFFFF）。0x07C00–0x07DFF	被规划为 MBR 加载位置（512B）。 在后来8086中，为了向下兼容就一直沿用了这个地址。


## 三、实模式
在8086CPU中，CPU 使用 16 位寄存器 和 20 位地址总线，直接访问 1MB 内存空间。这时候CPU的运行模式被称为实模式，程序可以 直接操作硬件（如端口 I/O、中断向量表、BIOS 调用）。没有内存保护机制，任何程序均可修改全部内存。实模式的寻址方式是： 物理地址 = (段基址 << 4) + 偏移量，CS:IP = 0xF000:0xFFF0 → 物理地址 0xFFFF0。

和实模式对应的是保护模式，现代计算机启动后也是先运行在实模式状态下，只能访问1M内存，启动后会由实模式切换为保护模式。这里我们后续再详细介绍保护模式。

### 3.1、实模式print
在实模式中，需要依赖BIOS提供的 0x10 中断进行字符的打印。0x10中断通过设置 AH 寄存器 选择子功能，AH=0x0E时子功能是以电传打字（TTY）模式显示字符。我们实现代码如下：
```asm
xchg bx, bx; bochs ; 魔术断点, bochs的功能，会自动在这行代码打断点

mov si, booting
call print

; 阻塞
jmp $

print:
    mov ah, 0x0e
.next:
    mov al, [si]
    cmp al, 0
    jz .done
    int 0x10
    inc si
    jmp .next
.done:
    ret

booting:
    db "Booting Onix...", 10, 13, 0; \n\r  10和13分别是\n\r的ASCII码值
```
如上所示，我们就在实模式下通过调用BIOS的中断实现了打印字符串的功能。


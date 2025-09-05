---
title:       自制操作系统 - 内核加载器与内存检测
subtitle:    "内核加载器与内存检测"
description: "主引导扇区用来读取并执行内核加载器，内核加载器用来加载内核，BIOS提供了内存检测的功能，本文我们将在内核加载器实现内存检测的代码，并在主引导扇区中加载并执行。"
excerpt:     "主引导扇区用来读取并执行内核加载器，内核加载器用来加载内核，BIOS提供了内存检测的功能，本文我们将在内核加载器实现内存检测的代码，并在主引导扇区中加载并执行。"
date:        2024-01-07T10:42:02+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/8b/d9/sun_sunset_abendstimmung_sea_water_clouds-1349389.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-loader-memory-test"
categories:  [ "操作系统" ]
---

## 一、内核加载器
前面我们说了主引导扇区的作用读取并执行内核加载器。内核加载器的作用很明显了，就是加载内核。我们现在还没有内核，因此先在内核加载器中输出一串字符，来实现内核加载器以及在引导扇区中读取并执行。先看内核加载器的代码：
```asm
[org 0x1000]

dw 0x55aa; 魔数，用于判断错误
mov si, loading ; 打印字符串
call print
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

loading:
    db "Loading Onix...", 10, 13, 0; \n\r
```
如上所示为内核加载器的代码，仅仅实现了一个显示字符串的功能。内核加载器中顶一个魔术，用于判断内核加载器是否被正确的读取了内存。接下来我们需要将内核加载器写入到磁盘并在主引导扇区读取内核加载器并执行。
```shell
dd if=loader.bin of=master.img bs=512 count=4 seek=2 conv=notrunc
```
如上为将内核加载器写入硬盘的指令。写入到硬盘的第二个扇区。
```asm
mov edi, 0x1000; 读取的目标内存
mov ecx, 2; 起始扇区
mov bl, 4; 扇区数量

call read_disk

cmp word [0x1000], 0x55aa
jnz error
jmp 0:0x1002
```
如上为主引导扇区执行内核加载器的代码。可以看到我们将内核加载器读入到内存为 0x1000的内置，这是因为在实模式下，这个位置属于可用区域。这里我们读取的扇区是4个，在一般情况下，2kb的内存足以容纳内核加载器的代码。

### 1.1、实模式内存布局
我们将内核加载器加载到内存 0x1000 的位置是因为这里属于可用区域，接下来我们看下实模式的内存布局。
![图片加载失败](/post_images/os/{{< filename >}}/1.1-01.png)
可以看到，有两块可用内存区域，之所这样，是因为cpu要向下兼容。0x500~0x7BFF属于第一款内存可用区，我们将内核加载器存放到这个位置，取0x1000是因为整数操作更加方便。


## 二、内存检测
内存检测是内存加载器一个核心功能。因为操作系统并不知道将来会运行在什么样的机器上，为了兼容不同型号的机器，因此需要检测所运行机器的内存情况。BIOS的0x15中断0xe820子功能就提供内存检测的功能。

### 2.1、内存检测功能
BIOS提供了内存检测功能，即0x15中断的0xe820子功能。它返回Address Range Descriptor Structure (ARDS)结构体。ARDS是一个20位的数据结构。
![图片加载失败](/post_images/os/{{< filename >}}/2.1-01.png)
如上所示，为ARDS的数据结构信息，因为我们是32位的操作系统，这里忽略高32位。TYPE为内存类型，有以下几种取值：
![图片加载失败](/post_images/os/{{< filename >}}/2.1-02.png)
接下来看一下内存检测功能的参数和返回。
![图片加载失败](/post_images/os/{{< filename >}}/2.1-03.png)
如上所示，第一次检测需要给EBX传入0，在ES:DI指向的内存中写入ARDS信息，返回时在EBX中存入下一个ARDS的位置，直到EBX为0，检测完成。

### 2.2、内存检测实现
我们在内核加载器中实现一下内核检测的代码，如下：
```asm
detect_memory:
    xor ebx, ebx    ; 将 ebx 置为 0
    mov ax, 0       ; es:di 结构体的缓存位置
    mov es, ax
    mov edi, ards_buffer
    mov edx, 0x534d4150  ; 固定签名

.next:
    mov eax, 0xe820        ; 子功能号
    mov ecx, 20            ; ards 结构的大小 (字节)
    int 0x15               ; 调用 0x15 系统调用
    jc error               ; 如果 CF 置位，表示出错
    add di, cx             ; 将缓存指针指向下一个结构体
    inc word [ards_count]  ; 将结构体数量加一
    cmp ebx, 0             ; 如果ebx不为0，则检测下一块内存
    jnz .next
    mov si, detecting      ; 内存检测成功，输出字符串
    call print

    ; 以下代码为了能更好查看内存检测是结果，实际中是不需要的，测试完可以进行删除 
    mov cx, [ards_count]   ; 结构体数量
    mov si, 0              ; 结构体指针
.show:
    mov eax, [ards_buffer + si]
    mov ebx, [ards_buffer + si + 8]
    mov edx, [ards_buffer + si + 16]
    add si, 20
    xchg bx, bx
    loop .show

ards_count:
    dw 0
ards_buffer:   ; 这里必须定义在内核加载器的末尾，因为不清楚ards会占用多少内存。
```
如下图，为内核加载器的运行结果。
![图片加载失败](/post_images/os/{{< filename >}}/2.2-01.png)
内存一共被分为了6部分。这里只截取了第一块内存，起始位置是 0x0000000，内存长度为0x0009f000，类型是1，可以被操作系统使用。
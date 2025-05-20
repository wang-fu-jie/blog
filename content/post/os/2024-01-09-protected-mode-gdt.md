---
title:       自制操作系统 - 保护模式与全局描述符
subtitle:    "保护模式与全局描述符"
description: "系统刚启动时是位于实模式，即8086模式，只可以访问1MB内存。实模式下任何应用程序都可以访问这1M内存，包括病毒程序。因此在80286中就引入了保护模式。保护模式中引入了全局描述符来保护内存，并将内存按照4k的大小进行分页。"
excerpt:     "系统刚启动时是位于实模式，即8086模式，只可以访问1MB内存。实模式下任何应用程序都可以访问这1M内存，包括病毒程序。因此在80286中就引入了保护模式。保护模式中引入了全局描述符来保护内存，并将内存按照4k的大小进行分页。"
date:        2024-01-09T20:00:31+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/d8/4b/walking_forest_couple_nature_green_tree_vacation_leaves-825751.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "protected-mode-gdt"
categories:  [ "操作系统" ]
---

## 一、保护模式
系统刚启动时是位于实模式，即8086模式，只可以访问1MB内存。实模式下任何应用程序都可以访问这1M内存，包括病毒程序。因此在80286中就引入了保护模式，80286的保护模式也是16位的。我们重点关注32位的保护模式。

### 1.1、32位保护模式
保护模式主要是为了保护信息。对以下资源进行了保护：
* 寄存器：    设置部分寄存器只可以被操作系统访问
* 高速缓存：  对应用程序不透明
* 内存：     使用描述符的形式保护内存，设置内存访问特权级
* 外部设备：  只允许操作系统访问 in/out 指令

## 二、全局描述符
全局描述符是保护模式中保护内存的一种方式。全局描述符描述了内存的起始位置、内存的长度、和内存的属性。如下图为80386的全局描述符。
![图片加载失败](/post_images/os/{{< filename >}}/2-01.png)
如图：80386的全局描述符共占用32个字节，这么设计是为了兼容80286。80386描述符的结构体如下：
```cpp
typedef struct descriptor /* 共 8 个字节 */
{
    unsigned short limit_low;      // 段界限 0 ~ 15 位
    unsigned int base_low : 24;    // 基地址 0 ~ 23 位 16M
    unsigned char type : 4;        // 段类型
    unsigned char segment : 1;     // 1 表示代码段或数据段，0 表示系统段
    unsigned char DPL : 2;         // Descriptor Privilege Level 描述符特权等级 0 ~ 3
    unsigned char present : 1;     // 存在位，1 在内存中，0 在磁盘上
    unsigned char limit_high : 4;  // 段界限 16 ~ 19;
    unsigned char available : 1;   // 该安排的都安排了，送给操作系统吧
    unsigned char long_mode : 1;   // 64 位扩展标志
    unsigned char big : 1;         // 32 位 还是 16 位;
    unsigned char granularity : 1; // 粒度 4KB 或 1B
    unsigned char base_high;       // 基地址 24 ~ 31 位
} __attribute__((packed)) descriptor;
```

### 2.1、段类型
段类型由type 和  segment控制，segment占1位，当为0时表示系统段，当位1时表示代码段或数据库。type占4位，当segment为1时，type的4位分别如下：

| X | C/E | R/W | A |
* A: Accessed 是否被 CPU 访问过
* X: 1/代码 0/数据
* X = 1：代码段
  * C: 是否是依从代码段
  * R: 是否可读
* X = 0: 数据段
  * E: 0 向上扩展 / 1 向下扩展
  * W: 是否可写
这里解释下依从代码段，如果是依从代码段，应用程序可以跳转到这里执行代码。依从代码段执行不需要修改特权级。


### 2.2、全局描述符表
全局描述符表GDT，是一个顺序表，即数组。GDT最多可以有8192个，第0个必须全为0，即NULL描述符。因此可用的描述符最多有8191个。全局描述符也需要存储在内存中，所以需要让CPU知道全局描述符表在内存的哪个位置。因此需要一个描述符指针来执行全局描述符表的基地址和界限，描述符指针结构如下：
```cpp
typedef struct pointer
{
    unsigned short limit; // size - 1
    unsigned int base;
} __attribute__((packed)) pointer;
```
CPU提供了一个寄存器 gdtr 用于获取全局描述符表的起始位置和长度。提供了两个指令支持操作GDT,
```asm
lgdt [gdt_ptr]   ; 加载gdt
sgdt [gdt_ptr]   ; 保存gdt
```
GDT可能会有很多块，通过这两个指令可以加载或保存不同的GDT。

### 2.3、段选择子
代码运行时候需要一个代码段，但是可能需要一个或多个数据段/栈段。因此提供段选择子用于定位GDT中的描述符，段选择子结构如下：
```cpp
typedef struct selector
{
    unsigned char RPL : 2; // Request PL 
    unsigned char TI : 1; // 0  全局描述符 1 局部描述符 LDT Local 
    unsigned short index : 13; // 全局描述符表索引
} __attribute__((packed)) selector;
```
段选择子会被加载到16位的段寄存器中，我们看到索引只有13位，这也就是为什么全局描述符最多支持8192个。


### 2.4、A20线
8086中只有20根地址总线，因此只能访问1M内存。但是80386中访问大于1M的内存，就需要打开A20线，通过 0x92 端口就可以打开A20线。最后还需要将cr0 寄存器 0 位 置为 1启用保护模式。


## 三、保护模式代码实现
本章节从定时全局描述符开始，一步步用代码实现如何进入保护模式。

### 3.1、定义全局描述符
我们只需要定义两个全局描述符即可，一个代码段，一个数据段。
```asm
memory_base equ 0      ; 内存开始的位置：基地址
memory_limit equ ((1024 * 1024 * 1024 * 4) / (1024 * 4)) - 1   ; 内存界限 4G / 4K - 1, 一页4k

gdt_ptr:                 ; 描述符指针
    dw (gdt_end - gdt_base) - 1  ; 描述符的大小
    dd gdt_base          ; 描述符的基地址
gdt_base:
    dd 0, 0            ; NULL 描述符
gdt_code:
    dw memory_limit & 0xffff           ; 段界限 0 ~ 15 位
    dw memory_base & 0xffff            ; 基地址 0 ~ 16 位
    db (memory_base >> 16) & 0xff      ; 基地址 0 ~ 16 位
    db 0b_1_00_1_1_0_1_0               ; 存在 - dlp 0 - S _ 代码 - 非依从 - 可读 - 没有被访问过
    db 0b1_1_0_0_0000 | (memory_limit >> 16) & 0xf  ; 4k - 32 位 - 不是 64 位 - 段界限 16 ~ 19
    db (memory_base >> 24) & 0xff      ; 基地址 24 ~ 31 位
gdt_data:
    dw memory_limit & 0xffff           ; 段界限 0 ~ 15 位
    dw memory_base & 0xffff            ; 基地址 0 ~ 16 位
    db (memory_base >> 16) & 0xff      ; 基地址 0 ~ 16 位
    db 0b_1_00_1_0_0_1_0               ; 存在 - dlp 0 - S _ 数据 - 向上 - 可写 - 没有被访问过
    db 0b1_1_0_0_0000 | (memory_limit >> 16) & 0xf  ; 4k - 32 位 - 不是 64 位 - 段界限 16 ~ 19
    db (memory_base >> 24) & 0xff      ; 基地址 24 ~ 31 位
gdt_end:
```

### 3.2、定义段选择子
同样的我们需要两个段选择子，分别用于选择代码段和数据段。
```asm
code_selector equ (1 << 3)  ; 代码段描述符位于全局描述符表的第一个
data_selector equ (2 << 3)  ; 数据段描述符位于全局描述符表的第二个
```

### 3.3、准备保护模式
前边已经将保护模式所需要的数据准备完毕了，接下来就可以准备进入保护模式了
```asm
prepare_protected_mode:
    cli   ; 关闭中断, 防止切换保护模式过程中被中断
    
    in al,  0x92  ; 打开 A20 线
    or al, 0b10
    out 0x92, al

    lgdt [gdt_ptr]; 加载 gdt

    mov eax, cr0  ; 启动保护模式
    or eax, 1
    mov cr0, eax

    ; 用跳转来刷新缓存，启用保护模式， 远跳转用于清空当前流水线中实模式的指令。
    jmp dword code_selector:protect_mode
```

### 3.4、进入保护模式
接下来我们初始化段寄存器，并测试修改1M以外的内存
```
[bits 32]  ; 告诉编译器已经进入32位模式
protect_mode:
    mov ax, data_selector
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax       ; 初始化段寄存器
    mov esp, 0x10000 ; 修改栈顶 此地址是随便写的

    mov byte [0xb8000], 'P'  ; 直接操作显示的内存
    mov byte [0x200000], 'P' ; 修改1M以外的内存
```

## 四、运行测试
执行bochs -q后，会发现gdtr寄存器是有数据的，这是因为bios进行自检做的。
![图片加载失败](/post_images/os/{{< filename >}}/4-01.png)
如图所示：GDT已经显示了我们定义的三个全局描述符。其中代码段描述符和数据段描述符都可以使用4GB的内存空间。并且可以对1MB以外的内存进行修改。
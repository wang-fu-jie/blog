---
title:       自制操作系统 - multiboot2引导
subtitle:    "multiboot2引导"
description: "Multiboot2是一个协议标准，GRUB 是这个协议的实现，在linux中一班时钟grub引导操作系统，而不是我们自己写的bootloader。使用grub引导需要遵循grub的规范来进行内核的一些列初始化"
excerpt:     "Multiboot2是一个协议标准，GRUB 是这个协议的实现，在linux中一班时钟grub引导操作系统，而不是我们自己写的bootloader。使用grub引导需要遵循grub的规范来进行内核的一些列初始化"
date:        2024-02-06T18:53:11+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/3a/94/brittany_beach_sea_sun_clouds_mirroring_perspective_hiking-1011647.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-multiboot2"
categories:  [ "操作系统" ]
---

## 一、multiboot2介绍
在linux系统中，是通过grub来引导操作系统程序。GRUB，全称 GRand Unified Bootloader，是一个多操作系统引导程序（Bootloader），主要用于在计算机启动时选择和引导操作系统内核，它是启动电脑后运行的第一个软件，它负责把控制权交给操作系统内核。

Multiboot2是一个协议标准，GRUB 是这个协议的实现者。要支持 multiboot2，内核必须添加一个 multiboot 头，而且必须在内核开始的 32768(0x8000) 字节，而且必须 64 字节对齐；

## 二、multiboot2头
在此之前我们自制的操作系统，在编译好内核程序后直接通过dd指令写入硬盘，且bochs是通过硬盘启动的。但是有了grub后，系统就不再通过硬盘启动， 而是通过cdrom启动。配置bochs配置文件：
```
boot: cdrom
ata0-master: type=cdrom, path="../build/kernel.iso", status=inserted
```
接下来就需要在进入内核的程序加上 multiboot2 头
```s
magic   equ 0xe85250d6
i386    equ 0
length  equ header_end - header_start

section .multiboot2
header_start:
    dd magic  ; 魔数
    dd i386   ; 32位保护模式
    dd length ; 头部长度
    dd -(magic + i386 + length); 校验和

    ; 结束标记
    dw 0    ; type
    dw 0    ; flags
    dd 8    ; size
header_end:
```
如上为multiboot2的内容，magic是魔数，按照multiboot2要求的格式定义模式，架构，头部长度，校验和，结束标记。这些内容无需细究，按照这么写即可。接下来还需要修改内核加载器，前边我们将内核加载了0x10000的位置，内核加载器会跳转到这个为止执行内核代码，现在内核最前边有了multiboot2头，因此0x10000位置现在存的是multiboot2头，进入内核就需要跳转到 0x10040。因为multiboot2是64字节对齐的。

接下来就是修改makefile生产iso文件，这里我们不做过多介绍，它并不涉及内核的相关知识。还需要写grub.cfg配置文件，内容如下：
```
set timeout=5
set default=0

menuentry "Onix" {
	multiboot2 /boot/kernel.bin
}
```
timeout是让grub界面显示5秒再进入内核。

## 三、multiboot2引导
在加入multiboot2头后，进入内核会报错内存初始化失败。因此我们需要处理multiboot2引导的内存初始化。以下为multiboot2引导下i386的状态：
* EAX：魔数 0x36d76289
* EBX：包含 bootloader 存储 multiboot2 信息结构体的，32 位 物理地址
* CS：32 位 可读可执行的代码段，尺寸 4G
* DS/ES/FS/GS/SS：32 位可读写的数据段，尺寸 4G
* A20 线：启用
* CR0：PG = 0, PE = 1，其他未定义
* EFLAGS：VM = 0, IF = 0, 其他未定义
* ESP：内核必须尽早切换栈顶地址
* GDTR：内核必须尽早使用自己的全局描述符表
* IDTR：内核必须在设置好自己的中断描述符表之前关闭中断

之前我们使用自己写的bootloader，但是现在使用grub就需要遵循grub的规则。我们先看下修改start.asm的内容：
```s
_start:
    push ebx    ;   ards_count 
    push eax    ; magic
    call console_init   ; 控制台初始化
    call gdt_init       ; 全局描述符初始化
    lgdt [gdt_ptr]
    jmp dword code_selector:_next
_next:
    mov ax, data_selector
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax; 初始化段寄存器
    call memory_init    ; 内存初始化
    mov esp, 0x10000; 修改栈顶
    call kernel_init    ; 内核初始化
    jmp $; 阻塞
```
先理一下grub引导执行流程。
1. 内核编译生成system.bin文件，（grub之前直接将该文件通过dd写入硬盘， 内核加载器去指定扇区读取）
2. 使用grub之后，通过grub-mkrescue将system.bin生成iso文件。
3. 通过iso文件启动内核，BIOS 读取光盘的引导扇区。加载到内存 0x7C00 并执行
4. 执行 /boot/grub/grub.cfg 中定义的菜单逻辑，加载内核。
5. 此时EAX中存储了Multiboot2 魔数， EBX中Multiboot2 信息结构的物理地址（包含内存布局、启动设备等）

接下来就进入内核代码开始执行了。从上边代码可以看出我们先初始化了控制台，因为我们需要打印功能。然后压入了EAX魔数，和EBX内存布局ards。 (这里回忆下我们自己写的bootloader做了内存布局检测， GDT初始化，然后在内核中拷贝直接并加载初始化好的GDT)。 使用GRUB后，是不会帮我们进行GDT初始化的，因此需要在内核中进行GDT初始化。如下为内核GDT初始化代码：
```cpp
#define KERNEL_CODE_IDX 1
#define KERNEL_DATA_IDX 2
descriptor_t gdt[GDT_SIZE]; // 内核全局描述符表
pointer_t gdt_ptr;          // 内核全局描述符表指针

void descriptor_init(descriptor_t *desc, u32 base, u32 limit)
{
    desc->base_low = base & 0xffffff;
    desc->base_high = (base >> 24) & 0xff;
    desc->limit_low = limit & 0xffff;
    desc->limit_high = (limit >> 16) & 0xf;
}

// 初始化内核全局描述符表
void gdt_init()
{
    DEBUGK("init gdt!!!\n");
    memset(gdt, 0, sizeof(gdt));  // 初始化第一个描述符全为0，即NULL描述符
    descriptor_t *desc;         // 全局描述符结构体
    desc = gdt + KERNEL_CODE_IDX;       // 第一个全局描述符为代码段
    descriptor_init(desc, 0, 0xFFFFF);
    desc->segment = 1;     // 代码段
    desc->granularity = 1; // 4K
    desc->big = 1;         // 32 位
    desc->long_mode = 0;   // 不是 64 位
    desc->present = 1;     // 在内存中
    desc->DPL = 0;         // 内核特权级
    desc->type = 0b1010;   // 代码 / 非依从 / 可读 / 没有被访问过

    desc = gdt + KERNEL_DATA_IDX;   // 初始化数据段全局描述符
    descriptor_init(desc, 0, 0xFFFFF);
    desc->segment = 1;     // 数据段
    desc->granularity = 1; // 4K
    desc->big = 1;         // 32 位
    desc->long_mode = 0;   // 不是 64 位
    desc->present = 1;     // 在内存中
    desc->DPL = 0;         // 内核特权级
    desc->type = 0b0010;   // 数据 / 向上增长 / 可写 / 没有被访问过
    gdt_ptr.base = (u32)&gdt;
    gdt_ptr.limit = sizeof(gdt) - 1;
}
```
在C语言中初始化后，结合上边的代码可以看到初始化gdt之后，在start.asm加载了gdt。 

接下来是内存初始化，grub会自动做内存检测，在内核中需要按照grub要求的规则进程内存初始化，例如判断EAX中的魔数，代码如下：
```cpp
else if (magic == MULTIBOOT2_MAGIC)
    {
        u32 size = *(unsigned int *)addr;
        multi_tag_t *tag = (multi_tag_t *)(addr + 8);

        LOGK("Announced mbi size 0x%x\n", size);
        while (tag->type != MULTIBOOT_TAG_TYPE_END)
        {
            if (tag->type == MULTIBOOT_TAG_TYPE_MMAP)
                break;
            // 下一个 tag 对齐到了 8 字节
            tag = (multi_tag_t *)((u32)tag + ((tag->size + 7) & ~7));
        }
        multi_tag_mmap_t *mtag = (multi_tag_mmap_t *)tag;
        multi_mmap_entry_t *entry = mtag->entries;
        while ((u32)entry < (u32)tag + tag->size)
        {
            LOGK("Memory base 0x%p size 0x%p type %d\n",
                 (u32)entry->addr, (u32)entry->len, (u32)entry->type);
            count++;
            if (entry->type == ZONE_VALID && entry->len > memory_size)
            {
                memory_base = (u32)entry->addr;
                memory_size = (u32)entry->len;
            }
            entry = (multi_mmap_entry_t *)((u32)entry + mtag->entry_size);
        }
    }
```
如上我们这里只贴了grub引导的内存初始化的分支，如果是我们自己写的加载器，那还是走原来的代码。这个分支的代码了解即可，它使用multiboot2要求的规则和结构。


## 四、可能遇到的问题
在使用grub生成iso文件时，虽然正常生产了iso，但是qemu报错cloud not read cdrom。 这是因为在i386中属于BIOS启动，会主引导扇区MBR启动运行，需要安装grub-pc-bin组件，但是目前的系统是属于UEFI启动，不包含grub-pc-bin，因此需要安装该组件。
```console
apt install grub-pc-bin xorriso
```
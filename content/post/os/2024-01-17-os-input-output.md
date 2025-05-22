---
title:       自制操作系统 - 输入输出与基础显卡驱动
subtitle:    "输入输出与基础显卡驱动"
description: "操作外部设备是通过端口实现的，本文将定义对端口进行输入输出的函数。并通过显卡的端口来实现一个基础的显卡驱动，操作屏幕和鼠标以及在屏幕上输出字符串等功能。"
excerpt:     "操作外部设备是通过端口实现的，本文将定义对端口进行输入输出的函数。并通过显卡的端口来实现一个基础的显卡驱动，操作屏幕和鼠标以及在屏幕上输出字符串等功能。"
date:        2024-01-17T16:32:18+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/da/23/london_bus_street_big_ben_city-64016.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-input-output"
categories:  [ "操作系统" ]
---

## 一、基本数据类型和字符串处理
前边我们提到，开发操作系统不能使用C的内置函数和标准库。但是我们又需要处理字符串和使用基本类型，因此需要自己定义基本类型和字符串处理函数。
```cpp
#define EOF -1 // END OF FILE
#define NULL ((void *)0) // 空指针
#define EOS '\0' // 字符串结尾
#define bool _Bool
#define true 1
#define false 0
#define _packed __attribute__((packed)) // 用于定义特殊的结构体
typedef unsigned int size_t;
typedef char int8;
typedef short int16;
typedef int int32;
typedef long long int64;
typedef unsigned char u8;
typedef unsigned short u16;
typedef unsigned int u32;
typedef unsigned long long u64;

char *strcpy(char *dest, const char *src);
char *strcat(char *dest, const char *src);
size_t strlen(const char *str);
int strcmp(const char *lhs, const char *rhs);
char *strchr(const char *str, int ch);
char *strrchr(const char *str, int ch);
int memcmp(const void *lhs, const void *rhs, size_t count);
void *memset(void *dest, int ch, size_t count);
void *memcpy(void *dest, const void *src, size_t count);
void *memchr(const void *ptr, int ch, size_t count);
```
如上所示：定义了基础数据类型和字符串处理，这里没有将字符串处理的函数实现贴出来，可自行查阅源码。

## 二、输入与输出
本小节先做一个简单的输入输出，控制屏幕的光标位置。在80386的体系架构中，文本模式下的光标位置 是通过 CRT 控制器（Cathode Ray Tube Controller） 的端口控制的，涉及的端口如下：
* CRT 地址寄存器 0x3D4   表示访问CRT的哪个寄存器
* CRT 数据寄存器 0x3D5   对地址寄存器里表示的寄存器进行读写
* CRT 光标位置 - 高位 0xE
* CRT 光标位置 - 低位 0xF

首先需要定义用于对端口读写的函数，函数原形使用C，但是实现使用汇编：
```asm
global inb ; 将 inb 导出
inb:
    push ebp; 
    mov ebp, esp ; 保存帧
    xor eax, eax ; 将 eax 清空
    mov edx, [ebp + 8]; 端口号，端口号被调用者作为参数传递进来，因此位于栈中
    in al, dx; 将端口号 dx 的 8 bit 输入到 al
    jmp $+2 ; 一点点延迟
    leave ; 恢复栈帧
    ret
```
这里只贴一下给端口输入一个字节的实现， 在头文件中定义函数原型  extern u8 inb(u16 port)， 给端口输入一个字，从端口读一个字节，读一个字等三个函数是类似的。接下来就可以在内核的main.c函数中测试输入输出函数了
```cpp
#define CRT_ADDR_REG 0x3d4
#define CRT_DATA_REG 0x3d5

#define CRT_CURSOR_H 0xe
#define CRT_CURSOR_L 0xf

void kernel_init()
{
    outb(CRT_ADDR_REG, CRT_CURSOR_H);  // 操作光标高8位
    u16 pos = inb(CRT_DATA_REG) << 8;  //将输入的数据左移8位，左移后低用0填充
    outb(CRT_ADDR_REG, CRT_CURSOR_L);  // 操作光标的低8为
    pos |= inb(CRT_DATA_REG);          // 将获取到的低8为填进去
    return;
}
```
运行这个代码，就可以获取到光标的当前位置，也可以修改光标位置，这里就不再贴修改光标位置的代码。

## 三、基础显卡驱动
显卡模式是指 显卡（GPU）与显示器 协同工作时采用的 图像输出格式，我们这里使用CGA的文本模式，40x25 或 80x25，16色。CGA 使用的 MC6845 芯片；
* CRT 地址寄存器 0x3D4
* CRT 数据寄存器 0x3D5
* CRT 光标位置 - 高位 0xE
* CRT 光标位置 - 低位 0xF
* CRT 显示开始位置 - 高位 0xC
* CRT 显示开始位置 - 低位 0xD

在往显示器写内容时需要考虑控制字符，控制字符是指 ASCII 码表开头的 32 个字符。例如ASCII的13为换行，相当于\n，当遇到是光标的位置应该进入下一行。还有其他的例如\r、 \b 等。

### 3.1、代码实现
在显卡驱动实现，我们只实现文本模式，包括一下三个函数
```cpp
void console_init();
void console_clear();
void console_write(char *buf, u32 count);
```
当然这里还需要做一些宏定义，例如定义CRT寄存器的端口，显卡开始的内存位置0xb8000，以及一些控制字符等。首先需要一些辅助函数：
```cpp
static void get_screen() // 获得当前显示器的开始位置，这里需要通过端口获取的屏幕位置需要*2，因为端口返回的是字符，但是内存单位是字节。
static void set_screen() // 设置当前显示器开始的位置
static void get_cursor() // 获得当前光标位置
static void set_cursor() // 设置光标的位置
```
这里省略了这些函数的函数体。因为上文中有获取光标位置的实例代码，就是通过给CRT对应的寄存器进行in/out操作。接下来看清空屏幕的实现
```cpp
void console_clear()
{
    screen = MEM_BASE;
    pos = MEM_BASE;
    x = y = 0;
    set_cursor();
    set_screen();

    u16 *ptr = (u16 *)MEM_BASE; // 将显存内所有字符设置为空格
    while (ptr < MEM_END)
    {
        *ptr++ = erase;
    }
}
```
console_clear函数直接调用console_clear函数清空屏幕就可以了。

### 3.2、console_write实现
console_write是最复杂的一个函数了。它需要实现控制字符，例如遇到\n进行换行，实现字符的显示，以及屏幕的滚屏动作。
```cpp
static void command_del()
{
    *(u16 *)pos = erase;  // erase是空格，意思是橡皮擦，在文件中定义
}

void console_write(char *buf, u32 count)
{
    char ch;
    char *ptr = (char *)pos;
    while (count--)
    {
        ch = *buf++;
        switch (ch)
        {
        case ASCII_NUL:
            break;
        case ASCII_BEL:
           //...
        case ASCII_DEL:
            command_del();
            break;
        default:
            if (x >= WIDTH)
            {
                x -= WIDTH;
                pos -= ROW_SIZE;
                command_lf();
            }
            *ptr = ch;
            ptr++;
            *ptr = attr;
            ptr++;

            pos += 2;
            x++;
            break;
        }
    }
    set_cursor();
}
```
如上所示，为了节约文章篇幅，这里只贴了del控制符的代码。这里需要注意的是，在遇到换行符时，如果当前行已经在屏幕最后一行，那屏幕就需要滚动，向上滚动一行。滚屏的函数此处不再粘贴，大致实现逻辑是，将当前屏幕下方的一行全部设置为空格，屏幕和鼠标的位置加上1行的大小。另外屏幕是循环滚动的，如果当前屏幕显示的位置已经到了显存的末端。就应该把当前屏幕显示的内存重新拷贝到显存开始的位置，这样就可以从头滚动了。
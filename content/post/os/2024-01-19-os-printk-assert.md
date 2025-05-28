---
title:       自制操作系统 - 打印与调试
subtitle:    "打印与调试"
description: "内核无法使用C语言中的打印函数，因此在内核中需要自己实现打印功能。本文将介绍可变参数是如何实现的，以及进行打印功能、断言功能和调试功能的实现，它们的核心都是打印相关信息，便于追踪。"
excerpt:     "内核无法使用C语言中的打印函数，因此在内核中需要自己实现打印功能。本文将介绍可变参数是如何实现的，以及进行打印功能、断言功能和调试功能的实现，它们的核心都是打印相关信息，便于追踪。"
date:        2024-01-19T13:38:19+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/f8/8e/bike_fog_forest_wood_tree-106172.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-printk-assert"
categories:  [ "操作系统" ]
---

## 一、可变参数原理
在C语言中，printf可以传入任意个参数，那在执行这个参数是如何知道传入了多少参数呢。这就用到了可变参数，通过上节我们知道，函数传参时按照从右到左的顺序压入栈中。因此从栈中可以依次获取到这些参数，我们需要先定义一下几个变量：
```cpp
typedef char *va_list;
#define va_start(ap, v) (ap = (va_list)&v + sizeof(char *))
#define va_arg(ap, t) (*(t *)((ap += sizeof(char *)) - sizeof(char *)))
#define va_end(ap) (ap = (va_list)0)
```
如上所示，我们逐个解释下：
* va_list保存可变参数指针。这里使用char *，因为char *的大小是固定的。
* va_start：v是变参前的一个已知参数，+ sizeof(char *)：跳过这个参数的大小，指向第一个变参的地址
* va_arg：先将 ap 前进一个参数的位置。再减回来，得到当前参数的位置（因为我们要“取”的是前一个参数）。最后解引用获取值
* va_end：这个宏用于结束变参处理，通常只是为了清理 va_list
 
我们直接结合示例来理解下：
```cpp
void test_args(int cnt, ...)
{
    va_list args;              // 定义一个指针
    va_start(args, cnt);       // 获取传入的第一个参数，赋给args

    int arg;                   // 这里测试传入的参数都是整型
    while (cnt--)
    {
        arg = va_arg(args, int);  // args指向下一个参数, 然后获取当前args的值
    }
    va_end(args);                // 结束可变参数
}
test_args(5, 1, 0xaa, 5, 0x55, 10);  // 调用函数
```
你可能会奇怪，在使用printf时并没有传入参数的数量，这里需要说明在使用printf时，传入的第一个字符为format，它里边有占位符，有多少个占位符就代表后边有多少个参数。

注意：这里在调试时，可以使用 -exec display/16xw $sp 观察栈的变化。在函数调用时，会压入旧栈帧和eip，eip是call指令自动压入的。如图所示：
![图片加载失败](/post_images/os/{{< filename >}}/1-01.png)

## 二、printk实现
printk是内核用来打印的函数，这里参考printf的功能。我们知道printf是使用%后跟一堆参数，格式如： %[flags][width][.prec][h|l|L][type] 其中：
* %：格式引入字符
* flags：可选的标志字符序列
* width：可选的宽度指示符
* .prec：可选的精度指示符
* h|l|L：可选的长度修饰符， 例如ld代表长整型
* type：转换类型，例如d代表整型 s代表字符串

关于这些参数更详细的介绍，请自行查阅资料。总之printk和printf的核心功能一样。printk的代码实现如下：
```cpp
static char buf[1024];

int printk(const char *fmt, ...)
{
    va_list args;
    int i;
    va_start(args, fmt);
    i = vsprintf(buf, fmt, args);    // 这里重点是vsprintf函数，它实现了字符串的格式化并写入到buf
    va_end(args);
    console_write(buf, i);           // 将buf的数据写到控制台
    return i;
}
```
vsprintf函数将入参进行格式化处理，根据传入的format参数进行格式化。例如如果给flag传入0，则进行0填充。根据%后的类型进行相应的类型处理，这块的代码比较大，不在文本内进行粘贴。

## 三、断言assert
断言是用于确定程序的运行状态，防止错误蔓延，如果出错提供尽可能多的错误信息，以便排查。首先定义一下头文件：
```cpp
void assertion_failure(char *exp, char *file, char *base, int line);

#define assert(exp) \     // 定义宏函数，在表达式exp出错时，打印出错行代码所在的文件和行信息
    if (exp)        \
        ;           \
    else            \  // __FILE__等 是 C/C++ 语言标准中的预定义宏
        assertion_failure(#exp, __FILE__, __BASE_FILE__, __LINE__)

void panic(const char *fmt, ...);     // 在程序无法运行时调用panic函数
```
如上所示，定义宏函数assert为方便断言的使用，例如assert(3<5)。 如果断言失败则执行assertion_failure函数，打印断言信息：
```cpp
void assertion_failure(char *exp, char *file, char *base, int line)
{
    printk(
        "\n--> assert(%s) failed!!!\n"
        "--> file: %s \n"
        "--> base: %s \n"
        "--> line: %d \n",
        exp, file, base, line);

    spin("assertion_failure()");       // 打印出在哪里阻塞了，spin函数里是一个while True 死循环。
    // 不可能走到这里，否则出错；
    asm volatile("ud2");
}
```
pinic的函数实现也是类似的，打印出错信息，然后进入死循环阻塞。

## 四、调试工具debug
内核使用的调试工具有两个，一个是bochs的魔术断点，一个是打印工具。先看宏定义
```cpp
#define BMB asm volatile("xchgw %bx, %bx") // bochs magic breakpoint
#define DEBUGK(fmt, args...) debugk(__BASE_FILE__, __LINE__, fmt, ##args)
```
如上魔术断点为内联汇编，打印工具只是实现了信息的打印，实现如下：
```cpp
static char buf[1024];

void debugk(char *file, int line, const char *fmt, ...)
{
    va_list args;
    va_start(args, fmt);
    vsprintf(buf, fmt, args);
    va_end(args);

    printk("[%s] [%d] %s", file, line, buf);
}
```
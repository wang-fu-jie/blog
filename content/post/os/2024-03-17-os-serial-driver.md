---
title:       自制操作系统 - 串口设备驱动
subtitle:    "串口设备驱动"
description: "串口是一种 IBM-PC 兼容机上的传统通信端口， 目前已经被USB取代。串口驱动对系统开发者来说实现起来要比 USB 简单的多，通常用于调试的目的，无需复杂的硬件操作，可以在操作系统初始化的早期使用串口。我们这里的目的也是把调试的信息放到串口里面。"
excerpt:     "串口是一种 IBM-PC 兼容机上的传统通信端口， 目前已经被USB取代。串口驱动对系统开发者来说实现起来要比 USB 简单的多，通常用于调试的目的，无需复杂的硬件操作，可以在操作系统初始化的早期使用串口。我们这里的目的也是把调试的信息放到串口里面"
date:        2024-03-17T15:44:46+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/97/04/heartsickness_lover's_grief_lovesickness_coupe_bench_sitting_argument_quarrel-1335874.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-serial-driver"
categories:  [ "操作系统" ]
---

## 一、串口介绍
串口是一种 IBM-PC 兼容机上的传统通信端口，使用串口连接外设已经被如 USB(Universial Serial Bus) 等新的通信方式所取代。不过串口仍然广泛应用于工业硬件领域，历史上很多拨号上网的调制解调器（猫）通常以串口连接计算机通信。PC 微机的串行通信使用的异步串行通信芯片是 INS 8250 或 NS16450 兼容芯片，统称为 UART(Universial Asynchronous Receiver/Transceiver 通用异步接收发送器)。负责串口的编码和解码工作。

串口驱动对系统开发者来说实现起来要比 USB 简单的多，通常用于调试的目的，无需复杂的硬件操作，可以在操作系统初始化的早期使用串口。我们这里的目的也是把调试的信息放到串口里面。

## 二、串口编程
对 UART 的编程实际上是对其内部寄存器执行读写操作。因此可将 UART 看作是一组寄存器集合，包含发送、接收和控制三部分。UART 内部有 10 个寄存器，供 CPU 通过 in/out 指令对其进行访问。我们这里不对串口的寄存器进行过多的介绍，可参考 [串口设备驱动](https://github.com/StevenBaby/onix/blob/9157a4e202f6f274f9a4fb222f551afb5e3ecc2a/docs/09%20%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8/111%20%E4%B8%B2%E5%8F%A3%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8.md)

## 2.1、qemu虚拟机配置串口
qemu 虚拟机配置串口 6 的方式如下：
```
QEMU+= -chardev stdio,mux=on,id=com1 # 字符设备 1
# QEMU+= -chardev vc,mux=on,id=com1 # 字符设备 1
QEMU+= -chardev udp,id=com2,port=6666,ipv4=on # 字符设备 2
QEMU+= -serial chardev:com1 # 串口 1
QEMU+= -serial chardev:com2 # 串口 2
```
stdio 表示将字符输出到终端
vc 是 qemu 默认的虚拟终端，可以在 TAB (View -> Show Tabs)中打开
udp 用 udp 协议传输字符数据，主要可以用 netcat 来调试。 netcat调式串口使用到的命令如下：
```cmd
nc -ulp 6666  # 建立服务器端，监听端口号 6666：
nc -u localhost 6666  #  nc 建立客户端，连接刚建好的服务器
```

## 三、串口代码实现
串口是也是作为一个设备，被虚拟设备统一管理。我们使用前两个串口，因为前两个串口的端口地址是固定的。
```cpp
#include <onix/io.h>
#include <onix/interrupt.h>
#include <onix/fifo.h>
#include <onix/task.h>
#include <onix/mutex.h>
#include <onix/assert.h>
#include <onix/device.h>
#include <onix/debug.h>
#include <onix/stdarg.h>
#include <onix/stdio.h>

#define LOGK(fmt, args...) DEBUGK(fmt, ##args)

#define COM1_IOBASE 0x3F8 // 串口 1 基地址
#define COM2_IOBASE 0x2F8 // 串口 2 基地址

#define COM_DATA 0          // 数据寄存器
#define COM_INTR_ENABLE 1   // 中断允许
#define COM_BAUD_LSB 0      // 波特率低字节
#define COM_BAUD_MSB 1      // 波特率高字节
#define COM_INTR_IDENTIFY 2 // 中断识别
#define COM_LINE_CONTROL 3  // 线控制
#define COM_MODEM_CONTROL 4 // 调制解调器控制
#define COM_LINE_STATUS 5   // 线状态
#define COM_MODEM_STATUS 6  // 调制解调器状态

// 线状态
#define LSR_DR 0x1
#define LSR_OE 0x2
#define LSR_PE 0x4
#define LSR_FE 0x8
#define LSR_BI 0x10
#define LSR_THRE 0x20
#define LSR_TEMT 0x40
#define LSR_IE 0x80

#define BUF_LEN 64

typedef struct serial_t
{
    u16 iobase;           // 端口号基地址
    fifo_t rx_fifo;       // 读 fifo
    char rx_buf[BUF_LEN]; // 读 缓冲
    lock_t rlock;         // 读锁
    task_t *rx_waiter;    // 读等待任务
    lock_t wlock;         // 写锁
    task_t *tx_waiter;    // 写等待任务
} serial_t;

static serial_t serials[2];   // 串口表，我们只使用前两个串口，因此指定大小为2

void recv_data(serial_t *serial)
{
    char ch = inb(serial->iobase);
    if (ch == '\r') // 特殊处理，回车键直接换行
    {
        ch = '\n';
    }
    fifo_put(&serial->rx_fifo, ch);
    if (serial->rx_waiter != NULL)
    {
        task_unblock(serial->rx_waiter);
        serial->rx_waiter = NULL;
    }
}

// 中断处理函数
void serial_handler(int vector)
{
    u32 irq = vector - 0x20;
    assert(irq == IRQ_SERIAL_1 || irq == IRQ_SERIAL_2);
    send_eoi(vector);
    serial_t *serial = &serials[irq - IRQ_SERIAL_1];
    u8 state = inb(serial->iobase + COM_LINE_STATUS);

    if (state & LSR_DR) // 数据可读
    {
        recv_data(serial);
    }
    if ((state & LSR_THRE) && serial->tx_waiter)  // 如果可以发送数据，并且写进程阻塞
    {
        task_unblock(serial->tx_waiter);
        serial->tx_waiter = NULL;
    }
}

int serial_read(serial_t *serial, char *buf, u32 count)
{
    lock_acquire(&serial->rlock);
    int nr = 0;
    while (nr < count)
    {
        while (fifo_empty(&serial->rx_fifo))   // 如果队列为空，阻塞当前任务，等待终端唤醒
        {
            assert(serial->rx_waiter == NULL);
            serial->rx_waiter = running_task();
            task_block(serial->rx_waiter, NULL, TASK_BLOCKED);
        }
        buf[nr++] = fifo_get(&serial->rx_fifo);
    }
    lock_release(&serial->rlock);
    return nr;
}

int serial_write(serial_t *serial, char *buf, u32 count)   // 串口写
{
    lock_acquire(&serial->wlock);    // 申请写锁
    int nr = 0;
    while (nr < count)
    {
        u8 state = inb(serial->iobase + COM_LINE_STATUS);
        if (state & LSR_THRE) // 如果串口可写
        {
            outb(serial->iobase, buf[nr++]);
            continue;
        }
        task_t *task = running_task();
        serial->tx_waiter = task;
        task_block(task, NULL, TASK_BLOCKED);
    }
    lock_release(&serial->wlock);
    return nr;
}


void serial_init()   // 初始化串口
{
    for (size_t i = 0; i < 2; i++)   // 我们只使用前两个串口，因此这里写死为2.
    {
        serial_t *serial = &serials[i];
        fifo_init(&serial->rx_fifo, serial->rx_buf, BUF_LEN);   // 初始化队列，用来读写数据
        serial->rx_waiter = NULL;
        lock_init(&serial->rlock);
        serial->tx_waiter = NULL;
        lock_init(&serial->wlock);

        u16 irq;
        if (!i)
        {
            irq = IRQ_SERIAL_1;
            serial->iobase = COM1_IOBASE;
        }
        else
        {
            irq = IRQ_SERIAL_2;
            serial->iobase = COM2_IOBASE;
        }

        outb(serial->iobase + COM_LINE_CONTROL, 0x80);   // 激活 DLAB
        outb(serial->iobase + COM_BAUD_LSB, 0x30);   // 设置波特率因子 0x0030  波特率为 115200 / 0x30 = 115200 / 48 = 2400
        outb(serial->iobase + COM_BAUD_MSB, 0x00);
        outb(serial->iobase + COM_LINE_CONTROL, 0x03);    // 复位 DLAB 位，数据位为 8 位
        outb(serial->iobase + COM_INTR_ENABLE, 0x0d);  // 0x0d = 0b1101   数据可用 + 中断/错误 + 状态变化 都发送中断
        outb(serial->iobase + COM_MODEM_CONTROL, 0b11011);   // 设置回环模式，测试串口芯片
        outb(serial->iobase, 0xAE);                        // 发送字节
        if (inb(serial->iobase) != 0xAE)                    // 收到的内容与发送的不一致，则串口不可用
        {
            continue;
        }
        outb(serial->iobase + COM_MODEM_CONTROL, 0b1011);       // 设置回原来的模式
        set_interrupt_handler(irq, serial_handler);             // 注册中断函数
        set_interrupt_mask(irq, true);                          // 打开中断屏蔽å
        char name[16];
        sprintf(name, "com%d", i + 1);

        device_install(                             // 进行虚拟设备的安装
            DEV_CHAR, DEV_SERIAL, serial, name, 0,
            NULL, serial_read, serial_write);
        LOGK("Serial 0x%x init...\n", serial->iobase);
    }
}
```
如上所示，为串口驱动的实现，我们可以通过设备读写来操作串口读写，串口的数据存放到队列中进行暂存。

## 四、串口测试
我们在test系统调用中进行串口的读写测试：
```cpp
static u32 sys_test()
{
    char ch;
    device_t *device;

    device_t *serial = device_find(DEV_SERIAL, 0);   // 进程串口读
    assert(serial);

    device_t *keyboard = device_find(DEV_KEYBOARD, 0);
    assert(keyboard);
    device_t *console = device_find(DEV_CONSOLE, 0);
    assert(console);

    device_read(serial->dev, &ch, 1, 0, 0);
    // device_read(keyboard->dev, &ch, 1, 0, 0);
    device_write(serial->dev, &ch, 1, 0, 0);
    device_write(console->dev, &ch, 1, 0, 0);

    return 255;
}
```
如上所示，test系统调用进行键盘读取，并写入到串口和终端。并在makefile中配置串口1为stdio, 串口2为vc。stdio即QEMU 启动的终端。


## 五、串口输出日志
我们的系统有了 shell 之后，直接将日志输出到控制台，开始有点乱了，可以将日志通过宏去掉，或者将日志输出到串口。另外，bochs 12 好像对串口的支持不太好，目前不知道是为什么所以直接禁了。内核调试的日志是实现函数debugk
```cpp
void debugk(char *file, int line, const char *fmt, ...)
{
    device_t *device = device_find(DEV_SERIAL, 0);
    if (!device)
    {
        device = device_find(DEV_CONSOLE, 0);
    }

    int i = sprintf(buf, "[%s] [%d] ", file, line);
    device_write(device->dev, buf, i, 0, 0);

    va_list args;
    va_start(args, fmt);
    i = vsprintf(buf, fmt, args);
    va_end(args);

    device_write(device->dev, buf, i, 0, 0);
}
```
如上所示，通过修改debugk，将日志输出到串口，这样调试日志就打印到了启动qemu的终端。而不再打印到我们操作系统的shell终端。 

最后，串口也输出设备，在设备文件初始化也需要在/dev目录下创建串口的设备文件。
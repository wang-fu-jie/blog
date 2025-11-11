---
title:       自制操作系统 - ISA总线与声霸卡驱动
subtitle:    "ISA总线与声霸卡驱动"
description: "ISA总线是CPU与内存以及外部设备进行数据交换的通道，它正在慢慢地被现代机器中常见的超级 I/O 芯片所取代。但是仍然有一些设备使用ISA，确切的说是使用ISA DMA功能。如内部软盘，声霸卡等。本文将通过ISA DMA来实现声霸卡驱动。"
excerpt:     "ISA总线是CPU与内存以及外部设备进行数据交换的通道，它正在慢慢地被现代机器中常见的超级 I/O 芯片所取代。但是仍然有一些设备使用ISA，确切的说是使用ISA DMA功能。如内部软盘，声霸卡等。本文将通过ISA DMA来实现声霸卡驱动。"
date:        2024-04-05T15:24:46+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/46/bd/horses_nature_animal_horse_animals_equine_mane_farm-1373792.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-isa-sound-driver"
categories:  [ "操作系统" ]
---

## 一、ISA总线
总线是CPU与内存以及外部设备进行数据交换的通道。工业标准体系结构(ISA Industry Standard Architecture) 1 总线是在 1981 年为最初的 IBM PC 创建的。彼时彼刻，它是一个 8 位，5 MHz 总线(2.39MB/s)，但后来升级到 16 位，8MHz (8.33MB/s)。它仍然常见于较老的机器中，许多最常见的基本设备都连接到它。由于这个原因，许多操作系统仍然支持它。它正在慢慢地被现代机器中常见的超级 I/O 芯片所取代。

### 1.1、ISA DMA要点
在前面编程中，我们在进行存储访问时都是使用的CPU的IO操作端口，这样是比较占用CPU资源的。DMA全称为Direct Memory Access 直接存储访问，使用DMA就可以把CPU空出来，去做其他事情。DMA 控制器（DMA Controller, DMAC）是一个独立的硬件模块。外设需要读写内存时，CPU 先告诉 DMA 控制器源地址、目标地址长度和方向。然后 CPU 下达“开始 DMA”命令，DMA 控制器接管总线，直接在内存和外设之间传输数据。传输完成后，DMA 控制器发出中断通知 CPU：“数据搬好了”。ISA DMA 的要点是：
* ISA DMA 通道 1、2 和 3 可用于 8 位传输到 ISA 外设;
* ISA DMA 通道 5、6 和 7 可用于向 ISA 外设的 16 位传输;
* 传输不能跨越物理 64KB 边界，并且不能大于 64KB;
* 传输必须在物理上是连续的，并且只能以最低的 16MB 物理内存为目标;
* ISA DMA 很慢-理论上是 4.77 MB/s，但由于 ISA 总线协议，更像是 400 KB/s;
* ISA DMA 释放了 CPU 资源，但是给内存总线增加了非常重的负载;

目前很少有设备使用 ISA DMA，只有：内部软盘、一些嵌入式声音芯片、一些并行端口、一些串行端口，


### 1.2、ISA DMA编程
DMA 不能访问任何虚拟内存地址，分页内存映射是由 CPU 独占控制的。DMA 的全部意义就是绕过 CPU。所有的 DMA 总是只在物理内存地址上完成。ISA DMA 有 16MB 的物理地址限制。ISA 架构下有两个 DMA 控制器。 主DMA控制器	通道号0–3，数据宽度8-bit，通常用于低速设备（声卡、软驱），从DMA控制器通道号4–7，数据宽度16-bit，用于高速设备（硬盘、网卡）。通道 4 用来把两个控制器“级联”成一个整体。
```cpp
#define DMA_MODE_CHECK 0x00   // 自检模式
#define DMA_MODE_READ 0x04    // 外部设备读出（写内存）
#define DMA_MODE_WRITE 0x08   // 写入外部设备（读内存）
#define DMA_MODE_AUTO 0x10    // 自动模式
#define DMA_MODE_DOWN 0x20    // 从高地址向低地址访问内存
#define DMA_MODE_DEMAND 0x00  // 按需传输；
#define DMA_MODE_SINGLE 0x40  // 单次 DMA 传输；
#define DMA_MODE_BLOCK 0x80   // 块 DMA 传输；
#define DMA_MODE_CASCADE 0xC0 // 级联模式(用于级联另一个 DMA 控制器)；
 
void isa_dma_mask(u8 channel, bool mask);   // 屏蔽/启用通道  控制是否允许 DMA 工作
void isa_dma_addr(u8 channel, void *addr);  // 设置内存起始地址  分低16位 + 高8位页
void isa_dma_size(u8 channel, u32 size);    // 设置传输长度 低通道按字节，高通道按字
void isa_dma_reset(u8 channel);             // 设置传输模式  决定方向与类型
void isa_dma_mode(u8 channel, u8 mode);    // 重置控制器  让 DMA 进入初始状态
```
如上所示，为ISA DMA的相关定义。在需要使用DMA时，基本流程如下：
1.	CPU 设置 DMA 通道号、模式、地址、长度
2.	DMA 通道被解屏蔽
3.	设备发 DMA 请求（DRQ
4.	DMA 控制器获得总线控制权（HRQ→HLDA
5.	DMA 自动搬运数据
6.	传输结束后发中断
7.	CPU 响应中断，做下一步处理

## 二、声霸卡驱动
Sound Blaster 1 系列是由 Creative 制作的声卡系列。多年以来，它们都是 IBM PC 上的标准声卡。 Sound Blaster 16 内置的 DSP 支持播放和录制 8 位和 16 位 PCM 编码的音频，以及播放其他几种格式( ADPCM 等)。I/O 寄存器基地址可以通过 PCI 型号的 PCI 总线找到，或者通过向几个常见 I/O 端口地址之一 (0x220、0x240 等) 发出 Get Version 命令来检测旧的 ISA Sound Blaster 的存在，并等待其响应。

### 2.1、声卡驱动实现
声卡作为一种声音设备，也需要接入虚拟设备进行管理，设备类型定义为 DEV_SB16。接下来看声霸卡驱动的实现：
```cpp
typedef enum sb16_cmd_t
{
    SB16_CMD_ON = 1,   // 声卡开启
    SB16_CMD_OFF,      // 声卡关闭
    SB16_CMD_MONO8,    // 八位单声道模式
    SB16_CMD_STEREO16, // 16位立体声模式
    SB16_CMD_VOLUME,   // 设置音量
} sb16_cmd_t;


typedef struct sb_t
{
    task_t *waiter;   // 等到声卡的进程
    lock_t lock;
    char *addr; // DMA 地址
    u8 mode;    // 模式
    u8 channel; // DMA 通道
} sb_t;

static sb_t sb16;   // 我们只有一个声卡

static void sb_handler(int vector)   // 声卡的中断处理函数
{
    send_eoi(vector);
    sb_t *sb = &sb16;
    inb(SB_INTR16);
    u8 state = inb(SB_STATE);
    if (sb->waiter)           // 如果有等待使用声卡的进程，就解除阻塞
    {
        task_unblock(sb->waiter, EOK);
        sb->waiter = NULL;
    }
}

static void sb_reset(sb_t *sb)  // 声卡重置
{
    outb(SB_RESET, 1);
    sleep(1);
    outb(SB_RESET, 0);
    u8 state = inb(SB_READ);
}

static void sb_intr_irq(sb_t *sb)
{
    outb(SB_MIXER, 0x80);
    u8 data = inb(SB_MIXER_DATA);
    if (data != 2)
    {
        outb(SB_MIXER, 0x80);
        outb(SB_MIXER_DATA, 0x2);
    }
}

static void sb_out(u8 cmd)
{
    while (inb(SB_WRITE) & 128)
        ;
    outb(SB_WRITE, cmd);
}

static void sb_turn(sb_t *sb, bool on)   // 开启/关闭声卡
{
    if (on)
        sb_out(CMD_ON);
    else
        sb_out(CMD_OFF);
}

static u32 sb_time_constant(u8 channels, u16 sample)
{
    return 65536 - (256000000 / (channels * sample));
}

static void sb_set_volume(u8 level)
{
    LOGK("set sb16 volume to 0x%02X\n", level);
    outb(SB_MIXER, 0x22);
    outb(SB_MIXER_DATA, level);
}

int sb16_ioctl(sb_t *sb, int cmd, void *args, int flags)
{
    switch (cmd)
    {
    // 设置 tty 参数
    case SB16_CMD_ON:
        sb_reset(sb);    // 重置 DSP
        sb_intr_irq(sb); // 设置中断
        sb_out(CMD_ON);  // 打开声霸卡
        return EOK;
    case SB16_CMD_OFF:
        sb_out(CMD_OFF); // 关闭声霸卡
        return EOK;
    case SB16_CMD_MONO8:
        sb->mode = MODE_MONO8;
        sb->channel = 1;
        isa_dma_reset(sb->channel);
        return EOK;
    case SB16_CMD_STEREO16:
        sb->mode = MODE_STEREO16;
        sb->channel = 5;
        isa_dma_reset(sb->channel);
        return EOK;
    case SB16_CMD_VOLUME:
        sb_set_volume((u8)args);
        return EOK;
    default:
        break;
    }
    return -EINVAL;
}

int sb16_write(sb_t *sb, char *data, size_t size)   // 给声卡写入数据，即播放声音
{
    lock_acquire(&sb->lock);
    assert(size <= DMA_BUF_SIZE);   // 写入大小硬小于缓冲区
    memcpy(sb->addr, data, size);   // 赋值数据到缓冲器
    isa_dma_mask(sb->channel, false);   // DMA进行写设备
    isa_dma_addr(sb->channel, sb->addr);
    isa_dma_size(sb->channel, size);

    sb_out(CMD_SOSR);                  // 44100 = 0xAC44   // 设置采样率
    sb_out((SAMPLE_RATE >> 8) & 0xFF); // 0xAC
    sb_out(SAMPLE_RATE & 0xFF);        // 0x44

    if (sb->mode == MODE_MONO8)      // 
    {
        isa_dma_mode(sb->channel, DMA_MODE_SINGLE | DMA_MODE_WRITE);
        sb_out(CMD_SINGLE_OUT8);
        sb_out(MODE_MONO8);
    }
    else
    {
        isa_dma_mode(sb->channel, DMA_MODE_SINGLE | DMA_MODE_WRITE);
        sb_out(CMD_SINGLE_OUT16);
        sb_out(MODE_STEREO16);
        size >>= 2; // size /= 4
    }

    sb_out((size - 1) & 0xFF);
    sb_out(((size - 1) >> 8) & 0xFF);
    isa_dma_mask(sb->channel, true);
    assert(sb->waiter == NULL);
    sb->waiter = running_task();
    assert(task_block(sb->waiter, NULL, TASK_BLOCKED, TIMELESS) == EOK);

rollback:
    lock_release(&sb->lock);
    return size;
}

void sb16_init()
{
    sb_t *sb = &sb16;
    sb->addr = (char *)DMA_BUF_ADDR;   // 设置声卡DMA地址， #define DMA_BUF_ADDR 0x40000 // 必须 64K 字节对齐
    sb->mode = MODE_STEREO16;
    sb->channel = 5;
    lock_init(&sb->lock);
    set_interrupt_handler(IRQ_SB16, sb_handler);   // 设置中断处理函数
    set_interrupt_mask(IRQ_SB16, true);
    device_install(DEV_CHAR, DEV_SB16, sb, "sb16", 0, sb16_ioctl, NULL, sb16_write);   // 安装设备
}
```
我们这里只实现了声卡写，即播放声音。但是没有实现声卡读，即录音功能。

## 三、 播放声音的应用程序
在完成声卡驱动时，就可以通过应用程序读取文件并播放声音。大体逻辑即使读取音乐文件，并循环写入显卡。

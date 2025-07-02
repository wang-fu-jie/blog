---
title:       自制操作系统 - 键盘驱动
subtitle:    "键盘驱动"
description: "键盘中断时8259a芯片的1号中断，中断向量为0x21。当键盘按键时，需要处理0x21中断，获取按下的扫描码，根据不同的扫描码进行处理，例如字符就打印。目前使用键盘都是第二套扫描码，但是内部会被8042转换为第一套扫描码，因此第一套不存在的扫描码就属于扩展码。"
excerpt:     "键盘中断时8259a芯片的1号中断，中断向量为0x21。当键盘按键时，需要处理0x21中断，获取按下的扫描码，根据不同的扫描码进行处理，例如字符就打印。目前使用键盘都是第二套扫描码，但是内部会被8042转换为第一套扫描码，因此第一套不存在的扫描码就属于扩展码。"
date:        2024-02-08T16:55:52+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/75/37/sun_grass_dune_denmark_sky_north_north_sea-1357857.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-keyboard-driver"
categories:  [ "操作系统" ]
---

## 一、键盘中断
在前面介绍中断我们知道 8259a 芯片的0号中断时时钟中断，1号中断就是键盘中断，因此键盘中断的向量是 0x21。如下所示：
```cpp
#define KEYBOARD_DATA_PORT 0x60
#define KEYBOARD_CTRL_PORT 0x64

void keyboard_handler(int vector)
{
    assert(vector == 0x21);
    send_eoi(vector);                       // 发送中断处理完成信号
    u16 scancode = inb(KEYBOARD_DATA_PORT); // 从键盘读取按键信息扫描码
    LOGK("keyboard input 0x%x\n", scancode);
}

void keyboard_init()
{
    set_interrupt_handler(IRQ_KEYBOARD, keyboard_handler);
    set_interrupt_mask(IRQ_KEYBOARD, true);
}
```
键盘中断涉及两个端口，0x60是数据端口，0x64读取时是状态寄存器，写入时是控制寄存器。从0x60端口读取扫描码到cpu中，键盘按下和弹起都会产生一个扫描码。键盘初始化设置了键盘中断的处理函数，并开启了键盘中断。当在键盘上按下字母a时并松开，产生两个扫描码，分为打印 0x2a， 0xaa。


## 二、键盘驱动
键盘扫描码分为通码和断码。通码：按下按键产生的扫描码，断码：抬起按键产生的扫描码。键盘共有三套扫描码，目前我们使用的键盘都是第二套扫描码，但是键盘产生了第二套扫描码之后会被8042芯片转换为第一套扫描码，因此键盘中断获取到的是第一套扫描码。

### 2.1、8042控制器
8042控制器有两个端口。0x60是数据端口，0x64读取时是状态寄存器，写入时是控制寄存器。

状态寄存器：
* 0位，输出缓冲区状态：1 表示输出缓冲区满
* 1位，输入缓冲区状态：1 表示输入缓冲区满
* 2位，系统标志位：加电时置为 0，自检通过时置为 1
* 3位，命令/数据位：1 表示输入缓冲区的内容是命令，0 表示输入缓冲区的内容是数据
* 4位，1 表示键盘启用，0 表示键盘禁用
* 5位，1 表示发送超时 （存疑）
* 6位，1 表示接受超时 （存疑）
* 7位，奇偶校验出错

控制寄存器使用不上。

### 2.2、键盘驱动实现
如下为键盘驱动的实现：
```cpp
#define INV 0 // 不可见字符

#define CODE_PRINT_SCREEN_DOWN 0xB7

typedef enum
{
    KEY_NONE,
    KEY_ESC,
    KEY_1,
    ...
    // 以下为自定义按键，为和 keymap 索引匹配
    KEY_PRINT_SCREEN,
} KEY;

static char keymap[][4] = {
    /* 扫描码 未与 shift 组合  与 shift 组合 以及相关状态 */
    /* ---------------------------------- */
    /* 0x00 */ {INV, INV, false, false},   // NULL
    /* 0x01 */ {0x1b, 0x1b, false, false}, // ESC
    /* 0x02 */ {'1', '!', false, false},
    ....
};

static bool capslock_state; // 大写锁定
static bool scrlock_state;  // 滚动锁定
static bool numlock_state;  // 数字锁定
static bool extcode_state;  // 扩展码状态
#define ctrl_state (keymap[KEY_CTRL_L][2] || keymap[KEY_CTRL_L][3])   // CTRL 键状态
#define alt_state (keymap[KEY_ALT_L][2] || keymap[KEY_ALT_L][3])  // ALT 键状态
#define shift_state (keymap[KEY_SHIFT_L][2] || keymap[KEY_SHIFT_R][2])  // SHIFT 键状态
```
如上所示，通过一个枚举类型定义了键盘所有的扫描码。通过结构体keymap定义了扫描码对应的字符，其中数组的第三个元素判断shift是否被按下，第四元素代表扩展按键是否被按下。接下来就是键盘中断的处理函数，这里不在贴函数的代码实现，因为比较长。核心逻辑是根据按下的扫描码在控制台打印对应的字符。

这里介绍下扩展码，扩展码是第二套编码比第一套编码扩展出来的按键，例如按下右边的alt，会先触发一次扫描码E0，说明这是扩展码，然后再产生一个扫描码38说明是alt键被按下。例如alter的两个状态分别判断是左alt还是右alt。


## 三、控制键盘LED灯
当滚动锁定，大写锁定，和数字锁定时，键盘上对应的LED灯应置于亮灯状态。PS/2 键盘接受很多种类型的命令（cpu向键盘发命令），命令有一个字节，一些命令有数据，这些数据必须在命令字节发送之后再发送。键盘通过一个 ACK(0xFA) (表示命令已收到) 或者 Resend(0xFE) (表示前一个命令有错误)；在发送命令之间需要等待键盘缓冲区为空。

控制键盘LED灯就需要给键盘发命令。实现如下：
```cpp
#define KEYBOARD_CMD_LED 0xED // 设置 LED 状态
#define KEYBOARD_CMD_ACK 0xFA // ACKcpp

static void keyboard_wait()
{
    u8 state;
    do
    {
        state = inb(KEYBOARD_CTRL_PORT);
    } while (state & 0x02); // 读取键盘缓冲区，直到为空
}

static void keyboard_ack()
{
    u8 state;
    do
    {
        state = inb(KEYBOARD_DATA_PORT);
    } while (state != KEYBOARD_CMD_ACK);
}

static void set_leds()
{
    u8 leds = (capslock_state << 2) | (numlock_state << 1) | scrlock_state;
    keyboard_wait();
    // 设置 LED 命令
    outb(KEYBOARD_DATA_PORT, KEYBOARD_CMD_LED);
    keyboard_ack();

    keyboard_wait();
    // 设置 LED 灯状态
    outb(KEYBOARD_DATA_PORT, leds);
    keyboard_ack();
}
```
如上所示，只需遵循键盘协议，当按下指定键时，设置键盘对应LED亮即可。这里不必深究，在运行时可以使用bochs，bochs可以模拟键盘的led灯。


## 四、循环队列
当键盘在被按下，但是此时确没有程序去使用键盘产生的字符。此时就需要把这些字符暂存到队列中去。队列属于常见的数据结构。定义如下：
```cpp
typedef struct fifo_t
{
    char *buf;
    u32 length;
    u32 head;
    u32 tail;
} fifo_t;

void fifo_init(fifo_t *fifo, char *buf, u32 length);
bool fifo_full(fifo_t *fifo);
bool fifo_empty(fifo_t *fifo);
char fifo_get(fifo_t *fifo);
void fifo_put(fifo_t *fifo, char byte);
```
队列是先进先出的数据结构，我们这里实现的循环队列，如果队列满了就移除最早入队的那个元素。接下来就可以修改键盘中断处理函数，当键盘产生字符将字符压入队列。不再直接打印。我们这里看下键盘对循环队列的应用：
```cpp
// LOGK("keydown %c \n", ch);
fifo_put(&fifo, ch);
if (waiter != NULL)
{
    task_unblock(waiter);
    waiter = NULL;
}
// ----------------------------

u32 keyboard_read(char *buf, u32 count)
{
    lock_acquire(&lock);
    int nr = 0;
    while (nr < count)
    {
        while (fifo_empty(&fifo))
        {
            waiter = running_task();  // 如果队列没有数据，就阻塞进行等待。
            task_block(waiter, NULL, TASK_WAITING);
        }
        buf[nr++] = fifo_get(&fifo);
    }
    lock_release(&lock);
    return count;
}
```
先看下键盘中断的修改。键盘每触发一次中断直接将字符写入队列。并且如果此时有任务在等待获取键盘输入，就将它唤醒。相当于通知它有键盘输入进来了。还实现了一个keyboard_read函数用于从队列读取字符。这个函数是不支持并发的，队列属于共享资源读取时需要上锁。


## 五、键盘读取测试
我们修改init任务，让它读取键盘。代码如下：
```cpp
void init_thread()
{
    set_interrupt_state(true);
    u32 counter = 0;

    char ch;
    while (true)
    {
        bool intr = interrupt_disable();
        keyboard_read(&ch, 1);
        printk("%c", ch);

        set_interrupt_state(intr);
    }
}
```
如上所示，当内核启动进入init线程后，因为keyboard_read函数的执行需要保证不能被中断，因此init进入后先做了中断关闭。我们来梳理下运行流程。
* init进程启动，关闭中断进入读取键盘输入
* 此时键盘缓冲区没有数据，阻塞进行等待。
* 当我们敲击键盘输入数据后，数据进入队列并环境init线程
* init线程去队列读取输入的输入并打印到控制台。

此时你是否好奇，既然init线程关了中断，那为何键盘中断可以执行。这是因为当缓冲区没数据时，keyboard_read函数中将init线程阻塞，此时切换到了idle线程，idle线程开启了中断等待键盘中断的到来。键盘中断到来并执行后唤醒了init线程，init线程继续读取数据。
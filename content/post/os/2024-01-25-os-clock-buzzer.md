---
title:       自制操作系统 - 计数器、时钟与蜂鸣器
subtitle:    "计数器、时钟与蜂鸣器"
description: "计算机硬件中使用CMOS实时时钟（RTC）数据, 它使用电池供电，维持系统时间和日期。通过 I/O 端口读写CMOS寄存器，用来获取CMOS中的时间和实时时钟信息，并触发周期性中断或时钟中断。"
excerpt:     "计算机硬件中使用CMOS实时时钟（RTC）数据, 它使用电池供电，维持系统时间和日期。通过 I/O 端口读写CMOS寄存器，用来获取CMOS中的时间和实时时钟信息，并触发周期性中断或时钟中断。"
date:        2024-01-25T18:19:33+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/83/d5/girl_beauty_model_pretty_outside_female_bella_young-665056.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-clock-buzzer"
categories:  [ "操作系统" ]
---

## 一、计数器
常用的可编程定时计数是8253。在 8253 内部有 3 个独立的计数器，分别是计数器 0 ~ 2，端口号分别为 0x40 ~ 0x42；每个计数器完全相同，都是 16 位大小，相互独立。8253 计数器是个减法计数器，从初值寄存器中得到初值，然后载入计数器中，然后随着时钟变化递减。三个计数器的作用如下：
* 计数器 0，端口号 0x40，用于产生时钟信号
* 计数器 1，端口号 0x41，用于 DRAM 的定时刷新控制；
* 计数器 2，端口号 0x42，用于内部扬声器发出不同音调的声音，原理是给扬声器输送某频率的方波；

计数器 0 用于产生时钟中断，就是连接在 IRQ0 引脚上的时钟，也就是控制计数器 0 可以控制时钟发生的频率，以改变时间片的间隔。计数器依赖振荡器提供脉冲作为输入，振荡器的频率大概是 1193182 Hz，假设希望中断发生的频率为 100Hz（10ms），那么计数器初值寄存器的值为 11931。
```cpp
#define PIT_CHAN0_REG 0X40   // 0号计数器端口
#define PIT_CHAN2_REG 0X42   // 2号计数器端口
#define PIT_CTRL_REG 0X43    // 计数器的控制字寄存器端口
#define HZ 100
#define OSCILLATOR 1193182
#define CLOCK_COUNTER (OSCILLATOR / HZ)   // 震荡这么多次发生中断，其实并不是准备的100hz, 因为不能被整除但是这不影响系统调度
#define JIFFY (1000 / HZ)   // 时间片 10ms

// 时间片计数器
u32 volatile jiffies = 0;
u32 jiffy = JIFFY;

void clock_handler(int vector)    // 时钟中断的处理函数
{
    assert(vector == 0x20);
    send_eoi(vector);
    jiffies++;
    DEBUGK("clock jiffies %d ...\n", jiffies);
}

void pit_init()   // 初始化，这块不在详细说明，不同的位指定了使用哪个计数器和哪种模式无需关注
{
    outb(PIT_CTRL_REG, 0b00110100);
    outb(PIT_CHAN0_REG, CLOCK_COUNTER & 0xff);
    outb(PIT_CHAN0_REG, (CLOCK_COUNTER >> 8) & 0xff);
}

void clock_init()
{
    pit_init();
    set_interrupt_handler(IRQ_CLOCK, clock_handler);  // 设置时钟中断的中断函数
    set_interrupt_mask(IRQ_CLOCK, true);           // 打开外中断控制器的时钟中断开关
}
```
如上：我们定义了时间片为10ms，并初始化了计数器和时间中断的处理函数。运行这块代码就会每10ms打印一行内容。

## 二、蜂鸣器
扬声器有两种状态，输入和输出，状态可以通过键盘控制器中的端口号 0x61 设置，该寄存器结构如下：
* 0	计数器 2 门有效
* 1	扬声器数据有效
* 2	通道校验有效
* 3	奇偶校验有效
* 4	保留
* 5	保留
* 6	通道错误
* 7	奇偶错误

需要将 0 ~ 1 位置为 1，然后计数器 2 设置成 方波模式，就可以播放方波的声音。这里不再贴代码实现，比较简单，打开键盘控制器，并将0x61端口寄存器的0~1位置1。在输出程序配置如果有\a就启动蜂鸣器就可以了。


## 三、时间
PC机的时间是由CMOS提供的，它使用电池供电。实际上是 64 或 128 字节 RAM 内存块，是系统时钟芯片的一部分。该 64 字节的 CMOS 原先在 IBM PC-XT 机器上用于保存时钟和日期信息，存放的格式是 BCD 码。由于这些信息仅用去 14 字节，剩余的字节就用来存放一些系统配置数据。

CMOS 的地址空间是在基本地址空间之外的。因此其中不包括可执行的代码。它需要使用在端口 0x70,0x71 使用 in 和 out 指令来访问。为了读取指定偏移位置的字节，首先需要使用 out 向端口 0x70 发送指定字节的偏移值，然后使用 in 指令从 0x71 端口读取指定的字节信息。不过在选择字节（寄存器）时最好屏蔽到 NMI 中断。不同的偏移值存储的不同信息，例如偏移0x00是当前的秒值， 0x02	是当前分钟。还有闹钟的秒值，分钟值等。
```cpp
typedef struct tm
{
    int tm_sec;   // 秒数 [0，59]
    int tm_min;   // 分钟数 [0，59]
    int tm_hour;  // 小时数 [0，59]
    int tm_mday;  // 1 个月的天数 [0，31]
    int tm_mon;   // 1 年中月份 [0，11]
    int tm_year;  // 从 1900 年开始的年数
    int tm_wday;  // 1 星期中的某天 [0，6] (星期天 =0)
    int tm_yday;  // 1 年中的某天 [0，365]
    int tm_isdst; // 夏令时标志
} tm;    // 定义时间结构

// 下面是 CMOS 信息的寄存器索引
#define CMOS_SECOND 0x00  // (0 ~ 59)
...   // 省略了一些定义

#define MINUTE 60          // 每分钟的秒数
#define HOUR (60 * MINUTE) // 每小时的秒数
#define DAY (24 * HOUR)    // 每天的秒数
#define YEAR (365 * DAY)   // 每年的秒数，以 365 天算

// 每个月开始时的已经过去天数
static int month[13] = {
    0, // 这里占位，没有 0 月，从 1 月开始
    0,
    (31),
    (31 + 29),
    (31 + 29 + 31),
    (31 + 29 + 31 + 30),
    (31 + 29 + 31 + 30 + 31),
    (31 + 29 + 31 + 30 + 31 + 30),
    (31 + 29 + 31 + 30 + 31 + 30 + 31),
    (31 + 29 + 31 + 30 + 31 + 30 + 31 + 31),
    (31 + 29 + 31 + 30 + 31 + 30 + 31 + 31 + 30),
    (31 + 29 + 31 + 30 + 31 + 30 + 31 + 31 + 30 + 31),
    (31 + 29 + 31 + 30 + 31 + 30 + 31 + 31 + 30 + 31 + 30)};

time_t startup_time;   // 定义开机启动的时间
int century;

// 这里生成的时间可能和 UTC 时间有出入
// 与系统具体时区相关，不过也不要紧，顶多差几个小时
time_t mktime(tm *time)
{
    time_t res;
    int year; // 1970 年开始的年数
    // 下面从 1900 年开始的年数计算，读取出来的年份是最后两位，这也是千年虫问题的根本原因
    if (time->tm_year >= 70)   // 这里未考虑2070年之后
        year = time->tm_year - 70;
    else
        year = time->tm_year - 70 + 100;
    res = YEAR * year;       // 这些年经过的秒数时间
    res += DAY * ((year + 1) / 4);    // 已经过去的闰年，每个加 1 天
    res += month[time->tm_mon] * DAY;  // 已经过完的月份的时间
    if (time->tm_mon > 2 && ((year + 2) % 4))  // 如果 2 月已经过了，并且当前不是闰年，那么减去一天
        res -= DAY;
    res += DAY * (time->tm_mday - 1);    // 这个月已经过去的天
    res += HOUR * time->tm_hour;        // 今天过去的小时
    res += MINUTE * time->tm_min;       // 这个小时过去的分钟
    res += time->tm_sec;                // 这个分钟过去的秒
    return res;
}

int get_yday(tm *time)
{
    int res = month[time->tm_mon]; // 已经过去的月的天数
    res += time->tm_mday;          // 这个月过去的天数

    int year;
    if (time->tm_year >= 70)
        year = time->tm_year - 70;
    else
        year = time->tm_year - 70 + 100;

    // 如果不是闰年，并且 2 月已经过去了，则减去一天
    // 注：1972 年是闰年，这样算不太精确，忽略了 100 年的平年
    if ((year + 2) % 4 && time->tm_mon > 2)
    {
        res -= 1;
    }

    return res;
}

u8 cmos_read(u8 addr)    // 从CMOS读取时间信息
{
    outb(CMOS_ADDR, CMOS_NMI | addr);
    return inb(CMOS_DATA);
};

void time_read_bcd(tm *time)
{
    // CMOS 的访问速度很慢。为了减小时间误差，在读取了下面循环中所有数值后，
    // 若此时 CMOS 中秒值发生了变化，那么就重新读取所有值。
    // 这样内核就能把与 CMOS 的时间误差控制在 1 秒之内。
    do
    {
        time->tm_sec = cmos_read(CMOS_SECOND);
        time->tm_min = cmos_read(CMOS_MINUTE);
        time->tm_hour = cmos_read(CMOS_HOUR);
        time->tm_wday = cmos_read(CMOS_WEEKDAY);
        time->tm_mday = cmos_read(CMOS_DAY);
        time->tm_mon = cmos_read(CMOS_MONTH);
        time->tm_year = cmos_read(CMOS_YEAR);
        century = cmos_read(CMOS_CENTURY);
    } while (time->tm_sec != cmos_read(CMOS_SECOND));
}

void time_read(tm *time)
{
    time_read_bcd(time);
    time->tm_sec = bcd_to_bin(time->tm_sec);
    time->tm_min = bcd_to_bin(time->tm_min);
    time->tm_hour = bcd_to_bin(time->tm_hour);
    time->tm_wday = bcd_to_bin(time->tm_wday);
    time->tm_mday = bcd_to_bin(time->tm_mday);
    time->tm_mon = bcd_to_bin(time->tm_mon);
    time->tm_year = bcd_to_bin(time->tm_year);
    time->tm_yday = get_yday(time);
    time->tm_isdst = -1;
    century = bcd_to_bin(century);
}

void time_init()
{
    tm time;
    time_read(&time);
    startup_time = mktime(&time);
    LOGK("startup time: %d%d-%02d-%02d %02d:%02d:%02d\n",
         century,
         time.tm_year,
         time.tm_mon,
         time.tm_mday,
         time.tm_hour,
         time.tm_min,
         time.tm_sec);
}
```	

## 四、实时时钟中断
CMOS提供的另一个功能就是实时时钟，实时中断可以提供闹钟功能和定时任务功能。通过时钟中断也可以实现，如每次时钟中断都去检测下有没有到任务的定时时间，但是这样对性能损耗较高。

实时时钟是CMOS提供的中断，实时时钟中断位于中断控制器的第八个IRQ接口。可以提供周期性中断和闹钟中断等，可以控制周期性中断的中断频率。我们这里仅贴出周期性中断的实现，闹钟中断其实基本类似，根据CMOS的手册调整相关的寄存器值即可。
```cpp
// 实时时钟中断处理函数
void rtc_handler(int vector)
{
    assert(vector == 0x28);   // 实时时钟中断向量号
    send_eoi(vector);         // 向中断控制器发送中断处理完成的信号
    cmos_read(CMOS_C);        // 读 CMOS 寄存器 C，允许 CMOS 继续产生中断
    LOGK("rtc handler %d...\n", counter++);
}

void rtc_init()
{
    u8 prev;
    cmos_write(CMOS_B, 0b01000010); // 打开周期中断
    cmos_read(CMOS_C); // 读 C 寄存器，以允许 CMOS 中断
    outb(CMOS_A, (inb(CMOS_A) & 0xf) | 0b1110);  // 1110代表中断周期为250ms
    set_interrupt_handler(IRQ_RTC, rtc_handler);  
    set_interrupt_mask(IRQ_RTC, true);
    set_interrupt_mask(IRQ_CASCADE, true);
}
```
如上所示，就打开CMOS的周期中断，每250ms触发一次。
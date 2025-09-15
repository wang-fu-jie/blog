---
title:       自制操作系统 - 任务状态
subtitle:    "任务状态"
description: "任务处理就绪态和运行态之外，还有阻塞态和休眠态。本文中将实现任务进入阻塞态和休眠态的实现，详细介绍进入阻塞和解除阻塞已经休眠和唤醒。进入阻塞和休眠的任务分别使用双向链表进行管理。"
excerpt:     "任务处理就绪态和运行态之外，还有阻塞态和休眠态。本文中将实现任务进入阻塞态和休眠态的实现，详细介绍进入阻塞和解除阻塞已经休眠和唤醒。进入阻塞和休眠的任务分别使用双向链表进行管理。"
date:        2024-02-02T18:30:38+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/9e/f7/architect_graphics_abstract_building_secret_success_quality-1057487.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-task-state"
categories:  [ "操作系统" ]
---

## 一、双向链表
任务的实现需要依赖双向链表这个数据结构，链表相信大家都很熟悉，不进行过多介绍，这里说明一下内核使用的链表和普通链表的不同之处。先看下基本定义：
```cpp
#define element_offset(type, member) (u32)(&((type *)0)->member)
#define element_entry(type, member, ptr) (type *)((u32)ptr - element_offset(type, member))

typedef struct list_node_t   // 链表结点
{
    struct list_node_t *prev; // 下一个结点
    struct list_node_t *next; // 前一个结点
} list_node_t;

typedef struct list_t      // 链表
{
    list_node_t head; // 头结点
    list_node_t tail; // 尾结点
} list_t;
```
注意这是的链表节点只有前后节点的两个指针，并没有数据。这是因为可能把一个数据块链接到不同的链表中。因此这是采用将node嵌入到数据区中，这样数据就可以链接到多个链表中。这里还有两个宏，它的作用是通过node的指针找到链表的指针。

其中 element_offset 计算结构体类型 type 中，成员 member 相对于结构体起始地址的字节偏移量。element_entry 通过一个指向结构体成员 (member) 的指针 ptr，求出这个成员所在的那个完整结构体的起始地址

## 二、任务的阻塞和就绪
为了保存阻塞的任务，在task的结构体中加入一个链表。如果任务被阻塞了就将任务压入链表中。我们这里为了测试，将0号系统调用改为阻塞和解除阻塞。如代码：调用test()函数即可调用0号系统调用。
```cpp
task_t *task = NULL;
static u32 sys_test()
{
    if (!task)
    {
        task = running_task();                  // 如果没有传入任务，调用0号系统调用就阻塞任务
        task_block(task, NULL, TASK_BLOCKED);
    }
    else
    {
        task_unblock(task);     // 如果传入了一个任务，就解除任务的阻塞态
        task = NULL;
    }
    return 255;
}

static list_t block_list;            // 任务默认阻塞链表
// 任务阻塞
void task_block(task_t *task, list_t *blist, task_state_t state)
{
    if (blist == NULL)     // 如果没有指定阻塞链表就是系统全局链表
    {
        blist = &block_list;
    }
    list_push(blist, &task->node);  // 将任务压入到链表中
    task->state = state;
    task_t *current = running_task();
    if (current == task)  // 如果当前任务被阻塞，则进行任务调度，切换到下一个就绪态任务
    {
        schedule();
    }
}

// 解除任务阻塞
void task_unblock(task_t *task)
{
    list_remove(&task->node);   // 将任务从链表中移除
    task->state = TASK_READY;   // 将任务状态改为就绪态。
}

u32 test()
{
    return _syscall0(SYS_NR_TEST);
}
```
这里为了节省篇幅，省略了一些断言。如上实现了任务的阻塞和解除阻塞，但是仍然存在一个问题，那就是所有的任务都被阻塞了，调度函数将会报错。我们在第三节解决这个问题。

我们先梳理下三个进程的执行流程，进行 A启动->切换B->解除A->切换A->解除B->切换C->解除A。 因此以此打印ABBAACC。依次这样循环打印AABBCC。但是如果删除c进程，当只有两个进程时，A被阻塞，B在执行中就找不到了就绪态的任务，因此task_search就会触发异常。

## 三、基础任务
在上一节中，如果所有任务都被阻塞了程序就会报错。因此当所有任务被阻塞时需要切换到一个空闲进程上。这涉及到了linux的两个基础进程，一个是空闲进程idle，进程id为0。一个是初始化进程init，进程号为1。

### 3.1、空闲进程
空闲进程永远不会被阻塞，它的主要工作有 开中断， 中断开之后关闭 CPU，等待外中断。外中断进来后通过 yield 调度执行其他程序。idle线程实现如下：
```cpp
void idle_thread()
{
    set_interrupt_state(true);
    u32 counter = 0;
    while (true)
    {
        LOGK("idle task.... %d\n", counter++);
        asm volatile(
            "sti\n" // 开中断
            "hlt\n" // 关闭 CPU，进入暂停状态，等待外中断的到来
        );
        yield(); // 放弃执行权，调度执行其他任务
    }
}
```

### 3.2、初始化进程
初始化进程的工作是用于执行初始化操作，进入到用户态等工作，进程 id 为 1。初始化线程的代码如下：
```cpp
void init_thread()
{
    set_interrupt_state(true);
    while (true)
    {
        LOGK("init task....\n");
        // test();             
    }
}
```

### 3.3、任务初始化
现在有了空闲进程和初始化进程，就不再需要之前测试的ABC进程了。需要基础任务初始化，如下：
```cpp
void task_init()
{
    list_init(&block_list);
    task_setup();
    idle_task = task_create(idle_thread, "idle", 1, KERNEL_USER);
    task_create(init_thread, "init", 5, NORMAL_USER);
}
```
如上所示，空闲进程属于内核进程，任务优先级较低为1。init进程为用户进程，任务优先级为5。这里会在idle和init进程间来回切换，但是init执行频率更高，因为它的优先级更高。如果给init进程增加了阻塞，那就会一直运行idle进程。


## 四、任务睡眠与唤醒
任务睡眠与唤醒核心涉及两个函数task_sleep 和 task_wakeup。在系统调用里需要加一个sleep的系统调用。sleep调用时需要传入一个参数，
```cpp
static _inline u32 _syscall1(u32 nr, u32 arg)
{
    u32 ret;
    asm volatile(
        "int $0x80\n"
        : "=a"(ret)
        : "a"(nr), "b"(arg));
    return ret;
}

void sleep(u32 ms)
{
    _syscall1(SYS_NR_SLEEP, ms);
}
```
如上所示，系统调用需要增加_syscall1的函数，传入的一个参数放到ebx寄存器中。这样当执行sleep函数时就会调用0x80中断。因此在中断初始化时也需要增加中断处理函数：
```cpp
syscall_table[SYS_NR_SLEEP] = task_sleep;   // 中断初始化加上这样。
```
接下来就是当触发睡眠中断时，中断函数的实现。如下：
```cpp
static list_t sleep_list;            // 任务睡眠链表
void task_sleep(u32 ms)
{
    assert(!get_interrupt_state()); // 不可中断，当前处于系统调用阶段，不可再被其他中断
    u32 ticks = ms / jiffy;        // 需要睡眠的时间片
    ticks = ticks > 0 ? ticks : 1; // 至少休眠一个时间片
    task_t *current = running_task();   // 记录目标全局时间片，在那个时刻需要唤醒任务
    current->ticks = jiffies + ticks;  // 这里将任务剩余可执行时间片设置的会很大

    // 从睡眠链表找到第一个比当前任务唤醒时间点更晚的任务，进行插入排序
    list_t *list = &sleep_list;
    list_node_t *anchor = &list->tail;

    for (list_node_t *ptr = list->head.next; ptr != &list->tail; ptr = ptr->next)
    {
        task_t *task = element_entry(task_t, node, ptr);
        if (task->ticks > current->ticks)
        {
            anchor = ptr;
            break;
        }
    }
    assert(current->node.next == NULL);
    assert(current->node.prev == NULL);
    list_insert_before(anchor, &current->node);   // 插入链表
    current->state = TASK_SLEEPING;  // 修改阻塞状态是睡眠
    schedule();   // 调度执行其他任务
} 

void task_wakeup()
{
    assert(!get_interrupt_state()); // 不可中断
    // 从睡眠链表中找到 ticks 小于等于 jiffies 的任务，恢复执行
    list_t *list = &sleep_list;
    for (list_node_t *ptr = list->head.next; ptr != &list->tail;)
    {
        task_t *task = element_entry(task_t, node, ptr);
        if (task->ticks > jiffies)
        {
            break;
        }
        // unblock 会将指针清空
        ptr = ptr->next;

        task->ticks = 0;   // 设置剩余时间片为0
        task_unblock(task);  // unblokc会将任务改为就绪态。并从链表中移除
    }
}
```
如上所示，实现了任务的睡眠与唤醒。但是还需要修改时钟中断，在每次时钟中断时都要去触发一次唤醒睡眠结束的任务。

## 五、睡眠唤醒测试
我们这里在init线程上加一个sleep(500)，再创建一个test线程加sleep(709)。
```cpp

void task_init()
{
    list_init(&block_list);
    list_init(&sleep_list);
    task_setup();
    idle_task = task_create(idle_thread, "idle", 1, KERNEL_USER);
    task_create(init_thread, "init", 5, NORMAL_USER);
    task_create(test_thread, "test", 5, KERNEL_USER);
}
```
运行结果就是这两个线程交替执行，但是打印的速度明显变慢，因为我们加了sleep。当两个线程都在睡眠时，就会运行idle进程。

我们这里梳理一下执行流程：

1. 初始化init进程和测试进程，init进程执行一次打印，进入sleep
2. sleep函数触发0x80中断进行系统调用，触发task_sleep函数的执行，将init进程休眠，去调度其他进程
3. 调度到了test进程也打印一次，进入休眠。此时两个进程都开始休眠，运行idle线程。
4. 每次时钟中断都会尝试进程任务唤醒，如果达到睡眠结束时间就唤醒任务。idle交出执行权。如此往复。

最后说明一下：jiffies是全局系统时间片。触发睡眠时将任务的可执行时间片设置为当前全局时间片上加睡眠时间。因此当全局时间片增长达到这个设置的时间后，就可以唤醒了。唤醒时剩余时间片被设置为0，进入就绪态，等待下一次调度，把可用时间片再次置为优先级相等的数字。


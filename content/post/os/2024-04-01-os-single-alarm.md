---
title:       自制操作系统 - 信号与闹钟
subtitle:    "信号与闹钟"
description: "在 UNIX 系统中，信号是一种 软件中断 处理机制。信号机制提供了一种处理异步事件的方法。例如执行 CTRL+C 组合键会产生一个 SIGINT 信号来终止一个程序的执行。通过信号还可以实现闹钟功能，指定多长时间后去执行某个函数。"
excerpt:     "在 UNIX 系统中，信号是一种 软件中断 处理机制。信号机制提供了一种处理异步事件的方法。例如执行 CTRL+C 组合键会产生一个 SIGINT 信号来终止一个程序的执行。通过信号还可以实现闹钟功能，指定多长时间后去执行某个函数。"
date:        2024-04-01T11:27:29+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/b3/d2/niagara_falls_niagara_water_waterfall_night_lighting_border_new_york-1157765.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-single-alarm"
categories:  [ "操作系统" ]
---

## 一、信号
在 UNIX 系统中，信号是一种 软件中断 处理机制。信号机制提供了一种处理异步事件的方法。例如：
1. 用户在终端键盘上键入 CTRL+C 组合键来终止一个程序的执行。该操作就会产生一个 SIGINT (SIGnal INTerrupt) 信号，并被发送到当前前台执行的进程中；
2. 当进程设置了一个报警时钟，到期时，系统就会向进程发送一个 SIGALRM 信号；
3. 当发生硬件异常时，系统也会向正在执行的进程发送相应的信号；
4. 另外，一个进程也可以向另一个进程发送信号，使用 kill() 函数向同组的子进程发送终止执行信号。

通常使用一个 32 位无符号长整数，中的比特位表示各种不同信号（一种位图结构），因此最多可表示 32 个不同的信号。

### 1.1、信号处理方式
对于进程来说，当收到一个信号时，可以由三种不同的处理或操作方式。
1. 忽略该信号： 大多数信号都可以被进程忽略。但有两个信号忽略不掉：SIGKILL 和 SIGSTOP。不能被忽略掉的原因是能为超级用户提供一个确定的方法来终止或停止指定的任何进程。另外，若忽略掉某些硬件异常而产生的信号（如除0错误），则进程的行为或状态就可能变得不可知了。
2. 捕获该信号： 为了进行该操作，我们必须首先告诉内核在指定的信号发生时调用我们自定义的信号处理函数。在该处理函数中，我们可以做任何操作，当然也可以什么不做，起到忽略该信号的同样作用。自定义信号处理函数来捕获信号的一个例子是：如果我们在程序执行过程中创建了一些临时文件，那么我们就可以定义一个函数来捕获 SIGTERM（终止执行）信号，并在该函数中做一些清理临时文件的工作。SIGTERM 信号是 kill 命令发送的默认信号。
3. 执行默认操作： 内核为每种信号都提供一种默认操作。通常这些默认操作就是终止进程的执行。


## 二、信号实现
信号的实现首先需要定义信号编号、信号掩码、范围宏定义、信号标志位、以及默认的信号忽略和处理程序等，代码如下:
```cpp
enum SIGNAL
{
    SIGHUP = 1,   // 挂断控制终端或进程
    SIGINT,       // 来自键盘的中断
    SIGQUIT,      // 来自键盘的退出
    SIGILL,       // 非法指令
    SIGTRAP,      // 跟踪断点
    SIGABRT,      // 异常结束
    SIGIOT = 6,   // 异常结束
    SIGUNUSED,    // 没有使用
    SIGFPE,       // 协处理器出错
    SIGKILL = 9,  // 强迫进程终止
    SIGUSR1,      // 用户信号 1，进程可使用
    SIGSEGV,      // 无效内存引用
    SIGUSR2,      // 用户信号 2，进程可使用
    SIGPIPE,      // 管道写出错，无读者
    SIGALRM,      // 实时定时器报警
    SIGTERM = 15, // 进程终止
    SIGSTKFLT,    // 栈出错（协处理器）
    SIGCHLD,      // 子进程停止或被终止
    SIGCONT,      // 恢复进程继续执行
    SIGSTOP,      // 停止进程的执行
    SIGTSTP,      // tty 发出停止进程，可忽略
    SIGTTIN,      // 后台进程请求输入
    SIGTTOU = 22, // 后台进程请求输出
};

#define MINSIG 1        // 最小信号编号
#define MAXSIG 31       // 最大信号编号
#define SIGMASK(sig) (1 << (sig - 1))  // 把信号编号转成一个位掩码


#define SIG_NOMASK 0x40000000   // 不阻止在指定的信号处理程序中再收到该信号
#define SIG_ONESHOT 0x80000000   // 信号句柄一旦被调用过就恢复到默认处理句柄

#define SIG_DFL ((void (*)(int))0) // 默认的信号处理程序（信号句柄）
#define SIG_IGN ((void (*)(int))1) // 忽略信号的处理程序

typedef u32 sigset_t;    // 表示一个信号集合（signal set），用 32 个比特表示哪些信号被阻塞
typedef struct sigaction_t  // 信号处理结构
{
    void (*handler)(int); // 信号处理函数
    sigset_t mask;        // 信号屏蔽码
    u32 flags;
    void (*restorer)(void); // 恢复函数指针
} sigaction_t;
```
信号掩码 SIGMASK 的作用是，把信号编号转成一个位掩码。比如：SIGMASK(2) = 1 << 1 = 0x00000002，这样你就可以用一个 u32 整数的 32 位来表示当前阻塞的信号集合。从代码中也可以看到使用 u32 表示信号集合，因此最多支持32种信号。定义完相关结构后，在实现信号处理函数之前，需要先给任务结构体添加元素进程信号位图、信号屏蔽位图和信号处理函数。因为每个进程确实都有自己独立的一套信号处理和信号屏蔽机制。如下：
```cpp
typedef struct task_t
{
    // ...
    u32 signal;                         // 进程信号位图
    u32 blocked;                        // 进程信号屏蔽位图
    sigaction_t actions[MAXSIG];        // 信号处理函数
} task_t;
```
接下来就可以实现信号的相关操作了，如下：
```cpp
typedef struct signal_frame_t
{
    u32 restorer; // 恢复函数
    u32 sig;      // 信号
    u32 blocked;  // 屏蔽位图
    u32 eax;        // 保存调用时的寄存器，用于恢复执行信号之前的代码
    u32 ecx;
    u32 edx;
    u32 eflags;
    u32 eip;
} signal_frame_t;

int sys_sgetmask()    // 获取信号屏蔽位图
{
    task_t *task = running_task();
    return task->blocked;
}

int sys_ssetmask(int newmask)    // 设置信号屏蔽位图
{ 
    task_t *task = running_task();
    int old = task->blocked;          // 获取当前信号屏蔽位图
    task->blocked = newmask & ~SIGMASK(SIGKILL);   // 添加新的屏蔽信号
    return old;      // 返回旧的信号屏蔽位图
}

int sys_signal(int sig, int handler, int restorer)    // 注册信号处理函数
{
    if (sig < MINSIG || sig > MAXSIG || sig == SIGKILL)  // 如果注册kill的处理函数则直接返回EOF，
        return EOF;
    task_t *task = running_task();          // 获取当前进程
    sigaction_t *ptr = &task->actions[sig - 1];  // 获取信号处理结构
    ptr->mask = 0;                              // 设置信号屏蔽码
    ptr->handler = (void (*)(int))handler;   // 设置信号处理函数
    ptr->flags = SIG_ONESHOT | SIG_NOMASK;   // 设置行为控制标志（一次性/允许重入）
    ptr->restorer = (void (*)())restorer;   // 设置函数恢复指针
    return handler;              // 返回信号处理函数
}

int sys_sigaction(int sig, sigaction_t *action, sigaction_t *oldaction)   // 注册信号处理函数，更高级的一种方式
{
    if (sig < MINSIG || sig > MAXSIG || sig == SIGKILL)
        return EOF;
    task_t *task = running_task();
    sigaction_t *ptr = &task->actions[sig - 1];
    if (oldaction)
        *oldaction = *ptr;

    *ptr = *action;
    if (ptr->flags & SIG_NOMASK)
        ptr->mask = 0;
    else
        ptr->mask |= SIGMASK(sig);
    return 0;
}

int sys_kill(pid_t pid, int sig)         // 发送kill信号
{
    if (sig < MINSIG || sig > MAXSIG)
        return EOF;
    task_t *task = get_task(pid);   // 获取进程任务结构
    if (!task)
        return EOF;
    if (task->uid == KERNEL_USER)   // 判断是否是内核用户
        return EOF;
    if (task->pid == 1)
        return EOF;

    LOGK("kill task %s pid %d signal %d\n", task->name, pid, sig);
    task->signal |= SIGMASK(sig);
    if (task->state == TASK_WAITING || task->state == TASK_SLEEPING)   // 如果进程状态为等待或者睡眠，就解除阻塞
    {
        task_unblock(task, -EINTR);
    }
    return 0;
}

void task_signal()        // 内核信号处理函数
{
    task_t *task = running_task();
    u32 map = task->signal & (~task->blocked);      // 获得任务可用信号位图
    if (!map)
        return;

    assert(task->uid);
    int sig = 1;
    for (; sig <= MAXSIG; sig++)
    {
        if (map & SIGMASK(sig))
        {
            task->signal &= (~SIGMASK(sig));    // 将此信号置空，表示已执行
            break;
        }
    }
    sigaction_t *action = &task->actions[sig - 1];     // 得到对应的信号处理结构
    if (action->handler == SIG_IGN)        // 忽略信号
        return;
    if (action->handler == SIG_DFL && sig == SIGCHLD)    // 子进程终止的默认处理方式
        return;
    if (action->handler == SIG_DFL)      // 默认信号处理方式，为退出
        task_exit(SIGMASK(sig));

    intr_frame_t *iframe = (intr_frame_t *)((u32)task + PAGE_SIZE - sizeof(intr_frame_t));   // 处理用户态栈，使得程序去执行信号处理函数，处理结束之后，调用 restorer 恢复执行之前的代码
    signal_frame_t *frame = (signal_frame_t *)(iframe->esp - sizeof(signal_frame_t));
    frame->eip = iframe->eip;         // 保存执行前的寄存器
    frame->eflags = iframe->eflags;
    frame->edx = iframe->edx;
    frame->ecx = iframe->ecx;
    frame->eax = iframe->eax;
    frame->blocked = EOF;        // 屏蔽所有信号
    if (action->flags & SIG_NOMASK)        // 不屏蔽在信号处理过程中，再次收到该信号
        frame->blocked = task->blocked;
    frame->sig = sig;               // 信号
    frame->restorer = (u32)action->restorer;    // 信号处理结束的恢复函数
    LOGK("old esp 0x%p\n", iframe->esp);
    iframe->esp = (u32)frame;
    LOGK("new esp 0x%p\n", iframe->esp);
    iframe->eip = (u32)action->handler;

    if (action->flags & SIG_ONESHOT)         // 如果设置了 ONESHOT，表示该信号只执行一次
        action->handler = SIG_DFL;
    task->blocked |= action->mask;      // 进程屏蔽码添加此信号
}
```
sys_kill、sys_signal、sys_sgetmask、sys_ssetmask、sys_sigaction 需要注册为系统调用，这样在用户态就可以进程信号的注册和通过kill命令发起信号了。信号处理是软中断，当进行信号处理时，需要保存当前寄存器，跳转去进行信号处理函数，当处理完成后，需要进行回复，如下为 恢复函数：
```asm
[bits 32]

extern ssetmask
section .text
global restorer

restorer:
    add esp, 4; sig
    call ssetmask
    add esp, 4; blocked
    ; 以下恢复调用方寄存器
    pop eax
    pop ecx
    pop edx
    popf
    ret
```
当我们发起信号时，信号应该是发送给前台的进程组，因此需要再tty设备的实现中来进行信号的发送，代码如下：
```cpp
int tty_intr()         // 向前台组进程发送 SIGINT 信号
{
    tty_t *tty = &typewriter;
    if (!tty->pgid)
    {
        return 0;
    }
    for (size_t i = 0; i < TASK_NR; i++)   // 遍历任务列表，如果任务的进程组等于tty设备的进程组，那就给这个进程发送信号。
    {
        task_t *task = task_table[i];
        if (!task)
            continue;
        if (task->pgid != tty->pgid)
            continue;
        kill(task->pid, SIGINT);
    }
    return 0;
}
```
最后进程也需要进行相关的调整，在进程退出，如果是会话首领退出那就需要给会话中所有进程发送退出信号并且释放tty设备，如果子进程退出需要通知父进程。
```cpp
static void task_kill_session(task_t *task)    // 如果进程是会话首领则向会话中所有进程发送信号 SIGHUP
{
    if (!task_leader(task))  // 如果不是会话首领，直接return
        return;

    for (size_t i = 0; i < TASK_NR; i++)
    {
        task_t *child = task_table[i];
        if (!child)
            continue;
        if (task == child || task->sid != child->sid)
            continue;
        child->signal |= SIGMASK(SIGHUP);
    }
}

static void task_free_tty(task_t *task)     // 释放 TTY 设备
{
    if (task_leader(task) && task->tty > 0)
    {
        device_t *device = device_get(task->tty);
        tty_t *tty = (tty_t *)device->ptr;
        tty->pgid = 0;
    }
}

static void task_tell_father(task_t *task)     // 子进程退出，通知父进程
{
    if (!task->ppid)
        return;
    for (size_t i = 0; i < TASK_NR; i++)
    {
        task_t *parent = task_table[i];
        if (!parent)
            continue;
        if (parent->pid != task->ppid)
            continue;
        parent->signal |= SIGMASK(SIGCHLD);
        return;
    }
    panic("No Parent found!!!");
}

void task_exit(int status)
{
    task_t *task = running_task();
    task_kill_session(task);
    task_tell_father(task);
    task_free_tty(task);
    // ...
}
```
到此为止，我们已经基本实现了信号功能。还需要在shell中还需要注册信号，每启动一个shell都要注册信号。我们这里只实现 ctrl+c 这个信号。
```cpp
static int signal_handler(int sig)
{
    // printf("pid %d signal %d happened\n", getpid(), sig);
    signal(SIGINT, (int)signal_handler);
    interrupt = true;
}

int main()
{
    signal(SIGINT, (int)signal_handler);  // 注册信号 CTRL + C
    // ...
}
```
如上，我们在shell中只注册了 CTRL + C 这个信号，注意这个函数 signal_handler，内部继续注册了自身，这是因为第一次 Ctrl+C 会触发 signal_handler，第二次 Ctrl+C 就会直接杀死进程（因为处理函数被还原为默认）。因此在处理函数内部重新注册自己，确保下一次信号仍然由它处理。这里的 interrupt 是全局变量，它的作用是当用户按下 Ctrl+C 时（即 SIGINT 到来）：内核中断当前进程, 调用你的信号处理函数signal_handler 把 interrupt 设置为 true。等信号处理函数返回，主程序循环检测到 interrupt == true 后，再安全地终止或清理资源。

最后一个需要实现的是kill命令，它的实现比较简单，直接进行kill系统调用即可。

## 三、信号处理流程调试
现在已经完成ctrl+c的信号处理了，当在shell中按下ctrl+c时，现在是就重新开启一行。当执行count命令时，中断一直打印内容，此时按下ctrl+c就会结束count命令。下面我们分别就这两种情况是如何运行的进行分析：
1. 运行系统，在tty处理ctrl+c出打断点，并且在终端发起ctrl+c。
2. 接下来会调用 tty_intr 向前台组进程发送 SIGINT 信号，此时tty的进程组首领为3， 即init进程。因为调试阶段我们的系统是在init中执行的。
3. 接下来会向init进程发送kill信号，kill系统调用将 task->single 设置为2，SIGINT 的编号就是 2， kill系统调用结束进行 return。
4. 系统调用结束会做中断退出，这里注意在中断退出时，会调用task_signal进行信号处理，这里是对handler.asm的修改，前面代码中没有体现。
5. 进入task_signal内部继续处理，如果当前任务有可用信号位图，则进行处理，如果没有函数直接结束返回，这样不影响其他的系统调用。
6. 当有信号需要处理事。task_signal内部就会注册信号处理函数并执行，执行完就会进入restore，这里最终会恢复到原来的进程中，我们在shell中设置了重新读取命令。因此这里终端就切换了一行重新等待命令的输入。

当有count在执行，我们按下ctrl+c 时，就走走到默认的信号处理方式，调用进程退出。这里我们大致的描述了信号的处理流程，具体的细节还需要进行代码调试。


## 四、闹钟
前面已经实现了信号，现在就可以利用信号 SIGALRM 来做下闹钟，使得程序可以在特定的时间之后执行某函数。alarm也是一个系统调用， 实现闹钟，首先需要再任务结构增加一个元素： struct timer_t *alarm 即闹钟定时器。如下为系统调用的实现
```cpp
extern int sys_kill();

static void task_alarm(timer_t *timer)  // 发送信号
{
    timer->task->alarm = NULL;
    sys_kill(timer->task->pid, SIGALRM);
}

int sys_alarm(int sec)    
{
    task_t *task = running_task();  
    if (task->alarm)    // 如果进程已经有闹钟了，就移除并重新设置一个。
    {
        timer_put(task->alarm);
    }
    task->alarm = timer_add(sec * 1000, task_alarm, 0);
}
```
如上所示，当通过系统调用 alarm 设置一个闹钟后，就会增加一个闹钟定时器。并设置这个定时器的处理函数为 task_alarm。 在task_alarm处理时，清楚闹钟并给进程发送 SIGALRM 信号。接下来是用户程序对闹钟的使用：
```cpp
static int signal_handler(int sig)
{
    printf("pid %d when %d signal %d happened \a\n", getpid(), time(), sig);
    signal(SIGALRM, (int)signal_handler);
    alarm(2);
}

int main(int argc, char const *argv[])
{
    signal(SIGALRM, (int)signal_handler);   // 给闹钟注册处理函数
    alarm(2);                               // 调用alarm系统调用
    while (true)
    {
        printf("pid %d when %d sleeping 1 second\n", getpid(), time());
        sleep(1000);
    }
    return 0;
}
```
如上程序在运行时，会注册闹钟处理函数，并定时2秒后执行。 因此主函数循环没一秒一次打印。当定时到时候后，定时器就会调用task_alarm发送SIGALRM信号，信号处理函数会执行一次打印，并重新注册再次设置闹钟。


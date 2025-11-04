---
title:       自制操作系统 - 任务会话与tty设备
subtitle:    "任务会话与tty设备"
description: "进程会话（session）**是 一组相关进程的集合。主要用于：管理 终端控制、实现 作业控制（job control）、组织进程之间的父子关系和权限。tty设备是一个抽象的终端设备，虽然目前我们的系统只支持一个终端，但是我们需要tty设备做信号处理。"
excerpt:     "进程会话（session）**是 一组相关进程的集合。主要用于：管理 终端控制、实现 作业控制（job control）、组织进程之间的父子关系和权限。tty设备是一个抽象的终端设备，虽然目前我们的系统只支持一个终端，但是我们需要tty设备做信号处理。"
date:        2024-03-29T16:48:56+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/a8/68/beach_sand_woman_ocean_coast-10524.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-task-session-tty"
categories:  [ "操作系统" ]
---

## 一、任务会话
进程会话（session）**是 一组相关进程的集合。主要用于：管理 终端控制、实现 作业控制（job control）、组织进程之间的父子关系和权限。一个会话可以拥有 唯一的控制终端。会话可以有一个控制 tty。一个会话中最多只能有一个进程组是前台进程组。Linux 中 jobs, fg, bg, CTRL + Z 可以用来控制进程。

一个会话包含多个进程组，一个进程组包含多个进程。组通常与管道联系在一起，多个由管道连接的进程属于一个组。会话通常与 shell 联系在一起，每一次打开 shell 产生一个会话。

### 1.1、相关系统调用
实现任务会话涉及四个系统调用，分别是setsid、setpgrp、setpgid、getpgrp。实现任务会话首先进程的结构需要添加两个元素，分别是进程组和进程会话：
```cpp
typedef struct task_t
{
    pid_t pid;                          // 任务 id
    pid_t ppid;                         // 父任务 id
    pid_t pgid;                         // 进程组
    pid_t sid;                          // 进程会话
    // ....
    u32 magic;                          // 内核魔数，用于检测栈溢出
} task_t;
```
在创建一个新进程时，pgid 和 sid 被初始化为0。

setsid 如果调用进程不是进程组首领，setsid() 函数将创建一个新会话。返回时，调用进程应该是这个新会话的会话首领，也是一个新进程组的进程组首领，并且应该没有控制终端，调用进程的进程组 ID 必须与调用进程的进程 ID 相等，调用进程必须是新进程组中唯一的进程，也是新会话中唯一的进程。

setpgrp 如果调用进程不是会话首领，setpgrp() 将调用进程的进程组 ID 设置为调用进程 ID，如果 setpgrp() 创建了一个新会话，此新会话没有控制终端。当调用进程是会话首领时，setpgrp() 函数不起作用。

setpgid() 函数加入一个现有的进程组，或者在调用进程的会话中创建一个新的进程组。会话领导者的进程组 ID 不能改变。进程号与 pid 匹配的进程的进程组 ID 将被设置为 pgid。特殊情况下，pid 为 0，表示调用进程应使用的进程号。如果 pgid 为 0，则表示指定进程应使用的进程 ID。

getpgrp 的作用比较简单，返回调用进程的进程组 ID。以下为代码实现：
```cpp
bool _inline task_leader(task_t *task)   // 设置指定进程为会话首领
{
    return task->sid == task->pid;
}

int sys_setpgid(int pid, int pgid)   // 给指定进程加入现有进程组
    task_t *current = running_task();   // 获取当前进程
    if (!pid)                       // 如果没有指定被设置的进程，那就是给当前进程设置进程组
        pid = current->pid;
    if (!pgid)                      // 如果没有指定进程组id，那就是设置当前进程为进程组id
        pgid = current->pid;

    for (int i = 0; i < TASK_NR; i++)   // 遍历任务列表，找到pid的任务结构
    {
        task_t *task = task_table[i];
        if (!task)            // 如果没有任务则跳过
            continue;
        if (task->pid != pid)   // 如果是当前进程，则跳过
            continue;
        if (task_leader(task))  // 如果进程已有会话首领了  返回错误，没有操作许可
            return -EPERM;
        if (task->sid != current->sid)  // 如果进程的会话首领不等于当前进程的会话首领，返回错误，没有操作许可
            return -EPERM;
        task->pgid = pgid;   // 设置进程的进程组为指定进程组id
        return EOK;
    }
    return -ESRCH;    // 返回错误，指定的进程不存在
}

int sys_getpgrp()     //  返回调用进程的进程组 ID
{
    task_t *task = running_task();   
    return task->pgid;
}

int sys_setsid()    // 设置会话id
{
    task_t *task = running_task();   // 获取当前进程
    if (task_leader(task))  // 如果当前进程是会话首领，返回错误，没有操作许可。
        return -EPERM;
    task->sid = task->pgid = task->pid;   // 设置进程会话id等于进程组id等于进程id， 并返回。
    return task->sid;
}
```
会话通常与 shell 联系在一起，每一次打开 shell 产生一个会话。 因此当启动一个shell时，应该首先给这个shell设置一个会话id。 整个设置流程如下：
1. 启动一个shell，设置会话id。此时该shell进程不是会话首领，因此会被设置为会话首领。设置sid 和 pgid 均等于自身pid，假设为 3。
2. 在shell中执行命令，builtin_exec函数中定义变量pid_t pgid = 0。并将pgid作为参数传给 builtin_command。
3. 在builtin_command中fork子进程执行命令，如果先执行父分支，父进程分支进程判断，如果pgid == 0， 就设置 pgid 为子进程的id。这里第一个子进程会成为进程组的组长。
4. 如果子进程的分支先执行，则调用 setpgid(0, *pgid)。 此时参数pgid为0， 这个子进程的 pgid 也设置进程组的组长。 
5. 当命令还有后续命令，如管道符后的命令，后续执行builtin_command时，都被和第一个命令属于同一个进程组。

这样在shell中，每个shell都会开启一个进程会话，并设置自身为会话首领和和进程组首领。当执行命令时，一次性指令的一组命令会被放到一个进程组中, 例如 ls | dup。shell 会把管道 ls | dup 看作一个 作业（job），这样，shell 可以通过 前台作业控制（Ctrl+C 等）统一管理这一组命令。当再次执行一组命令，会产生一个新的进程组。 shell自身这个进程，它的sid，gpid都是它自身， shell单独属于一个进程组。每个作业运行子进程都是shell进程fork出来的，因此子进程的sid等于shell的sid。


## 二、tty设备
(TeleTYpewriter, TTY) 原指电传打字机，电传打字机曾经在世界各地以大型网络方式连接起来，称为电传，用于传输商业电报，但彼时电传打字机尚未连接到任何计算机。 随着科技的发展与进步，很多电传打字机的模型和功能略有不同，于是产生了一个软件兼容层，来处理不同打字机之间的通信协议。现在物理的电传打字机已经绝迹，除非你参观电子设备的博物馆。不过 TTY 这种兼容层设备在 UNIX 世界保留了下来。

可以认为 TTY 电传打字机设备是一种中间的兼容层，用以处理不同字符设备之间的通信。这些设备可能是控制台、键盘、串口等。除此以外 TTY 通常还和进程的会话和任务相关。

### 2.1、为什么需要tty
假设我们现在有一个最简单的操作系统，它从键盘读输入,在屏幕上输出字符,能运行shell。 但是如果有多个进程要读写屏幕怎么办，两个程序都往屏幕写，此时字符会乱掉。如果我按下 Ctrl+C，希望终止前台程序，键盘只是发字符，中断处理函数不知道哪个进程该收到信号。如果我有多个用户登录系统，每个用户都需要独立的输入输出通道，不可能所有人都共享一个键盘缓冲区。 因此就需要tty这样一个抽象的终端设备，

tty在内核中扮演了一个“虚拟控制台”的角色，每个终端都有独立的输入/输出缓冲，所有写入终端的输出，都会进入 tty 的输出缓冲，tty 会统一管理写屏的节奏。只有绑定到该 tty 的会话的进程才能写，不同用户的 tty 输出互不干扰。假设没有 TTY，每个进程都直接从 同一个键盘缓冲区读取输入，两个进程同时在等待用户输入，那就谁抢到算谁的，当然现在我们的操作系统不具备多用户的能力，这里也只是为了让的大家理解tty存在的意义。并且我们这里实现tty是为了实现信号做准备的。

### 2.2、tty设备实现
tty属于设备的一种，因此需要创建设备文件。
```cpp
void dev_init()
{
    mkdir("/dev", 0755);
    // ....
    device = device_find(DEV_TTY, 0);
    mknod("/dev/tty", IFCHR | 0600, device->dev);
    link("/dev/tty", "/dev/stdout");
    link("/dev/tty", "/dev/stderr");
    link("/dev/tty", "/dev/stdin");
}
```
如上，在设备文件初始化中创建设备节点 /dev/tty。并将标准输入输出硬链接到/dev/tty。因此这三个文件名其实指向同一个设备节点（同一个 inode），也就是当前终端设备。 另外针对设备的输入输出封装一个系统调用 ioctl，如下:
```cpp
int sys_ioctl(fd_t fd, int cmd, void *args)  // 控制设备输入输出
{
    if (fd >= TASK_FILE_NR)   // 判断文件描述符是否超过最大限制
        return -EBADF;
    task_t *task = running_task();   // 获取当前进程
    file_t *file = task->files[fd];  // 获取文件描述符对应的文件结构
    if (!file)
        return -EBADF;

    int mode = file->inode->desc->mode;   // 文件必须是某种设备
    if (!ISCHR(mode) && !ISBLK(mode))
        return -EINVAL;
    dev_t dev = file->inode->desc->zone[0];   // 得到设备号
    if (dev >= DEVICE_NR)
        return -ENOTTY;
    return device_ioctl(dev, cmd, args, 0);   // 进行设备控制
}

```
有了ioctl这个系统调用，允许用户程序通过文件描述符 fd 向设备（如终端、磁盘、串口等）发送控制命令（cmd），并可传递额外参数（args）。接下来是tty设备的实现：
```cpp
typedef struct tty_t
{
    dev_t rdev; // 读设备号
    dev_t wdev; // 写设备号
    pid_t pgid; // 进程组
} tty_t;

#define TIOCSPGRP 0x5410   // ioctl 设置进程组命令
static tty_t typewriter;

int tty_rx_notify(char *ch, bool ctrl, bool shift, bool alt)   // tty设备处理输入字符
{
    switch (*ch)
    {
    case '\r':
        *ch = '\n';
        break;
    default:
        break;
    }
    if (!ctrl)   // ctrl没有被按下，返回0， 即字符不被处理。
        return 0;

    tty_t *tty = &typewriter;
    switch (*ch)
    {
    case 'c':
    case 'C':
        LOGK("CTRL + C Pressed\n");
        break;
    case 'l':
    case 'L':
        // 清屏
        device_write(tty->wdev, "\x1b[2J\x1b[0;0H\r", 12, 0, 0);  // 这里使用了CSI序列清屏
        *ch = '\r';
        return 0;
    default:
        return 1;
    }
    return 1;
}

int tty_read(tty_t *tty, char *buf, u32 count)  // TTY 读， 直接调用设备读
{
    return device_read(tty->rdev, buf, count, 0, 0);
}

int tty_write(tty_t *tty, char *buf, u32 count)    // TTY 写， 直接调用设备写
{
    return device_write(tty->wdev, buf, count, 0, 0);
}

int tty_ioctl(tty_t *tty, int cmd, void *args, int flags)  // 目前支持一个控制指令，即设置进程组
{
    switch (cmd)
    {
    case TIOCSPGRP:  // 设置 tty 参数
        tty->pgid = (pid_t)args;   // 设置tty的进程组为args 
        break;
    default:
        break;
    }
    return -EINVAL;
}

void tty_init()   // 初始化串口
{
    device_t *device = NULL;
    tty_t *tty = &typewriter;
    device = device_find(DEV_KEYBOARD, 0);  // 输入设备是键盘
    tty->rdev = device->dev;
    device = device_find(DEV_CONSOLE, 0);  // 输出设备是控制台
    tty->wdev = device->dev;
    device_install(DEV_CHAR, DEV_TTY, tty, "tty", 0, tty_ioctl, tty_read, tty_write);  // 安装tty设备
}

```
这里POSIX标准定义了 stty 和 gtty两个系统调用，这里我们用不到，因此不做实现。


## 三、tty设备应用
前面我们已经实现了tty设备，接下来需要去使用tty设备进程终端控制。首先在任务结构体中应该增加一个 tty 设备元素，即dev_t tty; 接下来就是键盘驱动，在键盘中断处理函数中调用 tty设备处理输入字符：
```cpp
void keyboard_handler(int vector)
{
    assert(vector == 0x21);
    send_eoi(vector); // 向中断控制器发送中断处理结束的信息
    // ...
    char ch = 0;
    if (tty_rx_notify(&ch, ctrl_state, shift_state, alt_state) > 0)  // 通知 tty 设备处理输入字符
        return;

    fifo_put(&fifo, ch);
    if (waiter != NULL)
    {
        task_unblock(waiter, EOK);
        waiter = NULL;
    }
}
```
如上所示，将字符首先通过tty进行助理，例如如果是 ctrl+c，tty就会打印内容，目前我们还没有对ctrl+c进行处理，因此直接打印。如果tty_rx_notify返回的结果小于等于0， 说明没有做处理，将该字符直接写入队列即可。在shell中设置tty的进程组为osh
```cpp
// 设置 TTY 进程组为 osh
ioctl(STDIN_FILENO, TIOCSPGRP, getpid());
```
在进程退出时同时要释放tty设备
```cpp
// 释放 TTY 设备
if (task_leader(task) && task->tty > 0)
{
    device_t *device = device_get(task->tty);
    tty_t *tty = (tty_t *)device->ptr;
    tty->pgid = 0;
}
```
这样当你在终端启动 shell 时，打开一个 TTY（或伪TTY）； 启动 shell 进程，并把这个 TTY 的文件描述符（stdin/stdout/stderr）绑定给 shell；设置这个 shell 为该 TTY 的前台进程组。最后我们来整理一下整个流程：
1. 启动shell时，shell 会通过 setsid() 创建一个新会话，当前进程（shell）成为 会话首领，同时成为 进程组首领
2. shell的主函数调用了ioctl系统调用，入参为标准输入0和当前进程pid。因为标准输入是/dev/tty的硬链接，所以这里就是操作的tty设备，设置了tty->pgid为osh的进程id， 即 将该 TTY 的前台进程组设置为 shell 的 pgid。
2. 在执执行一组命令（如 ls | grep txt）时，builtin_command为这一组命令（管道组）创建新的进程组。所有子进程（例如 ls、grep）加入这个新的进程组，TTY 的前台进程组切换为子进程组。
3. 这样用户按 Ctrl+C 时 → 信号发给子进程组。例如当按下Ctrl+C后，键盘产生中断，tty设备会进行字符处理，并执行相应的操作，例如中断执行，此时我们还未实现，因为这里需要信号。
4. 当命令执行完， 等待所有子进程运行结束后，exec再次调用ioctl，设置设置 TTY 进程组为 osh。

当我们现在在键盘上按下 ctrl+l 时，键盘产生中断。中断处理函数通过tty设备进行字符处理，做了清屏操作，这里其实和shell本身没有关系。但是现在linux操作系统下，当你在 bash/zsh 等 shell 中 按下 Ctrl+L 时，键盘中断触发，内核 tty 驱动把字符流（包括 Ctrl+L = 0x0C）放入终端输入缓冲区，shell（用户态进程）通过标准输入（文件描述符 0）读取并执行清屏指令。

---
title:       自制操作系统 - init进程与内核内存保护
subtitle:    "init进程与内核内存保护"
description: "osh 和 init应该属于用户程序，但是目前我们直接编译进内核了。到目前我们的系统已经可以执行程序了，因此需要将这两个程序独立出来，作为用户程序放到用户空间去执行。并且目前用户进程是可以访问内核内存的，我们需要对内核内存进行保护，不允许用户进程访问。"
excerpt:     "osh 和 init应该属于用户程序，但是目前我们直接编译进内核了。到目前我们的系统已经可以执行程序了，因此需要将这两个程序独立出来，作为用户程序放到用户空间去执行。并且目前用户进程是可以访问内核内存的，我们需要对内核内存进行保护，不允许用户进程访问。"
date:        2024-03-23T10:21:37+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/37/91/boys_school_teacher_education_asia_the_board_of_directors_cambodia_kids-783707.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-init-kernel-mem"
categories:  [ "操作系统" ]
---

## 一、init进程
osh 和 init应该属于用户程序，但是目前我们直接编译进内核了。到目前我们的系统已经可以执行程序了，因此需要将这两个程序独立出来，作为用户程序放到用户空间去执行。

### 1.1、独立osh
osh作为shell程序，首先需要修改他的主函数为main，因为它要作为一个独立的用户程序来运行。其次我们是在init进程中调用了osh，在内核中编译时可以直接调用，但是作为用户程序之后需要使用系统调用来执行。修改如下：
```cpp
static void user_init_thread()
{
    while (true)
    {
        u32 status;
        pid_t pid = fork();
        if (pid)
        {
            pid_t child = waitpid(pid, &status);
            printf("wait pid %d status %d %d\n", child, status, time());
        }
        else
        {
            int err = execve("/bin/osh.out", NULL, NULL);
            printf("execve /bin/osh.out error %d\n", err);
            exit(err);
        }
    }
}
```
如上所示，我们应该通过execve系统调用来执行osh程序。调整到用户空间后，就可以多次执行osh了。

### 1.2、独立init进程
init进程也需要运行在用户内存空间，因此首先需要编写init程序，如下：
```cpp
int main()
{
    if (getpid() != 1)
    {
        printf("init already running...\n");
        return 0;
    }
    while (true)
    {
        u32 status;
        pid_t pid = fork();
        if (pid)
        {
            pid_t child = waitpid(pid, &status);
            printf("wait pid %d status %d %d\n", child, status, time());
        }
        else
        {
            int err = execve("/bin/osh.out", NULL, NULL);
            printf("execve /bin/osh.out error %d\n", err);
            exit(err);
        }
    }
    return 0;
}
```
init进程的逻辑基本没有变化，只是独立成了一个程序文件。接下来在从内核态进入用户态时，就不需要需要内联汇编来通过jump跳转了，直接使用内核函数sys_execve调用即可
```cpp
void task_to_user_mode()
{
    task_t *task = running_task();
    ....
    int err = sys_execve("/bin/init.out", NULL, NULL);
    panic("exec /bin/init.out failure");
}
```
到此，用户态和内核态就已经完全隔离了。接下来就可以对内核的内存进行保护，防止用户程序使用内核内存。


## 二、内核内存保护
目前我们定义全局描述符时，定义了用户特权级也是完整访问整个4G的内存。现在用户程序也可以访问内核的内存，这样极不安全，所以需要一种机制来防止用户程序，无意或有意的访问和修改内核内存。我们先写一个用户程序来访问内核内存：
```cpp
#include <onix/stdio.h>

int main()
{
    char *video = (char *)0xB8000;
    printf("char in 0x%X is %c\n", video, *video);
    *video = 'E';
    printf("changed to E\n");
    return 0;
}
```
如上所示，我们给显存的第一个位置改写为字符E，这里因内核的映射是虚拟地址映射到相同的物理地址，因此实际访问的物理内存就是0xB8000。 当shell中执行err命令，控制台的第一个字符被修改为了E。说明用户程序是可以操作系统内存的。

将程序映射参数 user 置为 0，表示该页只能由内核访问。因此在内核内存映射时设置页表想的user元素为0，代表仅超级用户可以使用。
```cpp
typedef struct page_entry_t
{
    u8 present : 1;  // 在内存中
    u8 write : 1;    // 0 只读 1 可读可写
    u8 user : 1;     // 1 所有人 0 超级用户 DPL < 3
    u8 pwt : 1;      // page write through 1 直写模式，0 回写模式
    u8 pcd : 1;      // page cache disable 禁止该页缓冲
    u8 accessed : 1; // 被访问过，用于统计使用频率
    u8 dirty : 1;    // 脏页，表示该页缓冲被写过
    u8 pat : 1;      // page attribute table 页大小 4K/4M
    u8 global : 1;   // 全局，所有进程都用到了，该页不刷新缓冲
    u8 shared : 1;   // 共享内存页，与 CPU 无关
    u8 privat : 1;   // 私有内存页，与 CPU 无关
    u8 readonly : 1; // 只读内存页，与 CPU 无关
    u32 index : 20;  // 页索引
} _packed page_entry_t;
```
当用户尝试访问内核内存时，就会触发页错误，我们在页错误里进行判断访问的内存，如果访问时内核内存或者超过栈顶的内存，直接打印段错误。

除此以外，由于用到了平坦模型，所以内核必须检测用户程序系统调用传入的 内存空间，一定在用户的内存空间，否则也需要报错。但是这样在开发阶段有一个缺点，就是不方便调试。可以添加 `ONIX_DEBUG` 宏，判断是否是调试状态，然后分别对不同的状态进行处理。如果开启了DEBUG，则将init和osh编译进内核。

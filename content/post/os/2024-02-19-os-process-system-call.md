---
title:       自制操作系统 - 进程相关系统调用
subtitle:    "进程相关系统调用"
description: "进程涉及进程创建，进程退出，进程资源回收等多种操作，在用户态中这些动作依赖系统调用来实现。本文我们将完成brk、 fork、 exit、waitpid、time共五个系统调用。"
excerpt:     "进程涉及进程创建，进程退出，进程资源回收等多种操作，在用户态中这些动作依赖系统调用来实现。本文我们将完成brk、 fork、 exit、waitpid、time共五个系统调用。"
date:        2024-02-19T11:02:23+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/ff/a6/yellowstone_national_park_wyoming_landscape_sunrise_sunset_mountains_snow_winter-551109.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-process-system-call"
categories:  [ "操作系统" ]
---

## 一、系统调用brk
brk系统调用的作用是修改堆内存的上限。我们的操作系统从8M ~ 128M 是用户进程的内存空间。我们会把进程的ELF文件映射到 8M 开始的位置， 有text段、data段、bss段。这些段结束后就是堆内存。堆内存使用了多少就需要使用brk来标记。brk系统调用时 malloc/free 函数的基础。

当程序运行时，如果你使用了如 malloc 这样的内存分配函数，它最终可能会通过 brk 或 mmap 来向内核申请更多内存。而 brk 就是负责调节 堆的顶端位置 的系统调用。
```cpp
int32 sys_brk(void *addr)
{
    LOGK("task brk 0x%p\n", addr);
    u32 brk = (u32)addr;
    ASSERT_PAGE(brk);                         // 判断brk是否是页开始的位置
    task_t *task = running_task();
    assert(task->uid != KERNEL_USER);         // 判断是否是用户
    assert(KERNEL_MEMORY_SIZE < brk < USER_STACK_BOTTOM);  // 判断brk是否是
    u32 old_brk = task->brk;   // 获取进程当前的堆内存边界

    if (old_brk > brk)   // 如果当前边界大于新申请的边界，那就释放内存映射
    {
        for (; brk < old_brk; brk += PAGE_SIZE)
        {
            unlink_page(brk);
        }
    }
    else if (IDX(brk - old_brk) > free_pages)  // 如果新的增加brk大于了剩余的空闲页，就返回-1,没有可用内存了。
    {
        return -1;    // out of memory
    }

    task->brk = brk;
    return 0;
}

int32 brk(void *addr)
{
    return _syscall1(SYS_NR_BRK, (u32)addr);
}
```
如上所是：sys_brk函数中brk系统调用的实现函数。在系统调用里增加brk，brk为45号系统调用，这与当前的linux系统保持一致。我们在user_init_thread进行brk系统调用，当使用这块内存空间时会产生缺页异常，缺页异常处理函数会把申请的页进行映射。用户进程就可以访问这些内存了。缺页异常处理函数的代码也需要略微调整，增加判断 vaddr < task->brk。

## 二、任务id
这节涉及两个系统调用，获取任务id和父任务id。首先需要在task_t这个结构中增加 pid 和 ppid 两个元素。调用的实现比较简单, 直接返回任务的pid和ppid即可。
```cpp
pid_t sys_getpid()      // 获取进程 id
{
    task_t *task = running_task();
    return task->pid;
}
pid_t sys_getppid()      // 获取父进程 id
{
    task_t *task = running_task();
    return task->ppid;
}

static task_t *get_free_task()
{
    for (size_t i = 0; i < NR_TASKS; i++)
    {
        if (task_table[i] == NULL)
        {
            task_t *task = (task_t *)alloc_kpage(1); // todo free_kpage
            memset(task, 0, PAGE_SIZE);
            task->pid = i;
            task_table[i] = task;
            return task;
        }
    }
    panic("No more tasks");
}
```
如上所示：在从任务列表中获取空闲任务时，直接将任务的pid设置为数组的索引。运行测试：test进程的id号为2， init进程的进程id为1，他们的父进程id都是0，因为目前还没有父进程，没有给task_t结构体的pid元素赋值。

## 三、系统调用fork
fork的作用是创建一个子进程，子进程的返回值为0，父进程的返回值为子进程的id。该函数是一次调用，两次返回。如下为系统调用fork的实现：
```cpp
pid_t task_fork()
{
    task_t *task = running_task();  // 当前进程为父进程，需保证父进程没有阻塞，且正在执行。 
    // node是任务阻塞节点，确保当前任务不在任何链表中，且任务状态为运行态
    assert(task->node.next == NULL && task->node.prev == NULL && task->state == TASK_RUNNING);

    task_t *child = get_free_task();   // 从任务数组中获取空闲数组并将数组索引赋给id
    pid_t pid = child->pid;           // 此处的pid就是子进程在任务组的索引
    memcpy(child, task, PAGE_SIZE);  // 拷贝父进程的PCB和内核栈给子进程
    child->pid = pid;                // 设置子进程的pid
    child->ppid = task->pid;          // 设置子进程的父id
    child->ticks = child->priority;
    child->state = TASK_READY;
    child->vmap = kmalloc(sizeof(bitmap_t)); // 拷贝用户进程虚拟内存位图 todo kfree
    memcpy(child->vmap, task->vmap, sizeof(bitmap_t));   // 复制父进程的虚拟内存位图给子进程
    void *buf = (void *)alloc_kpage(1); // 拷贝虚拟位图缓存 todo free_kpage
    memcpy(buf, task->vmap->bits, PAGE_SIZE);   // 将父进程的虚拟内存位图缓存也拷贝给子进程
    child->vmap->bits = buf;
    child->pde = (u32)copy_pde();  // 拷贝页目录
    task_build_stack(child); // 构造 child 内核栈 ROP
    return child->pid;
}
```
task_build_stack函数用来构造子进程的内核栈，内核栈的概念可以参考前面进入用户模式的文章。task_build_stack函数中将 eax 寄存器赋为0。 因此子进程的返回值就是0。以下为task_build_stack的函数实现：
```cpp
static void task_build_stack(task_t *task)   // 入参为子进程所在的内存页
{
    u32 addr = (u32)task + PAGE_SIZE;
    addr -= sizeof(intr_frame_t);
    intr_frame_t *iframe = (intr_frame_t *)addr;
    iframe->eax = 0;           // 设置eax寄存器为0 ，也就是子进程的返回值为0
    addr -= sizeof(task_frame_t);
    task_frame_t *frame = (task_frame_t *)addr;
    frame->ebp = 0xaa55aa55;
    frame->ebx = 0xaa55aa55;
    frame->edi = 0xaa55aa55;
    frame->esi = 0xaa55aa55;
    frame->eip = interrupt_exit;
    task->stack = (u32 *)frame;
}
```
这里子进程的栈帧中 eip 被设置为了 中断退出。它的作用是调度器切换进程时，相当于跳转到 interrupt_exit，它负责恢复中断上下文并用 iret 跳回用户态。父进程调用 fork() 后，会从内核返回，继续跑它的代码（已经回用户态）。子进程的创建是“复制”了父进程的内存、寄存器等状态，但它还没有被 CPU 执行，此时子进程处于内核态。当调度器首次切换到子进程，子进程要“从 fork 返回”，继续执行它自己的用户态代码。

fork时还有一个事情需要做的就是拷贝页目录，如下为拷贝页目录的实现：
```cpp
page_entry_t *copy_pde()
{ 
    task_t *task = running_task();      // 获取当前运行的进程，也就是父进程
    page_entry_t *pde = (page_entry_t *)alloc_kpage(1); // 申请一页内存存储页目录 todo free
    memcpy(pde, (void *)task->pde, PAGE_SIZE);         // 拷贝父进程的页目录给子进程
    
    page_entry_t *entry = &pde[1023];   // 将最后一个页表指向页目录自己，方便修改
    entry_init(entry, IDX(pde));

    page_entry_t *dentry;
    for (size_t didx = 2; didx < 1023; didx++)    // 遍历页目录
    {
        dentry = &pde[didx];
        if (!dentry->present)
            continue;
        page_entry_t *pte = (page_entry_t *)(PDE_MASK | (didx << 12));   // 如果页表存在就继续遍历页表
        for (size_t tidx = 0; tidx < 1024; tidx++)
        {
            entry = &pte[tidx];
            if (!entry->present)
                continue;
            assert(memory_map[entry->index] > 0);  //如果页框存在 判断对应物理内存引用大于 0
            entry->write = false;                  // 对该页置为只读
            memory_map[entry->index]++;            // 对应物理页引用加 1
            assert(memory_map[entry->index] < 255);   // 物理页引用要小于255
        }
        u32 paddr = copy_page(pte);    // 拷贝一页8M以上的物理内存，并把这页内存映射到0这个位置
        dentry->index = IDX(paddr);
    }
    set_cr3(task->pde);
    return pde;
}

static u32 copy_page(void *page)      // 拷贝一页，返回拷贝后的物理地址，入参是一个虚拟地址
{
    u32 paddr = get_page();   // 获取一个8M以上的物理页，物理地址存储在 paddr 中。
    // 获取虚拟地址 0x00000000 对应的 页表项。也就是说，我们想 临时把虚拟地址 0 映射到 paddr 所指向的物理页
    page_entry_t *entry = get_pte(0, false); 
    entry_init(entry, IDX(paddr));  // 初始化这个页表项 entry，把它映射到物理页 paddr
    memcpy((void *)0, (void *)page, PAGE_SIZE); 通过 memcpy() 把 page 指向的数据，拷贝到地址 0x00000000 所指向的新物理页
    entry->present = false;   
    return paddr;
}
```
如上，巧妙了利用了第0页地址进行物理也数据的拷贝，因为第0页我们没有做映射。这里还有一个注意点，就是拷贝页目录时我们将页框设置为了只读。这是写时复制原理，如果直接拷贝所有物理内存页，会非常耗时和浪费内存。而实际上，很多页（尤其是代码段、只读数据段）根本不会被写入。所以 OS 采用策略：父子进程页表指向同一物理页（共享页框）将这些页设置为只读，当某个进程尝试写这些页时 → 触发页保护异常 → 内核捕获后再复制该页。

在用户进程调用fork
```cpp
static void user_init_thread()
{
    u32 counter = 0;
    while (true)
    {
        pid_t pid = fork();
        if (pid)
        {
            printf("fork after parent %d, %d, %d\n", pid, getpid(), getppid());
        }
        else
        {
            printf("fork after child %d, %d, %d\n", pid, getpid(), getppid());
        }
        hang();
        sleep(100);
    }
}
```
如上，如果fork调用返回为0，说明当前运行的是父进程，如果不为0，则当前运行的是子进程。

## 四、系统调用exit
exit的作用是终止当前进程。实现如下：
```cpp
void task_exit(int status)
{
    task_t *task = running_task();

    // 当前进程没有阻塞，且正在执行
    assert(task->node.next == NULL && task->node.prev == NULL && task->state == TASK_RUNNING);
    task->state = TASK_DIED;
    task->status = status;
    free_pde();
    free_kpage((u32)task->vmap->bits, 1);
    kfree(task->vmap);
    // 将子进程的父进程赋值为自己的父进程
    for (size_t i = 0; i < NR_TASKS; i++)
    {
        task_t *child = task_table[i];
        if (!child)
            continue;
        if (child->ppid != task->pid)
            continue;
        child->ppid = task->ppid;
    }
    LOGK("task 0x%p exit....\n", task);
    schedule();
}
```
exit主要来进行内存的释放。将子进程的父进程赋值为自己的父进程， 这个作用是如果爹要死了，就把儿子托管给爷爷。当前进程已经结束，因此需要进行一次调度。此时还有一页内存没有被释放，就是任务PCB所在的那页内存。因此需要父进程来给它收尸，如果父进程没有进行收尸，那这个进程就进入僵尸状态。因此还需要一个waitpid系统调用来给子进程收拾。


## 五、系统调用waitpid
waitpid用来获取子进程的退出状态，也就是帮子进程收尸。实现如下：
```cpp
pid_t task_waitpid(pid_t pid, int32 *status)
{
    task_t *task = running_task();
    task_t *child = NULL;

    while (true)
    {
        bool has_child = false;
        for (size_t i = 2; i < NR_TASKS; i++)
        {
            task_t *ptr = task_table[i];
            if (!ptr)
                continue;

            if (ptr->ppid != task->pid)
                continue;
            if (pid != ptr->pid && pid != -1)
                continue;

            if (ptr->state == TASK_DIED)
            {
                child = ptr;
                task_table[i] = NULL;
                goto rollback;
            }

            has_child = true;
        }
        if (has_child)
        {
            task->waitpid = pid;
            task_block(task, NULL, TASK_WAITING);
            continue;
        }
        break;
    }

    // 没找到符合条件的子进程
    return -1;

rollback:
    *status = child->status;
    u32 ret = child->pid;
    free_kpage((u32)child, 1);
    return ret;
}
```
父进程进程waitpid系统调用，首先会检查是否有子进程，如果没有就返回-1。


## 六、系统调用time
time的作用是获取当前时间戳，即从 1970-01-01 00:00:00 开始的秒数。实现比较简单，如下：
```cpp
time_t sys_time()
{
    return startup_time + (jiffies * JIFFY) / 1000;
}
```
如上，时间戳直接从系统启动时间，加上启动以来经过的 时钟滴答（tick）次数。



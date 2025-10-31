---
title:       自制操作系统 - 输入输出重定向与管道
subtitle:    "输入输出重定向与管道"
description: "输入输出重定向一般用于将标准输入输出重定向到文件，依赖系统调用dup和dup2完成。管道是一种进程间通信的方式，pipe 系统调用可以返回两个文件描述符，其中前一个用于读，后一个用于写，那么读进程就可以读取写进程写入的内容。管道只能在具有公共祖先的两个进程之间使用。"
excerpt:     "输入输出重定向一般用于将标准输入输出重定向到文件，依赖系统调用dup和dup2完成。管道是一种进程间通信的方式，pipe 系统调用可以返回两个文件描述符，其中前一个用于读，后一个用于写，那么读进程就可以读取写进程写入的内容。管道只能在具有公共祖先的两个进程之间使用。"
date:        2024-03-21T15:24:10+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/04/28/ocean_sea_beach_person_single_serene_calm_water-595543.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-dup-pipe"
categories:  [ "操作系统" ]
---

## 一、输入输出重定向
shell 输入输出重定向有很多类型，一般常见以下几种，用于操作标准输入输出的文件；
```shell
cmd < filename     # 输入重定向
cmd > filename     # 输出重定向
cmd >> filename    # 输出追加重定向
```
输入输出重定向的实现依赖两个系统调用 dup 和 dup2， 因此需要先实现这两个系统调用。

### 1.2、系统调用dup
系统调用 dup 的作用是制文件描述符；实现如下：
```cpp
static int dupfd(fd_t fd, fd_t arg)
{
    task_t *task = running_task();   /// 获取当前进程
    if (fd >= TASK_FILE_NR || !task->files[fd])  // 判断文件描述符是否大于进程支持的最大描述符
        return EOF;

    for (; arg < TASK_FILE_NR; arg++)    // 从 arg 开始，查找第一个空闲的文件描述符槽位
    { 
        if (!task->files[arg])   
            break;
    }

    if (arg >= TASK_FILE_NR)  // 如果没找到空闲的槽位（到达上限），返回错误
        return EOF;

    task->files[arg] = task->files[fd];  // 将新描述符 arg 指向与原描述符 fd 相同的文件结构（共享同一个 file_t）
    task->files[arg]->count++; // 文件结构的引用计数 count++，表示多个描述符共享同一文件
    return arg;      // 返回新的文件描述符号 arg
} 

fd_t sys_dup(fd_t oldfd)
{
    return dupfd(oldfd, 0);
}

fd_t sys_dup2(fd_t oldfd, fd_t newfd)
{
    close(newfd);   // 关闭文件描述符，释放这个槽位
    return dupfd(oldfd, newfd);
}
```
如上所示，有两个系统调用，分别是dup和dup2。dup系统调用是 复制文件描述符，分配最小空闲的新描述符。 系统调用dup2 是 将 fd2 复制为 fd1 的副本，如果 fd2 已打开则关闭再复制。注意dup2中先关闭新的文件描述符，说明指定的这个位置就是一个空槽位。

### 1.3、输入输出重定向实现
实现了文件描述符的复制后，就需要修改shell来试下重定向功能了
```cpp
static void dupfile(int argc, char **argv, fd_t dupfd[3])
{
    for (size_t i = 0; i < 3; i++)   // 初始化dupfd
    {
        dupfd[i] = EOF;
    }
    int outappend = 0;   // 输出是否采用追加模式
    int errappend = 0;   // 错误输出是否采用追加模式
    char *infile = NULL;
    char *outfile = NULL;
    char *errfile = NULL;
    for (size_t i = 0; i < argc; i++)   // 遍历argc
    {
        if (!strcmp(argv[i], "<") && (i + 1) < argc)  // 如果参数中有 <  则设置下一个参数为输入文件
        {
            infile = argv[i + 1];
            argv[i] = NULL;
            i++;
            continue;
        }
        // ... 这里省略了 > >> 2>  2>>等参数的判断
    }

    if (infile != NULL)
    {
        fd_t fd = open(infile, O_RDONLY | outappend | O_CREAT, 0755);
        if (fd == EOF)
        {
            printf("open file %s failure\n", infile);
            goto rollback;
        }
        dupfd[0] = fd;
    }
    // ...  省略打开标准输出文件和标准错误输出文件的的打开
    return;
}

pid_t builtin_command(char *filename, char *argv[], fd_t infd, fd_t outfd, fd_t errfd)  // 创建子进程执行命令
{
    int status;
    pid_t pid = fork();  // fork一个子进程
    if (pid)
    {
        if (infd != EOF)
        {
            close(infd);
        }
        // .... 父进程关闭输入输出文件描述符
        return pid;
    }
    if (infd != EOF)  // 子进程
    {
        fd_t fd = dup2(infd, STDIN_FILENO);  // 让标准输入重定向到新文件
        close(infd);
    }
    if (outfd != EOF)
    {
        fd_t fd = dup2(outfd, STDOUT_FILENO);
        close(outfd);
    }
    if (errfd != EOF)
    {
        fd_t fd = dup2(errfd, STDERR_FILENO);
        close(errfd);
    }

    int i = execve(filename, argv, envp);   // 启动目标程序
    exit(i);
}
```
如上所示，在shell中执行指令时，定义dupfile解析命令行参数，打开对应文件。 builtin_command 函数来创建子进程执行命令。 例如如果指定了输入文件，在这里就会让0号文件描述符指向指定的文件。这样当从0号文件描述符读取时，就是读取了我们指定的文件。


## 二、管道
管道是一种进程间通信的方式，pipe 系统调用可以返回两个文件描述符，其中前一个用于读，后一个用于写，那么读进程就可以读取写进程写入的内容。管道只能在具有公共祖先的两个进程之间使用。通常，一个管道由一个进程创建，在进程调用 fork 之后，这个管道就能在父进程和子进程之间使用了。


### 2.1、管道实现
管道是通过文件描述符实现的，因此inode_t结构需要增加三个元素，分别为读等待进程、写等待进程和管道标志，标志这个inode是否作为一个管道。如下：
```cpp
typedef struct inode_t
{
    ...
    struct task_t *rxwaiter; // 读等待进程
    struct task_t *txwaiter; // 写等待进程
    bool pipe;               // 管道标志
} inode_t;
```
有了pipe的inode，还需要提供申请和释放inode的函数。
```cpp
/ 获取根 inode
inode_t *get_pipe_inode()
{
    inode_t *inode = get_free_inode();  // 从系统inode表中申请一个空闲的inode
    inode->dev = -2;   // 区别于 EOF(-1) 这里是无效的设备，但是被占用了。因此我们这里使用 -2 
    inode->desc = (inode_desc_t *)kmalloc(sizeof(fifo_t));  // 申请内存，表示缓冲队列
    inode->buf = (void *)alloc_kpage(1);  // 管道缓冲区一页内存
    inode->count = 2;      // 两个文件， 使用同一个inode
    inode->pipe = true;    // 管道标志
    fifo_init((fifo_t *)inode->desc, (char *)inode->buf, PAGE_SIZE);   // 初始化输入输出设备
    return inode;
}

void put_pipe_inode(inode_t *inode)   // 释放管道inode
{
    if (!inode)
        return;
    inode->count--;
    if (inode->count)
        return;
    inode->pipe = false;
    kfree(inode->desc);   // 释放描述符 fifo
    free_kpage((u32)inode->buf, 1);  // 释放缓冲区
    put_free_inode(inode);      // 释放 inode
}
```
这里申请一个inode时设置了引用为2，是因为存在一个读端一个写端，当释放时count--，需要读端和写端均释放，引用才能为0，最终释放inode。为了统一接口，在inode的释放函数 iput 中进行判断，如果是管道，那就调用put_pipe_inode。

接下来就需要实现管道的读写和系统调用，如下：
```cpp
int pipe_read(inode_t *inode, char *buf, int count)   // 管道读
{
    fifo_t *fifo = (fifo_t *)inode->desc;  // 获取管道inode的数据块，设置类型为队列
    int nr = 0;
    while (nr < count)     // 循环读取，知道读取指定长度
    {
        if (fifo_empty(fifo))   // 如果队列为空
        {
            assert(inode->rxwaiter == NULL);
            inode->rxwaiter = running_task();  // 将当前进行设置为读等下进程，并阻塞
            task_block(inode->rxwaiter, NULL, TASK_BLOCKED);
        }
        buf[nr++] = fifo_get(fifo);   // 读取数据，写入到buf
        if (inode->txwaiter)   // 如果有写等待进程，则解除写等待的阻塞，因为现在没有数据，可以开始写了。
        {
            task_unblock(inode->txwaiter);
            inode->txwaiter = NULL;
        }
    }
    return nr;
}

int pipe_write(inode_t *inode, char *buf, int count)  // 管道写
{
    fifo_t *fifo = (fifo_t *)inode->desc;
    int nr = 0;
    while (nr < count)
    {
        if (fifo_full(fifo))   // 判断队列是否已满， 如果满了，就阻塞写进程
        {
            assert(inode->txwaiter == NULL);
            inode->txwaiter = running_task();
            task_block(inode->txwaiter, NULL, TASK_BLOCKED);
        }
        fifo_put(fifo, buf[nr++]);   // 将buf的数据写入队列，如果有读等待进程，解除读进程的阻塞，因此此时已经有数据了。
        if (inode->rxwaiter)
        {
            task_unblock(inode->rxwaiter);
            inode->rxwaiter = NULL;
        }
    }
    return nr;
}

int sys_pipe(fd_t pipefd[2])   // pipe 系统调用
{
    inode_t *inode = get_pipe_inode();  // 获取一个管道 inode
    task_t *task = running_task();   // 获取当前进程
    file_t *files[2];
    pipefd[0] = task_get_fd(task);   // 获取一个进程描述符
    files[0] = task->files[pipefd[0]] = get_file();  // 获取文件结构

    pipefd[1] = task_get_fd(task);   // 再获取一个进程描述符
    files[1] = task->files[pipefd[1]] = get_file();
    files[0]->inode = inode;
    files[0]->flags = O_RDONLY;  // 读端设置为只读
    files[1]->inode = inode;
    files[1]->flags = O_WRONLY;  // 写端设置为只写
    return 0;
}
```
这里我们实现了管道的读和写，为了统一接口，需要修改read和write两个系统调用，如果inode类型为管道，那就调用管道的读写函数。

### 2.3、管道测试
实现了管道，就可以再shell中进行测试了。这里我们使用内建的test命令进行测试，代码如下：
```cpp
void builtin_test(int argc, char *argv[])
{
    int status = 0;
    fd_t pipefd[2];   // 两个文件描述符
    int result = pipe(pipefd);  // 调用pipe系统调用， 为这两个文件描述符申请inode和文件结构

    pid_t pid = fork();  // fork一个子进程
    if (pid)
    {
        char buf[128];
        printf("--%d-- geting message\n", getpid());   // 父进程打印
        int len = read(pipefd[0], buf, 24);            // 父进程进行管道读，
        printf("--%d-- get message: %s count %d\n", getpid(), buf, len);  // 父进程打印
        pid_t child = waitpid(pid, &status);
        close(pipefd[1]);  // 关闭管道描述符
        close(pipefd[0]);
    }
    else
    {
        char *message = "pipe written message!!!"; 
        printf("--%d-- put message: %s\n", getpid(), message);  // 子进程打印
        write(pipefd[1], message, 24);       // 子进程进行管道写
        close(pipefd[1]);   // 关闭管道描述符
        close(pipefd[0]);
        exit(0);
    }
}
```
这里我们梳理一下这个命令的执行流程，
1. 父进程调用pipe系统调用申请了两个文件描述符，申请了一个管道inode，并将两个文件描述符都指向这一个inode。
2. 父进程fork子进程，子进程和父进程的文件描述符是一样的，子是父的副本，它们共享底层的打开文件对象。
3. 父进程进行管道读，但是此时没有数据，阻塞自身。
4. 子进程获取到CPU执行权，进行写管道，并唤醒父进程
5. 父进程进行管道读，并等待子进程退出
6. 子进程关闭文件描述符，并退出。父进程也关闭文件描述符，此时引用减为0，会释放inode。


## 三、管道序列
shell 中可以用 | 来表示管道。这种方式叫做管道序列，我们这里也实现下这种方式。
```cpp
int main(int argc, char const *argv[])
{
    char ch;
    while (true)
    {
        int n = read(STDIN_FILENO, &ch, 1);   // 读取标准输入
        if (n == EOF)
        {
            break;
        }
        if (ch == '\n')  // 如果字符为 \n 就退出
        {
            write(STDOUT_FILENO, &ch, 1);
            break;
        }
        write(STDOUT_FILENO, &ch, 1);  // 两次写入
        write(STDOUT_FILENO, &ch, 1);
    }
    return 0;
}
```
我们这里实现了一个dup命令，它读取一次标准输入，然后进行两次键盘输出。接下来需要修改shell，来支持管道符 |
```cpp
void builtin_exec(int argc, char *argv[])
{
    bool p = true;
    int status;
    char **bargv = NULL;
    char *name = buf;

    fd_t dupfd[3];
    if (dupfile(argc, argv, dupfd) == EOF)   // 解析命令行参数，打开重定向文件，并在 dupfd[0~2] 中保存
        return;

    fd_t infd = dupfd[0];  // 标准输入
    fd_t pipefd[2];
    int count = 0;
    for (int i = 0; i < argc; i++)  // 对命令的参数进行循环
    {
        if (!argv[i])
        {
            continue;
        }
        if (!p && !strcmp(argv[i], "|"))   // 判断是否出现管道符
        {
            argv[i] = NULL;
            int ret = pipe(pipefd);         // 调用pipe系统调用， 为管道描述符申请inode和文件结构
            builtin_command(name, bargv, infd, pipefd[1], EOF);  // 执行当前命令（输入来自 infd，输出写入 pipefd[1]）
            count++;
            infd = pipefd[0];    // 标准输入
            int len = strlen(name) + 1;
            name += len;
            p = true;  // 表示准备开始新命令
            continue;
        }
        if (!p)
        {
            continue;
        }

        stat_t statbuf;
        sprintf(name, "/bin/%s.out", argv[i]);
        if (stat(name, &statbuf) == EOF)
        {
            printf("osh: command not found: %s\n", argv[i]);
            return;
        }
        bargv = &argv[i + 1];
        p = false;
    }

    int pid = builtin_command(name, bargv, infd, dupfd[1], dupfd[2]);
    for (size_t i = 0; i <= count; i++)
    {
        pid_t child = waitpid(-1, &status);
    }
}
```
如上所示，如果我们执行了 ls | dup。首先会遍历ls, ls不是管道符。然后继续遍历到管道符，调用pipe系统调用，设置ls的标准输出为管道符的写端。最后执行dup命令，从管道输入端读取数据。
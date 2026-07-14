---
title:       Redis源码阅读 - Redis事件驱动模型(二)
subtitle:    "Redis事件驱动模型(二)"
description: ""
excerpt:     ""
date:        2026-06-05T14:27:19+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/28/a8/desert_rock_star_moonlight_outdoor-17717.jpg!d"
published:   false
tags:
    - redis
slug:        "redis-event-model-02"
categories:  [ "REDIS" ]
---

## 一、Reactor 模型
Reactor 模型就是网络服务器端用来处理高并发网络 IO 请求的一种编程模型。我把这个模型的特征用两个“三”来总结，也就是：
* 三类处理事件，即连接事件、写事件、读事件；
* 三个关键角色，即 reactor、acceptor、handler。

当一个客户端要和服务器端进行交互时，客户端会向服务器端发送连接请求，以建立连接，这就对应了服务器端的一个连接事件。一旦连接建立后，客户端会给服务器端发送读请求，以便读取数据。服务器端在处理读请求时，需要向客户端写回数据，这对应了服务器端的写事件。无论客户端给服务器端发送读或写请求，服务器端都需要从客户端读取请求内容，所以在这里，读或写请求的读取就对应了服务器端的读事件。

连接事件由 acceptor 来处理，负责接收连接；acceptor 在接收连接后，会创建 handler，用于网络连接上对后续读写事件的处理。其次读写事件都由handler处理。在高并发场景中，连接事件、读写事件会同时发生，所以，我们需要有一个角色专门监听和分配事件，这就是 reactor 角色。当有连接请求时，reactor 将产生的连接事件交由 acceptor 处理；当有读写请求时，reactor 将读写事件交由 handler 处理。

这三个角色都是 Reactor 模型中要实现的功能的抽象。当我们遵循 Reactor 模型开发服务器端的网络框架时，就需要在编程的时候，在代码功能模块中实现 reactor、acceptor 和 handler 的逻辑。我们实现这三者的交互就需要事件驱动模型了。

## 二、事件驱动模型
事件驱动模型，就是在实现 Reactor 模型时，需要实现的代码整体控制逻辑。简单来说，事件驱动模型包括了两部分：一是事件初始化；二是事件捕获、分发和处理主循环。

事件初始化是在服务器程序启动时就执行的，它的作用主要是创建需要监听的事件类型，以及该类事件对应的 handler。而一旦服务器完成初始化后，事件初始化也就相应完成了，服务器程序就需要进入到事件捕获、分发和处理的主循环中。在开发代码时，我们通常会用一个 while 循环来作为这个主循环。然后在这个主循环中，我们需要捕获发生的事件、判断事件类型，并根据事件类型，调用在初始化时创建好的事件 handler 来实际处理事件。


## 三、Redis 对 Reactor 模型的实现
Redis 的网络框架实现了 Reactor 模型，代码位于 ae.h 和 ae.c 中。从ae.h 头文件中就可以看到，Redis 为了实现事件驱动框架，相应地定义了事件的数据结构、框架主循环函数、事件捕获分发函数、事件和 handler 注册函数。

Redis 的事件驱动模型定义了两类事件：IO 事件和时间事件，分别对应了客户端发送的网络请求和 Redis 自身的周期性操作。

### 3.1、aeEventLoop 结构体与初始化
首先来看下 Redis 事件驱动框架循环流程对应的数据结构 aeEventLoop。这个结构体是在事件驱动框架代码ae.h中定义的，记录了框架循环运行过程中的信息。
```c
typedef struct aeEventLoop {      // 基于事件驱动的程序的状态
    int maxfd;   // 当前注册的最高文件描述符
    int setsize; // 跟踪的文件描述符的最大数量
    long long timeEventNextId;
    int nevents; /* Size of Registered events 注册事件的大小*/
    aeFileEvent *events; /* Registered events IO事件数组 */
    aeFiredEvent *fired; /* Fired events 已触发事件数组 */
    aeTimeEvent *timeEventHead;   //记录时间事件的链表头, 即按一定时间周期触发的事件。
    int stop;
    void *apidata; /* This is used for polling API specific data 和API调用接口相关的数据 */
    aeBeforeSleepProc *beforesleep;   //进入事件循环流程前执行的函数
    aeBeforeSleepProc *aftersleep;    //退出事件循环流程后执行的函数
    int flags;
    void *privdata
} aeEventLoop;
```
aeEventLoop这个结构在Redis服务启动时就通过调用 aeCreateEventLoop 函数进行初始化了，个函数的参数只有一个，是 setsize。它的核心实现如下：
```c
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;
   
  if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;    //给eventLoop变量分配内存空间
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);         //给IO事件、已触发事件分配内存空间
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
    …
    eventLoop->setsize = setsize;
    eventLoop->lastTime = time(NULL);
    
    eventLoop->timeEventHead = NULL;    //设置时间事件的链表头为NULL
  …
  if (aeApiCreate(eventLoop) == -1) goto err;   //调用aeApiCreate函数，去实际调用操作系统提供的IO多路复用函数
    for (i = 0; i < setsize; i++)               //将所有网络IO事件对应文件描述符的掩码设置为AE_NONE
        eventLoop->events[i].mask = AE_NONE;
    return eventLoop;
 
    //初始化失败后的处理逻辑，
    err:
    …
}
```
aeCreateEventLoop 函数执行的操作，大致可以分成以下三个步骤。
* 第一步，aeCreateEventLoop 函数会创建一个 aeEventLoop 结构体类型的变量 eventLoop。然后，该函数会给 eventLoop 的成员变量分配内存空间，比如，按照传入的参数 setsize，给 IO 事件数组和已触发事件数组分配相应的内存空间。此外，该函数还会给 eventLoop 的成员变量赋初始值。
* 第二步，aeCreateEventLoop 函数会调用 aeApiCreate 函数。aeApiCreate 函数封装了操作系统提供的 IO 多路复用函数，假设 Redis 运行在 Linux 操作系统上，并且 IO 多路复用机制是 epoll，那么此时，aeApiCreate 函数就会调用 epoll_create 创建 epoll 实例，同时会创建 epoll_event 结构的数组，数组大小等于参数 setsize
* 第三步，aeCreateEventLoop 函数会把所有网络 IO 事件对应文件描述符的掩码，初始化为 AE_NONE，表示暂时不对任何事件进行监听。

### 3.2、IO事件
我们先看 IO 事件的结构定义：
```c
/* IO事件结构体 */
typedef struct aeFileEvent {
    int mask;  // mask 是用来表示事件类型的掩码， 包括可读事件、可写事件和屏障事件
    aeFileProc *rfileProc;  // 指向 AE_READABLE 事件的处理函数
    aeFileProc *wfileProc;  // 指向 AE_WRITABLE 事件的处理函数
    void *clientData;       // 指向客户端私有数据的指针
} aeFileEvent;
```
屏障事件的主要作用是用来反转事件的处理顺序。比如在默认情况下，Redis 会先给客户端返回结果，但是如果面临需要把数据尽快写入磁盘的情况，Redis 就会用到屏障事件，把写数据和回复客户端的顺序做下调整，先把数据落盘，再给客户端回复。

IO 事件的创建是通过 aeCreateFileEvent 函数来完成的，函数的原型定义，如下所示：
```c
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask, aeFileProc *proc, void *clientData)
```
这个函数的参数有 5 个，分别是循环流程结构体 *eventLoop、IO 事件对应的文件描述符 fd、事件类型掩码 mask、事件处理回调函数*proc，以及事件私有数据*clientData。紧接着，aeCreateFileEvent 函数会调用 aeApiAddEvent 函数，添加要监听的事件。



### 3.4、支撑模型运行的函数
支撑模型运行的函数包括框架主循环的 aeMain 函数、负责事件捕获与分发的 aeProcessEvents 函数，以及负责事件和 handler 注册的 aeCreateFileEvent 函数。我们逐个学习下：

#### 4.4.1、主循环：aeMain 函数
aeMain 函数的逻辑很简单，就是用一个循环不停地判断事件循环的停止标记。如果事件循环的停止标记被设置为 true，那么针对事件捕获、分发和处理的整个主循环就停止了；否则，主循环会一直执行。aeMain 函数的主体代码如下所示：
```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```
在redis服务启动的代码中，可以看到调用了aeMain函数。

#### 4.4.2、事件捕获与分发：aeProcessEvents 函数
aeProcessEvents 函数实现的主要功能，包括捕获事件、判断事件类型和调用具体的事件处理函数，从而实现事件的处理。代码如下：
```c
nt aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;   // 若没有事件处理，则立刻返回

    if (eventLoop->maxfd != -1 ||                                           // 如果有IO事件发生，或者紧急的时间事件发生，则开始处理
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        struct timeval tv, *tvp = NULL; /* NULL means infinite wait. */
        int64_t usUntilTimer;

        if (eventLoop->beforesleep != NULL && (flags & AE_CALL_BEFORE_SLEEP))
            eventLoop->beforesleep(eventLoop);

        if ((flags & AE_DONT_WAIT) || (eventLoop->flags & AE_DONT_WAIT)) {
            tv.tv_sec = tv.tv_usec = 0;
            tvp = &tv;
        } else if (flags & AE_TIME_EVENTS) {
            usUntilTimer = usUntilEarliestTimer(eventLoop);
            if (usUntilTimer >= 0) {
                tv.tv_sec = usUntilTimer / 1000000;
                tv.tv_usec = usUntilTimer % 1000000;
                tvp = &tv;
            }
        }
        
        numevents = aeApiPoll(eventLoop, tvp);      // 调用aeApiPoll函数捕获事件

        if (!(flags & AE_FILE_EVENTS)) {
            numevents = 0;
        }

        /* After sleep callback. */
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        for (j = 0; j < numevents; j++) {
            int fd = eventLoop->fired[j].fd;
            aeFileEvent *fe = &eventLoop->events[fd];
            int mask = eventLoop->fired[j].mask;
            int fired = 0; /* Number of events fired for current fd. */

            int invert = fe->mask & AE_BARRIER;

            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
            }

            if (fe->mask & mask & AE_WRITABLE) {      // 处理写事件
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            if (invert) {
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
                if ((fe->mask & mask & AE_READABLE) &&
                    (!fired || fe->wfileProc != fe->rfileProc))
                {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            processed++;
        }
    }

    if (flags & AE_TIME_EVENTS)      // 检查是否有时间事件，若有，则调用processTimeEvents函数处理
        processed += processTimeEvents(eventLoop);

    return processed; // 返回已经处理的文件或时间
}
```
如上所示，可以看到主要有三个 if 条件分支，分别是：
* 情况一：既没有时间事件，也没有网络事件；
* 情况二：有 IO 事件或者有需要紧急处理的时间事件；
* 情况三：只有普通的时间事件。

我们主要来看情况二，情况发生时，Redis 需要捕获发生的网络事件，并进行相应的处理。那么从 Redis 源码中我们可以分析得到，在这种情况下，aeApiPoll 函数会被调用，用来捕获事件。在 aeApiPoll 函数中直接调用了 epoll_wait 函数，并将 epoll 返回的事件信息保存起来的逻辑。

### 4.4.3、事件注册：aeCreateFileEvent 函数
当 Redis 启动后，服务器程序的 main 函数会调用 initSever 函数来进行初始化，而在初始化的过程中，aeCreateFileEvent 就会被 initServer 函数调用，用于注册要监听的事件，以及相应的事件处理函数。

AE_READABLE 事件就是客户端的网络连接事件，而对应的处理函数就是接收 TCP 连接请求。下面的示例代码中，显示了 initServer 中调用 aeCreateFileEvent 的部分片段
```c
void initServer(void) {
    …
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic("Unrecoverable error creating server.ipfd file event.");
            }
  }
  …
}
```
aeCreateFileEvent 如何实现事件和处理函数的注册呢？首先，Linux 提供了 epoll_ctl API，用于增加新的观察事件。而 Redis 在此基础上，封装了 aeApiAddEvent 函数，对 epoll_ctl 进行调用。aeCreateFileEvent 就会调用 aeApiAddEvent，然后 aeApiAddEvent 再通过调用 epoll_ctl，来注册希望监听的事件和相应的处理函数。
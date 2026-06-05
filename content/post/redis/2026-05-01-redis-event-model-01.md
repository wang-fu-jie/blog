---
title:       Redis源码阅读 - Redis事件驱动模型(一)
subtitle:    "Redis事件驱动模型(一)"
description: "Redis在进行网络通信的实现时分为了四层，其中网路基础设施封装了socket模型，也封装了sock套接字。日常我们大多使用TCP协议，因此本文只重点介绍了对Socket的封装。 Redis使用了事件循环机制，底层利用了IO多路复用。考虑到对不同平台的兼容，提供了统一的API接口。"
excerpt:     "Redis在进行网络通信的实现时分为了四层，其中网路基础设施封装了socket模型，也封装了sock套接字。日常我们大多使用TCP协议，因此本文只重点介绍了对Socket的封装。 Redis使用了事件循环机制，底层利用了IO多路复用。考虑到对不同平台的兼容，提供了统一的API接口。"
date:        2026-05-01T16:46:08+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/b8/72/coastline_coast_beach_sea_ocean-99120.jpg!d"
published:   true
tags:
    - Redis
slug:        "redis-event-model-01"
categories:  [ "REDIS" ]
---

## 一、引言
Redis是一个C/S架构的数据库，通常系统实现网络通信的基本方法是使用 Socket 编程模型。但是，由于基本的 Socket 编程模型一次只能处理一个客户端连接上的请求，所以当要处理高并发请求时，一种方案就是使用多线程，让每个线程负责处理一个客户端的请求。

处理并发请求的另一种方案就是IO多路复用，Linux 操作系统，就提供了 select、poll 和 epoll 三种编程模型。运行在linux上的Redis通常就使用 epoll 模型。

## 二、为什么不直接使用socket模型
使用用 Socket 模型实现网络通信时，需要经过创建 Socket、监听端口、处理连接和读写请求等多个步骤。首先，当我们需要让服务器端和客户端进行通信时，可以在服务器端通过以下三步，来创建监听客户端连接的监听套接字（Listening Socket）：
* 调用 socket 函数，创建一个套接字。我们通常把这个套接字称为主动套接字（Active Socket）；
* 调用 bind 函数，将主动套接字和当前服务器的 IP 和监听端口进行绑定；
* 调用 listen 函数，将主动套接字转换为监听套接字，开始监听客户端的连接。

在完成上述三步之后，服务器端就可以接收客户端的连接请求了。为了能及时地收到客户端的连接请求，我们可以运行一个循环流程，在该流程中调用 accept 函数，用于接收客户端连接请求。

accept 也是阻塞函数，如果此时一直没有客户端连接请求，服务器端的执行流程会一直阻塞在 accept 函数。一旦有客户端连接请求到达，accept 将不再阻塞，而是处理连接请求，和客户端建立连接，并返回已连接套接字（Connected Socket）。最后，服务器端可以通过调用 recv 或 send 函数，在刚才返回的已连接套接字上，接收并处理读写请求，或是将数据发送给客户端。
```c
listenSocket = socket(); //调用socket系统调用创建一个主动套接字
bind(listenSocket);  //绑定地址和端口
listen(listenSocket); //将默认的主动套接字转换为服务器使用的被动套接字，也就是监听套接字
while (1) { //循环监听是否有客户端连接请求到来
   connSocket = accept(listenSocket); //接受客户端连接
   recv(connsocket); //从客户端读取数据，只能同时处理一个客户端
   send(connsocket); //给客户端返回数据，只能同时处理一个客户端
}
```
上述代码中，虽然它能够实现服务器端和客户端之间的通信，但是程序每调用一次 accept 函数，只能处理一个客户端连接。因此，如果想要处理多个并发客户端的请求，就需要使用多线程的方法，来处理通过 accept 函数建立的多个客户端连接上的请求。

但是对于Redis，主执行流程是由一个线程在执行，无法使用多线程的方式来提升并发处理能力。 因此Redis并没有直接使用socket模型。

## 三、Redis对socket模型的封装
Socket 模型提供了最基本的网络通信机制，因此尽管Redis使用了 epoll 模型，本质还是需要通过 socket 进行网络通信。因此Redis在创建服务器时封装了socket，后边我们结合代码实际体会下。

### 3.1、Redis服务端整体架构
Redis 服务端网络层的实现架构分为了四层，如下：
```
┌─────────────────────────────────────────────────────┐
│                    networking.c                     │
│   协议处理 + 客户端管理 + 读写逻辑                      │
│   readQueryFromClient()  /  writeToClient()         │
├─────────────────────────────────────────────────────┤
│                    connection.c                     │
│   连接抽象层 (Redis 6.0+)                             │
│   connRead() / connWrite() / connAccept()           │
├─────────────────────────────────────────────────────┤
│                      anet.c                         │
│   网络基础设施：socket 创建、bind、listen、选项设置      │
│   anetTcpServer() / anetTcpAccept() / anetNonBlock()│
├─────────────────────────────────────────────────────┤
│                      ae.c                           │
│   事件循环：epoll/kqueue/select 封装                   │
│   aeCreateFileEvent() / aeMain()                    x│
└─────────────────────────────────────────────────────┘
```
* ae.c — 事件循环，基于 Reactor 模式使用IO多路复用创建的事件循环
* anet.c — 网络基础设施，负责创建和配置 socket，不参与数据读写。
* connection.c — 连接抽象层
* networking.c — 业务逻辑

以上我们对这四层先做一个了解，后续章节再逐层次进行介绍。

### 3.2、网络基础设施层
我们知道socket模型开发服务端需要多个步骤，redis封装了整个流程，提供一个对外的函数。在anet.c、anet.h 源文件中实现， 这两个文件中除了实现了基于socket的通信，还实现了基于unixsock的通信模型，因为我们大多都是使用socket，这里不对unix套接字做过多介绍。
```c
static int _anetTcpServer(char *err, int port, char *bindaddr, int af, int backlog)   // 封装一个服务端，设置地址和绑定端口
{
    int s = -1, rv;
    char _port[6];  /* strlen("65535") */  // 存储端口号的字符串
    struct addrinfo hints, *servinfo, *p;

    memset(&hints,0,sizeof(hints));
    hints.ai_family = af;
    hints.ai_socktype = SOCK_STREAM;       // 设置协议为TCP
    hints.ai_flags = AI_PASSIVE;

    if ((rv = getaddrinfo(bindaddr,_port,&hints,&servinfo)) != 0) {  // 调用 getaddrinfo 获取可用于 bind 的地址列
        anetSetError(err, "%s", gai_strerror(rv));
        return ANET_ERR;
    }
    for (p = servinfo; p != NULL; p = p->ai_next) {       // 遍历地址列表尝试绑定
        if ((s = socket(p->ai_family,p->ai_socktype,p->ai_protocol)) == -1)     // 创建一个主动套接字
            continue;

        if (af == AF_INET6 && anetV6Only(err,s) == ANET_ERR) goto error;
        if (anetSetReuseAddr(err,s) == ANET_ERR) goto error;     // 设置端口重用
        if (anetListen(err,s,p->ai_addr,p->ai_addrlen,backlog,0) == ANET_ERR) s = ANET_ERR;     // 绑定ip并 监听端口
        goto end;
    }

error:
    if (s != -1) close(s);
    s = ANET_ERR;
end:
    freeaddrinfo(servinfo);
    return s;
}
```
如上所示，为redis创建一个服务端的通用实现函数。这是粘贴的代码中笔者删掉了部分错误处理代码以节省篇幅。在Redis对套接字封装的实现中，还实现了 accept、connect等函数，包括阻塞模式和非阻塞模式。这里我们不做过多展开。


## 四、IO多路复用

### 4.1、select模型
Linux 针对每一个套接字都会有一个文件描述符，用来唯一标识该套接字。在多路复用机制的函数中，Linux 通常会用文件描述符作为参数。有了文件描述符，函数也就能找到对应的套接字，进而进行监听、读写等操作。

第一种实现IO多路复用的是select模型，它提供一个select函数，原型如下：
```c
int select (int __nfds, fd_set *__readfds, fd_set *__writefds, fd_set *__exceptfds, struct timeval *__timeout)
```
select 函数使用三个集合，表示监听的三类事件，分别是读数据事件（对应readfds集合）、写数据事件（对应writefds集合）和异常事件（对应__exceptfds集合）。fd_set 结构体的定义，其实就是一个 long int 类型的数组，该数组中一共有 32 个元素（1024/32=32），每个元素是 32 位（long int 类型的大小），而每一位可以用来表示一个文件描述符的状态。因此，select 函数对每一个描述符集合，都可以监听 1024 个描述符。

select函数一次监听1024个描述符，每次调用会返回已经就绪的描述符数量。因此我们需要对这些描述符进行循环，判断是哪个已经就绪并进行处理。因此可以看到select模型的两个显著缺陷：
* select 函数对单个进程能监听的文件描述符数量是有限制的，它能监听的文件描述符个数由 __FD_SETSIZE 决定，默认值是 1024。
* 当 select 函数返回后，我们需要遍历描述符集合，才能找到具体是哪些描述符就绪了。这个遍历过程会产生一定开销，从而降低程序的性能

### 4.2、pool模型
为了解决select模型的缺陷，linux又提出了pool模型。主要函数是 poll 函数，我们先来看下它的原型定义，如下所示：
```c
int poll (struct pollfd *__fds, nfds_t __nfds, int __timeout);
```
，参数 *__fds 是 pollfd 结构体数组，参数 nfds 表示的是 *fds 数组的元素个数，而 __timeout 表示 poll 函数阻塞的超时时间。pollfd 结构体里包含了要监听的描述符，以及该描述符上要监听的事件类型。

在使用pool机制时，一样需要循环调用pool函数检查文件描述符是否有就绪的，返回就绪的文件描述符个数。不同的是，它允许一次监听超过 1024 个文件描述符。是当调用了 poll 函数后，我们仍然需要遍历每个文件描述符，检测该描述符是否就绪。

### 4.3、epool模型
为了解决遍历问题，linux再次提出了epool模型。epoll 机制是使用 epoll_event 结构体，来记录待监听的文件描述符及其监听的事件类型的，这和 poll 机制中使用 pollfd 结构体比较类似。

对于 epoll_event 结构体来说，其中包含了 epoll_data_t 联合体变量，以及整数类型的 events 变量。epoll_data_t 联合体中有记录文件描述符的成员变量 fd，而 events 变量会使用不同的宏定义值，来表示 epoll_data_t 变量中的文件描述符所关注的事件类型，比如一些常见的事件类型包括以下这几种:
* EPOLLIN：读事件，表示文件描述符对应套接字有数据可读。
* EPOLLOUT：写事件，表示文件描述符对应套接字有数据要写。
* EPOLLERR：错误事件，表示文件描述符对于套接字出错。

下面的代码展示了 epoll_event 结构体以及 epoll_data 联合体的定义:
```c
typedef union epoll_data
{
  ...
  int fd;  //记录文件描述符
  ...
} epoll_data_t;

struct epoll_event
{
  uint32_t events;  //epoll监听的事件类型
  epoll_data_t data; //应用程序数据
};
```
我们知道，在使用 select 或 poll 函数的时候，创建好文件描述符集合或 pollfd 数组后，就可以往数组中添加我们需要监听的文件描述符。 但是对于 epoll 机制来说，我们则需要先调用 epoll_create 函数，创建一个 epoll 实例。这个 epoll 实例内部维护了两个结构，分别是记录要监听的文件描述符和已经就绪的文件描述符，而对于已经就绪的文件描述符来说，它们会被返回给用户程序进行处理。所以，我们在使用 epoll 机制时，就不用像使用 select 和 poll 一样，遍历查询哪些文件描述符已经就绪了。

在创建了 epoll 实例后，我们需要再使用 epoll_ctl 函数，给被监听的文件描述符添加监听事件类型，以及使用 epoll_wait 函数获取就绪的文件描述符。如下为 epool机制的使用流程示例：
```c
int sock_fd,conn_fd; //监听套接字和已连接套接字的变量
sock_fd = socket() //创建套接字
bind(sock_fd)   //绑定套接字
listen(sock_fd) //在套接字上进行监听，将套接字转为监听套接字
  
epfd = epoll_create(EPOLL_SIZE); //创建epoll实例，
//创建epoll_event结构体数组，保存套接字对应文件描述符和监听事件类型  
ep_events = (epoll_event*)malloc(sizeof(epoll_event) * EPOLL_SIZE);

//创建epoll_event变量
struct epoll_event ee
//监听读事件
ee.events = EPOLLIN;
//监听的文件描述符是刚创建的监听套接字
ee.data.fd = sock_fd;

//将监听套接字加入到监听列表中  
epoll_ctl(epfd, EPOLL_CTL_ADD, sock_fd, &ee); 
  
while (1) {
   //等待返回已经就绪的描述符 
   n = epoll_wait(epfd, ep_events, EPOLL_SIZE, -1); 
   //遍历所有就绪的描述符   
   for (int i = 0; i < n; i++) {
       //如果是监听套接字描述符就绪，表明有一个新客户端连接到来 
       if (ep_events[i].data.fd == sock_fd) { 
          conn_fd = accept(sock_fd); //调用accept()建立连接
          ee.events = EPOLLIN;  
          ee.data.fd = conn_fd;
          //添加对新创建的已连接套接字描述符的监听，监听后续在已连接套接字上的读事件    
          epoll_ctl(epfd, EPOLL_CTL_ADD, conn_fd, &ee); 
            
       } else { //如果是已连接套接字描述符就绪，则可以读数据
           ...//读取数据并处理
       }
   }
}
```

### 2.2、Redis的IO多路复用实现
Redis 在设计和实现网络通信框架时，就基于 epoll 机制中的 epoll_create、epoll_ctl 和 epoll_wait 等函数和读写事件，进行了封装开发，实现了用于网络通信的事件驱动框架，从而使得 Redis 虽然是单线程运行。Redis的实现主要位于 ae_epool.c 中，如下：
```c
typedef struct aeApiState {
    int epfd;                    // 保存epoll实例的文件描述符
    struct epoll_event *events;  // epoll_event数组，内核把就绪事件写到这里。
} aeApiState;

static int aeApiCreate(aeEventLoop *eventLoop) {
    aeApiState *state = zmalloc(sizeof(aeApiState));
    if (!state) return -1;
    state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);  // 根据事件循环的大小申请内存
    if (!state->events) {
        zfree(state);
        return -1;
    }
    state->epfd = epoll_create(1024); /* 1024 是一个提示值， */
    if (state->epfd == -1) {
        zfree(state->events);
        zfree(state);
        return -1;
    }
    anetCloexec(state->epfd);   // 设置 CLOEXEC 如果 Redis 调用：fork或者 exec 执行新程序时，自动关闭 epfd。否则子进程继承了epfd会导致fd泄露
    eventLoop->apidata = state;
    return 0;
}
```
如上展示了创建一个 epool 实例的实现。核心调用了 epoll_create 创建实例，并把实例描述符和事件数组封装到 aeApiState 结构中。接下来我们看下是如何添加一个事件的：
```c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee = {0};  // 避免未初始化数据 
    int op = eventLoop->events[fd].mask == AE_NONE ?    // 判断 ADD 还是 MOD
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* 合并旧事件 */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;  // 调用 epoll_ctl 实现监听
    return 0;
}
```
这个函数实现了把一个 fd 注册到 epoll，或者修改这个 fd 已经注册的监听事件。在Redis中自定义了事件类型 AE_READABLE、AE_WRITABLE，因此在实际监听时需要转换为linux定义的EPOLLIN 和 EPOLLOUT。同样的Redis封装了删除时间监听以及释放epool实例的函数，请自行阅读代码。

这里你可能会好奇aeEventLoop是什么样的结构，因为我们这里并未做介绍，它定义在ae.c文件中。 aeEventLoop 是 Redis 的“事件循环对象”，保存了整个 Redis 运行时所有网络事件、时间事件以及底层 IO 多路复用状态，redis利用它实现的时间循环。 在这一节我们主要介绍redis多epool的封装，下一篇文件我们重点来聊聊aeEventLoop这个结构。


## 五、总结
Redis在进行网络通信的实现时分为了四层，其中网路基础设施封装了socket模型，也封装了sock套接字。日常我们大多使用TCP协议，因此本文只重点介绍了对Socket的封装。 Redis使用了事件循环机制，底层利用了IO多路复用。考虑到对不同平台的兼容，提供了统一的API接口，如aeApiCreate。 
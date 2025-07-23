---
title:       Python基础 - IO模型
subtitle:    "IO模型"
description: "客户端和服务端通信时，设计数据准备和数据copy两个过程，这过程中会遇到网络IO，前面我们默认都是阻塞IO，但是阻塞IO的效率较低。本文中我们会介绍非阻塞IO和IO多路复用，以及它们的代码实现。"
excerpt:     "客户端和服务端通信时，设计数据准备和数据copy两个过程，这过程中会遇到网络IO，前面我们默认都是阻塞IO，但是阻塞IO的效率较低。本文中我们会介绍非阻塞IO和IO多路复用，以及它们的代码实现。"
date:        2025-07-23T10:35:09+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/10/13/couple_cloudy_conversation_seaside_date-29746.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-io-model"
categories:  [ "PYTHON" ]
---

## 一、IO模型
这里研究的IO模型指的是网络IO。客户端和服务端进行通信，需要经历以下过程客户端发送数据拷贝到系统内核，系统将数据通过网线发送到目标系统，再把数据从目标系统拷贝到服务端程序。操作系统之间传输的过程叫做等待数据准备。

在网络编程中，我们使用到的 accept, recv等都属于网络IO操作。

### 1.1、阻塞IO
阻塞IO是我们第一个研究的IO模型，阻塞 I/O 是最基本的 I/O 模型，指的是当进程发起I/O操作（如 read()、write()）时，如果数据未就绪，进程会被挂起（阻塞），直到 I/O 操作完成才继续执行。

### 1.2、非阻塞IO
非阻塞IO做的事情是尽可能的将CPU执行权抓在自己手里。例如recv接收数据，如果操作系统没有数据，就立刻返回。然后客户端执行其他操作，或者轮询请求数据直到有数据返回。这样CPU就不会因为有IO等待让出执行权。

上一章节的协程就是一个非阻塞IO的实现，协程就是遇到阻塞就进行代码层切换，让CPU以为我们的程序没有IO。


### 1.3、非阻塞IO实现
我们利用非阻塞IO来实现一个TCP的服务端，代码如下：
```python
import socket

server = socket.socket()
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('127.0.0.1', 8080))
server.listen(5)
server.setblocking(False)  # 所有的网络阻塞都会变成非阻塞

c_list = []
d_list = []
while True:
    try:
        conn, addr = server.accept()
        c_list.append(conn)   // 有客户端建链就添加到列表
    except BlockingIOError:
        for conn in c_list:
            try:
                data = conn.recv(1024)  // 接收客户端数据，如果数据为空就断开连接，加入到待删除列表
                if not data:
                    conn.close()
                    d_list.append(conn)
                conn.send(data.upper())
            except BlockingIOError:   // 如果没有数据发送过来，即客户端也在阻塞，就什么也不做
                pass
            except ConnectionError:   // 在windows平台会抛出这个异常
                conn.close()
                d_list.append(conn)

        for conn in d_list:
            c_list.remove(conn)
        d_list.clear()
```
注意：这里的服务端因为被修改为了非阻塞模型，如果不做异常处理，例如conn，没有客户连上来，就会报错。我们这个代码虽然解决了阻塞IO的问题，但是仍然存在一个缺点，就是如果没有客户端进行通信，那CPU就会持续空转，对CPU资源时一种浪费。这个示例是为了帮助大家理解非阻塞IO，现实开发中并不会这么做。


## 二、IO多路复用
IO多路复用是操作系统提供一个select监管机制，我们把可能进入IO的对象提供给这个监管机制，比如socket对象、conn对象。这个监管机制会等待多个套接字对象其中一个变成可读，这时监管机制就会给我们返回相应的对象，我们就可以拿这个对象去读取数据。

对于IO多路复用来说，如果监管的对象只有一个，那它的效率甚至会低于阻塞IO，因为它相比阻塞IO多了监听并返回可读对象的步骤。但是它监管的对象很多的时候，它的效率就比阻塞IO高很多。

### 2.1、select模块
select是操作系统提供的监管机制，使用这个监管机制，也需要导入select模块。
```python
import socket
import select

server = socket.socket()
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('127.0.0.1', 8080))
server.listen(5)
server.setblocking(False)

input_list = [server]

while True:
    rlist, wlist, xlist = select.select(input_list, [], [])
    for i in rlist:
        if i is server:
            conn, addr = i.accept()
            input_list.append(conn)
            continue

        data = i.recv(1024)
        if not data:
            i.close()
            input_list.remove(i)
            continue
        i.send(data.upper())
```
如上，我们实现一个select的IO多路复用，将socker对象加入到监管机制，如果所有对象都在等待，那代码将会在select.select 这里阻塞，如果其中一个对象可读了，代码才会继续向下执行。

select需要传入三个列表参数，并且返回三个列表参数。分别是rlist 可读对象列表, wlist 可写对象列表, xlist 异常对象列表。我们值使用到可读对象列表，因为针对写操作是立即执行的，基本不会长时间阻塞。

### 1.2、select、poll与epoll
select监管机制有三种实现，上边我们的示例代码使用的是select，还有poll和epoll。select最大监控1024个对象，pool和epool则没有限制。select 和 poll 都是会轮循监管的对象，依次询问是否可读，因此当监管的对象比较多的时候，可能会出现极大的延迟响应。

但是epoll就不同了，epoll提供了一个就绪链表，它给每个监管对象都绑定了一个回调机制，一旦有响应回调机制就会把可读对象放进就绪链表。因此epoll只需要判断监管对象是否为空即可，不需要每次把监管对象进行遍历。poll和epoll也被封装到了select模块中，可以像select一样使用
```python
# select.poll
# select.epoll
```

### 1.3、selectors模块
poll和epool只有linux支持，windows只支持select。那如何实现跨平台呢，可以在代码中进行判断，如果运行的平台是windows，那就使用select。但是这个功能已经有人壮壮好了，那就是selectors模块。
```python
import socket
import selectors

def accept(server):
    conn, addr = server.accept()
    sel.register(conn, selectors.EVENT_READ, read)

def read(conn):
    data = conn.recv(1024)
    if not data:
        conn.close()
        sel.unregister(conn)
        return
    conn.send(data.upper())


server = socket.socket()
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('127.0.0.1', 8080))
server.listen(5)
server.setblocking(False)

sel = selectors.DefaultSelector()     // 创建selectors对象
sel.register(server, selectors.EVENT_READ, accept)  // 给socker对象注册监听，accept是回调函数
while True:
    events = sel.select()
    for key, mask in events:
        callback = key.data
        callback(key.fileobj)
```
如上所示为selectors的使用方式，根据不同的平台它会自主选择IO多路复用的实现。我们需要不断的循环获取就绪链表，从就绪链表中获取可读事件，然后来做响应的处理。


### 三、总结
本文中我们使用了三个IO模型，分别为阻塞IO，非阻塞IO和IO多路复用。除了这三个IO模型外，还有一个异步IO，我们下一篇文章再详细说一下异步IO以及异步IO的实现。
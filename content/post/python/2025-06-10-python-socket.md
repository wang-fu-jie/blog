---
title:       Python基础 - socket套接字
subtitle:    "socket套接字"
description: "socket套接字帮我们封装了网络通信的协议，直接使用socket可以很简单的实现网络编程。本文将基于socket套接字实现一个简单的C/S程序，并解决TCP协议的粘包问题，以及通过标准库的sockerserver模块实现客户端的并发服务。"
excerpt:     "socket套接字帮我们封装了网络通信的协议，直接使用socket可以很简单的实现网络编程。本文将基于socket套接字实现一个简单的C/S程序，并解决TCP协议的粘包问题，以及通过标准库的sockerserver模块实现客户端的并发服务。"
date:        2025-06-10T17:41:45+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/13/5e/moose_bull_elk_yawns-760371.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-socket"
categories:  [ "PYTHON" ]
---

## 一、socket套接字
早期的套接字是用来解决一台计算机上多个进程间通信的，最早进程间通信是基于文件(套接字类型AF_UNIX)，AF是地址家族的缩写。有了互联网之后就有了另一个套接字家族AF_INET，基于网络通信的套接字家族。接下来就可以基于套接字编写网络通信的程序了。

### 2.1、服务端
C/S架构需要服务端和客户端，我们先编写服务端的代码，如下：
```python
import socket
sk = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  # 创建套接字对象 socket.SOCK_STREAM是流式协议（即tcp协议）
sk.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)  # 这行代码可以重用端口
sk.bind(('127.0.0.1', 5002))                             # 绑定地址
sk.listen(5)  # 监听连接请求， 5是半连接池大小
print('服务端启动成功，在5002端口等待连接')

while True:
    conn, addr = sk.accept()     # 取出连接请求，开始服务
    print('连接对象：', conn)
    print('客户端ip+端口', addr)
   
    while True:   # 5、数据传输
        data = conn.recv(1024)  # 1024是一次接收的数据量
        if not data:        # 如果客户端发空或者异常断开，就退出。
            break
        data = data.decode('utf-8')
        print('客户端发过来的数据：', data)
        conn.send(data.upper().encode('utf-8'))
    conn.close()      # 结束服务
# sk.close()   # 可选，相当于把服务器关了
```
如上就完成了服务端的编写，服务端采用的是TCP协议。这里的代码，重用端口是因为服务端停止后，操作系统不会立刻回收端口，再次启动会报错端口已被占用。套接字提供的accept、recv、send都是阻塞方法，即如果没有客户端连接，那代码会一直阻塞在accept这里。

还有一点需要说明的是，当客户端异常断开连接后，服务端会进入异常状态。这个异常状态并不是抛异常，而是客户端的recv会一直循环收空数据，不再处于阻塞状态。这里说的客户异常断开后，服务端进入循环收空状态，是在Liunx和Mac系统上的行为，在Windows上服务端会直接抛出异常，因此在Windows上只能通过try来解决。

半连接池的作用是暂时存储客户端的连接，客户端发起连接后三次握手完成进入半连接池，需要等待服务端从半连接池取出并进行通信，即accept函数会从半连接池取出连接。半连接池不宜设置过大。因为真正是要提升服务端的并发能力，半连接池过大，虽然可以更多的客户端连上来，但是服务端并没有处理，并不会提升客户端的体验。目前我们的服务端只能响应一个客户端因为我们还没有学并发编程。如果半连接池满了再有客户端连接就会报超时错误。

### 2.2、客户端
完成了服务端，就可以写客户端了，代码如下：
```python
import socket
sk = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  # 流式协议（tcp协议）
sk.connect(('127.0.0.1', 5002))
while True:
    msg = input('请输入内容>>>')
    sk.send(msg.encode('utf-8'))
    if not msg:
        continue
    if msg == 'q':
        break
    data = sk.recv(1024)
    print(data.decode('utf-8'))
sk.close()
```
这里说明一下为什么要判断发送的数据是否为msg， 这里因为如果发送数据是发空时可以正常发送的，但是服务端收不到空(只有异常断连服务端才会收空)。因此客户端发送完空进行接收状态阻塞，但是服务端收不到空也在阻塞，就都进入了阻塞状态。那为什么可以发空呢，原因是执行send时其实调用了系统调用，让操作系统把缓存中的数据发出去，但是缓存中并没有数据，因此操作系统并没有真的发送，但是send这个系统调用确实执行了。


## 二、UDP套接字
UDP协议又称数据报协议，它不需要建立连接，直接发送数据即可。
```python
## UDP服务端
import socket
sk = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)  # 数据包协议（tcp协议）
sk.bind(('127.0.0.1', 5002))
while True:
    data, addr = sk.recvfrom(1024)  # 1024是一次接收的数据量
    print('客户端发过来的数据：', data)
    sk.sendto(data.upper(), addr)

## UDP客户端
import socket
sk = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)  # 数据包协议（tcp协议）
while True:
    msg = input('请输入内容>>>')
    if msg == 'q':
        break
    sk.sendto(msg.encode('utf-8'), ('127.0.0.1', 5002))
    data, addr = sk.recvfrom(1024)
    print(data)
sk.close()
```
如上所示，UDP协议的客户端和服务端实现，UDP协议不需要监听和连接，直接发送数据即可。和TCP不同的是，UDP是可以发空的，服务端会直接回空。因为UDP是数据报协议即使没有数据也会发送一次。但是TCP是流式协议，数据从缓存中获取，一次发送可以分多次接收， 也可以多次发送一次接收， 因为最终是从缓存中读取数据。


## 三、粘包问题
在第一章节中，我们写了TCP的客户端和服务端，但是这里存在一个问题，就是我们每次发送和接收的数据都是1024字节，那如果超过1024字节就会出问题，这就是粘包。粘包问题的根本原因是一次发送的数据没有收完, 下次再接收就会继续接收上次剩余的数据，并和本次的数据粘在一起发过来。

没有收完有几种原因，第一种是我们设置接收的字节比发送的少，这种情况就需要多次接收直到收完。 第二种是接收太快，服务端是刚发了一部分客户端就已经接收了，这种情况接口需要客户端判断是否接受完成，即使用自定义协议，发送的数据加上结束标识。第三张情况是客户端缓存有限，发送的数据量超过缓存大小，解决这个问题也是多次接收直到数据接收完。


### 3.2、自定义协议
上边说了，解决粘包问题就需要判断发送了多少长度，保证接收完成，或者加上数据结束标识。这种动作就属于自定义协议，因为我们自定义了要发生的数据内容。 我们这里实现一个下载文件的功能，这里为了代码简洁只贴一下自定义协议的代码
```python
header = {
    'filename': 'a.mp4',
    'filesize': 111111,
    'md5': uiswqhdiwhckishdi
}
header = json.dumps(header).encode('utf-8')
headder_n = bytes(str(len(header)), 'utf-8').zfill(4)    # 固定长度，这样客户端就可以知道获取多少了
conn.send(headder_n)
conn.send(header)
cond.send(data)
```
如上所示，我们要发生文件，发送文件名大小和md5用于校验。因为头部大小不固定，因此先发送头部长度，客户端读取指定长度的头部来解析头部，通过头部获取到了文件的长度，接下来就可以读取知道大小的字节了。 这里我们只贴了解决粘包的代码实现，其他部分可以自行编写。

## 四、UDP不粘包
UDP没有粘包问题，UDP协议每一条消息都是完整的报文。 那如果UDP发送的数据量大，但是客户端接受的少会出现什么问题呢？ UDP协议一次发送就必须对应一次接收。假设发送了5个字节，但是只接受了3个，另外两个就会被认为丢掉不要了，再次接受也是收到新的消息，这是在linux上的行为。 在windows上会直接报错用户接受数据的缓冲区比数据报小。UDP最大长度是1472字节，因此也一般不会使用UDP发送大数据。


## 五、 socketserver介绍
前面我们实现的服务端只能同时响应一个客户端。如果想是服务端同时响应多个客户端，就需要用到并发编程，除了使用多线程多进程并发技术外，也可以使用封装好的模块socketserver来实现并发效果。socketserver 是 Python 标准库中的一个模块，它简化了网络服务器的创建过程，是对底层 socket 模块的高级封装。

socketserver模块支持线程并发也支持进程并发，进程并发使用的fork方式，属于unix的系统调用，因此windows不支持，接下来的示例代码我们使用线程并发。

### 5.1、TCP并发服务端
基于socketserver就可以实现TCP的并发客户端，使用socketserver要求必须定义一个类并实现handle方法，该方法用来实现数据通信。代码如下：
```python
import socketserver

class RequestHandle(socketserver.BaseRequestHandler):
    def handle(self):
        print(self.request)  # self.request = conn
        print(self.client_address)
        while True:
            data = self.request.recv(1024)  # 1024是一次接收的数据量
            if not data:
                break
            data = data.decode('utf-8')
            print('客户端发过来的数据：', data)

            self.request.send(data.upper().encode('utf-8'))

sk = socketserver.ThreadingTCPServer(('127.0.0.1', 5002), RequestHandle)  # 这是使用的线程，也可以使用进程
sk.serve_forever()   # 每获取一个连接对象，就启动一个线程
```
如上我们就实现了TCP并发服务端，每连接一个客户端服务端就会启动一个新线程用于和客户端通信，该线程自动调用handle方法。客户端的代码不需要修改，因为客户端本身是不需要并发的。


### 5.2、UDP并发客户端
在原来的UDP中，启动服务端后再启动两个客户端，此时两个客户端发出消息服务都能进行回应，而不是像TCP那样第二个客户端直接阻塞。看起来UDP默认就支持并发，其实这不是真正的并发，是因为服务端响应比较快，在短时间内分别响应了多个客户端，如果其中一个客户端数据量比较大，那另一个客户端会阻塞等待响应，可以在客户端使用time模块阻塞来进行模拟延迟响应。

UDP真正的并发实现也需要使用sockerserver模块，代码如下：
```python
import socketserver

class RequestHandle(socketserver.BaseRequestHandler):
    def handle(self):
        print('客户端发来的数据：', self.request[0])
        print(self.request[1].sendto(self.request[0].upper(), self.client_address))

sk = socketserver.ThreadingUDPServer(('127.0.0.1', 5006), RequestHandle)
sk.serve_forever()
```
如上所示，就实现UDP并发了，sk.serve_forever()接收数据后，调用handle方法进行处理。

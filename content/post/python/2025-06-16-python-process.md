---
title:       Python基础 - 进程
subtitle:    "Python基础 - 进程"
description: "进程就是正在运行的程序实例，也是操作系统进行资源分配和调度的基本单位，在python中有多种创建进程的方式，但是主要通过multiprocessing模块实现。多进程编程中会存在共享资源访问，进程间通信等问题，分别使用互斥锁和消息队列解决，最后本文将介绍设计模型生产者-消费者模型。"
excerpt:     "进程就是正在运行的程序实例，也是操作系统进行资源分配和调度的基本单位，在python中有多种创建进程的方式，但是主要通过multiprocessing模块实现。多进程编程中会存在共享资源访问，进程间通信等问题，分别使用互斥锁和消息队列解决，最后本文将介绍设计模型生产者-消费者模型。"
date:        2025-06-16T09:50:59+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/93/16/desk_office_business_apple_iphone_cell_mac_keyboard-892309.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-process"
categories:  [ "PYTHON" ]
---

## 一、进程介绍
进程就是正在运行的程序实例，是一个静态的概念，而是程序在计算机中动态执行的过程。进程是操作系统进行资源分配和调度的基本单位，操作系统以进程为单位分配CPU时间、内存等资源。

进程的运行有三种状态，在进程创建好后进入就绪态等待CPU执行，当进程获取到执行权时进入执行态进行执行，进程遇到IO或者时间片执行完后进行阻塞态，当阻塞态时间完成再进入就绪态等待执行，直到程序执行完终止释放资源。

### 1.1、同步和异步
同步和异步是用来描述任务的提交方式的。同步是任务提交之后，原地等待任务的放回结果，等待的过程中不做任何事情。异步指任务提交后，去做其他事情，等任务完成后结果通过异步回调机制自动进行处理。

### 1.2、阻塞与非阻塞
阻塞与非阻塞描述的是进程的运行状态。阻塞就是进程处于阻塞态，非阻塞就是进程处于运行态或就绪态。同步和阻塞经常会结合使用，例如异步非阻塞是效率最高的。

## 二、创建进程
创建进程的方式有三种，分别的fork系统调用，subprocess模块和multiprocessing模块。os.fork在windows上不支持，因此一般不会使用这个模块。subprocess一般于启动和控制外部进程， 而multiprocessing模块用于实现多进程，因此我们重点介绍multiprocessing模块

multiprocessing模块创建进程的方式有两种，先看第一种方式：
```python
from multiprocessing import Process
import time

def func(name):
    print(f'{name}任务开始')
    time.sleep(5)
    print(f'{name}任务执行完毕')

p = Process(target=func, args=('写讲话稿',))   # 1、得到进程操作对象
p.start()                   # 2、创建进程
print('主进程')
```
该实例的运行结果是会先打印主进程，然后再执行子进程的函数。这个代码在windows执行会报错，这是因为在linux中创建子进程时会把当前进程的代码和数据拷贝一份，然后在子进程执行我们的任务。但是在windows中创建子进程会使用类似模块导入的方式把当前文件执行一遍，因此又执行了创建进程的代码，进入死循环，在windows中创建进程的代码加上if __name__ == '__main__':即可解决。接下来看创建进程的第二种方式：
```python
from multiprocessing import Process
import time

class MyProcess(Process):
    def __init__(self, name):
        super().__init__()
        self.task_name = name
    def run(self):
        print(f'{self.task_name}任务开始')
        time.sleep(5)
        print('任务结束')

if __name__ == '__main__':
    p = MyProcess('约会')
    p.start()
    print('主进程')
```
第二种方式通过定义类继承Process实现，必须实现run方法。总结创建进程就是申请一块内存空间，把需要运行的代码放进去，它们彼此之间隔离，进程与进程之间的数据是没法直接交互的，如果想交互需借助第三方工具/模块。

### 2.1、join方法
join方法的作用是让主进程等待子进程运行结束后在执行。如下：
```python
from multiprocessing import Process
import time
def func(name, n):
    print(f'{name}任务开始')
    time.sleep(n)
    print(f'{name}任务执行完毕')

if __name__ == '__main__':
    start = time.time()
    l = []
    for i in range(1,4):
        p = Process(target=func, args=(f'写讲话稿{i}', i))
        p.start()
        l.append(p)
    for p in l:
        p.join()
    print('主进程')
    end = time.time()
    print(end-start)
```
如上所示，就启动了三个进程，等等三个进程都执行完，主进程才继续执行。

### 2.2、进程间数据隔离
进程间数据是隔离的，不能直接交互，下面我们来验证一下：
```python
from multiprocessing import Process
age = 18
def func():
    global age
    age = 16
if __name__ == '__main__':
    p = Process(target=func)
    p.start()
    p.join()
    print(age)

## 执行结果
18
```
如上子进程修改了age的值，但是主进程中依然是18，因此进程间的数据是隔离的。

### 2.3、进程号
操作系统为了管理进程，给每个进程分配一个pid号。获取进程号的方式也有两种：
```python
from multiprocessing import Process, current_process
import time
import os

def task(name='子进程'):
    print(f'{name}{current_process().pid}执行中')
    print(f'{name}的父进程{os.getpid()}执行中')
    # time.sleep(100)

if __name__ == '__main__':
    p = Process(target=task)
    p.start()
    print('主进程', current_process().pid)
```
如上所示，可以用过current_process().pid 或者 os.getpid()获取到进程号，还可以通过os.getppid()获取父进程的进程号。这里再补充两个进程的其他方法，p.terminate()   # 杀死当前进程  相当于 kill -9。p.is_alive()判断进程是否存活。

### 2.4、僵尸进程和孤儿进程
在子进程结束后，还会有一些系统资源占用(进程号、 进程运行状态、 运行时间等)， 等待父进程进行资源回收。除init进程之外，所有进程最后都会步入僵尸进程。但是如果子进程退出父进程没有及时处理，僵尸进程就会占用资源一直不释放，如果有大量的僵尸进程占用进程号，最终会导致操作系统无法创建新的进程。

孤儿进程是子进程处于存活状态，但是父进程意外结束了，这时子进程就会变成孤儿进程。这时init进程就会接管这个孤儿进程，负责回收孤儿进程的资源。


### 2.5、守护进程
守护进程是一类特殊的进程，它随着系统启动而启动，随着系统结束死亡。我们看示例：
```python
from multiprocessing import Process
import time
def task(name):
    print(name, '还活着')
    time.sleep(3)
    print(name, '正常死亡')

if __name__ == '__main__':
    p = Process(target=task, kwargs={'name': '苏妲己'})
    p.daemon = True
    p.start()
    time.sleep(1)
    print('纣王驾崩了')
```
如上，子进程被设置为守护进程，当主进程结束后，子进程立刻死亡，而不会运行到正常结束。

## 三、互斥锁
当多个进程访问同一个共享数据时，就会出现数据错乱问题，演示如下：
```python
import json
from multiprocessing import Process, Lock
import time
import random

# 查票
def search_ticket(name):
    # 读取文件，查询余票数量
    with open('data/tickets', 'r', encoding='utf-8') as f:
        dic = json.load(f)
    print(f'用户{name}查询余票：{dic.get("tickets_num")}')


def buy_ticket(name):
    with open('data/tickets', 'r', encoding='utf-8') as f:
        dic = json.load(f)
        time.sleep(random.randint(1, 5))
        if dic.get('tickets_num') > 0:
            dic['tickets_num'] -= 1
            with open('data/tickets', 'w', encoding='utf-8') as f:
                json.dump(dic, f)
            print(f'用户{name}买票成功')
        else:
            print(f'余票不足，用户{name}买票失败')

def task(name, mutex):
    search_ticket(name)
    mutex.acquire()
    buy_ticket(name)
    mutex.release()

if __name__ == '__main__':
    mutex = Lock()
    for i in range(1, 9):
        p = Process(target=task, args=(i, mutex))
        p.start()
```
如上，我们来模拟一个买票的问题，在不加锁的情况下，只有2张票但是8个人买票成功。注意这里我们把数据存到文件中，这样演示代码虽然比较复杂，但是解决了多个进程间数据不共享的问题，如果直接写在代码中，那每个进程的数据都是独立的。

在对代码加锁后(不加锁的代码可去除锁自行演示)，把并发变为了串行，这样虽然牺牲了效率，但是保证了数据安全。

## 四、消息队列
进程间的数据是隔离的，但是依然可以实现进程间通信，消息队列就是实现进程通信的方式。导入队列的方式有三种：
```python
import queue
queue.Queue()
from multiprocessing import queues
queues.Queue()
from multiprocessing import Queue
```
除了队列之外，管道也可以实现进程间的通信。但是我们一般不会使用管道，因为队列可以理解为管道+锁的实现。接下来看一下进程基于队列通信的示例，进程间通信也叫IPC机制
```python
from multiprocessing import Process, Queue
def task1(q):
    q.put('宫保鸡丁')
def task2(q):
    print(q.get())
if __name__ == '__main__':
    q = Queue()
    p = Process(target=task1, args=(q,))
    p.start()
    p1 = Process(target=task2, args=(q,))
    p1.start()
```
如上所示，就实现了两个子进程的通信，当然也可以子进程和主进程之间进行通信，这里就不再演示了。

## 五、生产者消费者模型
生产者-消费者模型是并发编程中最经典的设计模式之一，用于解决生产者和消费者之间速度不匹配的问题。通过队列来存储生产和消费的数据。例：
```python
from multiprocessing import Process, Queue, JoinableQueue
import time
import random
def producer(name, food, q):
    for i in range(8):
        time.sleep(random.randint(1, 3))
        print(f'{name}生产了{food}{i}')
        q.put(f'{food}{i}')

def consumer(name, q):
    while True:
        food = q.get()
        time.sleep(random.randint(1, 3))
        print(f'{name}吃了{food}')
        q.task_done()   # 告诉队列，已经拿走了一个数据， 并且已经处理完了

if __name__ == '__main__':
    q = JoinableQueue()
    p1 = Process(target=producer, args=('中华小当家', '黄金炒饭', q))
    p2 = Process(target=producer, args=('神厨小福贵', '佛跳墙', q))
    c1 = Process(target=consumer, args=('八戒', q))
    c2 = Process(target=consumer, args=('悟空', q))
    c1.daemon = True
    c2.daemon = True
    p1.start()
    p2.start()
    c1.start()
    c2.start()
    p1.join()
    p2.join()

    q.join()  # 等待队列所有数据被取完  计数器变成0
    # 主进程死了，消费者也要陪葬
```
先介绍下JoinableQueue，它在Queue的基础上多了一个计数器机制，每put一个数据，计数器就加1，每调用一次task_done计数器就减1，当计数器为0是，执行q.join后的代码。如上，设置了生产者进程join，保证生产者能生产完成，生产完整后执行q.join等队列数据去玩就结束主进程，消费者进程被设置成了守护，因此主进程结束消费者进程一起结束，而不会阻塞在q.get函数。
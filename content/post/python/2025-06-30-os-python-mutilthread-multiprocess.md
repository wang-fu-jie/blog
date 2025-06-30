---
title:       Python基础 - 多线程vs多进程
subtitle:    "多线程vs多进程"
description: "Cpython多线程因为GIL的存在无法利用多核能力，但并不代表多线程没用。多线程适用于IO密集型场景。本文中将对计算密集型和IO密集型场景进行演示，同时介绍死锁、递归锁、信号量以及Event事件。"
excerpt:     ""
date:        2025-06-30T15:18:05+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/1c/cf/man_village_shepherd_buffalo_nature_foggy-770087.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "os-python-mutilthread-multiprocess"
categories:  [ "PYTHON" ]
---

## 一、多线程vs多进程
在Cpython中，由于全局解释器锁GIL的存在，导致python多线程无法利用到多核优势。那是不是多线程就没什么用了呢？ 不是的，需要分情况来考虑。首先在单核情况下，只有单个核，即便能利用多核优势也只有一个核在工作。 再看多个核情况，目前的计算机基本都是多核，多核分IO密集型任务和计算密集型任务

### 1.1、计算密集型
计算密集型是需要CPU不断工作的，在多进程情况下，计算密集型可以利用到多核能力，因此计算密集型适合使用多进程。但是多线程无法利用到多核。
```python
def task():
    res = 0
    for i in range(10000000):
        res += 1

if __name__ == '__main__':
    l = []
    start = time.time()
    for i in range(8):
        # p = Process(target=task)    # 花费时间: 1.427130937576294
        p = Thread(target=task)  # 花费时间: 5.094439744949341
        p.start()
        l.append(p)
    for p in l:
        p.join()
    end = time.time()
    print('花费时间:', end - start)
```
如上所示，在计算密集型的任务重，多线程执行任务消耗的时候是小于多进程的。

### 1.2、IO密集型
在任务频繁出现IO操作的任务中，多线程和多进程使用的时间是差不多的，此时多线程更加节省资源。
```python
def task():
    time.sleep(1)
if __name__ == '__main__':
    l = []
    start = time.time()
    for i in range(2000):
        p = Process(target=task)    # 花费时间: 8.94227409362793
        # p = Thread(target=task)  # 花费时间: 1.3317527770996094
        p.start()
        l.append(p)
    for p in l:
        p.join()

    end = time.time()
    print('花费时间:', end - start)
```
在IO密集型的任务中，因为开启进程更消耗资源，可以看到多线程是更节省时间的。

总结： 多线程和多进程都有各自的优势，在写项目中，通常会在多线程的情况下同时开启多进程。此时多进程和多线程的的优势都会被利用到了。


## 二、死锁
死锁是指 多个进程（或线程） 在执行过程中，因为 竞争资源 或 通信不当 而陷入一种 互相等待对方释放资源 的状态，导致所有进程都无法继续执行下去的情况。我们看一个演示:
```python
from threading import Thread, Lock, current_thread
import time

mutex1 = Lock()
mutex2 = Lock()

def task():
    mutex1.acquire()
    print(current_thread().name, '抢到锁1')
    mutex2.acquire()
    print(current_thread().name, '抢到锁2')
    mutex2.release()
    mutex1.release()

    mutex2.acquire()
    print(current_thread().name, '抢到锁2')
    time.sleep(1)
    mutex1.acquire()
    print(current_thread().name, '抢到锁1')
    mutex1.release()
    mutex2.release()


if __name__ == '__main__':
    for i in range(8):
        t = Thread(target=task)
        t.start()
```
如上所示，第一个线程先抢到锁，然后释放锁1和锁2， 第一个线程继续向下执行抢到锁2进行睡眠。这时候第二个线程会抢到锁1。但是第二个线程抢不到锁2因为被占用了，第一个线程苏醒后抢锁1也抢不到，就造成整个任务阻塞。这就是死锁。

## 三、递归锁
递归锁内部有一个计数器，每acquire一次计数器就会加1，每release一次计数器就会减1。递归锁是可以重复抢占的。只要计数器不为0，别的线程就不能抢锁。
```python
from threading import Thread, Lock, current_thread, RLock
mutex1 = RLock()
mutex2 = mutex1
```
递归锁是Rlock，还是上边的代码，改成递归锁就不会出现阻塞的问题。

## 四、信号量
信号量在不同的阶段可能对应不同的技术点。对于并发编程来说它指的就是“锁”。它可以控制同时访问特定资源的数量，也就是限流。比如数据库连接池只有能有限个链接。
```python
from threading import Thread, Semaphore
import time
import random
sp = Semaphore(5)
def task(name):
    sp.acquire()
    print(name, '抢到车位')
    time.sleep(random.randint(3, 5))
    sp.release()

if __name__ == '__main__':
    for i in range(25):
        t = Thread(target=task, args=(f'宝马{i + 1}号',))
        t.start()
```
如上所示为信号量的示例，这个程序的运行结果，开始有5个线程抢到资源，之后每有一个线程释放资源，才会有一个新线程继续抢占。

## 五、Event事件
在多线程和多进程编程中，主进程或主线程可以通过join()方法等待子进程或子线程运行完毕后再继续晕车。但是子进程和子进程、子线程和子线程之间是不能相互等待的。如果想实现子进程或子线程之间的相互等待，就需要用到Event事件。
```python
from threading import Thread, Event
import time

event = Event()

def bus():
    print('公交车即将到站')
    time.sleep(5)
    print('公交车到站了')
    event.set()  # 发射信息

def passenger(name):
    print(name, '正在等车')
    event.wait()
    print(name, '上车出发')

if __name__ == '__main__':
    t = Thread(target=bus)
    t.start()

    for i in range(10):
        t = Thread(target=passenger, args=(f'乘客{i}',))
        t.start()
```
如上所示，子线程在进行等待，当被等待的子线程发射信号时，等待的子线程才继续执行。

## 六、Queue补充
前边说过进程间通信需要使用队列作为中转。同一个进行下的多线程数据是共享的，不需要使用队列。除了我们之前使用的Queue，还有两个分别为LifoQueue 和 PriorityQueue。
```python
import queue

# 先进先出Queue
q = queue.Queue()

# 后进先出Queue
q = queue.LifoQueue()
q.put('a')
q.put('b')
q.put('c')
print(q.get())

# 优先级Queue
q = queue.PriorityQueue()
q.put((18, 'a'))
q.put((69, 'b'))
q.put((36, 'c'))
q.put((-1, 'd'))
print(q.get())
print(q.get())
```
优先级Queue数字越小优先级越高。
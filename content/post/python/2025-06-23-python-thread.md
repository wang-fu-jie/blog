---
title:       Python基础 - 线程
subtitle:    "线程"
description: "由于进程和线程模块的开发遵循了鸭子类型，线程的创建和进程的操作基本是一样的。但是在Cpython中由于GIL全局解释器锁的存在，线程无法真正的并行，无法有效利用CPU的多核优势。"
excerpt:     "于进程和线程模块的开发遵循了鸭子类型，线程的创建和进程的操作基本是一样的。但是在Cpython中由于GIL全局解释器锁的存在，线程无法真正的并行，无法有效利用CPU的多核优势。"
date:        2025-06-23T15:33:09+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/6b/c1/child_girl_kid_grass_tree-52730.jpg!d"
published:   true
tags:
    - Python
    - python基础
slug:        "python-thread"
categories:  [ "PYTHON" ]
---

## 一、线程理论
前面说进程都是一块独立的内存空间，进程是资源单位。但是真正工作工作的是线程，因此创建一个进程后是自带一个线程的。在进程内创建多个线程，不需要再申请内存空间也不需要拷贝代码。同一个进程内，多个线程资源是共享的。

## 二、创建线程
操作线程的代码和操作进程基本是一致的，因为开发者在开发这两个模块时用了鸭子类型。创建线程同样也有两种方式。
```python
from threading import Thread
import time
def task(name):
    print(f'{name} 任务开始')
    time.sleep(3)
    print(f'{name} 任务结束')
if __name__ == '__main__':
    t = Thread(target=task, args=('悟空',))
    t.start()
    print('主线程')

## 执行结果
悟空 任务开始
主线程
悟空 任务结束
```
这里可以发现，运行结果，先打印了子线程，然后才打印了主线程。 但是在进程中先打印主进程，原因就是创建线程不需要申请内存空间与拷贝代码，所以线程的启动很快先执行了。创建线程的第二种方式也是通过继承：
```python
from threading import Thread
import time

class MyThread(Thread):
    def __init__(self, name):
        super().__init__()
        self.name = name

    def run(self) -> None:  # None是类型提示。标识没有返回值
        print(f'{self.name} 任务开始')
        time.sleep(3)
        print(f'{self.name} 任务结束')

if __name__ == '__main__':
    t = MyThread('八戒')
    t.start()
    print('主线程')
```

### 2.1、join方法
在前边示例中，子线程还没执行完成，主线程先执行完成。如果需要主线程等待子线程执行完成，就需要join方法，join方法的使用和进程是一样的，这里不再演示。


### 2.2、线程间数据共享
一个进程成创建多个线程，每个线程获取进程号都是一样的，且他们之间数据共享，来验证一下：
```python
import time
from threading import Thread, currentThread, activeCount
import os

age = 18
def task():
    print('子线程', os.getpid())
    global age
    age = 16
    print(currentThread().name)
    time.sleep(1)

if __name__ == '__main__':
    t = Thread(target=task)
    t.start()
    time.sleep(0.001)
    print('主线程', os.getpid())
    print(age)
    print(currentThread().name)
    print('活跃的线程数量',activeCount())

## 执行结果
子线程 17507
Thread-1
主线程 17507
16
MainThread
活跃的线程数量 2
```
如上所示： 子线程和主线程的进程id是一致的，并且子线程修改的是全局的数据。这里还通过currentThread().name获取线程名，这个名字是自动生成的。activeCount()可用来获取活跃现场的数量。

### 2.3、守护线程
守护线程和守护进程的概念也是类似的，一旦主线程结束，守护线程也会跟着一起结束。默认情况下：主线程运行完毕后不会立即结束，要所有子线程运行完毕后才会结束。因为主线程结束意味着进程结束了，设置守护线程也是把 daemon 属性置为 True。

## 三、线程互斥锁
当多线程操作同一个共享资源时，也会出现资源争抢的问题，因此也需要给共享资源上锁，我们这里贴出加锁的代码：
```python
from threading import Thread, Lock
import time

num = 180
def task(mutex):
    global num
    # mutex.acquire()
    with mutex:
        temp = num
        time.sleep(0.05)
        num = temp - 1
    # mutex.release()

if __name__ == '__main__':
    l = []
    mutex = Lock()
    for i in range(180):
        t = Thread(target=task, args=(mutex,))
        t.start()
        l.append(t)

    for j in l:
        j.join()

    print(num)
```
如上所示，使用with上下文管理器就可实现加锁和释放锁。也可以去掉这行代码验证不加锁的效果。另外说明一点。threading中的Lock和muiltprocessing中的Lock对象是一样的。


## 四、全局解释器锁GIL
GIL是cpython独有的特点。在Cpython解释器中，GIL是一把互斥锁，用来阻止同一个进程下多个线程同时执行，也就是说同一个进程下的线程不能并行。python的多线程无法利用多核优势，这是因为Cpython的内存管理是线程不安全的。

我们先假设cpython可以并行，但其中一个线程要定义变量，刚申请了内存还没有绑定变量，这时候垃圾回收线程运行把该内存回收了，就会导致原来的工作线程报错。因此GIL保证的是解释器级别的线程安全。
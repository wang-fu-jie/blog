---
title:      Python基础 - 协程
subtitle:    "协程"
description: "协程也被称为微线程，它是一种用户态的上下文切换技术，简单来说就是单线程下实现并发效果。当程序遇到IO时，来完成代码的上下文切换，达到欺骗CPU的效果，让CPU以为我们的程序一直在运行。"
excerpt:     "协程也被称为微线程，它是一种用户态的上下文切换技术，简单来说就是单线程下实现并发效果。当程序遇到IO时，来完成代码的上下文切换，达到欺骗CPU的效果，让CPU以为我们的程序一直在运行。"
date:        2025-07-15T09:43:43+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/cd/2a/outdoor_landscape_hiking_sun_hiker-20963.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-coroutine"
categories:  [ "PYTHON" ]
---

## 一、协程
协程也被称为微线程，它是一种用户态的上下文切换技术，简单来说就是单线程下实现并发效果。当程序遇到IO的时候，通过我们写的代码来完成代码的自动切换。也就是说我们来写代码监听IO，一旦程序遇到了IO，就在代码层面自动切换。

切换不一定会提升效率，对于IO密集型的程序来说，切换会提升效率。但是对弈计算密集型的程序来说，切换会降低效率。每次切换要保存当然任务的状态，python提供的yiled关键字就能实现状态的保存。

### 1.1、计算密集型程序切换
前边我们说了，计算密集型的程序进行上下文切换会降低效率，可以通过以下代码来验证：
```python
import time
def f1():     #   2.8911221027374268
    n = 0
    for i in range(10000000):
        n += i
        yield

def f2():
    g = f1()
    n = 0
    for i in range(10000000):
        n += i
        next(g)

start = time.time()
f2()
end = time.time()
print(end - start)
```
如上，通过yield函数将f1函数变成生成器，f2每循环一次就调用next()回到f1。这样运行完代码的总时间是大于f1和f2的串行时间的。另外yield关键字虽然能实现切换加保存状态的效果，但是它做不到遇到IO就切换。

### 1.2、gevent模块
gevent模块可以监听程序的IO，让我们的程序遇到IO就进行切换。但是它本身不能监听常见的IO操作，只能检测它自己的IO操作。

## 二、协程实现
通过gevent来监听程序的IO， 就可以实现代码的上下文切换与保存。
```python
from gevent import monkey
monkey.patch_all()
from gevent import spawn
import time

def da():
    for _ in range(3):
        print('da')
        time.sleep(2)

def mie():
    for _ in range(3):
        print('mie')
        time.sleep(2)

start = time.time()
g1 = spawn(da)
g2 = spawn(mie)
g1.join()
g2.join()
end = time.time()
print(end - start)
```
如上，针对两个IO密集型的任务，通过gevnt模块可以检测IO来实现任务的切换，这样任务执行的时间就变成了6秒，两个任务串行的执行时间是12秒多。这个模块日常我们一般不会用到，一般一些框架的底层会使用到。

这里的spawn在提交任务的时候也是异步提交，因此需要通过join方法来等待任务执行完毕，否则就会继续向下执行代码。

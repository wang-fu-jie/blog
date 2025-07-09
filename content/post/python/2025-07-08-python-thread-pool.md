---
title:       Python基础 - 线程池与进程池
subtitle:    "线程池与进程池"
description: "在开设线程和进程时是会消耗资源的，因此python提供了池的技术，一次创建多个进程或线程，在使用线程池或进程池过程中这些进程或线程不会被频繁创建和销毁。并且池一次性创建了指定数量的进程和进程，有效保护了硬件资源，防止软件过多的创建进程和线程。"
excerpt:     "在开设线程和进程时是会消耗资源的，因此python提供了池的技术，一次创建多个进程或线程，在使用线程池或进程池过程中这些进程或线程不会被频繁创建和销毁。并且池一次性创建了指定数量的进程和进程，有效保护了硬件资源，防止软件过多的创建进程和线程。"
date:        2025-07-08T15:35:59+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/01/f3/coffee_cup_lake_water_silhouette_hand_holding_dring-693736.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-thread-pool"
categories:  [ "PYTHON" ]
---


## 一、池的概念
在每次开设线程或开设进程的时候，都是需要消耗资源的。如果一个服务开启了上亿的线程那可能造成计算机罢工，计算机根本跑不起来。因此就引出了池的概念，池是用来保证计算机硬件安全的情况下，最大限度利用计算机的资源。降低了程序的运行效率，但保证了计算机的硬件安全。注意这里说的降低效率是指假设一亿个客户端需要创建一亿个线程来服务，用了池之后就不能一次服务一亿个客户端了。

## 二、线程池
线程池在创建时候就会一次性开设指定数量的线程，它省去了自己创建线程和销毁线程过程中占用的资源。我们先看示例：
```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import time
import os

pool = ThreadPoolExecutor(10)  # 不传参的话。默认开设的线程数量是当前cpu的个数乘以5
def task(name):
    print(name, os.getpid())
    time.sleep(3)
    return name + 10

def call_back(res):
    print('call_back', res.result())


# f_list = []
for i in range(50):
    # future = pool.submit(task, i)  # 往线程池提交任务
    # f_list.append(future)
    pool.submit(task, i).add_done_callback(call_back)
# pool.shutdown()  # 关闭线程池，等待所有线程运行完毕

# for f in f_list:
#     print(f.result())

print('主线程')
```
如上所示，创建线程池开设了10个线程，在使用时需要通过pool.submit()往线程池内提交任务，线程池提交任务属于异步提交，异步提交时任务提交后不等待任务的返回结果，代码继续向下执行。

那如何拿到任务的返回值呢，submit()是具有返回值的，它返回一个future对象。future对象具有一个result函数可以获取返回结果。如果提交任务后直接获取异步结果却是同步的，要在原地等待任务执行结果。因此我们可以先把所有任务都提交上去，将返回结果暂存到列表，最后在获取异步提交结果。

pool.shutdown()的功能是关闭线程池，等待线程池中所有运行完毕。这样就会保证所有任务运行完毕，主线程再向下执行。


## 三、进程池
进程池的使用和线程池是一模一样的，只需要将创建线程池的代码替换成创建进程池的代码：
```python
pool = ProcessPoolExecutor(3)
```
在使用进程池可以打印进程的id, 通过id号就可以证明进程池的进程没有被新创建和销毁，每次使用都是通用的进程号。


## 四、异步回调机制
前面我们通过异步任务提交返回的future对象来获取任务的结果，这种方法是不推荐的。获取异步任务的返回结果应该使用回调机制。回调机制是给每个任务都绑定了一个函数，一旦该任务有了结果就会自动触发该函数的执行。
```python
def call_back(res):
    print('call_back', res.result())
pool.submit(task, i).add_done_callback(call_back)
```
submit返回一个future对象，因此这段代码时调用了future对象的add_done_callback方法。在add_done_callback方法中，它会调用传入的call_back方法，并把自身self作为参数传给call_back，因此call_back中通过result可以获取异步提交结果。

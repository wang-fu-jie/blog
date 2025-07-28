---
title:       Python基础 - 异步IO
subtitle:    "异步IO"
description: "异步IO用于在等待 IO 操作时不中断程序的其他部分执行, 通过单独的模块asyncio来实现。它依赖事件循环机制，事件循环会自动检测并执行我们添加给它的任务，遇到IO就进行任务的切换。"
excerpt:     "异步IO用于在等待 IO 操作时不中断程序的其他部分执行, 通过单独的模块asyncio来实现。它依赖事件循环机制，事件循环会自动检测并执行我们添加给它的任务，遇到IO就进行任务的切换。"
date:        2025-07-25T10:20:53+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/96/bf/helmet_light_bokeh_bike_person-182515.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-aio"
categories:  [ "PYTHON" ]
---

## 一、异步IO
当我们的程序发起系统调用后，会立刻拿到一个结果。此时我们的程序不等待继续执行，而操作系统来做等待，等操作执行等待数据后，将数据copy到程序并发出一个信号，我们的程序就可以处理这个数据了。

异步IO是需要和操作系统做交互的，因此原生的python代码无法实现，例如C语言可以控制操作系统。但是已经有人封装好了模块可以直接使用。例如asyncio就是python内置的异步IO模块。


### 1.1、事件循环
事件循环可以理解为一个死循环，它会自动检测并执行我们添加给它的任务。它有一个任务列表，它会在死循环内检测已完成的任务和可执行的任务。然后遍历执行可执行的任务，然后遍历已完成任务并从任务列表中移除。当任务列表中的任务全部完成就终止这个事件循环。产生事件循环只需要一行代码：
```python
loop = asyncio.get_event_loop()
loop.run_until_complete()
```
如上，通过asyncio.get_event_loop()函数即可穿件一个事件循环，run_until_complete函数往事件循环中添加任务。


### 1.2、yield关键字
前边学习生成器的时候我们使用过 yield 关键字。在函数中使用yield关键字后就变成了生成器函数，调用函数并不会执行而是得到一个生成器。每次调用next()就会返回一个 yield关键字后的值。还有一个知识点是yield表达式：
```python
def f1():
    res = yiled 'hello'
    yiled res

g = f1()
next(g)    # 返回hello
g.send('world')  # 返回world
```
第一次调用next会返回yield后边的hello，然后停在这里。当调用send时，会把world赋值费res并继续向下执行。

### 1.3、yield from
yield from是python3引进的语法，它有两个作用。第一个作用是链接子生成器。我们先看什么是子生成器
```python
def f1():
    for s in 'abc':
        yiled s
    for i in range(3):
        yiled i

for i in f1():
    print(i)
```
这里我们把'abc'和range(3)理解为子生成器，因为它有yield关键字。那函数f1()就是主生成器。但是这么写比较麻烦，因此就有了yield from语法：
```python
def f1():
    yield from 'abc'
    yiled from range(3)
```
以上就是yiled from的第一个作用。yield from的第二个作用是作用连接调用者和子生成器之间的通道。先看例子：
```python
def sub():
    yield 'a'
    yield 'b'
    return 'c'

def link():
    res = yield from sub()
    print(res)

g = link()
next(g)
```
这里的next虽然驱动的是link。但是拿到是sub里边的值。当sub结束后，返回的值就会交给link函数的res。

## 二、协程函数
协程函数直接调用返回的是一个协程对象，这里和普通函数并不一样。它需要通过装饰器 @asyncio.coroutine 来装饰。
```python
@asyncio.coroutine
def f1():
    print('f1 start', current_thread())
    yield from asyncio.sleep(1)
    print('f1 end', current_thread())

loop = asyncio.get_event_loop()
loop.run_until_complete(f1())
```
如上所示，我们就把 f1 定义成了协程函数。往事件循环中添加的任务必须是协程对象。此时把f1添加进事件循环，是看不出效果的，因为只有一个任务，肯定要睡眠1秒。假设我们再提交一个f2，也添加进事件循环。一样没有效果，两个任务串行执行。这是因为run_until_complete会运行f1任务直到完成。因此我们需要先将多个任务添加到列表中：
```python
tasks = [f1(), f2()]
loop = asyncio.get_event_loop()
loop.run_until_complete(asyncio.wait(tasks))
```
以上用法是python3.4版本的用法。在python3.5以后提供了更简单的方法和两个关键字 async/await，就不再需要用到装饰器了。如下：
```python
async def f1():
    print('f1 start', current_thread())
    await asyncio.sleep(1)
    print('f1 end', current_thread())

tasks = [f1(), f2()]
loop = asyncio.get_event_loop()
loop.run_until_complete(asyncio.wait(tasks))
asyncio.run(asyncio.wait(tasks))  # python3.7以后才支持
```

## 三、可等待对象与异步函数
前边示例中，yield from 后面不能跟普通函数，await也一样不能跟普通的函数。它后边只能跟可等待对象。python中可等待对象有三种，分别是协程对象、task对象和future对象。我们看一个示例：
```python
async def recv():
    print('进入IO')
    time.sleep(3)
    print('结束IO')
    print('结果 hello')

async def f1():
    print('f1 start', current_thread())
    data = await recv()
    print(data)
    print('f1 end', current_thread())

async def f2():
    print('f2 start', current_thread())
    await recv()
    print('f2 end', current_thread())


tasks = [f1(), f2()]
asyncio.run(asyncio.wait(tasks))  # python3.7以后才支持
```
这个代码运行的效果是先运行第一个任务，然后等待三秒，再进入下一个任务。awiat的作用就是如果后边的任务遇到了IO, async就去自动调度其他的任务。但是这里并没有切换，是因为如果想要遇到IO切换，这个造成IO的任务是有要求的，必须支持await语法。但是time.sleep函数并不支持await语法。

time.sleep是一个同步函数，对于asyncio来说协程的阻塞必须使用异步库提供的函数。比如
```python
time.sleep()  -> asyncio.sleep()
server.accept -> loop.sock_accept
conn.recv()  ->  loop.sock_recv()
```
因此上边示例中的 time.sleep() 替换成 asyncio.sleep()即可。


## 四、task对象
task对象是用于协程调度，通过 asyncio.create_task() 等函数可以将协程封装为一个task，该协程会被自动调度执行。create_task函数会返回一个task对象，这个task对象就是对协程对象的一层包装。里面有这个任务的状态。
```python
import asyncio

async def nested():
    await asyncio.sleep(3)
    return 42

async def main(name):
    print(name, 'start')
    # 将 nested() 加入计划任务  # 立即与 "main()" 并发运行。
    task = asyncio.create_task(nested())
    # 现在可以使用 "task" 来取消 "nested()"，or  或简单地等待它直到它被完成：
    res = await task
    print(res)

asyncio.run(asyncio.wait([main('任务一'), main('任务二')]))
```
如上所示，提交了两个任务，任务一和任务二，任务一和任务又各自提交了一个nested任务，因此事件循环共有四个任务。 这个例子中，我们在main函数中只提交了一个task任务，如果提交多个task任务，就可以用到列表，如下：
```python
import asyncio

async def nested():
    await asyncio.sleep(3)
    return 42

async def main(name):
    print(name, 'start')
    task_list = [asyncio.create_task(nested()), asyncio.create_task(nested())]
    done, pending = await asyncio.wait(task_list)
    for task in done:
        print(task.result())

asyncio.run(main('任务一'))
```
这里的done是我们提交进去，且已经完成的task任务，通过这个集合可以拿到任务的返回值。pending目前用处不大，它可以加一个超时参数，例如设置为1秒，1秒后完成的任务放入done，未完成的任务放入pending。

如果done集合里面的任务比较多，如何区分哪个结果是哪个任务的呢？ 在python3.8中，每个task都有一个name属性。


## 五、future对象
future更加底层，它是task对象的基类，主要作用就是用来等待结果的。我们这里不做futrue对象的演示。

## 六、异步上下文管理器
上下文管理器就是with语法。定义一个上下文管理器需要定义__enter__方法和__exit__方法，进入上下文管理器会自动执行__enter__方法。对于异步上下文管理器对应的是__aenter__方法和__aexit__方法。通用的异步上下文管理器需要使用 async with来调用。

最后介绍一个uvloop，它是一个第三方模型提供的事件循环机制，可以替代asyncio内置的事件循环，据说性能可以提升一倍以上。


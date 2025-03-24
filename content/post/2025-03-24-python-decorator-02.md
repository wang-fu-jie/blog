---
title:       Python基础 - 装饰器02
subtitle:    "装饰器02"
description: "装饰器在不修改原对象的基础上给它增加功能，装饰器支持有参装饰器和无参装饰器两种。本文将重点介绍装饰器模板使用、有参装饰器和装饰器叠加使用。以及如何完美伪装原函数。"
excerpt:     "装饰器在不修改原对象的基础上给它增加功能，装饰器支持有参装饰器和无参装饰器两种。本文将重点介绍装饰器模板使用、有参装饰器和装饰器叠加使用。以及如何完美伪装原函数。"
date:        2025-03-24T15:29:01+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/89/b5/portrait_flash_got_you_girl_woman_person_female_young-816939.jpg!d"
published:   true
tags:
    - tag1
    - tag2
slug:        "python-decorator-02"
categories:  [ "PYTHON" ]
---

## 一、装饰器模板
通过上一篇文章，我们已经知道了如何给函数写装饰器，这里总结一下装饰器的模板：
```python
def outer(func)
    def wrapper(*args, **kwargs):
        res = func(*args, **kwargs)
        return res
    return wrapper
```
这就是装饰器实现的模板，这个模板并没有增加功能，需要增加功能根据这个模板添加即可。pycharm提供了实时模板功能，例如在pycharm键入main就会自动弹出if __name__ == '__main__': 这就是实时模板功能。你也可以自定义装饰器模板到实时模板中去。


## 二、完美伪装
我们先看一下上篇文章中的示例，它伪装的还不够完美。
```python
import time
def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        response = func(*args, **kwargs)
        end = time.time()
        print(f'函数运行时间{end - start}秒')
        return response

    return wrapper

@timer
def inside(n):
    sum = 0
    for i in range(n + 1):
        sum += i
    return sum


print(inside)

## 执行结果
<function timer.<locals>.wrapper at 0x7fb163ef3290>
## 而原始函数打印结果是
<function inside at 0x7fbda66e4d40>
```
这是因为函数有内置属性例如 __name__指定当前函数名，__doc__执行当前函数注释。封装后的wrapper函数的名字是wrapper。因此我们需要这是wrapper的内置属性等于原函数的内置属性。
```python
import time
def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        response = func(*args, **kwargs)
        end = time.time()
        print(f'函数运行时间{end - start}秒')
        return response
    wrapper.__name__ = func.__name__
    wrapper.__doc__ = func.__doc__
    return wrapper

@timer
def inside(n):
    sum = 0
    for i in range(n + 1):
        sum += i
    return sum

print(inside)
```
但是这样做仍然不够，因为函数的内置属性太多了，如果一个一个都这么些太繁琐，Python也考虑到了这个问题，提供了功能来实现这个操作。
```python
from functools import wraps
import time

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        response = func(*args, **kwargs)
        end = time.time()
        print(f'函数运行时间{end - start}秒')
        return response
    return wrapper


@timer
def inside(n):
    sum = 0
    for i in range(n + 1):
        sum += i
    return sum

print(inside)

## 执行结果
<function inside at 0x7fde70ef3290>
```
这里我们使用了functools模块提供的wraps功能实现了完美伪装，wraps也是一个装饰器，它对wrapper函数进行了装饰，它实现的功能就是把原函数的属性都复制给wrapper函数，当然我们也可以自己再写一个装饰器实现这些功能，只是没有必要而已。

## 三、有参装饰器
在函数内部需要参数时，第一种方式是直接给函数传参，第二种方式是闭包函数。在装饰器中，如果wrapper函数需要参数怎么办，肯定不能通过第一种方式。我们先看下模板：
```python
def outer(func)
    def wrapper(*args, **kwargs):
        res = func(*args, **kwargs)
        return res
    return wrapper
```
如果wrapper需要一个参数name，不能直接给wrapper传参，这是因为wrapper的参数是原封不动传给原函数的。也不能给outer传参，因为受限于语法糖，@outer相当于 func = outer(func)，语法糖只能接受一个参数，那就是只能传函数名，不能传多余的参数了。所以我们只能采用闭包函数的方式，在outer函数外再包一层。
```python
from functools import wraps

def g_outer(name):
    def outer(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            print(name)
            res = func(*args, **kwargs)
            return res

        return wrapper
    return outer

@g_outer('wfj')  # 他就相当于@outer，因为g_outer返回了outer的内存地址
def home():
    print('hello world')

print(home)
```
这里我们实现了参数装饰器。注意@g_outer('wfj') 中g_outer('wfj')相当于函数调用，这个函数返回了outer的内存地址，所以相当于@outer。

## 四、装饰器叠加
装饰器时可以叠加使用的，如下：
```python
@outer3  # home = outer3(home)     ouer3.wrapper
@outer2  # home = outer2(home)     ouer2.wrapper
@outer1  # home = outer1(home)     ouer1.wrapper
def home():
    pass
```
叠加装饰器的执行顺序时，先执行原函数头顶上的，也就是先执行outer1。看一个示例：
```python
def outer1(func):
    def wrapper(*args, **kwargs):
        print('开始执行装饰器outer1')
        res = func(*args, **kwargs)
        print('结束执行装饰器outer1')
        return res
    return wrapper

def outer2(x):
    def outer(func):
        def wrapper(*args, **kwargs):
            print('开始执行装饰器outer2')
            res = func(*args, **kwargs)
            print('结束执行装饰器outer2')
            return res
        return wrapper
    return outer

def outer3(func):
    def wrapper(*args, **kwargs):
        print('开始执行装饰器outer3')
        res = func(*args, **kwargs)
        print('结束执行装饰器outer3')
        return res
    return wrapper


@outer3
@outer2(10)
@outer1
def home():
    print('执行函数home')

home()

### 执行结果：
开始执行装饰器outer3
开始执行装饰器outer2
开始执行装饰器outer1
执行函数home
结束执行装饰器outer1
结束执行装饰器outer2
结束执行装饰器outer3
```
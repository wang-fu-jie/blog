---
title:      Python基础 - 装饰器01
subtitle:    "装饰器01"
description: "装饰器时在不修改原函数或类的情况下，动态的添加额外的功能，通过闭包函数来实现，Python通过语法糖简化了装饰器的使用。本文中我们将利用示例逐步推导装饰器的原理和使用方法。"
excerpt:     "装饰器时在不修改原函数或类的情况下，动态的添加额外的功能，通过闭包函数来实现，Python通过语法糖简化了装饰器的使用。本文中我们将利用示例逐步推导装饰器的原理和使用方法。"
date:        2025-03-21T09:43:12+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/05/60/girls_beach_sunset_beautiful_bridge_sky_cloud_portrait-764924.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-decorator-01"
categories:  [ "PYTHON" ]
---

## 一、装饰器概念
在编程中有一种思想叫开放封闭原则，它的核心思想是：对扩展开放，对修改封闭。扩展是说在不改变原来代码的基础上为其增加功能，封闭是对修改源代码是封闭的。
装饰器用于在不修改原函数或类的情况下，动态地添加额外功能。在 Python 里，装饰器本质上是高阶函数，即接受一个函数作为输入，并返回一个新的函数。它可以极大提升代码的复用性、可读性和扩展性。

## 二、装饰器的实现
现在有一个函数，计算从1加到n的结果，实现如下：
```python
def inside(n):
    sum = 0
    for i in range(n + 1):
        sum += i
    print(sum)
```
现在需求给这个函数增加功能，统计这个函数的运行时间。我们可以使用time模块，记录起始时间和终止时间。实现如下：  
**方案一**
```python
import time

start = time.time()
def inside(n):
    start = time.time()
    sum = 0
    for i in range(n + 1):
        sum += i
    end = time.time()
    print(f'函数运行时间{end - start}秒')
    print(sum)

inside(20000000)

## 执行结果
函数运行时间1.2022600173950195秒
200000010000000
```
这个实现方式虽然没改变函数的调用方式，但是修改了代码，因此不符合开放封闭原则。  
**方案二**
```python
import time
def inside(n):
    sum = 0
    for i in range(n + 1):
        sum += i
    return sum
start = time.time()
print(inside(20000000))
end = time.time()
print(f'函数运行时间{end - start}秒')
```
在调用函数前后分别记录时间，这样也实现了统计运行时间的方案。但是这样如果函数多次调用时，每次都需要统计时间，就会造成大量重复代码，降低可读性。  
**方案三**  
前面我们也说到避免代码重复的方式就是封装成函数，那么我们把方案二中的调用封装一下。如下：
```python
import time
def inside(n):
    sum = 0
    for i in range(n + 1):
        sum += i
    print(sum)
def wrapper():
    start = time.time()
    inside(20000000)
    end = time.time()
    print(f'函数运行时间{end - start}秒')

wrapper()
```
这种方式解决了方案二的代码冗余问题，也没有修改被装饰对象的源代码，同时还为其增加了功能。但是被装饰对象的调用方式被修改了。  
**方案四**
```python
import time
def inside(n):
    sum = 0
    for i in range(n + 1):
        sum += i
    print(sum)
def wrapper(*args, **kwargs):
    start = time.time()
    inside(*args, **kwargs)
    end = time.time()
    print(f'函数运行时间{end - start}秒')

wrapper(20000000)
```
我们使用以上这种方式就解决了调用传参不一致的问题，wrapper函数的参数被原封不动的传给了inside函数，这样将来即使inside函数需要修改参数，wrapper的函数体也不需要改变。   
**方案五**  
如果我们现在再新增一个需求，又一个函数也需要实现统计功能运行的时间。可以采用方案四的方法再写一个wrapper, 但是这样就有造成了代码冗余，因此我们需要修改方案四，里边调用的inside函数不能写死。
```python
import time
def inside(n):
    sum = 0
    for i in range(n + 1):
        sum += i
    print(sum)

def outer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        func(*args, **kwargs)
        end = time.time()
        print(f'函数运行时间{end - start}秒')
    return wrapper

inside = outer(inside)
inside(20000000)
```
这个方案采用了闭包函数，将原始函数传递给装饰函数进行了进行扩展，即没有修改原函数，也没有修改调用方式。只是调用的底层发生了变化，不再是调用inside函数而是调用的wrapper函数，但是调用者是完全感知不到，这样我们就实现了装饰器。但是这个方案仍然存在优化点，接下来看方案六。   
**方案六**  
方案五中的inside函数是没有返回值，那如果有返回值呢。我们看一下效果。
```python
import time
def inside(n):
    sum = 0
    for i in range(n + 1):
        sum += i
    return sum

def outer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        func(*args, **kwargs)
        end = time.time()
        print(f'函数运行时间{end - start}秒')
    return wrapper

inside = outer(inside)
res = inside(20000000)
print(res)

## 执行结果
函数运行时间1.2029309272766113秒
None
```
可以看到，原函数是返回计算结果的，但是伪装后的返回了None，并没有做到和原函数一模一样。因为伪装后调用了其实是wrapper函数， wrapper并没有返回值。因此我们需要在wrapper中返回inside函数的返回值。
```python
import time
def inside(n):
    sum = 0
    for i in range(n + 1):
        sum += i
    return sum

def outer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        response = func(*args, **kwargs)
        end = time.time()
        print(f'函数运行时间{end - start}秒')
        return response
    return wrapper

inside = outer(inside)
res = inside(20000000)
print(res)

## 执行结果
函数运行时间1.217651128768921秒
200000010000000
```
这样装饰器就和原函数一模一样了。这也是装饰器的最终版本。

## 三、语法糖
第二章节中的装饰器虽然伪装的和原函数一模一样，但是存在语法不够简洁的缺点。例如，有两个函数都需要使用outer进行伪装。如
```python
func1 = outer(func1)
func2 = outer(func2)
```
每次装饰一个对象是都需要调用一次outer。语法糖就是解决这个问题的。我们来看下语法糖的使用
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

res = inside(20000000)
print(res)
```
只要在原函数上边添加 @装饰器名。 在调用@timer这行代码时，python就会去调用timer对inside进行封装。因此这里装饰器必须定义在前面。另外这里我们把装饰器名改为了timer，这属于命名规范了，通过名称见名知意，我们就可以知道装饰器扩展了什么功能。
---
title:       Python基础 - 函数的传递与闭包
subtitle:    "函数的传递与闭包"
description: "函数名属于特殊的变量名，它可以作为函数的参数或返回值，这种特性叫做函数的传递。闭包是指一个内部函数在其外部作用域（但不是全局作用域）中引用了变量，并且在外部函数返回后仍然能够访问这些变量。在 Python 中，当一个嵌套函数（内部函数）引用了外部函数的变量，并且外部函数的返回值是这个内部函数时，就形成了闭包。"
excerpt:     "函数名属于特殊的变量名，它可以作为函数的参数或返回值，这种特性叫做函数的传递。闭包是指一个内部函数在其外部作用域（但不是全局作用域）中引用了变量，并且在外部函数返回后仍然能够访问这些变量。在 Python 中，当一个嵌套函数（内部函数）引用了外部函数的变量，并且外部函数的返回值是这个内部函数时，就形成了闭包。"
date:        2025-03-19T11:33:58+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/5f/3e/man_jumping_joyful_happy_athletic_outdoor_fun_sport-735705.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-function-passing-closure"
categories:  [ "PYTHON" ]
---

## 一、函数的传递
函数名指向的是函数的内存地址，因此可以将函数名指向的内存地址绑定给另一个变量。
```python
def func():
    print('我是函数func')

x = func
print(x)
x()

## 执行结果
<function func at 0x7fd0b6f12950>
我是函数func
```
既然函数名可以赋值给别的变量，那意味它也可以当做参数进行传递。
```python
def func():
    print('我是函数func')
def func2(x):
    print(x)

func2(func)

## 执行结果
<function func at 0x7faa46f15b00>
```
函数名可以当做参数进行传递，同样也可以作为返回值进行返回。其实可以这么理解，函数名也是一个特殊的变量，因此函数名也可以作为列表、字典、元组等数据类型的元素。

## 二、闭包函数
闭是封闭的意思，闭函数是被封闭起来的函数。
```python
def func1():
    def func2():
        pass
```
这里的func2就是闭函数，因为它只能在func1的局部进行访问。接下来看包函数，包函数的意思是函数内部包含对外层函数作用域名词的引用。
```python
def func1():
    x = 10
    def func2():
        print(x)
```
现在func2定义在了func1的内部，全局是访问不到的，那如果全局需要访问func2呢，就可以使用上面提到函数传递，在func1中将func2返回出去。
```python
def func1(x):
    def func2():
        print(x)
    return func2

res = func1()
print(res)

## 执行结果
<function func1.<locals>.func2 at 0x7fa5e4f1e170>
```
这样就打破了作用域的限制。 总结：在 Python 中，当一个嵌套函数（内部函数）引用了外部函数的变量，并且外部函数的返回值是这个内部函数时，就形成了闭包

那闭包函数的应用场景是什么呢？ 一般用于函数修改需要增加参数，但是又不想改变原来的形参，就可以通过闭包函数传入新的参数。最大的应用场景就是下面要说的装饰器。
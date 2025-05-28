---
title:       Python基础 - 异常处理
subtitle:    "异常处理"
description: "异常处理对程序逻辑错误可能的逻辑错误进行处理，否则程序会报错终止。异常处理提升了程序的健壮性，但是降低了代码的可读性，因此异常处理尽可能不要使用，只有出现不可预知的异常时再进行异常处理。"
excerpt:     "异常处理对程序逻辑错误可能的逻辑错误进行处理，否则程序会报错终止。异常处理提升了程序的健壮性，但是降低了代码的可读性，因此异常处理尽可能不要使用，只有出现不可预知的异常时再进行异常处理。"
date:        2025-05-28T10:42:45+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/d7/7d/girls_sparklers_fireworks_celebration_happy_people_young_cheerful-846299.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-exception"
categories:  [ "PYTHON" ]
---

## 一、程序错误
程序错误分为语法错误和逻辑错误。语法错误属于写代码时产生的，属于程序运行前就需要修正的错误，例如使用了中文的括号等。逻辑错误是运行时错误。逻辑错误又分为两类，可预知和不可预知。可预知指的是我们可以知道错误发生的条件，例如int()函数传入非数字的字符串，这时候我们就可以进行if判断。不可预知是无法预先知道错误发生的条件，例如网络编程，客户端和服务端断开，无法知道什么时间因为什么原因断开，就只能使用异常处理来检测代码的运行。

异常指的是程序遇到错误后会抛出异常，程序立刻终止运行。为了提升程序的健壮性，在程序异常后，应该进行捕捉并处理，而不是报错终止程序。

## 二、异常处理
异常打印的信息包含跟踪信息和异常类型。我们可以使用try进行异常的捕获和处理。try语法的格式如下：
```python
try:
    子代码块
except 异常类型1 as e:
    子代码块
except 异常类型2 as e:
    子代码块
except 异常类型3 as e:
    子代码块
except Exception as e:
    子代码块
else:
    子代码块
finally:
    子代码块
```
如上所示，try对子代码块进行监测，如果捕获到了异常，就执行对应异常类型的子代码块。如果指定的异常类型都没捕获到， 则会走except Exception这块代码，Exception这用于捕获所有异常， 这里也可以只写一个except， 也可以捕获所有异常，但是就不能获取到异常信息了。如果运行没有异常，则执行else的子代码块。 finally的子代码块比较特殊，无论是否产生异常都会被执行。
```python
try:
    num = 'abc'
    # int(num)
    print('2'.center(20,'='))
    # print(name)
    print('3'.center(20,'='))
    dic = {'name':'张大仙'}
    dic['age']
except (ValueError, NameError) as e:  // 如果多个异常类型的处理方式相同，那就可以一起处理
    print(e)
    print('异常处理成功')
except Exception as e:
    print(e)
else:
    print('没有异常')
finally:
    print('总会执行')
```
另外需要注意的是： try和else不能连用，中间必须有except处理。但是try和finally可以连用。最后异常处理会降低代码的可读性，因此不推荐大量使用。在针对可预知的逻辑错误时尽量使用if判断进行处理，只有不可预知的异常再进行异常处理。
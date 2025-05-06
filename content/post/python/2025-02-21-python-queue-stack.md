---
title:       Python基础 - Python队列与堆栈
subtitle:    "Python队列与堆栈"
description: "队列和堆栈是两种很常见的数据结构，队列的特点是先进先出，即FIFO。堆栈的特点是后进先去，即LIFO。这里我们使用python的列表来实现以下这两种数据结构。"
excerpt:     "队列和堆栈是两种很常见的数据结构，队列的特点是先进先出，即FIFO。堆栈的特点是后进先去，即LIFO。这里我们使用python的列表来实现以下这两种数据结构。"
date:        2025-02-21T13:58:59+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/16/b6/landscape_hiking_fall_autumn_mountain-19622.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-queue-stack"
categories:  [ "PYTHON" ]
---

## 一、队列与堆栈
队列和堆栈是两种很常见的数据结构，都是用来存储数据的。队列可以理解为一个排队的队伍，先进入队伍的先排到，即fist in fist out，简写为FIFO。但是堆栈是后进先出，即last in first out， 简写为LIFO。


## 二、列表实现队列
我们可以使用列表来模拟实现队列这种结构。首先我们模拟入队操作，入队可以使用列表的insert()功能实现，也可以使用append()实现。注意如果使用insert()应该每次把数据插到队尾。
```python
>>> l = []
>>> l.append("张三")
>>> l.append("李四")
>>> l.append("王五")
>>> print(l)
['张三', '李四', '王五']
```
如上所示，张三第一个入队，位于队首。接下来我们模拟出队操作，那也应该是张三先出队。因此我们可以使用pop()模拟出队。
```python
>>> l.pop(0)
'张三'
>>> l.pop(0)
'李四'
>>> l.pop(0)
'王五'
>>> print(l)
[]
```
这时候我们就模拟了列表，先进去的的先出俩。

## 三、列表实现堆栈
堆栈的特点是后进先出。首先我们先模拟入栈操作，入栈和队列是一样的。
```python
>>> l = []
>>> l.append("张三")
>>> l.append("李四")
>>> l.append("王五")
>>> print(l)
['张三', '李四', '王五']
```
但是出栈就不一样了，应该是王五先出。
```python
>>> l.pop()
'王五'
>>> l.pop()
'李四'
>>> l.pop()
'张三'
```
我们通用使用pop()模拟了出栈操作，只不是pop弹出的不再是队首，而是队尾了。
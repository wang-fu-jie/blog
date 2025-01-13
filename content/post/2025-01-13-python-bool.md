---
title:       Python基础 - Python基本数据类型之布尔类型
subtitle:    "Python基本数据类型之布尔类型"
description: "bool类型在Python中是基本的数据类型，用于表示逻辑值“真”或“假”，广泛应用于条件判断和循环控制。它有两个值：True和False，并支持多种布尔运算。"
excerpt:     "bool类型在Python中是基本的数据类型，用于表示逻辑值“真”或“假”，广泛应用于条件判断和循环控制。它有两个值：True和False，并支持多种布尔运算。"
date:        2025-01-13T14:37:47+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/00/89/landscape_cloud_mountain-501384.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-bool"
categories:  [ "PYTHON" ]
---

## 一、布尔类型介绍
布尔类型（Boolean Type）是一种基本的数据类型，以英国数学家、布尔代数的奠基人乔治·布尔（George Boole）命名.主要用于表示逻辑值。布尔类型只有两个可能的值：True 和 False。这是计算机科学中用于表示逻辑判断的基础。我们来看下布尔类型的定义和 type。
```python
>>> a = True
>>> b = False
>>> print(type(a))
<class 'bool'>
```
可以看到，布尔类型的 type 就是 bool。

## 二、布尔类型使用
布尔类型只有真和假两种状态。在真正使用过程中，不一定必须用 True 和 False 来表示真和假两种状态。使用 1和0 一样可以。
```python
a = 1
b = 0
```
布尔值一般不会直接来定义，更多是当成条件判断，例如：
```python
a = 20
print(a>18)
True
```

## 三、None类型
在Python中，还存在一个特殊的常量 None。它表示没有值，也就是空值。Node 的数据类型属于 NoneType。
```python
>>> a = None
>>> print(type(a))
<class 'NoneType'>
```
需要注意的是，None 是 NoneType 类型的唯一值。 

## 四、真假判断
代表真的并不是只有 True，代表假的也并不是只有 False。 python规定所有有值的都是真。
0、None, 空字符串、空列表、空字典 都认为是假。其余的值都是真。
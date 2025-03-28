---
title:       Python基础 - 迭代器
subtitle:    "迭代器"
description: "迭代器是 Python 中用于遍历集合元素的一种机制，供了一种高效、统一的方式来访问序列（如列表、元组、字典等）或其他可迭代对象的元素。迭代器是重复取值的工具，每一次重复都是和上一次有关联的。"
excerpt:     "迭代器是 Python 中用于遍历集合元素的一种机制，供了一种高效、统一的方式来访问序列（如列表、元组、字典等）或其他可迭代对象的元素。迭代器是重复取值的工具，每一次重复都是和上一次有关联的。"
date:        2025-03-26T09:41:49+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/d3/06/leaves_books_color_coffee_cup_still_life_book_reading-670379.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-iterator"
categories:  [ "PYTHON" ]
---

## 一、迭代器
迭代器是 Python 中用于遍历集合元素的一种机制，供了一种高效、统一的方式来访问序列（如列表、元组、字典等）或其他可迭代对象的元素。迭代器是重复取值的工具，每一次重复都是和上一次有关联的。

## 二、可迭代对象
只要内置了__iter__()方法的对象都是可迭代对象。
```python
d = {'a': 1, 'b': 2}
res = d.__iter__()
print(res)

## 执行结果
<dict_keyiterator object at 0x7f9506915c50>
```
可以看到，\_\_iter\_\_()方法返回的是一个迭代器对象。因此可以这么说。可以转换为迭代器的对象就是可迭代对象。 

## 三、迭代器对象
迭代器对象是内置__next__()方法并且内置__iter__()方法的对象。调用__next__()方法就是取下一个值，调用__iter__()方法方法返回迭代器对象本身。
```python
d = {'a': 1, 'b': 2}
res = d.__iter__()
print(res.__next__())
print(res.__next__())
```
当迭代器对象取到最后一个值时，再次调用__next__()方法就会抛出异常。

## 四、for循环原理
for循环执行的过程如下：
* 调用对象的__iter__()方法，得到它的一个迭代器版本
* 调用该迭代器的__next__()方法，拿到一个值，赋值给key
* 循环执行上一步，直到抛出异常，进行异常捕获并结束循环

这里对象可以是可迭代对象也可以是迭代器对象，因为迭代器对象的__iter__()方法返回的是它自身。


---
title:       Python基础 - Python数据类型之集合类型
subtitle:    "Python数据类型之集合类型"
description: "集合类型用来存储多个值，功能主要是去重和关系运算。它是无序的数据类型，因此不支持索引取值。集合也属于可迭代对象支持循环。"
excerpt:     "集合类型用来存储多个值，功能主要是去重和关系运算。它是无序的数据类型，因此不支持索引取值。集合也属于可迭代对象支持循环。"
date:        2025-02-28T16:32:38+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/91/22/track_railway_photographer_vintage_railroad-44186.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-set"
categories:  [ "PYTHON" ]
---

## 一、集合类型介绍
集合也是用来存多个值的，主要用来去重和做关系运算。集合的定义是在{}之间使用逗号隔开多个元素，集合的元素必须是不可变类型。
```python
>>> s = {1, 2, 3}
>>> print(s, type(s))
{1, 2, 3} <class 'set'>
>>> 
>>> s = {1, 2, 3, 3}
>>> s
{1, 2, 3}
>>> s = {1, 2, 3, [4, 5]}
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
```
可以看到，集合会自动去重，在定义时存储了重复的元素，集合只会保留一个。集合也是无序的，也就意味着集合不能做取值操作。另外一个需要注意的点是，集合和字典都是使用{}进行定义，如果定义空集合需要使用set()，而不能使用{}，{}定义的是空字典。

这里的set()也就是定义集合时触发的功能，它也可以进行类型转换，可以将可迭代类型转换为集合类型，注意转换前的类型不能有可变类型。


## 二、关系运算
关系运算包含取交集，并集、差集、对称差集和父子集
```python
>>> s1 = {'a', 'b', 1, 2}
>>> s2 = {'a', 'b', 3, 4}
>>> s1 & s2                      # 取交集
{'b', 'a'}
>>> s1 | s2
{1, 2, 3, 4, 'a', 'b'}           # 取并集
>>> s1 - s2                      # 取差集 s1特有的元素
{1, 2}
>>> s2 - s1                      # 取差集 s2特有的元素
{3, 4}
>>> s2 ^ s1                      # 对称差集，等同于取两个集合特有元素的并集
{1, 2, 3, 4}
>>> (s1 - s2) | (s2 - s1)
{1, 2, 3, 4}
```
父子集用与比较两个集合的大小，这个大小指的是一个集合是否完全包含于另一个集合。
```python
>>> s1 = {1, 2, 3}
>>> s2 = {1, 2}
>>> s1 > s2
True
```
以上我们对几个做关系运算都是通过符号实现的，集合自身也提供了一些内置方法实现这些关系运算。
* 取交集  s1.intersection(s2)
* 取并集  s1.union(s2)
* 取差集  s1.difference(s2)
* 取堆成差集  s1.symmetric_difference(s2)
* 父子集（如果两者相等，则互为父子，结果都是True）
* 父集  s1.issuperset(s2)
* 子集  s1.issubset(s2)

## 三、集合的其他操作
1、通过集合去重，集合具备去重功能，它只能对不可变类型去重，并且不保证去重后的顺序。
```python
>>> l = [1, 2, 3, 3, 1]
>>> set(l)
{1, 2, 3}
```
2、取长度，集合也是支持len()函数获取长度
```python
>>> s = {1, 2, 3}
>>> len(s)
3
```
3、成员运算，in和not in
```python
>>> s = {1, 2, 3}
>>> 1 in s
True
>>> 4 not in s
True
```
4、循环，集合也属于可迭代类型
```python
>>> s = {1, 2, 3}
>>> for i in s:
...     print(i)
... 
1
2
```
5、集合更新，包含update、intersection_update、difference_update、symmetric_difference_update
```python
>>> s1 = {1, 2, 3}
>>> s2 = {3, 4, 5}
#  update()和字典的效果一样， 把s2中特有的元素更新到s1, 已有的不做改变
>>> s1.update(s2)
>>> s1
{1, 2, 3, 4, 5}
#  intersection_update 是取s1 he s2的交集，然后更新到s1。相当于取交集重新再赋值给2
>>> s1 = {1, 2, 3}
>>> s2 = {3, 4, 5}
>>> s1.intersection_update(s2)
>>> s1
{3}
```
difference_update、symmetric_difference_update分别是去差集和对称差集再赋值给s1，这里就不再做演示。

6、判断两个集合是否有交集
```python
>>> s1 = {1, 2}
>>> s2 = {3, 4}
>>> s1.isdisjoint(s2)
True
```

7、集合的其他内置方法
```python
>>> s = {1, 2, 3}
>>> 
>>> s1 = s.copy()    # 集合的拷贝
>>> s1
{1, 2, 3}
>>> s1.clear()       # 清空集合
>>> s1
set()
>>> s.pop()          # 删除并返回一个值
1
>>> s
{2, 3}
>>> s.remove(3)      # 删除一个元素，如果传入的元素不存在就会报错
>>> s
{2}
>>> s.discard(3)     # 删除一个元素，如果传入的元素不存在也不会报错
>>> s
{2}
>>> s.add(3)         # 给集合添加一个元素
>>> s
{3, 2}
```

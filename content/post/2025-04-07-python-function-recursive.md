---
title:       Python基础 - 函数递归
subtitle:    "函数递归"
description: "递归是指函数在执行过程中直接或间接调用自身的行为。递归函数通常包含两个关键部分：基准条件（Base Case）：递归终止的条件，递归条件（Recursive Case）：函数调用自身的条件。本文我们将介绍递归并使用递归实现一个全排列算法。"
excerpt:     "递归是指函数在执行过程中直接或间接调用自身的行为。递归函数通常包含两个关键部分：基准条件（Base Case）：递归终止的条件，递归条件（Recursive Case）：函数调用自身的条件。本文我们将介绍递归并使用递归实现一个全排列算法。"
date:        2025-04-07T09:50:21+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/c6/db/lions_animal_male_female_lions_ready_to_pounce_carnivore_feline_lioness-1120474.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-function-recursive"
categories:  [ "PYTHON" ]
---

## 一、函数递归
递归是在调用一个函数的过程中，又调用到了自己本身。例：
```python
def func():
    print('hello world')
    func()

func()

# 执行结果：
......
hello world
hello world
......
RecursionError: maximum recursion depth exceeded while calling a Python object
```
可以看执行结果，一开始会正常的嵌套调用，但是最终产生了报错，原因是这段代码一直在递归进行调用，函数一直不会结束。如果持续调用下去内存终会被撑爆，python为了避免这种情况，设置了最高调用层数为1000。python也提供了查看和修改递归深度的方式。
```python
import sys

sys.setrecursionlimit(1500)
print(sys.getrecursionlimit())

## 执行结果
1500
```

## 二、间接递归
函数直接调用自身可以实现递归，函数间相互调用也可实现递归，如：
```python
def func1():
    print('func1')
    func2()
def func2():
    print('func2')
    func1()
func1
```
以上两个函数互相调用，实现了函数间接调用自身。通过前面的示例，我们知道了函数递归默认最大为1000层，但是在编程中我们不应该让递归层级过深，否则会消耗大量资源。在其他编程语言中，有尾递归优化，但是Python不具备尾递归优化的功能，因此在使用函数递归时，应该给递归设置退出条件。
```python
i = 10
def my_sum(i):
    if i == 0:
        return 0
    return i + my_sum(i-1)
```

## 三、全排列算法
现在需求对一个字符串 abcd 进行全排列，就可以使用到递归了。全排列如图所示：
![图片加载失败](/post_images/{{< filename >}}/3-01.png)
如图所示：我们首先取第一个字符进行排列，然后剩余3个字符，接下来再将剩余的3个字符中的第一个字符进行排列，以此类推。接下来我们实现一下：
```python
s = 'abcd'
l = list(s)
def permutation(l, level):
    if level == len(l):
        print(l)
    for i in range(level, len(l)):
        l[level], l[i] = l[i], l[level]
        permutation(l, level + 1)
        l[level], l[i] = l[i], l[level]

permutation(l, 0)
```
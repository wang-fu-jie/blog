---
title:       Python基础 - for循环
subtitle:    "for循环"
description: "Python的循环结构实现的第一种方式是while语句，第二种实现方式是for语句。for在循环取值上比while循环更简洁。for 循环可用于遍历序列中的元素（如列表、字符串、字典、元组等），是一种简洁且高效的迭代工具。它的核心功能是按顺序逐个访问可迭代对象中的每个元素，并执行特定操作。"
excerpt:     "Python的循环结构实现的第一种方式是while语句，第二种实现方式是for语句。for在循环取值上比while循环更简洁。for 循环可用于遍历序列中的元素（如列表、字符串、字典、元组等），是一种简洁且高效的迭代工具。它的核心功能是按顺序逐个访问可迭代对象中的每个元素，并执行特定操作。"
date:        2025-02-12T10:41:36+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/ec/08/light_bulb_lights_electricity_energy_evening_party_celebration_idea-599181.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-for"
categories:  [ "PYTHON" ]
---

## 一、for循环介绍
实际上，for循环能实现的需求，while循环一样能做到。那为什么还需要for循环呢。原因for循环取值比while循环更简洁。它是一种简洁高效的迭代工具，核心功能是按顺序逐个访问可迭代对象中的每个元素，并执行特定操作。语法格式如下：
```python
for 变量名 in 可迭代对象:
    子代码块
```
这里出现了新名词可迭代对象，目前暂时不需要知道它是什么，后续函数章节会具体描述。暂时智需要知道列表、字符串、字典等是可迭代对象，可用于for循环。示例：
```python
l = ['a', 'b', 'c']
for a in l:
    print(a)
```
这段代码运行的结果是，它会依次打印a,b,c。for需要循环多少次，取决于可迭代对象有多少个值。因此for循环也叫遍历循环，while循环也叫条件循环。

## 二、for循环遍历字典
我们先来看一个for循环遍历字典的示例：
```python
d = {'name': 'wfj', 'age': 18}
for i in d:
    print(i)
```
运行结果是打印 name age。因此for循环遍历字典是变量字典的key, 如果需要打印值直接打印d[i]就可以了。

## 三、range函数
如果需要用for循环，来循环固定的次数。例如固定循环5次。第一种方法是写一个长度为5的列表，对这个列表做循环。但是如果需要循环1000次呢，不可能写一个长度1000的列表吧。这时就需要用到python提供的range函数，它可以用来控制for循环的循环次数。我们先来看range()在python2的运行结果。
```python
# python2
>>> range(10)
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
>>> range(1,9)
[1, 2, 3, 4, 5, 6, 7, 8]
>>> range(1,9,2)
[1, 3, 5, 7]
```
可以看到range函数会生成一个列表。它有三种形式：
```python
# range(num) 当传入一个参数时，它会生成一个从0开始到num结束的列表，不包含num，即左闭右开
# range(num1, num2) 当传入两个参数时，它会生成一个从num1开始到num2结束的列表，不包含num2，即左闭右开
# range(num1, num2, num3) 当传入三个参数时，它会生成一个从num1开始到num2结束的列表，步长为num3即每次加num3。不包含num3，即左闭右开
```
range函数默认步长为1，你可以自行测试，range(-1) 或者 range(2, 1)的结果是空列表。但是在Python3版本中，range()函数做了优化，生成的不再是列表。
```python
# python3
>>> range(10)
range(0, 10)
>>> range(1, 9)
range(1, 9)
>>> range(1, 9, 2)
range(1, 9, 2)
```
可以看到，python3 range()函数生成的是一个可迭代对象。它比列表更省空间。但是用法是一致的：
```python
for i in range(10):
    print(i)
```

## 四、for循环与break、else、continue
上一篇文章我们讲到了while循环和break、else、continue的用法，for循环也一样支持这三个语句，并且用法一模一样。：
```python
# break退出循环
for i in range(10):
    if i < 5:
        print(i)
    else:
        break
```

```python
# continue结束本次循环
for i in range(10):
    if i == 5:
        continue
```

```python
# continue结束本次循环
for i in range(10):
    if i == 5:
        continue
```

```python
# else。正常结束就执行，被break打断的不执行
for i in range(10):
    if i == 5:
        continue
else:
    print('循环结束了')
```
这里给出了三个示例代码，请自行运行测试，并查看实验结果。

## 五、for循环嵌套
for循环也同样支持嵌套。
```python
for i in range(3):
    print('外层循环-->', i)
    for j in range(5):
        print('内层循环-->', j)
```
这个代码运行结果是，0 0 1 2 3 4   1 0 1 2 3 4 2 0 1 2 3 4。 实际运维打印结果是竖着显示的，这里为了介绍文本空间按行进行了编辑。注意：for循环结束只能使用break语句，它不能像while循环那样改变条件。for循环和while循环也支持互相嵌套。
---
title:       Python基础 - while循环
subtitle:    "while循环"
description: "编程除了顺序结构和选择结构，第三中结构就是循环结构。通过巡检结构，可以使计算机重复的做一件事。本文中介绍python第一种实现循环结构的方式，while循环。"
excerpt:     "编程除了顺序结构和选择结构，第三中结构就是循环结构。通过巡检结构，可以使计算机重复的做一件事。本文中介绍python第一种实现循环结构的方式，while循环。"
date:        2025-02-10T19:09:50+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/f0/89/field_grass-99194.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-while"
categories:  [ "PYTHON" ]
---

## 一、while循环介绍
假如我们有这样一个需求，计算1+2+3，可以直接编写。但是如果需求改为计算1+2+...+5000，在编程时我们不能从1敲到5000吧，这时候就需要利用循环结构来实现。python循环结构有两种实现方式，本文我们先介绍第一种，while循环。它的基本语法格式如下：
```python
while 条件:
    子代码块
    ...
```
它的执行逻辑时，执行到whiil循环代码时，会先判断条件，如果条件成立则执行子代码块，然后再进行条件判断，如果一直成立一直循环执行。直到条件不成立退出循环，执行后续的代码。示例：
```python
num = 0
while num < 10:
    print(num)
    num += 1
```
上述代码运行结果是打印0到9，我们分析下当num打印9后，再进行加1为10，判断num<10为假，因此10就不再进行打印，直接结束程序。

## 二、死循环
当一个循环结构条件一直为真的时候，那这个循环就无法结束，我们称之为死循环。示例:
```python
num = 0
while num < 10:
    print(num)
```
这个实例中，num永远为0，每次判断都是小于10为真，因此这个代码一旦运行会无休止的打印0。我们再来看两个案例：
```python
# 案例一
while True:
    info = input("请输入内容: ")
    print(info)

# 案例二
while 1:
    10 + 10
```
这两个案例，条件分别使用了True和1，永远未真都是死循环。案例一在input会被阻塞等待用户输入，案例二是循环计算10+10，cpu会一直计算压力较大，这类程序会造成效率问题。因此我们再编写程序应尽量避免纯计算无IO的死循环。

## 三、退出while循环
在执行循环过程中，当我们发现运行已经满足我们的需求后，就应该退出循环，这样就不会产生死循环。我们还是拿一个例子演示：
```python
while True:
    info = input("请输入1-10之间的一个数字: ")
    if info == '5':
        print('猜对了')
    else:
        print(没猜对)
```
这个示例中，让用户猜数字，如果输入5就猜对了，此时应该退出循环，不应该继续猜了。退出循环有两种方式第一种是修改判断条件，第二种是break语句。我们先看第一种：
```python
condition = True
while condition:
    info = input("请输入1-10之间的一个数字: ")
    if info == '5':
        print('猜对了')
        condition = False
    else:
        print(没猜对)
     print('代码继续')
```
如上所示，我们把条件用变量存储，如果猜对了就修改变量为假这样就可以退出循环了，但是通过这种方式修改变量后续的代码还是会继续执行。我们再看第二种方式：
```python
while True:
    info = input("请输入1-10之间的一个数字: ")
    if info == '5':
        print('猜对了')
        break
    else:
        print(没猜对)
     print('代码继续')
```
break这种方式，只要代码运行到break，会立刻结束本层循环。break本层的循环体代码不再执行。即代码继续不会再被打印。

## 四、循环嵌套
注意我们上一小节是break会结束本层循环，意味着循环是可以嵌套的。例如我们需求找出2-100之间的素数。
```python
i = 2
while i < 100:
   j = 2
   while j <= i/j:
      if not i%j: 
        break
      j = j + 1
   if j > i/j: 
    print(i, " 是素数")
   i = i + 1
 
print("Good bye!")
```
如上所示：break的是结束本层循环，退回到外层循环。

## 五、终止本次循环
终止本次循环使用continue实现，注意它和break终止本次循环的区别。需求：打印0~9，但是不要打印4。
```python
num = 0
while num < 10:
    if num == 4:
        num += 1
        continue
    print(num)
    num += 1
```
如上所示：当num值为4时，就结束本次循环不进行打印，直接进入下一次循环。

## 六、while与else
while循环使用else语句，意思是在while循环正常结束后会执行else的代码，但是被break打断的循环不会执行else分支的代码。语法格式如下：
```python
while 条件:
    子代码
else:
    子代码
```
我们看一个示例：
```python
num = 0
while num < 10:
    if num == 4:
        num += 1
        continue
        # break
    print(num)
    num += 1
else:
    print('循环结束了')
```
当使用continue时，会打印到9并打印循环结束了，当使用break结束循环时，只会打印0~3。代码的执行可以自行验证
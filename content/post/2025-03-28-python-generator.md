---
title:       Python基础 - 生成器
subtitle:    "生成器"
description: "生成器是一种特殊的迭代器，通过生成器可以实现自定义迭代器，使程序按需生成值，通过yiled关键字实现。yield还可以使用表达式方式接收传值。可以使用更简洁的生成式语法创造一个生成器，生成式不仅支持生成器，还支持列表生成式、字典生成式和元组生成式。"
excerpt:     ""
date:        2025-03-28T10:39:47+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/99/f3/woman_female_bike_beauty_model_photography-893103.jpg!d"
published:   true
tags:
    - Python
    - python基础
slug:        "python-generator"
categories:  [ "PYTHON" ]
---

## 一、生成器
生成器是 Python 中一种特殊的迭代器，它允许你按需生成值，而不是一次性生成所有值并存储在内存中。这使得生成器非常适合处理大量数据或无限序列。也就是说生成器可以让我们自定义一个迭代器。生成器定义和函数定义语法类型，只是生成器不使用return返回数据，而是使用yield关键字代替return。
```python
def func():
    print('第一次执行')
    yield 1
    print('第二次执行')
    yield 2

res = func()
print(res)
print(res.__next__())

# 执行结果
<generator object func at 0x7f7f45f22450>
第一次执行
1
```
可以看到，当调用生成器函数时，返回值是一个生成器对象。函数体并没有执行，当调用__next__方法后，才执行了第一次打印，当运行到yield关键字时，函数就停止了运行，并降yield函数后的数据进行了返回，如果yield关键字后面没有数据默认返回None。

这里补充一点。生成器的__iter__方法和__next__方法可以使用iter()和next()方法代替，这样看起来更简洁。例如next(res), 和res.\_\_next\_\_()的效果是一样的。

## 二、yield表达式
yield表示式是将yield赋值给一个变量。yield有两个功能，第一个功能是把函数暂停在这里。第二个功能就是可以给它传值。
```python
def func(x):
    print(f'{x}开始执行')
    while True:
        y = yield None
        print('\n', x, y, '\n')


g = func(10)
g.send(None)
g.send(20)

## 执行结果
10开始执行

 10 20
```
这里解释下这段代码，首先send函数的作用是给yield函数传值并赋值给y。第一次send必须传值为None，这样才能触发函数的执行，打印第一行代码并停留在yield表达式那行，当第二次调用send传值为10，这时候y的值是10，send函数返回值依然为yield关键字后边的None，只是我们这里没有进行打印。

## 三、生成式
生成式包含列表生成式、字典生成式、 集合生成式。我们先看列表生成式。它是 Python 中一种简洁、高效的创建列表的方式，它可以用一行代码替代传统的 for 循环+append 操作来生成列表。
```python
# 列表生成式方式
squares = [x**2 for x in range(10)]
# 结果: [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# 带条件的列表生成
evens = [x for x in range(10) if x % 2 == 0]
# 结果: [0, 2, 4, 6, 8]

# 使用条件表达式。
numbers = [x if x % 2 == 0 else -x for x in range(10)]
# 结果: [0, -1, 2, -3, 4, -5, 6, -7, 8, -9]
```
这里for循环也是可以嵌套的，条件判断也可以使用and or等加多重条件，这里你可以自行去测试验证。总之，生成式虽然简化了代码，但是过于复杂会降低代码的可读性。

接下来看一下其他生成式。
```python
# 字典生成式
square_dict = {x: x**2 for x in range(5)}
# 结果: {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# 集合生成式
unique_chars = {char for char in 'abracadabra' if char not in 'abc'}
# 结果: {'d', 'r'}
```
那元素有没有生成式呢，答案是没有，因为元组是不可变的。 我们来测试一下：
```python
res = (x for x in range(10) if x % 2 == 0)
print(res)

## 执行结果
<generator object <genexpr> at 0x7f8178f22450>
```
可以看到，当我们使用（）时，得到的是一个生成器，因此这个语法叫做生成式表达式。这时候就可以是next()来调用生成器了。这里再补充一个知识点，如果一个函数调用生成器表达式时，它的括号是可以省略的。
```python
count = sum(x for x in range(10) if x % 2 == 0)
print(count)

## 执行结果
20
```
如上，在sum()函数调用生成器表达式进行求和，就可以省略生成器表达式的括号，使代码更简洁。



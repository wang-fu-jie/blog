---
title:       Python基础 - Python输入与输出
subtitle:    "Python输入与输出"
description: "程序经常需要与用户进行交互，即用户输入一段信息，运行后给用户返回信息。这就涉及到了输入和输出功能，Python通过input函数提供输入功能，通过print函数提供输出功能。"
excerpt:     "程序经常需要与用户进行交互，即用户输入一段信息，运行后给用户返回信息。这就涉及到了输入和输出功能，Python通过input函数提供输入功能，通过print函数提供输出功能。"
date:        2025-01-17T13:35:34+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/images/87/2d/3215cb8f0d65ba24013767a1cf9b-1637858.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-input-output"
categories:  [ "PYTHON" ]
---

## 一、程序输入
我们编写的程序经常需要与用户进行交互，例如翻译软件需要用户输入英文，程序才能给用户返回对应的中文。Python提供了接收用户输入的功能，通过input函数实现。在使用input时候可以接受一些提示性的信息，来对用户输入的内容进行指导说明。
```python
>>> myname = input("请输入你的名字：>>>")
请输入你的名字：>>>wfj
>>> print(myname, type(myname))
wfj <class 'str'>
```
如上例： 在执行完input这行代码后，程序就阻塞在这里等待用户输入。当输入内容后，Python将输入内容封装成字符串然后赋予变量myname。这里说明，在Python3版本中，会将用户输入的任何内容都存储为字符串类型。

这时如果我们需求是用户输入数字，Python还是会把数字转为字符串。这时候就需要我们手动处理将字符串再转换为数据，第一种方式我们可以通过int函数实现，但是int函数不能转换浮点型。

### 1.1、Python2的实现
在Python3版本中，input将输入都转换为字符串类型。但是在Python2中，提供了两个函数，一个是raw_input，它等同于Python3中的input函数。Python2还提供了自身的input函数要求用户必须输入明确类型，用户输入什么类型就会存为什么类型。

Python3为什么取消了呢，原因是这样对软件使用者不友好。例如要求用户输入字符串就必须加引号。也就是使用Python开发的软件还要熟悉Python的数据类型。因此在Python3中，输入统一为字符串，类型的转换由程序开发者完成。

## 二、程序输出
程序输出使用print函数实现，这个函数我们使用了很多次了。此处不做基本的演示。日常程序输出结果结构都是比较清晰的，例如输出表格形式，Python一样提供了规范输出的能力，即格式化输出。

### 2.1、格式化输出%s
Python提供的第一种格式化输出的方式是 %s。%s可以理解为一个占位符，我们先定义一个格式，不确定的内容使用%s先占位，输出时将内容填充到%s的位置。
```python
>>> info = 'my name is %s, I come from %s' %('wfj', 'tj')
>>> print(info)
my name is wfj, I come from tj
>>> info = 'my name is %(name)s, I come from %(hometown)s' %{'name':'wfj', 'hometown':'tj'}
>>> print(info)
my name is wfj, I come from tj
```
从上面示例中可以看出，使用%s占位符有两种传值方式，第一种是按照位置传值，括号的第一个值传到第一个占位符号，第二个值传给第二个占位符···。如果只有一个占位符，那括号可以省略不写，直接在%后面写值即可。 第二种传值方式是按照key传值，这样需要传入一个字典，key对应的value值传入对应的位置。%s我们可以传入任意类型的值，都会当成字符串处理。

除了%s还有一个占位符是%d，它表示接受一个整型。
```python
>>> info = 'my age is %d' % 18
>>> print(info)
my age is 18
>>> info = 'my age is %d' % 'nihao'
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: %d format: a number is required, not str
```

### 2.2、格式化输出format
Python字符串类型内置了一个format函数，它也是实现了格式化输出。format函数使用大括号作为占位符。
```python
>>> info = 'my name is {}, I come from {}'.format('wfj', 'tj')
>>> print(info)
my name is wfj, I come from tj
>>> info = 'my name is {0}{0}, I come from {1}{1}{1}'.format('wfj', 'tj')
>>> print(info)
my name is wfjwfj, I come from tjtjtj
>>> info = 'my name is {name}, I come from {hometown}{hometown}'.format(name='wfj', hometown='tj')
>>> print(info)
my name is wfj, I come from tjt
```
如上，format也可以有多种传值方式，第一种按位置，默认{}不填充内容是按照位置传值，也可以填入索引号取对应的值。也可以通过字典的形式取值。

### 2.3、格式化填充
format和%s一样，可以传入任意类型的值，最终都转化为字符串。Python还支持格式化填充，例如需求输出一段字符，字符长度不满足10时两边使用*填充。
```python
>>> info = '{0:*^10}'.format('begin')
>>> print(info)
**begin**
```
此处0指的是0号索引，只有一个值0也可以不写，如果传入字典的话方式一样，把0换成对应的key就行。^表示在两边填充，也可以使用 < 和 > ， 分别表示在右边填充和左边填充。
```python
>>> info = '{num:.2f}'.format(num=3.1415926)
>>> print(info)
3.14
```
这个实例中。.2f是四舍五入，保留两位小数。

### 2.4、格式化输出f
f这种格式化方式是Python3.5版本才引入的功能，因此低版本可能不支持。
```python
>>> name = 'wfj'
>>> home = 'tj'
>>> info = f'my name is {name}, I come from {home}'
>>> print(info)
my name is wfj, I come from tj
```
如示例所示，f的使用方法，使用大括号包裹变量即可实现填充。

## 三、总结
以上三种格式化方式执行效率是不同的，其中f的效率最高，format其次，%s最低。因此我们尽量使用f这种方式，除非你要考虑兼容性。例如将程序拿到Python2的环境运行。其实还有第四桌格式化方式，是基于Python的标准库string，这种方式基本很少使用，此处不做具体介绍
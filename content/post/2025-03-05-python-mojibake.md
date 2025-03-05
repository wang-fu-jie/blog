---
title:       Python基础 - Python的乱码问题
subtitle:    "Python的乱码问题"
description: "Python的乱码问题分两种，第一种是读取文件乱码，导致Python解释器无法读取到内存。第二种是执行时乱码，执行乱码在Python3中不会出现，因为Python3统一使用了Unicode编码。Unicode码点转换为其他编码格式的过程叫做编码，其他编码转换为Unicode码点的过程叫做解码。"
excerpt:     "Python的乱码问题分两种，第一种是读取文件乱码，导致Python解释器无法读取到内存。第二种是执行时乱码，执行乱码在Python3中不会出现，因为Python3统一使用了Unicode编码。Unicode码点转换为其他编码格式的过程叫做编码，其他编码转换为Unicode码点的过程叫做解码。"
date:        2025-03-05T10:02:04+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/3a/91/girl_summer_youth_coke_coca_cola-438.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-mojibake"
categories:  [ "PYTHON" ]
---

## 一、py文件乱码问题
运行python的程序的本质是，先编写python源程序，然后使用运行解释器执行源程序，pycharm工具也是一样的，可以认为pycharm是一个功能强大的文本编辑器。python解释器需要从硬盘读取源文件，把源文件的内存按照python语法解析并执行。这里读取源文件的过程就涉及字符编码的转换。

首先我们创建一个gbk编码格式的文件保存到硬盘，可以用pycharm编辑。pycharm的右下角支持设置文件保存到硬盘的编码格式。我们这里不再演示，在源程序输入一句打印字符。
```python
print('你好')
```
分别使用python3 和 python2 运行这个代码。
![图片加载失败](/post_images/{{< filename >}}/1-01.jpg)
结果如图所示：我们使用gbk存储到硬盘的源文件，并未告诉python解释器使用哪种编码方式去读取到内存转为Unicode格式。 Python3默认按照UTF-8来读取，Python2按照ASCII来读取。结果都产生了报错。

### 1.1、指定python文件头
基于以上问题，应该如何解决呢？也就是如何告诉python解释器按照gbk编码读取磁盘上的源文件。答案是使用python文件头指定编码。
```python
# coding:gbk
print('你好')
```
或者
```python
# -*- coding: gbk -*-
print('你好')
```
加上文件头后，我们再看运行结果：
![图片加载失败](/post_images/{{< filename >}}/1.1-01.jpg)
python3运行成功了，但是python2运行虽然没有报错，确打印了乱码。执行没有报错原因是，我们在第一行指定了文件头告诉解释器按照哪种编码格式读取，解释器在读取第一行字符仍然按照默认编码读取，因为文件头是英文，因此无论哪种编码都可以正常读取，读取文件之后再按照文件头指定的编码继续向下读取。

### 1.2、解决执行乱码
在python读取文件到内存中，字符 '你好'  包括单引号在内存中都是以Unicode二进制形式存在的。当时python解释器运行代码时，就会申请内存空间，把这个字符串值存入到堆区。在Python3中存储变量直接把 你好 的Unicode二进制丢到申请的内存空间中。因此打印的时候不会乱码，因为Unicode是兼容万国码的。

在Python2中，存储变量值默认不是按照Unicode存储的，因为发布python时候还没有Unicode编码。python2是按照文件头指定的编码在新申请的内存空间存入值。在print打印时是按照终端的编码方式打印，我们使用的终端是UTF-8的，所以print使用UTF-8来解析GBK的值，打印处理就产生了乱码。

针对此现象，python2也提供了解决方案，即在定义字符串前加上u指定在内存中存储为Unicode格式。
```python
# coding:gbk
print(u'你好')
```

### 1.3、结论
通过以上测试，我们可以得出结论：
* 在Python源程序尽量加上指定文件头为UTF-8编码，因为现在的系统和软件基本都是UTF-8
* 在Python3中不会出现执行乱码，因为Python3执行时的值是按照Unicode在内存中存储的
* 在Python2中，在内存中存储字符是按照文件头指定的编码存储，如果存储Unicode需要在在字符串前加 u
* python2中print打印是根据系统终端的字符集进行解码的，但是python3的print函数都是按照Unicode解码的

在python2和python3中，还有一些细微的差异，例如repr()函数，有兴趣可以自行研究。总之，目前开发程序一般使用python3版本，加上指定文件头使用UTF-8编码，就基本不会出现乱码问题了。

## 二、编码与解码
Unicode码点转换为其他编码格式的过程叫做编码，其他编码转换为Unicode码点的过程叫做解码。在Python3中字符串存储是Unicode格式，因此字符串存在解码函数encode。
```python
>>> a = '中'
>>> res = a.encode('gbk')
>>> print(res, type(res))
b'\xd6\xd0' <class 'bytes'>
```
这就是把Unicode码点转换为了GBK编码。编码后类型的bytes类型，也就是gbk编码的二进制数，只是print是按照十六进制进行打印的。解码是decode函数实现的：
```python
>>> res.decode('gbk')
'中'
```
解码也是按照gbk进行解码为Unicode格式。因为这串二进制就是gbk编码。解码后把Unicode的二进制给了print函数，print按照Unicode再解码为字符串，显示出来。


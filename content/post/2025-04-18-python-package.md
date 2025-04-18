---
title:       Python基础 - 包的介绍
subtitle:    "包的介绍"
description: ""
excerpt:     ""
date:        2025-04-18T10:22:11+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/81/62/field_girl_woman_walking_grass-21058.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-package"
categories:  [ "PYTHON" ]
---

## 一、包介绍
在上篇文章介绍模块时，提到一个python文件就是一个模块。其实除了文件，还有一种组织代码的方式就是文件夹，文件夹中包含一个__init__.py的文件，它有一个名字叫做包。包避免把所有代码写在一个文件里，提高可维护性。包可以将相关的 模块（Module） 和子包组合在一起，形成一个层次化的结构。

导包也是使用import或from关键字，导包过程也是和导入模块类似。先产生一个名称空间，然后执行python文件，这里执行的就是__init__.py，最后在当前执行文件中创建一个名字指向包产生的名称空间。
```python
## 创建包pack，在__init__文件写代码
print('我是init')

# test.py
import pack

## 执行结果
我是init
```
这时访问包的功能其实就是访问__init__.py文件中的功能。在python3中，如果没有__init__.py文件，导入包也不会报错。

## 二、包使用
现在有一个需求，假设我们需要开发一系列功能，如果使用模块全堆放在一个文件中，随着功能越来越多，那这个模块的维护有越来越复杂。因此我们将没类功能封装到一个模块中，把这些模块用包组织起来。
```python
# 组织结构
game
├── __init__.py
├── chat.py
├── kanpsack.py
└── map.py

# chat.py
def chat():
    print('聊天功能')
def chat2():
    print(1)
# kanpsack.py
def kanpsack():
    print('背包功能')
# map.py
def map():
    print('地图功能')

## __init__.py
# 绝对导入
from game.chat import chat, chat2
from game.kanpsack import kanpsack
from game.map import map

import game.chat

# 相对导入
# . 当前文件夹
# ..上层文件夹

from .chat import chat
```
这里我们看一下__init__.py文件。前边我们说了导包就是执行__init__.py， 如果__init__.py文件中没有名字，那就调用不到包里子模块的功能。因此我们需要将子模块的名字导入到__init__.py文件中，这里分为相对导入和绝对导入。

我们先看绝对导入，from game.map import map， 这里从game文件夹下的map文件中导入了map函数。这里从game开始找，原因是__init__.py文件的导包时的查找顺序和正在执行文件的查找顺序是一样的，都是参照执行文件的sys.path。这里可以自行验证，在两个文件中都打印一下sys.path就发现他们是一样的。这里说明一下包是可以嵌套的，即包里面可以嵌套子包，如果想要使用子包的功能，就需要在顶级包的__init__.py文件中进行导入。

接下来早看相对导入，相对导入使用俩符号，一个是 . 代表当前目录，一个是 .. 代表上级目录。这里 from .chat import chat 在__init__.py当前所在路径查找chat文件。
```python
import game
game.chat()
```
这里在使用包的时候，导入包名就可以调用到包的功能了，或者可以使用from，效果是一样的。
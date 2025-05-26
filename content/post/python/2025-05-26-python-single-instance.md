---
title:       Python基础 - 单例模式
subtitle:    "单例模式"
description: "单例模式是一种常用的软件设计模式，它的目的是保证一个类只能有一个实例对象存在。单例模式减少了内存的消耗，避免频繁创建销毁对象。python实现单例模式的方法有多种，可以通过模块导入、类装饰器、类绑定方法、__new__方法、元类、并发编程方式等来实现"
excerpt:     "单例模式是一种常用的软件设计模式，它的目的是保证一个类只能有一个实例对象存在。单例模式减少了内存的消耗，避免频繁创建销毁对象。python实现单例模式的方法有多种，可以通过模块导入、类装饰器、类绑定方法、__new__方法、元类、并发编程方式等来实现。"
date:        2025-05-26T09:52:00+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/03/2b/traffic_intersection_lane_road_street-2681.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-single-instance"
categories:  [ "PYTHON" ]
---

## 一、单例模式
单例模式是一种常用的软件设计模式，它的目的是保证一个类只能有一个实例对象存在。单例模式减少了内存的消耗，避免频繁创建销毁对象。python实现单例模式的方法有多种，可以通过模块导入、类装饰器、类绑定方法、__new__方法、元类、并发编程方式等来实现。

### 1.1、模块实现
模块实现是最常用的单例模式实现，在模块中实现一个类，然后实例化一个对象，在其他文件中直接导入这个对象即可。

### 1.2、类装饰器实现
通过装饰器来实现单例模式的代码如下：
```python
def singleton_mode(cls):
    obj = None

    def wrapper(*args, **kwargs):
        nonlocal obj
        if not obj:
            res = cls(*args, **kwargs)
        return obj

    return wrapper


@singleton_mode  # Human = singleton_mode(Human)
class Human:
    def __init__(self, name, age):
        self.name = name
        self.age = age
```
如上：在装饰器中定义好类对象，如果存在了就直接返回，这样每次实例化都是返回原来的对象。


### 1.3、类绑定方法
如下为通过类绑定方法实现的单例模式：
```python
class Human:
    obj = None

    def __init__(self, name, age):
        self.name = name
        self.age = age

    def get_obj(cls, *args, **kwargs):
        if not cls.obj:
            cls.obj = cls(*args, **kwargs)
        return cls.obj
```
如上所示，获取类对象通过get_obj方法实现，这里并没有禁止实例化对象，和java不一样，java可以直接将构造函数私有禁止实例化。python也可以自定义元类或重写__new__方法禁止实例化，但是一般不会这么做。

### 1.4、__new__方法
通过__new__方法也可以实现单例模式，代码如下：
```python
class Human:
    obj = None

    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __new__(cls, *args, **kwargs):
        if not cls.obj:
            cls.obj = super.__new__(cls)
        return cls.obj
```
前面我们说过，在实例化对象时，会先调用类的__new__方法，这里我们直接在__new__方法中实现单例即可。


### 1.5、元类
通过自定义元类来实现，代码都是差不多的，在元类的__call__方法中定义obj对象。

## 二、python内置函数
python有很多内置函数，并且许多我们都已经用过了。这里将常用函数都列举在此：
参考：[python内置函数](https://www.aliyundrive.com/s/XFB9gh3KApd/folder/647b1180de24778561a74d48b0ab56bb8c28668d)
---
title:       Python基础 - 反射机制
subtitle:    "反射机制"
description: "反射机制指的是在程序运行过程中，动态获取对象信息以及动态调用对象方法的功能。反射机制也是动态语言的一个特性，因为只有在运行时，才知道传过来的数据是什么，它有什么属性。python提供一些内置方法在某些时刻自动触发运行，例如构造函数，析构函数等。"
excerpt:     "反射机制指的是在程序运行过程中，动态获取对象信息以及动态调用对象方法的功能。反射机制也是动态语言的一个特性，因为只有在运行时，才知道传过来的数据是什么，它有什么属性。python提供一些内置方法在某些时刻自动触发运行，例如构造函数，析构函数等。"
date:        2025-05-20T17:57:26+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/10/bc/wheat_field_girl_walking_field_outdoors_meadow_walking_caucasian_portrait-665991.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-reflection"
categories:  [ "PYTHON" ]
---

## 一、反射机制
反射机制指的是在程序运行过程中，动态获取对象信息以及动态调用对象方法的功能。为什么需要反射机制呢？例如，我们定义了一个函数，入参是一个对象，函数内部调用了一个对象的属性，那计算机如何知道这个对象是否有这个属性呢，这就用到了反射机制。

反射机制也是动态语言的一个特性，因为只有在运行时，才知道传过来的数据是什么，它有什么属性。

## 二、反射实现
在python中，提供了一个dir方法可以用来获取一个对象有哪些属性。
```python
class Human:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def f1(self):
        print(self.name, self.age)

obj = Human('wfj', 18)
print(dir(obj))

## 执行结果
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'age', 'f1', 'name']
```
如上所示，获取到了obj对象的所有属性，但是这是一个字符串列表列表，我们不能通过对象.字符串来访问对象的属性。虽然有的对象可以通过__dict__来获取获取并访问，但并不是所有的对象都可以访问__dict__的。因此就需要反射机制将字符串反射到对应的属性上。python提供了四个内置函数来实现这一需求：
```python
hasattr()   # 判断对象是否有某一个属性
getattr()   # 获取对象的某个属性
setattr()   # 给某个属性赋值
delattr()   # 删除某一个属性

print(hasattr(obj, 'name'))
print(getattr(obj, 'name'))
print(setattr(obj, 'name', '李白'))
print(getattr(obj, 'name'))
## 执行结果
True
wfj
None
李白
```

## 三、反射案例
我们再看一个使用反射实现的案例
```python
class Ftp:
    def put(self):
        print("上传数据")
    def get(self):
        print("下载数据")
    def interact(self):
        opt = input(">>>")
        getattr(self, opt, slef.warning())
    def warning(self):
        print("你输入的功能不存在")
```
如上所示，当用户输入字符串时，通过反射找到对应的功能，其实前边我们通过字典实现过类似的功能，也属于反射的一种应用。

## 四、内置方法
内置方法是以为__开头和结尾的方法，会在满足条件的时候自动执行。它的作用是定制化我们的对象或者类，比如构造函数__init__。

### 4.1、__str__方法
我们在执行打印时，例如列表我们会打印出来，但是自定义的类打印出来确是内存地址。原因就是python内置的列表类定义了__str__方法。在打印的时候方便我们查看。
```python
class Human:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __str__(self):
        print('__str__运行了')
        return f'{self.name}:{self.age}'

obj = Human('wfj', 18)
print(obj)

## 执行结果
__str__运行了
wfj:18
```
如上所示，我们打印一个对象时，它的__str__方法就被自动运行了。

### 4.2、__del__方法
对象在销毁时会优先调用__del__方法，这个方法也被叫做析构函数。我们给上边那个例子加一个__del__方法。
```python
def __del__(self):             # 可以理解为析构函数
    print('__del__运行了')
```
在代码执行结束时，就执行了这个方法，这是因为程序运行结束，会销毁对象。析构函数多用来回收资源。例如关闭文件，关闭网络连接。
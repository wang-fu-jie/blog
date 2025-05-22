---
title:        Python基础 - 元类
subtitle:    "元类"
description: "元类是用于创造类的一个特殊类，默认的元类是type，我们也可以自定义元类，继承了type的类都是元类。本文将介绍类产生过程以及如何自定义一个元类。"
excerpt:     "元类是用于创造类的一个特殊类，默认的元类是type，我们也可以自定义元类，继承了type的类都是元类。本文将介绍类产生过程以及如何自定义一个元类。"
date:        2025-05-22T10:28:43+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/d0/c1/grass_sparkler_spark_sparkle_hand-58846.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-metaclass"
categories:  [ "PYTHON" ]
---

## 一、元类
在自定义类的过程中，通过class关键字来创造一个类，那class是如何把类造出来的呢，那就是通过元类。元类是一个用于实例化产生类的类。可以使用type函数查看类的元类
```python
class Human:
    pass
print(type(Human))

## 执行结果
<class 'type'>
```
可以看到，type类就是Human的元类。因此用class关键字定义的类以及内置的类，都是调用内置的元类type实例化产生的。

### 1.1、class分析
我们来看一下关键字class定义一个类，内部做了哪些动作。既然类是调用元类type()产生的，那首先要给元类传参，这里需要准备参数, 例如我们定义一个Human类
```python
#  类名
class_name = 'Human'
# 基类, 默认继承pbject类
class_bases = (object,)

# 执行类子代码，产生名称空间
class_dic = {}
class_body = '''
def __init__(self, name, age):
    self.name = name
    self.age = age
def f1(self):
    print(self.name, self.age)
'''
exec(class_body, {}, class_dic)  # 中间一个字典传入对全局变量的引用，如果没有传入空字典
```
如上可以看到，我们需要准备类名，类继承的父类，类的子代码作为字符串传入生成类的名称空间。接下来就可以调用元类了：
```python
print(type(class_name, class_bases, class_dic))
Human = type(class_name, class_bases, class_dic)

## 执行结果
<class '__main__.Human'>
```
如上所示，我们就自己通过调用type元类创造了一个类，没有通过class关键字。

### 1.2、自定义元类
前面我们通过调用type元类创造了新类，也可以自定义一个元类，就可以使用高度定制化新类。自定义的元类需要继承type。
```python
class MyType(type):
    def __init__(self, class_name, class_bases, class_dic):
        if '_' in class_name:
            raise NameError('类名不允许有下划线')

        if not class_dic.get('__doc__'):
            raise SyntaxError('定义类必须写注释')
        print(class_bases)
        print(self.__bases__)

    def __new__(cls, *args, **kwargs):
        print('Mytype.__new__')
        print(cls)
        print(args)
        print(kwargs)
        return super().__new__(cls, *args, **kwargs)

class Human(metaclass=MyType):
    """
    测试元类
    """
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def f1(self):
        print(self.name, self.age)
```
如上我们就自定义了一个元类，并使用自定义元类创造了Human类，在元类中，我们做了类型不能带下划线，类必须有文档等约束。这里说明一下造Human类的三个步骤
* 调用Mytype的__new__方法 产生一个空对象Human
* 调用MyType的init方法，初始化对象Human
* 返回初始化好的对象 Human

使用元类造类的第一步，就是调用了__new__方法，__new__方法帮类继承了object,并创建一个空对象进行返回。

这三个步骤在底层调用了 __call_方法完成了这三步，我们这里研究一下__call_方法
```python
class Test:
    def __call__(self, *args, **kwargs):
        print("__call__")
obj = Test()
obj()
```
如上，在一个类中定义了__call_方法， 当对象后加括号时，就自动调用了这个方法。因此我们在使用MyType()造类时，调用了它的父类也就是type类的__call_方法。因为前边我们说了使用class定义类的第四步就是调用元类。

## 二、属性查找
如果通过对象找属性，还是和以前一样，一直找到object类，没有就会报错，但是通过类找属性时，object类没有，还会去元类里面找。这里就不再进行演示了。最后需要说明，一般编程中，基本不会自定义元类，关于元类的知识这里可以只做了解。
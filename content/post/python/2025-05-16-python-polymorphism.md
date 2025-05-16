---
title:       Python基础 - 面向对象之多态
subtitle:    "面向对象之多态"
description: "多态也是一种编程思想，即同一操作作用于不同对象时，能产生不同的结果。python可以通过继承、鸭子类型，抽象基类实现多态，本身比较提倡鸭子类型，类似于linux中一切皆文件的理念。本文还讲介绍类方法和静态方法，类方法是绑定给类的方法，会自动传入类。静态方法属于普通函数，类和对象皆可调用，默认不传入参数。"
excerpt:     "多态也是一种编程思想，即同一操作作用于不同对象时，能产生不同的结果。python可以通过继承、鸭子类型，抽象基类实现多态，本身比较提倡鸭子类型，类似于linux中一切皆文件的理念。本文还讲介绍类方法和静态方法，类方法是绑定给类的方法，会自动传入类。静态方法属于普通函数，类和对象皆可调用，默认不传入参数。"
date:        2025-05-16T10:13:00+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/8e/88/tree_road_fall_travel_car-24101.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-polymorphism"
categories:  [ "PYTHON" ]
---

## 一、 多态介绍
多态也是一种编程思想。多态就是同一操作作用于不同对象时，能产生不同的结果。例如，鸡鸭鹅三个类都是继承了家禽类。家禽类中存在一个eat方法。那无论一个对象属于鸡鸭鹅哪个类，都可以调用eat方法。多态的优点是，在使用类对象时，只需要知道他们的父类有哪些方法就可以直接使用。

继承和多条的区别，继承：子类"是什么"（狗是一种动物） 多态：子类"怎么做"（狗怎么叫和猫怎么叫不同）。 但是在python中，并不推崇使用继承的方式表达多态，

### 1.1、鸭子类型
在python中，可以通过继承来表达多态，但是并不推崇，因为继承需要有父类，有了父类就会有耦合。python提倡鸭子类型，即有三个类，鸡鸭鹅并不继承父类，但是他们都自己实现了eat方法。比如linux操作系统一切皆文件的思想，把硬盘、内存、网卡都实现read，wirite方法，那就可以把我们都当成文件去读写，这样在操作linux的对象时都调用read write方法就可以了。这就是多态的一种体现。

### 1.2、抽象基类
如果不想使用鸭子类型，就想用父类达到规范子类的效果。就可以引入抽象基类的概念。父类本身是规范子类的，而不是具体实现方法，需要子类自行实现。这时就需要通过抽象基类强制子类来实现。这优点类似于java中的接口，抽象基类需要引入abc模块
```python
import abc

class Car(metaclass=abc.ABCmeta):
    @abc.abstractmethod
    def run(self):
        pass
```
如上所示，Car是抽象基类，给类Car定义了抽象方法，继承了Car的子类必须实现run方法，否则实例化会报错。同样抽象基类是不允许实例化的，它的作用是用来规范标准的。

## 二、类方法
前面学习面向对象时，我们知道了绑定方法，它是绑定给对象使用的，调用时会自动把对象本身作为第一个参数传入。 除此之外，还有另外一种绑定方法，是绑定给类使用的，叫做类方法。类方法调用时会自动把类传进去。
```python
class Mysql:
    def __init__(self, ip, port):
        self.ip = ip
        self.port = port

    def f1(self):
        print(self.ip, self.port)

    @classmethod
    def instance_from_conf(cls):      # 类方法，类调用的时候会把类传进去
        print(cls)
        obj = cls(settings.IP, settings.PORT)
        return obj

obj1 = Mysql.instance_from_conf()
print(obj1.__dict__)

## 执行结果
<class '__main__.Mysql'>
{'ip': '127.0.0.1', 'port': 3306}
```
如上所示，定义了一个类方法，在调用时会自动把传穿进去，类方法的使用场景是比较窄的，一般用来造对象。例如上例中为了避免每次实例化对象都传入相同的配置，就可以使用类方法来造对象。这里的cls也是一个变量名，和self一样的，换成任意名字也都可以，但是规范如此。


## 三、静态方法
静态方法既不需要用到对象，也不需要用到类，因此它调用时不需要自动传类或对象。
```python
class Mysql:
    def __init__(self, ip, port):
        self.ip = ip
        self.port = port

    def f1(self):
        print(self.ip, self.port)

    @staticmethod             # 静态方法，不会自动传参
    def f2(self):
        print('hello world')
```
如上所示，f2函数被装饰成非绑定方法，也就是静态方法。静态方法是一个普通函数，类和对象都可以调用它，但是没有自动传参的效果。


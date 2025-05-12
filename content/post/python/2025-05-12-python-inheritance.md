---
title:       Python基础 - 面向对象之继承
subtitle:    "面向对象之继承"
description: "继承是面向对象的第二大特性，它是一种创建新类的方式，通过继承创建的类称之为子类，被继承的类称之为父类或基类。Python是支持多继承的，并且使用MixIns机制解决多继承引发的代码可读性变差的缺点。"
excerpt:     "继承是面向对象的第二大特性，它是一种创建新类的方式，通过继承创建的类称之为子类，被继承的类称之为父类或基类。Python是支持多继承的，并且使用MixIns机制解决多继承引发的代码可读性变差的缺点。"
date:        2025-05-12T09:52:18+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/05/57/new_year's_eve_leipzig_fireworks_fire_fireworks_art_beacon_rocket_light-1202475.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-inheritance"
categories:  [ "PYTHON" ]
---

## 一、继承介绍
继承是面向对象的三大特性之一，继承是一种创建新类的方式，通过继承创建的类称之为子类，被继承的类称之为父类或基类。我们先看示例：
```python
class Parent1:
    x = 10

class Parent2:
    pass

class Child1(Parent1):  # 单继承
    pass

class Child2(Parent1, Parent2):  # 多继承
    pass

print(Child1.__bases__)
print(Child2.__bases__)
print(Parent1.__bases__)
print(Parent2.__bases__)

## 执行结果
(<class '__main__.Parent1'>,)
(<class '__main__.Parent1'>, <class '__main__.Parent2'>)
(<class 'object'>,)
(<class 'object'>,)
```
如上所示，在定义类时使用()传入要继承的类名，即可继承类。Python支持多继承，一个类可以继承多个类。通过类的内置属性__bases__可以获取当前类的父类。object是所有类的基类。python3中类默认会继承object，即python3中都是新式类。object类提供了一些常用的属性，例如__dict__。

补充：在python2中，有经典类和新式类之分。新式类是继承了object类的子类，以及继承了这个类的子子孙孙类。经典类则相反，经典类是没有继承object类的子类，以及继承了这个类的子子孙孙类。python2中默认定义的都是经典类，指定继承了object之后才是新式类。

### 1.1、继承的特性
继承的特性就是遗传，即父类有的属性和功能子类，子类都可以访问到。继承也是为了解决代码重复的问题，多个类共有的属性可以用同一个父类
```
print(Child1.x)

## 执行结果
10
```
这里子类Child1没有x属性，但是它的父类有，那Child1就可以访问到。如果子类自身也有x属性，那它会优先访问自己的属性。


### 1.2、多继承
多继承是一个类同时继承多个类，它可以遗传多个父类的属性。但是多继承也是存在缺点的，多继承第一个缺点是违背了人类的思维习惯，第二个缺点会使代码的可读性变差。因此一般情况下不建议使用多继承。


## 二、继承的实现
在编写代码中，我们将多个类共有的属性部分抽象成一个父类。这样就可以解决代码重复的问题，
```python
class Human:
    star = 'earth'

    def __init__(self, name, age, gender):
        self.name = name
        self.age = age
        self.gender = gender


class Chinese(Human):
    nation = 'China'

    def __init__(self, name, age, gender, balance):
        Human.__init__(self, name, age, gender)
        self.balance = balance

    def speak_chinese(self):
        print(f'{self.name}会说普通话')


class American(Human):
    star = 'sourth_earth'
    nation = 'America'

    def speak_english(self):
        print(f'{self.name}会说英语')


obj = Chinese('wfj', 18, '男', 3000)
print(obj.__dict__)
obj.speak_chinese()
print(obj.nation)
print(obj.star)
```
如上，子类Chinese和American除了继承父类的属性外，还有自己的属性nation和speak_xxx函数，这就是派生的第一种情况，子类派生父类没有的属性。American自己也定义了star属性，这属于派送的第二种情况，子类存在与父类的同名属性，以子类自己的属性为准。

再看Chinese类的__init__方法，它属于派生的第三种情况，Chinese类需要有自己的属性balance，但是不能修改父类的__init__函数，因为这个属性不是共有的。因此Chinese类需要定义自己的__init__函数，再调用父类的__init__函数。这个方法不是全新的方法，也不是父类的方法，而是结合了父类的方法做了进一步扩展。

### 2.1、单继承的属性查找
在通过子类创造的对象，会优先查找对象自身的属性，对象没有再去查找类的属性，类也没有再去查找父类的属性。我们看一个特殊的例子：
```python
class Test1:
    def f1(self):
        print('Test1.f1')

    def f2(self):
        print('Test1.f2')
        self.f1()


class Test2(Test1):
    def f1(self):
        print('Test2.f1')


obj = Test2()
obj.f2()

## 执行结果
Test1.f2
Test2.f1
```
这里惯性思维可能会认为打印Test2.f1。 但是这里调用f1函数的时候，是调用self的f1，self是Test2的对象，因此调用的是Test2的f1方法。如果想调用Test1的f1方法，可以直接用类名调用，或者把f1改为隐藏属性，这样即使Test2的f1也是隐藏属性，他们被python修改后的属性名也不一样。


### 2.2、菱形继承与MRO
菱形继承是一个子类继承自两个父类，这两个父类又都继承自同一个基类。先看代码：
```python
class A:
    def f1(self):
        print('A.f1')

class B(A):
    def f1(self):
        print('B.f1')

class C(A):
    def f1(self):
        print('C.f1')

class D(B, C):
    def f2(self):
        print('D.f2')

obj = D()
obj.f1()
print(D.mro())  # 属性是基于这个mro列表查找到的

## 执行结果
B.f1
[<class '__main__.D'>, <class '__main__.B'>, <class '__main__.C'>, <class '__main__.A'>, <class 'object'>]
```
如上所示，类D的对象访问f1，那会访问哪个父类的f1呢，即多继承的属性查找问题。在多继承中，属性的查找是根据MRO列表来进行查找的。MRO列表是底层通过一个C3算法实现的。如上所示，D的MRO列表的查找顺序是 D -> B -> C -> A -> object


### 2.3、非菱形继承的属性查找
对于非菱形继承，新式类和经典类的属性查找顺序是一样的，唯一的区别是经典类不会查找object。
![图片加载失败](/post_images/python/{{< filename >}}/2.3-01.png)
如上图所示，这里F类的MRO查找顺序是 F -> C -> A -> D -> B -> E -> object。这里可以自行写代码验证。注意：这里经典类不会查找object，但是如果你的python2代码中，一部分类是新式类，一部分是经典类，那继承关系就不移动了，可以会在中间查找object，但是一般不会这么做，要么全是新式类，要么全是经典类。

### 2.4、菱形继承的属性查找
针对菱形继承，新式类和经典类的查找方式是不一样的。我们看下这个图在经典类和新式类分别是如何查找的
![图片加载失败](/post_images/python/{{< filename >}}/2.4-01.png)
经典类是深度优先查找： 即 F -> C -> A -> Z -> D -> B -> E

新式类是广度优先查找： F -> C -> A -> D -> B -> E -> Z -> object

## 三、MixIns机制
多继承虽然减少了冗余代码，但是代码可读性变差。因此python引入了MixIns机制来解决这个问题。举个例子，鸡鸭鹅三个类都是家禽，他们的父类是家禽类，但是鸭和鹅会游泳，因此鸭和鹅需要单独继承一个游泳类。这样解决了代码冗余，但是不符合人类的思维习惯。因此我们把游泳作为一个附加功能混入，
```python
class Fowl:
    pass

class SwimMixIn:
    def swimming(self):
        pass

class Chicken(Fowl):
    pass

class Duck(SwimMixIn, Fowl):
    pass

class Goose(SwimMixIn, Fowl):
    pass
```
代码如上，MixIns机制并没有改变多继承的本质，只是从命名上表示，这是引入的一个功能。在使用MixIns类的功能应该单一，不依赖与其他类，就像一个插件一样的存在。不引入这个插件就没有这个功能。在java中不支持多继承，但是java肯定会有通用的问题，java提供了一个接口的概念，它定义了应该具有什么功能，但是它不实现这个功能。java这种实现方式其实和python的MixIn是很像的。


## 四、super关键字
在前面示例中，子类调用父类的方法，我们是直接通过父类的类名调用的，因此它没有依赖继承关系。当然也有另外一种方法，那就需要使用super关键字，super是严格依赖继承关系实现的重用父类代码。
```python
class Human:
    star = 'earth'

    def __init__(self, name, age, gender):
        self.name = name
        self.age = age
        self.gender = gender


class Chinese(Human):
    nation = 'China'

    def __init__(self, name, age, gender, balance):
        # Human.__init__(self, name, age, gender)
        super(Chinese, self).__init__(name, age, gender)
        self.balance = balance

    def speak_chinese(self):
        print(f'{self.name}会说普通话')
```
如上所示：我们把Human替换成了super(Chinese, self)。这里调用super会产生一个特殊的对象，这个对象会参照self所属类的mro列表，从当前类所处位置的下一个类找属性。需要注意的是，super只能用到新式类里面。而且python2中必须这么写 super(Chinese, self)， 但是在python3中直接写super()即可。

重点说明，super在找属性的时候，参照的是对象所属类的mro列表，并不一定是我们眼睛看到的父类。且从super所在类的下一个类开始找，并不是从第一个类开始找。
```python
class A:
    def f1(self):
        print('A.f1')
        super().f1()

class B:
    def f1(self):
        print('B.f1')

class C(A, B):
    pass

obj = C()
print(C.mro())
obj.f1()

## 执行结果
[<class '__main__.C'>, <class '__main__.A'>, <class '__main__.B'>, <class 'object'>]
A.f1
B.f1
```
如上，C的对象，调用f1，根据mro列表，调用了A类的f1函数，但是A类的f1中有super，因此下一个类是B，因此super就调用了B类的f1。
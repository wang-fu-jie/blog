---
title:       Python基础 - 面向对象与类
subtitle:    "面向对象与类"
description: "面向对象是一种编程思想，对象就是数据和行为的集合体，主要通过类进行实现。面向对象的三大特点是，封装、继承、多态。本文我们将介绍面向对象思想和类的定义及使用，以及类的构造函数，属性的查找顺序，类的绑定方法、隐藏属性和类装饰器"
excerpt:     "面向对象是一种编程思想，对象就是数据和行为的集合体，主要通过类进行实现。面向对象的三大特点是，封装、继承、多态。本文我们将介绍面向对象思想和类的定义及使用，以及类的构造函数，属性的查找顺序，类的绑定方法、隐藏属性和类装饰器"
date:        2025-05-08T10:15:14+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/44/25/horse_wild_horse_marsh_pony_swamp_grazing_gull_seagull-1144952.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-object-class"
categories:  [ "PYTHON" ]
---

## 一、面向对象
在此之前，我们编程用的都是面向过程的思想，即程序流程化。面向对象是另一种编程思想，对象就是数据和行为的集合体，以 “对象” 为核心，通过 封装、继承、多态 等机制组织代码，使程序更模块化、易维护、易扩展。

面向对象只是一种思想，它并没有固定的实现方式，例如C语言中可以通过结构体将数据和功能封装到一起，Python也可以字典将数据和功能封装到一起，这都属于利用面向对象的思想编程。但是在Python中，还提供一种语法支持面向对象的思想，那就是类。

## 二、类
类是对象的模板，可以用来定义对象的属性和方法。类解决了不同的对象存相同数据的问题。在编程中，需要先定义类，也就是先把公有的数据定义出来，将类作为模板再创建对象。python提供了class关键字用来对类做定义。
```python
class Hero:
    hero_work = '射手'
    def get_hero_info(hero_obj):
        print(f'英雄属性：名字{hero_obj.name} 移速{hero_obj.speed} 生命值 {hero_obj.hp} 攻击力 {hero_obj.atk}')
    def ser_hero_speed(hero_obj, speed_plus):
        hero_obj.speed += speed_plus
    print("类的定义")



# 执行结果
类的定义
```
如上所示，我们定义了一个类，类名为Hero。python的规范要求类型使用驼峰命名方式，和变量不同，变量使用下线线命名方式。在运行这段函数时，可以看到打印了内容，说明类是在定义阶段就运行了。同样在定义阶段就产生了类的名称空间。

## 三、类的使用
一般情况下，类作为对象的模板，类的数据和功能是给对象来使用的。目前我们还没有创建对象，类也是可以用这些功能的。我们可以通过 类名.__dict__获取类名称空间的名字。
```python
print(Hero.__dict__)

## 执行结果
{'__module__': '__main__', 'hero_work': '射手', 'get_hero_info': <function Hero.get_hero_info at 0x7fa3ce700d40>, 'ser_hero_speed': <function Hero.ser_hero_speed at 0x7fa3ce70d290>, '__dict__': <attribute '__dict__' of 'Hero' objects>, '__weakref__': <attribute '__weakref__' of 'Hero' objects>, '__doc__': None}
```
可以看到，我们定义的数据和功能都在这个字典内，python还内置了一些功能，这些我们先忽略。接下来看一下如何使用类的功能。
```python
print(Hero.__dict__['hero_work'])
print(Hero.hero_work)
```
如上所示，我们可以通过__dict__来获取类的属性或功能，但是python提供了更简洁的方式，类名.属性名 就可以直接访问了。

### 3.1、创造对象
既然类是对象的模板，那就可以基于类来创造一个对象，创造方式是 类名()，就会返回一个对象。对象通用有__dict__属性。我们来看一下
```python
hero_obj = Hero()
print(hero_obj.__dict__)
print(hero_obj.hero_work)

## 执行结果
{}
射手
```
对象的__dict__属性返回了空字典，这是因为只创建了对象，还没有往对象里存内容。但是类中定义数据和功能对象是可以访问到的。我们可以给hero_obj.__dict__这个字典添加元素实现给对象添加数据或者通过对象名+点的方式简写，但是这样太麻烦了而且创建的多个对象都需要单独添加数据，又会造成代码重复。前面我们知道解决代码重复的问题的方法就是封装为函数，因此python提供了__init__函数解决这个问题。

### 3.2、构造函数__init__
构造方法可以用来在创建对象时，给对象独有的属性值。并且在创建对象时自动执行，这个就是__init__函数。
```python
class Hero:
    hero_work = '射手'

    def __init__(self, name, speed, hp, atk):   # self只是个变量名，换成其他都可以，规范要求使用self
        self.name = name
        self.speed = speed
        self.hp = hp
        self.atk = atk

    def get_hero_info(self):
        print(f'英雄属性：名字{self.name} 移速{self.speed} 生命值 {self.hp} 攻击力 {self.atk}')

    def ser_hero_speed(self, speed_plus):
        self.speed += speed_plus
hero_obj = Hero('鲁班七号', 450, 3000, 110)
```
如上所示，我们创建了一个对象，它创建是会自动调用__init__函数函数，这里self参数会自动传入对象本身，当然也可以把self参数改为其他名字，self只是规范。

### 3.3、属性的查找顺序
在通过调用类产生的对象，就会同时产生一个对象的命名空间，里面存放了对象的属性。对象在调用属性时会优先查找对象自己的属性，如果没有就去查找类的属性。
```python
print(id(hero_obj.hero_work))
print(id(Hero.hero_work))

## 执行结果
140470233685040
140470233685040
```
可以看到对象和类访问hero_work时，是访问的同一块内存地址。如果把类的属性给修改了，那对象访问也是被修改后的值。如果通过对象修改hero_work，那就会给对象增加一个hero_work属性，而不会去修改类的属性，这里可以自行验证。

### 3.4、绑定方法
接下来再看一下类的方法，首先类定义的方法类自身是可以使用的。通过类来进行调用需要严格按照函数的用法来，有几个参数就要传几个参数，例：
```python
print(Hero.get_hero_info)
Hero.get_hero_info(hero_obj)

## 执行结果
<function Hero.get_hero_info at 0x7ff1c6ef3320>
英雄属性：名字鲁班七号 移速450 生命值 3000 攻击力 110
```
如上所示，类型调用，需要传入一个对象赋值给self，我们这里把hero_obj传入就正常执行了。但是，类定义的函数主要给对象使用，而且是绑定给对象使用的。虽然所有对象有相同的功能，但是绑定给不同对象后就会变成不同的绑定方法。
```python
print(Hero.get_hero_info)
print(hero_obj.get_hero_info)

## 执行结果
<function Hero.get_hero_info at 0x7ff1c6ef3320>
<bound method Hero.get_hero_info of <__main__.Hero object at 0x7fbffb705d50>>
```
可以看到，对象的方法是绑定方法，它的内存地址发生了变化，但并不意为函数的代码被重新复制了一份。可以理解为类的函数绑定给对象后，这个函数的内存地址被包装成一个绑定方法，当访问这个绑定方法时会访问原来类的方法，相当于一个装饰器。为什么要这么绕一圈呢，因为绑定方法做了一些手脚，在通过对象调用这些方法时，就不需要传入对象本身了，它默认把对象作为第一个参数传入。
```python
Hero.get_hero_info(hero_obj)
hero_obj.get_hero_info()
```
因此在类中定义方法时，至少需要一个形参，因为对象调用会默认把自身作为第一个参数，规范建议第一个参数为self。

## 四、封装
封装是面向对象的第一大特性，我们前面的把数据和函数封到一个类里，这其实就是封装。但是针对封装到容器里的属性，还可以做进一步的处理，例如封装时可以把属性隐藏起来，类装饰器等。

### 4.1、隐藏属性
隐藏属性可以是使用者没有办法直接使用。先来看如何对属性进行隐藏：
```python
class Test:
    __x = 10

    def __f1(self):
        print("f1")

# print(Test.__x)  ## 报错 AttributeError: type object 'Test' has no attribute '__x'
print(Test.__dict__)

## 执行结果
{'__module__': '__main__', '_Test__x': 10, '_Test__f1': <function Test.__f1 at 0x7f997a722170>, '__dict__': <attribute '__dict__' of 'Test' objects>, '__weakref__': <attribute '__weakref__' of 'Test' objects>, '__doc__': None}
```
我们在属性名前加在__，这个属性就变成了隐藏属性，不能直接调用了。我们可以类的名称空间，隐藏属性被自动加了前缀_Test。所以不同直接使用了，但是如果通过python修改后的名字仍然是可以调用的。所以python并不是真正意义的隐藏。

隐藏属性的第二个特点是对外不对内，我们在类的外部无法直接访问隐藏属性，但是在类的内部是可以访问的。因为在类定义时，类内部所有隐藏属性名字都会一起被修改。 隐藏属性第二个特点是这个改名操作只会发生在类定义阶段检查子代码语法时执行一次，之后定义的__开头的属性不会被改名。给对象定义隐藏属性的方法也是一样的，在__init__函数中的属性名也加上__就可以了。

为什么需要隐藏属性呢，因为将属性隐藏后，使用者就不能随便修改了，但是定义类时可以定义访问属性的接口。例如我的数据是整型，接口就可以限制修改的数据也只能是整型，如果不隐藏那使用者就可以改为任意类型。示例：
```python
class Test:

    def __init__(self, age):
        self.__age = age

    def get_age(self):
        return self.__age

    def set_age(self, age):
        if type(age) is not int:
            print("必须传入整型")
            return
        self.__age = age
```
隐藏类的方法原因也很简单，如果一个函数提供的功能，需要多步完成，就可以把步骤封装为单独的属性，但是这些单独的步骤又不需要对外提供。这样就隔离了复杂度


### 4.2、类装饰器
上文中，使用隐藏属性隐藏了age，因此提供了接口get_age获取age值，但是获取属性值正常应该使用 . 来获取，这里确改成了调用函数。如果还用 . 直接获取属性，就需要使用python内置的property装饰器。property是一个类实现的装饰器，以往我们都是使用函数实现的装饰器。property的作用是把绑定给对象的方法伪装成一个数据属性。
```python
class Test:
    def __init__(self, name, age):
        self.__name = name
        self.__age = age

    @property
    def age(self):
        return self.__age

    @age.setter
    def age(self, new_age):
        self.__age = new_age

    @age.deleter
    def age(self):
        del self.__age

    # age = property(get_age, set_age, del_age)

obj = Test('xxxx', 18)
# print(obj.get_age)
print(obj.age)
obj.age = 20
print(obj.age)
del obj.age
```
如上所示，我们就可以通过访问属性的方式来调用操作属性的接口了。对使用者来说使用属性的方式没变，但其实内部已经在调用我们的接口了。
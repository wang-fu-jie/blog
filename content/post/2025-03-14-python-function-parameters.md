---
title:       Python基础 - 函数的参数与返回值
subtitle:    "函数的参数与返回值"
description: "函数支持传入数据并在函数内部使用，这些数据通过参数传入。在函数运行结束，可以返回结果给调用者就是返回值。函数的参数包括位置参数、关键字参数、默认参数、可变长参数。在调用函数时传递给函数的值称为实际参数，而函数定义时列出的变量则称为形式参数"
excerpt:     "函数支持传入数据并在函数内部使用，这些数据通过参数传入。在函数运行结束，可以返回结果给调用者就是返回值。函数的参数包括位置参数、关键字参数、默认参数、可变长参数。在调用函数时传递给函数的值称为实际参数，而函数定义时列出的变量则称为形式参数"
date:        2025-03-14T10:48:24+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/bf/39/wedding_couple_love_wedding_couple_groom_woman_young_romance-813951.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-function-parameters"
categories:  [ "PYTHON" ]
---

## 一、函数的参数
函数的参数分为形式参数和实际参数，简称为形参和实参。在调用函数时传递给函数的值称为实际参数，而函数定义时列出的变量则称为形式参数：
```python
def func(a, b):
    print(a, b)
func(1, 2)

##执行结果
1 2
```
在调用函数时，1会传给a，2会传给b。这里a和b是形式参数，1和2时实际参数。

## 二、函数的返回值
函数的返回值是指函数在执行完毕后，通过 return 语句将计算结果或处理结果传递给调用者的数据，可以使用一个变量进行接收返回值，也可以直接使用这个返回值，例如这里直接使用返回值作为打印函数的参数。
```python
def func(a, b):
    return a + b
print(func(1, 2)）

## 执行结果3
```
如果不写return或者return后没有内容，那默认返回值是None。另外return代表函数结束的标志，如果在return后还有代码，那这些代码是不会执行的。reruen还可以返回多个值，使用逗号隔开，多个值会作为一个元组返回。
```python
def func():
    return 1, 2
res = func()
print(res, type(res))

## 执行结果
(1, 2) <class 'tuple'>
```
这里返回的元组也可以使用多个变量进行接收，也就是进行了解压赋值。

## 三、参数的类型
Python函数的参数可以支持多种类型，分别为位置参数、关键字参数、默认参数、可变长参数。

### 3.1、位置参数
位置参数是从左到右依次定义的参数。位置形参在调用函数的时候必须给它传值
```python
def func(x, y, z):
    print(x, y, z)
func(1, 2, 3)
```
这里的x,y,z就是位置形参，1,2,3是位置实参，他们按照参数位置一一对应。 

### 3.2、关键字参数
关键字参数指的是实参，在调用函数是给实参传值也可以不按照顺序，按照key=value的格式传值，这种格式就叫关键字参数
```python
def func(x, y, z):
    print(x, y, z)
func(y=1, x=2, z=3)
func(1, 2, z=3)
```
注意，关键字参数这里的key必须是形参中定义的。第二种方式就是位置实参和关键字实参的混用，1和2按照位置传给x和y, z=3通过关键字传给了z。位置实参必须放在关键字实参之前。

### 3.3、默认参数
默认参数指的是形参，就是在定义阶段，就被赋予了默认值的形参。在调用时默认形参可以传值，也可以不传值，如果不传值那就使用默认值。
```python
def func(x, y=2):
    print(x, y)
func(1)
func(1, 2)
func(x=1)
func(x=1, y=2)
func(1, y=2)
```
以上代码，这几次函数调用运行的结果都是一样的。另外定义时，默认形参的定义必须放在位置形参之后，这也是Pytho的语法规定。默认参数的值是函数定义阶段被赋值的。默认形参的值可以指定为任意类型，但是通常不建议使用可变类型。

### 3.4、可变长参数
可变长参数是指在调用函数时，传入的参数个数是不固定的。*形参名就可以接收多出来的位置实参，**形参名可以接收多出来的关键字实参。
```python
def func(x, y, *args):
    print(x, y, xrgs)
func(1, 2, 3, 4, 5)

## 执行结果
1 2 (3, 4, 5)
```
如上实例，x和y是位置实参，分别接收1和2。3,4,5就是多出来的位置实参，用z接收，多出来的参数就会被处理成元组。
```python
def func(x, y, **kwargs):
    print(x, y, kwargs)
func(1, 2, a=3, b=4, c=5)

## 执行结果
1 2 {'a': 3, 'b': 4, 'c': 5}
```
如上所示：多出来的关键词实参会被处理为字典然后赋予kwargs。这里可变长参数一般建议使用 *args 和 **kwargs，这样比较规范，当然也可以用其他变量名。在给函数传实参时，也可以使用变长参数。
```python
def func(x, y, z):
    print(x, y, z)
func(*[1,2,3])
```
*的作用把列表打散为位置实参。这里传入的不一定是列表，只要可迭代对象都会都\*打散，当然字典也可以，只不过字典被打散的是key。
```python
def func(x, y, z):
    print(x, y, z)
func(**{'x':1, 'y':2, 'z':3})
```
关键字参数也可以使用**打散，只不过这里只能是字典。

可变长参数可以混用，但是*args必须放在**kwargs之前。


## 四、形参的选择
在定义函数的参数时，如果参数变化比较频繁，就推荐使用位置参数，如果参数基本不会变化，推荐使用默认参数。

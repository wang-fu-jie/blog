---
title:       Python基础 - 名称空间与作用域
subtitle:    "名称空间与作用域"
description: "在栈区中存放的变量名、函数名等名字被存放在三个不同的名称空间，分别为内置名称空间、全局名称空间，局部名称空间。作用域是名词作用的范围，分为全局作用域和局部作用域。内置名称空间、全局名称空间属于全局作用域，局部名称空间属于局部作用域。"
excerpt:     "在栈区中存放的变量名、函数名等名字被存放在三个不同的名称空间，分别为内置名称空间、全局名称空间，局部名称空间。作用域是名词作用的范围，分为全局作用域和局部作用域。内置名称空间、全局名称空间属于全局作用域，局部名称空间属于局部作用域。"
date:        2025-03-17T13:54:07+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/9c/94/foggy_olive_tree_sun_sun_stripes_nature_landscape_tree_countryside-1057043.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-namespace-scope"
categories:  [ "PYTHON" ]
---

## 一、名称空间与作用域
名称空间(Namespaces)是一个用于存储变量（包括函数、类、模块等）名称与对应对象之间映射的环境。前边我们说过，在定义一个变量时，值存放在堆区，变量名存在于栈区，名称空间就是给栈区的变量名进行了分类。分为全局名称空间、内置名称空间和局部名称空间。这样存在于不同名称名称空间的变量名重复也不会冲突，导致被覆盖的问题。

作用域是根据作用范围的不同进行的分类，全局名称空间、内置名称空间属于全局作用域，局部名称空间属于局部作用域。

### 1.1、内置名称空间
内置名称空间存放的是python解释器内置的名字。解释器一启动，就会产生内置名称空间。解释器关闭，内置名称空间销毁。
```python
>>> print
<built-in function print>
```
例如print函数，它是一个内置函数，就位于内置名称空间。

### 1.2、 全局名称空间
全局名称空间包括：Python文件内定义的变量名，包含函数名、类名和模块名。总结就是只要不是函数内部定义的名字和内置名字，剩余的都是全局空间的名字。全局名称空间会在Python文件执行前产生，运行完毕后销毁。

### 1.3、局部名称空间
局部名称空间是函数内部定义的名字，包括函数参数。在函数调用时产生，调用结束销毁。

### 1.4、名字的查找顺序
名称空间查找优先级，局部名称空间>全局名称空间>内置名称空间。查找时从当前位置开始查找。例如如果在全局空间使用名词，会先查找全局名称空间，没有的话再去查找内置名称空间，仍然没有就会抛出错误
```python
input = 1
def func():
    input = 2
    print(input)

func()

## 执行结果
2
## 如果注释掉input = 2，则执行结果
1
## 如果再注释掉input = 1，则执行结果
<built-in function input>
```
如上所示，就验证我们前边说的查找结果。再看两个示例：
```python
## 示例一
def func():
    print(x)
x = 10
func()

## 执行结果
10

## 示例二
x = 10
def func1():
    print(x)

def func2():
    x = 20
    func1()
func2()

## 执行结果
10
```
第一个示例中，可以正常找到10，因为x变量的定义在前，调用在后。如果把x=10放在函数调用后就会报错。示例二说明名称空间的查找顺序是以定义阶段为准，和调用的位置没有关系。


## 二、函数嵌套
函数也是可以嵌套的。
```python
input =1
def func1():
    input = 2
    def func2():
        input = 3
        print(input)
    func2()
func1()

## 执行结果
3
```
这个示例中，会先在func2的局部名称空间查找，其次是func1的局部名称空间。这块可以分别注释掉input来验证结果。
```python
input =1
def func1():
    def func2():
        print(input)
    func2()
    input = 2
func1()

## 执行结果
NameError: free variable 'input' referenced before assignment in enclosing scope
```
这个报错原因是，在定义阶段就确定了名称空间的查找顺序，确定了名称空间有input变量。但是在运行时先调用了后定义的变量，因此就产生了报错。

## 三、作用域
全局作用域包括全局名称空间和内置名称空间， 它的特点是全局存活和全局有效。局部作用域包含局部名称空间，它的特点是临时存活，局部有效，函数调用时存活，调用完成就销毁。也有一种说法是Python的名称空间和作用域遵循LEGB规则。
* L（Local）局部作用域 → 当前函数内定义的变量
* E（Enclosing）嵌套作用域 → 外层函数的变量（仅适用于嵌套函数）
* G（Global）全局作用域 → 模块级别变量
* B（Built-in）内置作用域 → Python 预定义的变量（如 len()） 

LEGB 规则在 def 定义阶段就已确定，但变量的实际值是在运行时决定的。变量查找顺序不会因为调用位置不同而改变，Python 始终按照 LEGB 规则查找。

### 3.1、global关键字
如果需求在函数内部修改全局作用域的变量，需要使用global关键字。
```python
x = 10
def func():
    global x
    x = 20
func()
print(x)

## 执行结果
20
```
注意这里的变量修改是指的不可变类型，针对可变类型例如列表，可以用列表的功能进行修改，但是注意不能使用定义的语法进行修改，否则会认为在局部作用域定义了新的同名变量。

### 3.2、nonlocal关键字
nonlocal是非本地的意思，通过nonlocal声明的名字指的是上一层的变量，根据LEGB规则。
```python
x = 10
def func1():
    x = 20
    def func2():
        nonlocal x
        x = 30
    print('func2调用前：', x)
    func2()
    print('func2调用后：', x)

func1()
print('全局：', x)

## 执行结果
func2调用前： 20
func2调用后： 30
全局： 10
```
如果x =20没有定义，就会报错SyntaxError: no binding for nonlocal 'x' found


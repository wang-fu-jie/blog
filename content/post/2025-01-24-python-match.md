---
title:       Python基础 - Python模式匹配之match语法
subtitle:    "Python模式匹配之match语法"
description: "在Python3.10版本中，增加了mactch-case语法用作条件的分支选择，可用于多条件判断时替代if。和其他编程语言中的case-when语句类似。在Python中，case关键字后边匹配的叫做模式，因此match-case语法也叫模式匹配。"
excerpt:     "在Python3.10版本中，增加了mactch-case语法用作条件的分支选择，可用于多条件判断时替代if。和其他编程语言中的case-when语句类似。在Python中，case关键字后边匹配的叫做模式，因此match-case语法也叫模式匹配。"
date:        2025-01-24T11:04:02+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/9c/1e/piano_rose_red_flower_love_romance_white_relationship-1086677.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-match"
categories:  [ "PYTHON" ]
---

## 一、模式匹配介绍
如果你学过其他编程语言，就会知道许多语言支持一种叫做case-when的语法。这种语法比if语句的多条件判断可读性更好。但是Python旧版本并没有提供对case-when语句的支持，因此多条件判断只能使用if语句。Python10版本开始，支持了match-case语句用作多条件的分支选择，在Python中，case关键词后面的内容叫做模式。

## 二、match-case用法
match-case语句的功能非常强大，不仅可以支持简单的匹配，还可以支持匹配多个值，匹配一份范围，匹配列表等多种复杂匹配。基本语法格式如下：
```python
match subject:
    case pattern1:
        # 处理 pattern1
    case pattern2:
        # 处理 pattern2
    case _:
        # 默认情况
```

### 2.1、匹配字面值
匹配字面值是最基础的用法，示例如下：
```python
status = 200  # 输出: OK
# status = 404  # 输出: Not Found
# status = 418  # 输出: Unknown status
match status:
    case 200:
        print("OK")
    case 404:
        print("Not Found")
    case 500:
        print("Internal Server Error")
    case _:
         print("Unknown status")
```
如上所示，如果匹配到200，就会输出OK。其他值可以修改通过注释自行测试。 _为通配符，类似于其他编程语言default的功能。它必须在case的最后一个分支。大多数字面值都是按照相等性进行比较的，但是单例对象 True、False、None是按照标识号进行比较的。

### 2.2、使用|匹配多个模式
使用 | 符号可以组合多个字面值，这样一个分支就可以匹配多个不同的字面值。
```python
value = 2  # 输出: Small number
#value = 5  # 输出: Medium number
#value = 10 # 输出: Large number
match value:
    case 1 | 2 | 3:
        print("Small number")
    case 4 | 5 | 6:
        print("Medium number")
    case _:
        print("Large number")
```

### 2.3、匹配序列
序列可以是列表、集合、元组等。这里以列表示例：
```python
lst = []           # 输出: 空列表
# lst = [42]         # 输出: 只有一个元素的列表: 42
# lst = [1, 2]       # 输出: 有两个元素的列表: 1, 2
# lst = [1, 2, 3, 4]  # 输出: 第一个元素是 1，剩余元素是 [2, 3, 4]
# lst = "not a list" # 输出: 未知的列表结构

match lst:
    case []:
        print("空列表")
    case [first]:
        print(f"只有一个元素的列表: {first}")
    case [first, second]:
        print(f"有两个元素的列表: {first}, {second}")
    case [first, *rest]:
        print(f"第一个元素是 {first}，剩余元素是 {rest}")
    case [_, _, third]:
        print(f"有三个元素的列表, 第三个元素为: {third}")
    case _:
        print("未知的列表结构")
```
更复杂的还可以匹配嵌套列表。说明
* 匹配序列时，会按照元素顺序逐一比较。
* 按照从上到下的顺序匹配模式，匹配成功就执行子句返回结果
* 序列中可以使用 _ 占位符，表示那里有元素，但是不关注元素值
* 不能匹配迭代器和字符串
* 模式中可以使用变量进行解构赋值
如示例中的[first, *rest]即为解构赋值，\*之后的名称也可以是占位符 _ 。

### 2.4、匹配字典
在匹配字典时，是按照字典的key进行匹配，value被当成变量进行接收。注意：字典一样支持 ** 用来结构，此处不再做具体演示。但是 **_ 不允许使用，是冗余的。
```python
d = {"name": "Alice", "age": 30}  # 输出: 姓名: Alice, 年龄: 30
# d = {"name": "Bob"}               # 输出: 只有姓名: Bob
# d = {"age": 25}                   # 输出: 只有年龄: 25
# d = {}                            # 输出: 空字典
# d = {"city": "New York"}          # 输出: 未知的字典结构

match d:
    case {"name": name, "age": age}:
        print(f"姓名: {name}, 年龄: {age}")
    case {"name": name}:
        print(f"只有姓名: {name}")
    case {"age": age}:
        print(f"只有年龄: {age}")
    case {}:
        print("空字典")
    case _:
        print("未知的字典结构")
```

### 2.5、匹配类
match还是可以匹配类的对象。如示例所示：
```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

def match_class(obj):
    match obj:
        case Point(x=0, y=0):
            print("Origin")
        case Point(x=0, y=y):
            print(f"Y axis, y = {y}")
        case Point(x=x, y=0):
            print(f"X axis, x = {x}")
        case Point(x=x, y=y):
            print(f"Point is at ({x}, {y})")
        case _:
            print("Not a point")

match_class(Point(0, 0))  # 输出: Origin
match_class(Point(0, 5))  # 输出: Y axis, y = 5
match_class(Point(3, 0))  # 输出: X axis, x = 3
match_class(Point(2, 3))  # 输出: Point is at (2, 3)
```
如果你是根据本网站学习，此处还不知道什么时类没关系，先跳过，后续再回来看。

### 2.5、匹配内建类
在 Python 中，match 语句不仅可以匹配常量、列表、字典等数据结构，还可以匹配内置类（如 int、str、float 等）的实例。下面是一个匹配内置类的示例。
```python
value = 42                # 输出: 这是一个整数: 42
# value = 3.14              # 输出: 这是一个浮点数: 3.14
# value = "Hello, World!"   # 输出: 这是一个字符串: Hello, World!
# value = [1, 2, 3]         # 输出: 这是一个列表: [1, 2, 3]
# value = {"key": "value"}  # 输出: 这是一个字典: {'key': 'value'}
# value = True              # 输出: 未知类型: <class 'bool'>

match value:
    case int():
        print(f"这是一个整数: {value}")
    case float():
        print(f"这是一个浮点数: {value}")
    case str():
        print(f"这是一个字符串: {value}")
    case list():
        print(f"这是一个列表: {value}")
    case dict():
        print(f"这是一个字典: {value}")
    case _:
        print(f"未知类型: {type(value)}")
```

## 三、模式扩展
模式还支持嵌套、捕捉、条件匹配等功能

### 3.1、嵌套模式
模式可以任意的嵌套，即列表中可以嵌套列表，；列表中也可以嵌套类对象。此处不再给出示例，有兴趣的读者请自行编写代码测试。

### 3.2、模式捕获
模式捕获是为了模式匹配成功后，获取到该模式或者其子模式的值，使用关键字 as 实现。
```python
# value = 2          # 输出: 2
value = [(5, 2), 3]  # 输出: (5, 2)
match value:
    case 1 | 2 | 3 as num:
        print(num)
    case [(a, 2) as t, 3]:
        print(t)
```


### 3.3、条件匹配
模式还可以使用if添加守卫条件，如果守护项的值为假，则继续匹配下一个case语句块。注意，值的捕获发生在守护项被求值之前。
```python
value = -5  # 输出: Negative
# value = 0   # 输出: Zero
# value = 5   # 输出: Positive
match value:
    case x if x < 0:
        print("Negative")
    case x if x == 0:
        print("Zero")
    case x if x > 0:
        print("Positive")
```

## 四、总结
可以看出，Python提供的模式匹配功能非常强大，足以完成各种复杂的操作。但是过于复杂的匹配也会降低代码的可读性，请谨慎使用。Python3.10版本才支持了模式匹配功能，如果需要考虑代码兼容性应尽量避免使用match-case语法。
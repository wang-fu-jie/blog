---
title:       Python基础 - Python常量
subtitle:    "使用常量存储不变的数据"
description: "Python是一个灵活的语言，在语法层面没有变量的概念，但是在程序开发过程中又需要常量。因此在Python开发中约定用大写的变量名表示常量。"
excerpt:     "Python是一个灵活的语言，在语法层面没有变量的概念，但是在程序开发过程中又需要常量。因此在Python开发中约定用大写的变量名表示常量。"
date:        2024-12-27T09:37:22+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/01/94/silhouette_child_beach_boy_sunset-19323.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-constant"
categories:  [ "PYTHON" ]
---

## 一、 常量的基本使用
常量就是不变的变量。在编程开发中，经常会遇到不变的值就需要用常量来存储，例如圆周率π。但是在Python中没有常量的概念，因此通常在Python中使用大写的变量名表示常量。例：
```
PI = 3.1415926
```
事实上，PI仍然是一个变量。Python不提供任何机制来保证PI不被改变，如果你非要修改PI的值，那也没人拦得住你。
```python
>>> PI = 3.1415926
>>> PI = 3
>>> print(PI)
3
```

## 二、扩展
在其他编程语言中，可能会有明确的常量的定义方法。例如在C/C++，使用const修饰符来定义常量。const修饰过后的常量是不能被修改的，如果你硬要修改，那程序会直接报错终止。
```c++
const PI = 3.1415926
```
综上：虽然Python自身不限制你对常量进行修改，但是在开发过程中还是要养成良好的编程习惯，遵守这些约定俗称的规范。
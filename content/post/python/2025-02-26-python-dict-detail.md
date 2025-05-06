---
title:       Python基础 - Python字典操作详解
subtitle:    "Python字典操作详解"
description: ""
excerpt:     ""
date:        2025-02-26T10:04:33+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/dd/c2/portrait_woman_girl_blond_hair_long_hair_blonde_hair_female-1221096.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-dict-detail"
categories:  [ "PYTHON" ]
---

## 一、字典的生产和类型转换
字典使用{}定义，它的key必须是不可变类型。字典的定义底层是调用了dict()这个功能，接下来我们看一下几种造字典的方式：
```python
>>> d1 = {'a':1, 'b':2, 'c':3}
>>> print(d1, type(d1))
{'a': 1, 'b': 2, 'c': 3} <class 'dict'>

>>> dict({'a':1, 'b':2, 'c':3})
{'a': 1, 'b': 2, 'c': 3}

>>> dict(a=1, b=2, c=3)
{'a': 1, 'b': 2, 'c': 3}

>>> l = ['a', 'b', 'c']
>>> d2 = {}.fromkeys(l, None)
>>> d2
{'a': None, 'b': None, 'c': None}
```
我们可以通过以上四种方式产生一个新的字典。1、直接使用大括号  2、dict()功能传入大括号格式的数据 3、dict()功能传入关键字参数 4、使用字典的formkeys()功能。接下来我们再看下字典的类型转换。

dict()可以把可迭代对象转换为字典类型，前提是这个可迭代对象的每个元素都是存了两个值才可以。
```python
>>> l = [('name','wfj'), ['age', 18]]
>>> dict(l)
{'name': 'wfj', 'age': 18}
```

## 二、字典的基本操作
和列表、字符串一样，python也为字典内置了多种操作方法。

### 2.1、按key取值
字段按照key进行取值。
```python
>>> d = {'a':1, 'b':2, 'c':3}
>>> d['c']
3
>>> d['d']
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'd'
>>> print(d.get('d'))
None
```
可以看出，字典使用[]取值，取不存在的key就会报错。但是通过get()方法取值，不存在会返回None而不会报错。

### 2.2、字典增加和修改值
在列表中，对不存在的索引改值会报错，因此列表提供了insert()和append()功能。但是字典不会报错：
```python
>>> d = {'a':1, 'b':2, 'c':3}
>>> d['d'] = 4
>>> d
{'a': 1, 'b': 2, 'c': 3, 'd': 4}
```
字典是无序的，不存在把值插入到某一位置的说法，因此字典也就没有insert()方法。

### 2.3、字典删除值
字典用多种方法删除值，分别是 1、del 2、pop() 3、popitem() 4、clear()。我们分别来演示下这四种方法：
```python
>>> d = {'a':1, 'b':2, 'c':3, 'd':4, 'e': '5'}
>>> del d['a']
>>> d
{'b': 2, 'c': 3, 'd': 4, 'e': '5'}
>>> print(d.pop('b'))
2
>>> d
{'c': 3, 'd': 4, 'e': '5'}
>>> print(d.popitem())
('e', '5')
>>> d
{'c': 3, 'd': 4}
>>> d.clear()
>>> d
{}
``` 
del还是解除引用，这不是字典本身的方法，一样能达到删除值的效果。pop()传入要删除的key, 返回的是这个key的values。pipitem()是删除并返回最后一组的key和value，返回的是一个元组。clear()是清空字典。

### 2.4、获取字典长度
字典的长度获取一样是使用len()函数，可以说是统计key的个数，也可以说统计key value键值对的个数。
```python
>>> d = {'a':1, 'b':2, 'c':3, 'd':4, 'e': '5'}
>>> len(d)
5
```

### 2.5、成员运算
成员运算in 和 not in是判断的字典key，而不是value。
```python
>>> d = {'a':1, 'b':2, 'c':3, 'd':4, 'e': '5'}
>>> 'a' in d
True
>>> 1 in d
False
```

## 三、字典的内置方法
python为字典内置了多中方法以便对字典进行操作使用。

### 3.1、keys、values、items
这三种方法在python2版本和python3版本的结果不一样，我们先来看python2:
```python
>>> d = {'a':1, 'b':2, 'c':3}
>>> d.keys()
['a', 'c', 'b']
>>> d.values()
[1, 3, 2]
>>> d.items()
[('a', 1), ('c', 3), ('b', 2)]
```
在python2中，这三个方法得到的是列表，item()返回的列表嵌套键值对元组。但是在python3中，这三个方法返回的是迭代器：
```python
>>> d = {'a':1, 'b':2, 'c':3}
>>> d.keys()
dict_keys(['a', 'b', 'c'])
>>> d.values()
dict_values([1, 2, 3])
>>> d.items()
dict_items([('a', 1), ('b', 2), ('c', 3)])
```
这时候我们就可以使用循环来操作迭代器了。我们知道for循环默认遍历的是字典的key。 那如果想要遍历字典的值就可以用value()方法了。
```python
for i d.values():
    print(i)

for key, value in  d.items():
    print(key, value)
```
这里的运行结果可以自行测试，需要说明一点的是，在变量items()时用两个变量分别接收了元组的两个值，这种操作叫做解压赋值。

### 3.2、更新字典
字典的更新通过update()函数实现。我们先看一个例子:
```python
>>> d = {'a':1, 'b':2, 'c':3}
>>> new = {'c':7, 'd':8, 'e':9}
>>> d.update(new)
>>> d
{'a': 1, 'b': 2, 'c': 7, 'd': 8, 'e': 9}
```
可以看到对原字典进行更新，原字典已经存在的key值被更新，原字典不存在的key会被直接加入。

### 3.2、setdefault
这个方法从字面意思是设置默认值，我们先看效果：
```python
>>> d = {'name':'wfj', 'age':18}
>>> print(d.setdefault('age'))
18
>>> print(d.setdefault('gender'))
None
>>> d
{'name': 'wfj', 'age': 18, 'gender': None}
>>> d = {'name':'wfj', 'age':18}
>>> print(d.setdefault('gender', 'secret'))
secret
>>> d 
{'name': 'wfj', 'age': 18, 'gender': 'secret'}
```
可以看到，如果已经存在的key，这个方法会返回默认值，如果不存在会返回None。

### 3.3、拷贝函数
拷贝函数copy()，它属于浅拷贝，前面已经讲过，此处不再进行演示。
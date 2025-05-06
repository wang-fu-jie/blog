---
title:       Python基础 - Python字符串类型操作详解
subtitle:    "Python字符串类型操作详解"
description: "字符串是通过str()实现的，它内置了诸多方法，包括索引取值、切片、strip去空格、split拆分、字符串for循环、获取长度len、成员运算等等"
excerpt:     "字符串是通过str()实现的，它内置了诸多方法，包括索引取值、切片、strip去空格、split拆分、字符串for循环、获取长度len、成员运算等等"
date:        2025-02-17T14:01:16+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/21/a2/beach_wafe_ocean_water_wave-127492.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-varchar-detail"
categories:  [ "PYTHON" ]
---

## 一、字符串类型的生成和类型转换
前面我们知道了定义数字型其实是调用了int()和float()这两个功能，通用在定义字符串时是触发了str()这个功能。str()可以把任意类型转换为字符串。
```python
>>> a = str(['1', 'a'])
>>> print(a, type(a))
['1', 'a'] <class 'str'>
```

## 二、字符串的内置方法
Python字符串内置多种方法，用来对字符串的操作。

### 2.1、索引取值
虽然字符串是一个值，但它是由多个字符按顺序排练组成的。因此python也为它设置了索引的概念，所以字符串也可以按索引取值。
```python
>>> info = "good good study, day day up!"
>>> print(info[0])
g
>>> print(info[-1])
!
>>> info[0] = 'G'
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'str' object does not support item assignment
```
从示例我们可以看出，字符串支持通过索引取值，支持正向索引和反向索引。但是不支持通过索引修改，因此字符串是不可变类型。

### 2.2、切片
切片是索引的拓展应用，索引是取字符串的某一个字符，切片是根据索引取字符串中的某一段字符。即可以理解为切片是将原字符串中通过索引指定一个子串复制出来。原字符串并不发生改变。
```python
# 语法格式
>>> print(info[::-1])
!pu yad yad ,yduts doog doog
```
我们先来看几个代码示例：
```python
>>> info = "good good study, day day up!"
>>> print(info[0:4])
good
>>> print(info)
good good study, day day up!
>>> print(info[0:])
good good study, day day up!
>>> print(info[:4])
good
>>> print(info[:])
good good study, day day up!
>>> print(info[0:4:2])
go
>>> print(info[4:0:-1])
 doo
>>> print(info[::-1])
!pu yad yad ,yduts doog doog
```
通过示例我们做一个总结，步长step默认为1，begin不写默认从0开始，end不写默认到末尾结束。如果步长为正数，则end必须大于begin。步长如果为负数，则end必须小于begin。整体遵循左闭右开原则。

### 2.3、strip去空格
去空格是字符串专有的功能，通过strip()函数实现。
```python
>>> name = "   wf j   "
>>> print(name.strip())
wf j
>>> myname = "!@wfj@#"
>>> print(myname.strip("!@#"))
wfj
>>> print(myname.lstrip("!@#"))
wfj@#
>>> print(myname.rstrip("!@#"))
!@wfj
```
通过示例，我们可以得到结论，strip()函数默认去除字符串两边的空格。如果strip()传入参数，则去除字符串两边指定的参数。lstrip()的功能是只去除左边的空格或指定字符，rstrip()指的是只去除右边的空格或指定字符。

### 2.4、split拆分
拆分是把一个字符串按照某种分割符拆分，得到一个列表。主要用于拆分有规律的字符串，通过split()函数实现。
```python
>>> names = "小明 小红 小刚 小燕"
>>> res = names.split()
>>> print(res, type(res))
['小明', '小红', '小刚', '小燕'] <class 'list'>
>>> names = "小明-小红-小刚-小燕"
>>> names.split('-')
['小明', '小红', '小刚', '小燕']
>>> names.split('-', 2)
['小明', '小红', '小刚-小燕']
```
如示例所示，split()函数默认按照空格进行拆分，也可以按照指定字符进行拆分。也可以指定拆分次数。

### 2.5、字符串for循环
字符串也属于可迭代对象，是支持for循环，在前边文章讲述for循环时有说过，这里再做一下代码演示：
```python
s = "abc"
for a in s:
    print(a)
```
这三行代码分别会打印a b c。

### 2.6、获取长度len
python提供了内置函数len()可以用来统计字符串的长度，即计算一个字符串包含的字符个数。
```python
>>> info = "My name is wfj!"
>>> print(len(info))
15
```

### 2.7、成员运算
成员运算主要用来统计一个字符串是否属于另一个字符串的子串，用关键字in 和 not in实现。
```python
>>> "wfj" in "My name is wfj!"
True
>>> "wfj" not in "My name is wfj!"
False
```

### 2.8、其他常用操作汇总
1、去空格
```python
>>> name = "  wfj  "
>>> print(name.strip())  # 去除两边空格，也可以去除指定字符
wfj
>>> print(name.lstrip()) # 去除左边空格，也可以去除指定字符
wfj  
>>> print(name.rstrip()) # 去除右边空格，**也可以去除指定字符**
  wfj
```

2、拆分
```python
>>> names = "小明-小红-小刚-小燕"
>>> print(names.split('-'))    # 不加拆分次数的没有区别
['小明', '小红', '小刚', '小燕']
>>> print(names.rsplit('-'))
['小明', '小红', '小刚', '小燕']
>>> print(names.split('-', 1))  # 从左往右拆分
['小明', '小红-小刚-小燕']
>>> print(names.rsplit('-', 1))  # 从右往左拆分
['小明-小红-小刚', '小燕']
```

3、格式化format
```python
# 在输入输出章节我们讲过了format格式化，这是把示例贴过来复习一下，不再具体讲述
>>> info = 'my name is {}, I come from {}'.format('wfj', 'tj')
>>> print(info)
my name is wfj, I come from tj
>>> info = 'my name is {0}{0}, I come from {1}{1}{1}'.format('wfj', 'tj')
>>> print(info)
my name is wfjwfj, I come from tjtjtj
>>> info = 'my name is {name}, I come from {hometown}{hometown}'.format(name='wfj', hometown='tj')
>>> print(info)
my name is wfj, I come from tjt
```

4、大小写转换
```python
>>> info = "AbCd"
>>> print(info.lower())   # 转换为小写
abcd
>>> print(info.upper())   # 转换为大写
ABCD
```

5、判断开头结尾
```python
>>> "my name is wfj".startswith('my')   # 判断以...开头
True
>>> "my name is wfj".endswith('wfj')    # 判断以...结尾
True
>>> 
```

6、字符串拼接join
```python
>>> l = ['小明', '小红', '小刚', '小燕']
>>> '-'.join(l)
'小明-小红-小刚-小燕'
```
join和功能和split正好相反，这里需要注意的是，被拼接的列表里的元素必须是字符串。

7、字符串替换replace
```python
>>> names = "小明-小红-小刚-小燕"
>>> names.replace('-', ':')       # 把 - 替换为 :
'小明:小红:小刚:小燕'
>>> names.replace('-', ':', 1)    # 把 - 替换为 :， 只替换一个
'小明:小红-小刚-小燕'
```

7、判断字符串是否由于纯数字组成
```
>>> "18".isdigit()
True
>>> "18.2".isdigit()
False
```
注意，判断是否由纯数字组成，不是判断是不是数字类型。因此带小数点也返回False。

## 三、其他操作
除了以上比较常用的操作之外，python字符串还有很多的其他功能，只是日常用到的比较少。这里可做了解：
1. capitalize()   将字符串的第一个字符转换为大写。
2. center(width, fillchar)   返回一个指定宽度 width 居中的字符串，fillchar 为填充字符（默认空格）。
3. count(str, beg=0, end=len(string)) 返回 str 在字符串中出现的次数，可指定范围 beg 和 end。
4. expandtabs(tabsize=8)   将字符串中的 tab 符号转为空格，默认空格数为 8。
5. find(str, beg=0, end=len(string))   检测 str 是否包含在字符串中，可指定范围查找。
6. rfind(str, beg=0, end=len(string))  类似 find()，但从右侧开始查找。
7. index(str, beg=0, end=len(string)   与 find() 相同，但若未找到会抛出异常。
8. rindex(str, beg=0, end=len(string))   类似 index()，但从右侧开始查找。
9. isalnum()   若字符串至少有一个字符且全为字母或数字，返回 True。
10. isalpha()  若字符串至少有一个字符且全为字母或中文字，返回 True。
11. islower()  若字符串中至少有一个区分大小写的字符且全为小写，返回 True。
12. isupper()  若字符串中至少有一个区分大小写的字符且全为大写，返回 True。
13. isspace()  若字符串中只包含空白字符（空格、换行、制表符等），返回 True。
14. ljust(width, fillchar)  返回左对齐的字符串，并用 fillchar 填充至长度 width（默认空格）。
15. rjust(width, fillchar)  返回右对齐的字符串，并用 fillchar 填充至长度 width（默认空格）。
16. max(str) 返回字符串中最大的字母（按ASCII值比较）。
17. min(str) 返回字符串中最小的字母（按ASCII值比较）。
18. swapcase()  将字符串中的大写转小写，小写转大写。
19. title()  返回标题化字符串，每个单词首字母大写，其余小写。
10. istitle()  若字符串是标题化格式，返回 True。

---
title:       Python基础 - 序列化与反序列化
subtitle:    "序列化与反序列化"
description: "序列化是将内存中对象转换为字符串过程，反序列化则相反，通过序列化可以存储对象或跨平台交互。python提供了两个序列化的模块json和pickle，json是通过的数据格式，任何编程语言都支持，pickle是python特有的格式。另外本文额外介绍了给项目打补丁的思想之猴子补丁。"
excerpt:     "序列化是将内存中对象转换为字符串过程，反序列化则相反，通过序列化可以存储对象或跨平台交互。python提供了两个序列化的模块json和pickle，json是通过的数据格式，任何编程语言都支持，pickle是python特有的格式。另外本文额外介绍了给项目打补丁的思想之猴子补丁。"
date:        2025-04-28T09:31:01+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/07/c1/portrait_person_woman_girl_greenhouse-20956.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-serialization"
categories:  [ "PYTHON" ]
---

## 一、序列化与反序列化介绍
序列化（Serialization）就是把对象（比如 Python 的列表、字典、类实例）转成特定的字节流或者字符串的过程。这种格式可以用来存储或者发送给其他平台使用。反序列化就是把字节流或者字符串还原成对象的过程。

序列化的第一个用途就是把内存里的数据保存到硬盘，例如我们在玩游戏时的存档功能。第二个用途是跨语言通信，例如前端产生的数据发送到后端进行处理，就需要用json数据格式进行序列化。json是一种通用的数据

python中有两种格式可以用来进行序列化，一种是json，另一种是pickle。json是一种通用的数据格式，各种编程语言都支持python，因此可以用来跨语言通信。但是对于python它的缺点是并不能序列化独有的类型例如集合。pickle属于python独有的序列化格式，它能序列化所有python类型，但是不能跨语言通信。

## 二、json模块
首先我们看一下利用json模块进行序列化的过程。
```python
import json
d = {'a': 1, 'b': 2, 'c': [1, 2, 3], 'd': '你好'}
res = json.dumps(d)
print(res, type(res))

## 运行结果
{"a": 1, "b": 2, "c": [1, 2, 3], "d": "\u4f60\u597d"} <class 'str'>
```
从示例可以看出，在通过json进行序列化后，所有的单引号都被改变成了双引号，因为双引号字符串是所有编程语言的通用格式。另外中文编程了Unicode编码，这是因为默认序列化后的字符用ASCII表示。可以给json.dumps()函数加个参数ensure_ascii=False就可以显示中文了。最后可以看到序列化后的类型是字符串。

这里可以把序列化后的字符串写到本地文件或者在网络上进行传输，序列化后重新取数据就需要用到反序列化，反序列化使用json.loads()函数。
```python
import json
s = '{"a": 1, "b": 2, "c": [1, 2, 3], "d": "\u4f60\u597d"}'
res = json.loads(s)
print(res, type(res)) 

## 执行结果
{'a': 1, 'b': 2, 'c': [1, 2, 3], 'd': '你好'} <class 'dict'>
```
这里我们又把刚才的字符串重新恢复成了字典，但是实际编程一般不会这个干，都是从文本读取或者接收网络传过来的数据，示例中直接写字符串是为了方便演示。

json模块还提供了dump和load函数，它进一步简化的序列化和反序列化的流程，将序列化和读写文件两步骤通过一行代码实现。
```python
# 序列化
with open('data/test1.json', mode='wt', encoding='utf-8') as f:
    json.dump(dic, f, ensure_ascii=False)


# 反序列化
with open('data/test1.json', mode='rt', encoding='utf-8') as f:
    json_res = json.load(f)
    print(json_res)
```
在python3.6以后，在反序列化过程中，可以传入bytes类型，也就是可以通过rb模式读取文件，一样可以正常进行反序列化。最后我们再看一下json类型的限制，那就是不能序列化python特有的类型。
```python
import json
s = {1, 2, 3, 4}
res = json.dumps(s)
print(res, type(res))

## 执行结果
TypeError: Object of type set is not JSON serializable
```

## 三、pickle模块
pickle是python特有的序列化功能，它和json模块一样，也有dump、 dumps、load、loads四个函数。
```python
import pickle
d = {'a': 1, 'b': 2, 'c': [1, 2, 3], 'd': '你好', 'f': {7, 8, 9}}
res = pickle.dumps(d)
print(res, type(res))

## 执行结果
b'\x80\x03}q\x00(X\x01\x00\x00\x00aq\x01K\x01X\x01\x00\x00\x00bq\x02K\x02X\x01\x00\x00\x00cq\x03]q\x04(K\x01K\x02K\x03eX\x01\x00\x00\x00dq\x05X\x06\x00\x00\x00\xe4\xbd\xa0\xe5\xa5\xbdq\x06X\x01\x00\x00\x00fq\x07cbuiltins\nset\nq\x08]q\t(K\x08K\tK\x07e\x85q\nRq\x0bu.' <class 'bytes'>
```
可以看到pickle序列化后是字节类型，因此写入文件需要用b模式。写入文件后打开是乱码的，正常显示文本文件需要给pickle.dump()函数添加protocol=0参数。读者可以自行测试。


## 四、猴子补丁
猴子补丁本身和序列化关系不大，之所以放在这里是因为它的内容比较少，没必要单独开一篇文章。猴子补丁是一种编程思想，用来给代码打补丁的。猴子补丁的核心思想是用你自己的代码替换模块的源代码。比如我们写了一个项目，项目中很多文件都使用到了json模块，但是你觉得官方的json不能满足你的需求，自己实现了一个。就需要通过打补丁的方式来应用自己的模块。

这时就需要在你的执行文件，例如run.py导入自己写的json模块，这样其他文件再导入时会优先从内存来查找json，找到的就是自定的json模块了。修改只需要修改执行文件即可，这就是猴子补丁的核心思想。示例：
```python
# 猴子补丁：用自己的代码替换你所用模块的源代码
import json
import ujson

def monkey_patch_json():
    json.__name__ = 'ujson'
    json.dumps = ujson.dumps
    json.dump = ujson.dump

monkey_patch_json()
```
如上所示，就利用猴子补丁将json下的函数替换成了ujson下的函数。
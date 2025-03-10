---
title:       Python基础 - 操作文件(二)
subtitle:    "操作文件(二)"
description: ""
excerpt:     ""
date:        2025-03-10T09:52:58+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/cd/3c/forest_landscape_autumn_trees_germany_nature_mood_leaves-1370886.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-file-advanced"
categories:  [ "PYTHON" ]
---

## 一、文件的操作函数
上篇文章中介绍了操作文件的三大基本流程，打开文件、读写文件、关闭文件。Python还提供了更多的操作文件的方式，例如读取一行，写入一行等，本文降对这些函数进行介绍。

### 1.1、read函数
前边我们使用了read函数，用来读取文件，它默认从当文件指针位置读取到文件末尾。这样存在一个问题，如果文件非常大，一次性读取内存，可能会把内存撑爆。因此read还支持传入参数，指定读取的字节数。
```python
with open('a.txt', mode='rt') as f:
    print(f.read(1))
    print(f.read(1))
## 执行结果
你
好

with open('a.txt', mode='rb') as f:
    print(f.read(1))
## 执行结果
b'\xe4'
```
从示例我们可以看出，在读取文本文件时，read指定的参数是指定读取字符数，在读取二进制文件时是指定读取字节数。在一次读取后，文件指针根据参数向后移动。

### 1.2、readline与readlines
readline实现的功能是一次读取一行，readlines是一次读取多行。
```python
with open('a.txt', mode='rt') as f:
    print(f.readline())
    print(f.readline())
## 执行结果
你好

我也好

with open('a.txt', mode='rt') as f:
    print(f.readlines())
## 执行结果
['你好\n', '我也好']
```
可以看到，readlines是一次读取文件索引内容，按照每行为元素返回一个列表。

### 1.2.3、writeline函数
writeline的功能并不是写入多行，它是将一个可迭代对象写入文件，可以理解为就是做了一个for循环。
```python
with open('a.txt', mode='wt') as f:
    f.writelines(['abc', '123'])

## 执行结果
文件内容: abc123
```
如果把示例中的'123'改为数字123，它执行就会报错，这里可以自行测试。


## 二、bytes类型
bytes是二进制类型，它也属于不可变类型。如果针对英文和数字的bytes类型，直接用b定义即可，因为所有编码都支持英文和数字。但是中文用b定义就会报错。
```python
>>> a = b'123abc'
>>> print(a)
b'123abc'
>>> 
>>> b = b'你好'
  File "<stdin>", line 1
    b = b'你好'
             ^
SyntaxError: bytes can only contain ASCII literal characters.
```
通过报错信息可以得到，bytes类型只能包含ASCII码表的文字字符。针对不在ASCII码表中的文字字符，需要使用encode编码为二进制格式。
```python
>>> b = '你好'.encode('utf-8')
>>> b = bytes('你好', encoding='utf-8')
>>> print(b)
b'\xe4\xbd\xa0\xe5\xa5\xbd'
```
python在打印时做了优化处理，不在ASCII中的字符用十六进制打印，在ASCII中的字符直接打印输出，只在在前边加了b''，这样增加了输出的可读性。

bytes类型支持索引，索引返回的是int，而不是bytes子对象。还支持切片，也包含其他功能如拼接，查找。这里不再对这些功能进行一一演示。读者可自行进行测试。

## 三、flush功能
操作系统有一个写入优化机制，叫做写入缓存。即当应用程序请求写入数据时，操作系统不会立即将数据写入物理磁盘，而是：将数据写入内存缓存（Page Cache），并立即返回给应用程序（看起来写入很快）。在合适的时机（如缓存满或定期刷新）再将数据批量写入磁盘，以减少磁盘 I/O 开销。但是这个优化的缺点是，如果系统崩溃或断电，这些数据就会丢失。

当然如果不重要的数据，如日志可以容忍数据丢失则适用这种场景，如果很重要的数据要求必须落盘，python也提供了flush功能强制刷数据入盘。
```python
with open('a.txt', mode='wt') as f:
    f.write('abc')
    f.flush()
```

## 四、文件的其他功能
文件还存在一些其他功能，这里可做了解。
* f.readable()  # 判断文件是否可读
* f.writeable() # 判断文件是否可写
* f.closed      # 判断文件是否关闭
* f.encoding    # 获取文件当前的编码方式，如果以b模式打开文件，则没有改属性
* f.name        # 获取文件名

## 五、文件指针移动
文件指针是下一个要操作字节的位置。可以控制文件指针进行移动，不过这里的移动都是字节。只有read()函数在t模式下参数是字符。python提供了控制指针移动的函数是seek，它有两个参数，第一个是移动的字节数量，第二个是参数位置。
```python
f.seek(n, position)
# position = 0  是参照文件开头
# position = 1  是参照文件当前位置
# position = 2  是参照文件末尾
# n为正数就是往后移，如果n为负数就是往前移动
```
这里需要需要的是，当参照位置为1和2时，不能在t模式下使用。虽然参照位置为0时可以在t模式使用，但是一般不用，比如一个中文在utf-8编码下占3字节，如果移动了一个字节，会导致出现编码错误问题。我们来演示一下b模式下的指针移动。
```python
with open('a.txt', mode='rb') as f:
    f.seek(4, 0)
    print(f.tell())

## 执行结果
4
```
这里tell()的功能是获取当前指针的位置。前面提到t模式下不能使用参照位置1和2，会报错 不能进行非零端相对寻道。但是如果参照位置为1和2时，移动0字节是不会报错的。因此移动到文件末尾可以使用f.seek(0, 2)

## 六、文件修改
如果需求在文件中间新增内容。不能使用a模式，因为a模式是追加模式，即使移动指针到指定位置，在执行写入时也会写到文件末尾。 也不能使用r+模式，r+模式虽然能在指定位置写入，但是会覆盖该位置原来的字符。因此是不能直接该硬盘文件的，如果改数据，就需要把文件全部读入内存，修改后再覆盖回磁盘。一般的文本编辑器就是这么修改文件的。

还有第二种修改文件的方式，那就是打开源文件，逐行读入内存进行修改，修改后保存到一个临时文件中，修改完成后进行文件名字的交换。这也是linux系统用来修改文件的方式。
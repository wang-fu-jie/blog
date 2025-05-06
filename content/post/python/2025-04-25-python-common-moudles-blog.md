---
title:        Python基础 - 常用的内置模块
subtitle:    "常用的内置模块之random、os、sys、shutil模块等"
description: "python自己内置了许多模块，本文介绍几个比较常用的内置模块。其中random模块用来生产随机数，os模块用来和操作系统交互，sys模块用来和python解释器交互，shutil模块是对文件的高级操作适合批量任务。还有其他的如configparse、subprocess、hash模块等。"
excerpt:     "python自己内置了许多模块，本文介绍几个比较常用的内置模块。其中random模块用来生产随机数，os模块用来和操作系统交互，sys模块用来和python解释器交互，shutil模块是对文件的高级操作适合批量任务。还有其他的如configparse、subprocess、hashlib模块等。"
date:        2025-04-25T13:59:16+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/c6/df/fox_wild_nature_water_mirror_nature_photography_wildlife-564985.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-common-moudles-blog"
categories:  [ "PYTHON" ]
---

## 一、random模块
random模块主要用来生产随机数。可以生成随机整数，随机小数等。我们直接来看代码：
```python
>>> import random
>>> random.random()     # 生成一个随小数，范围 0 ~ 1
0.9221925269550556
>>> 
>>> random.uniform(1, 10)   # 生成指定范围内的随机小数
2.3963348248178153
>>> random.randint(-10, 10)  # 生成指定范围的随机整数，范围属于闭区间
2
>>> random.randrange(1, 3)   # 生成指定范围的随机整数，范围区间是左闭右开 
1
>>> random.choice(['a', 'b', 'c'])   # 从一个序列随机取一个值
'a'
>>> random.sample(['a', 'b', 'c'], 2)   # 从一个序列随机取多个值
['c', 'a']
>>> l = ['a', 'b', 'c']       # 打乱序列的顺序，必须是可变类型。
>>> random.shuffle(l)
>>> l
['c', 'a', 'b']
```
随机数其实是一种伪随机，就是说，它们看起来是随机的，但其实是根据一个初始值（种子 seed）经过某种算法计算出来的。这个种子决定了随机数序列的“起点”。如果你设置相同的种子，后续产生的随机数序列就会完全相同。
```python
>>> random.seed(42)  # 设置种子为 42
>>> print(random.randint(1, 100))  # 每次运行都会输出相同的值，例如 82
82
>>
```
随机数种子在调试和重现实验结果非常有用。默认的随机数种子是当前的系统时间，所以我们默认产生是随机数，因为每次运行的时间肯定是有差异的。

## 二、os模块
os模块是 operating system的缩写，所以os模块提供的功能是python程序与操作系统交互的接口。可以利用os模块操作目录、文件等
```python
os.getcwd()
# 获取当前工作目录
os.chdir("dirname") # 改变当前脚本工作目录；相当于终端里面的cd命令
os.listdir('dirname') # 获取指定目录下的所有文件和文件夹，包括隐藏文件，并返回列表
os.mkdir('dirname') # 创建文件夹；相当于终端里面的mkdir dirname
os.makedirs('dirname1/dirname2') # 递归创建多层目录
os.rmdir('dirname') # 删除单级空目录，若目录不为空则无法删除,则报错
os.rename('oldname','newname') # 重命名文件/目录
os.system("rm -rf /") # 运行终端命令os.environ
os.remove()  # 删除一个文件# 获取系统环境变量
os.environ.get('KEY') # 获取系统环境变量的某一个值
os.getenv('KEY') # 获取系统环境变量的某一个值
os.stat('path/filename')# 获取文件/目录信息
os.name # 输出字符串指示当前使用平台。win->'nt'; Linux->'posix'
os.path.split(path) # 将path分割成目录和文件名,返回元组
os.path.dirname(path) # 返回path的父级目录。其实就是os.path.split(path)的第一个元素os.path.basename(path) # 返回path最后的文件名。如path以／或\结尾，那么就会返回空值。即os.path.split(path)的第二个元素
os.path.exists(path) # 如果path存在，返回True；如果path不存在，返回False
os.path.isabs(path) # 如果path是绝对路径，返回True
os.path.isfile(path) # 如果path是一个存在的文件，返回True。否则返回Falseos.path.isdir(path) # 如果path是一个存在的目录，则返回True。否则返回False
os.path.join(path1[, path2[, ...]]) # 将多个路径组合后返回，第一个绝对路径之前的参数将被忽略
os.path.getatime(path) # 返回path所指向的文件或者目录的最后读取时间os.path.getmtime(path) # 返回path所指向的文件或者目录的最后修改时间
os.path.getctime(path) # # 返回path所指向的文件或者目录的创建时间(windows平台中)
os.path.getsize(path) # 返回path的大小
```

## 三、sys模块
sys 模块是一个与 Python 解释器交互 的核心模块，提供了对解释器变量和函数的访问。前边我们使用过了sys.path获取 Python 的 模块搜索路径。
```python
# 获取运行 Python 脚本时传递的 命令行参数（返回 list）：
import sys
# 假设运行：python script.py arg1 arg2
print(sys.argv)        # ['script.py', 'arg1', 'arg2']
print(sys.argv[1])     # 'arg1'（第一个参数）

# 标准输入/输出 (sys.stdin, sys.stdout, sys.stderr)
sys.stdout.write("Hello\n")  # 等同于 print("Hello")
error_msg = sys.stderr.write("Error!\n")  # 输出到错误流
```
sys还有一些其他功能，这里就不再一一列举了，后续用的时候可以再查资料。

## 四、shutil模块
shutil 模块是 Python 标准库中用于 高级文件操作 的模块，主要提供对文件和目录的 复制、移动、删除、压缩 等操作。它比 os 模块更高级，适合处理批量文件任务。
```python
import shutil

# 拷贝文件状态信息 如：atime mtime ctime(只拷贝状态信息，b.txt必须存在)
shutil.copystat('data/a.txt', '基础/data/b.txt')

# 拷贝文件的权限
shutil.copymode('data/a.txt', '基础/data/b.txt')

# 拷贝文件
shutil.copyfileobj(open('data/a.txt', 'r'), open('基础/data/z.txt', 'w'))
shutil.copyfile('data/a.txt', '基础/data/z.txt')
shutil.copy('data/a.txt', '基础/data/z.txt')   # 拷贝文件以及权限
shutil.copy2('data/a.txt', '基础/data/z.txt')   # 拷贝文件以及状态信息


# 拷贝文件夹，ignore的意思是排除
shutil.copytree('data', 'data1', ignore=shutil.ignore_patterns('*.py', 'user.*'))

# 移动文件或文件夹，如果移动的路径相同名字不同，则相当于重命名
shutil.move('data', 'data2')


# 创建压缩包 (压缩包种类，zip、 tar、 bztar、 gztar)
shutil.make_archive("path", 'zip', root_dir='data')
```
此外还有用于压缩的模块zipfile和tarfile，此处不再逐个介绍，使用时再查资料即可。

## 五、configparser模块
configparser模块用来解析ini格式的配置文件，例如mysql的配置文件就是ini格式。例：
```ini
# 注释
; 也是注释

[default]
delay = 10
salary = 3.5
compression = true
compression_level = 9
language_code = en-us
time_zone = UTC

[db]
db_type = mysql
database_name = catalogue_data
user = root
password = root
host = 127.0.0.1
port = 3306
charset = utf8
```
如上这个配置文件。有两个部门称之为section，每部分里的变量称之为option，option可以使用等号也可以使用冒号。接下来我们看python如何获取到配置信息：
```python

import configparser
config = configparser.ConfigParser()
config.read('data/test.ini', encoding='utf-8')

print(config.sections())       # 获取所有section的名称
print(config.options('db'))    # 获取指定section的所有option的名称
print(config.items('db'))      # 获取指定section的所有option的名称和值
delay = config.getint('default', 'delay')    # 获取指定section下指定option的值
print(delay, type(delay))

salary = config.getfloat('default', 'salary')
print(salary)
```
configparser获取到的值默认都是字符串，如果有整型的配置那就需要获取到之后再单独做类型转换。当然这个模块也提供了获取指定数据类型，如getint，getfloat。

## 六、subprocess模块
subprocess 模块是 Python 中用于创建和管理子进程的标准库模块，它允许你启动新的应用程序、连接到它们的输入/输出/错误管道，并获取它们的返回码。subprocess模块常用来执行终端命令，和os.system一样。这个模块下的功能很多，但是常用的不多。
```python
import subprocess
obj = subprocess.Popen('ls /Users/wangfujie/Desktop',
                       shell=True,
                       stdout=subprocess.PIPE,
                       stderr=subprocess.PIPE)

res = obj.stdout.read()
print(res.decode('utf-8'))

err = obj.stderr.read()
print(err.decode('utf-8'))
```
这是 Python 3.5后，引入了最简单的run方法用来执行系统命令。
```python
result = subprocess.run(['ls', '-l'], capture_output=True, text=True)
print(result.stdout)
```
同样获取输出，就比Popen简单了很多，这里其他功能你可以自行去研究。

## 七、hashlib模块
hash是一类算法，常见的有md5,sha256等，它用来将一串输入转换成固定长度的值。常用来进行文件的完整性校验和密码加密。它的特点是对输入敏感、不可逆、计算快且输出长度固定。python内置了hashlib模块可以供我们使用hash算法。
```python
import hashlib

h1 = hashlib.md5()
h1.update('abc'.encode('utf-8'))   # 这里必须传入bytes类型
h1.update('123'.encode('utf-8'))
print(h1.hexdigest())              # 计算的是abc123的hash结果。
```
这里我们使用md5算法示例，通过updtae给md5对象传入字节类型，可以多次传值，最终按照多次传值的拼接进行计算hash值。虽然hash是不可逆的，但是仍然可以通过hash碰撞得到你的密码，于是就出现 了密码加盐操作。接下来我们看下如何对密码进行加盐。
```python
# 密码加盐
pwd = 'xyxxy520'
import hashlib
m = hashlib.md5()
m.update('天青色等烟雨'.encode('utf-8'))
m.update(pwd.encode('utf-8'))
m.update('而我在等你'.encode('utf-8'))
print(m.hexdigest)
```
通过示例可以看到，密码加盐就是在原有的密码上增加一些额外的字符，这样即便是简单的密码加盐后也会变的复杂，通过常用的密码库就不容易破解密码了，增加了攻击者的破解难度。
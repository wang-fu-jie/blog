---
title:       Python基础 - 模块介绍
subtitle:    "模块介绍"
description: ""
excerpt:     ""
date:        2025-04-16T13:41:17+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/7a/24/forest_sky_winter_landscape_r_gen_trees-766978.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-module"
categories:  [ "PYTHON" ]
---

## 一、模块介绍
python模块就是一系列功能的集合体。模块分为三类，分别为内置模块，第三方模块和自定义模块。内置模块大多是用c/c++编写的，内置到python解释器中，例如我们前面使用过的os、 time等都是内置模块。第三方模块可能是c或python编写，可以进行安装使用。自定义模块一般使用python编写，单独存放在一个文件中，那这个文件就变成了模块。例如：在我们需要重复调用一个功能，可以使用函数封装起来，那这个功能需要在多个文件中调用呢，那就需要使用模块了，把这写函数单独存放到一个文件。
```python
# moudle.py
print('moudle模块')
name = 'wfj'

def func1():
    print('这是函数func1')

def func2():
    print('这是函数func2')
```

## 二、模块导入
上面我们在一个文件moudle.py中创建了两个函数，现在在另一个文件中进行调用，需要先进行模块导入。python使用关键字import进行模块导入：
```python
# test.py
import moudle

## 执行结果
moudle模块
```
通过以上示例发现，在首次导入模块的时候，会执行模块的代码。这里注意是首次，如果在同一个文件中多次导入一个模块，也只会执行一次。模块导入支持多种导入方法，例如：
```python
import os, time  # 一行代码导入多个模块，虽然不支持，但是不推荐这么写
import moudle as md  # 导入模块起别名，这样就可以通过别名调用模块内的功能了，对与名称比较长的模块推荐使用
```
在导入模块中，一般先导入内置模块，其次导入第三方模块，最后导入自定义模块。在Python3中，自定义模块的命名规范是纯小写加下划线组成。另外导模块的语句也可以写在函数体内只是这样模块就是局部的了，一般不会这么使用。


## 三、模块的名称空间
在导入模块后，例如上文实例导入moudle模块，会立即产生名称空间，名称moudle指向这个名称空间，并且moudle.py运行产生的名字都保存在这个名称空间中。这时就可以通过moudlue这个名称空间调用moudle.py的功能。
```python
import moudle
print(moudle.name)
moudle.func1()

## 执行结果：
moudle模块
wfj
这是函数func1
```

## 四、python代码的运行方式和__name__
现在我们知道，python代码有两种运行方式，第一种是是直接运行，第二种是作为模块被导入。我们看一个例子：
```python
x = 10
y = 20
import module
```
运行这个代码，首先会产生一个当前文件的名称空间，存储x, y两个全局变量。moudle也是一个名字，在执行这行代码时，就执行moudle.py这个文件，于是会产生moudle自己的名称空间。 当直接运行代码时，代码运行结束名称空间就会被回收。但是代码被导入产生的名称空间是被moudle引用的，当引用结束后，如果模块的名称空间引用计数变为0，那就会被垃圾回收机制回收。

既然python运行有两种方式，那是否有办法判断是通过那种方式运行的呢，当然是有的，就是通过__name__属性。
```python
# moudle.py
print('moudle的__name__属性', __name__)
## 执行结果
moudle的__name__属性 __main__

# test.py
import moudle
## 执行结果
moudle的__name__属性 moudle
```
从示例可以看出，当直接执行python代码是，它的__name__属性就是__main__，当作为模块被调用时，它的__name__属性就是 moudle。这样我们就可以通过if来判断模块是被导入还是被直接执行了。这个功能经常用来调试，例如:
```python
def func2():
    print('这是函数func2')

if __name__ == '__main__':
    func2()
```
这样，我们就可以直接执行代码来验证func2函数的运行结果了，但是作为模块被导入是，func2又不会被执行。

## 五、from导入
前面我们使用import直接导入模块，调用功能时需要使用 模块名.功能 来调用。python还提供了from导入的方式，可以在调用的时候不用加模块名。
```python
from moudle import name, func1
print(name)
print(func1)

## 执行结果
wfj
<function func1 at 0x7ff743f12950>
```
如上所示，当通过from导入时，在当前名称空间产生的名字指向的是，模块名称空间对应功能的内存地址，因此是不用模块名作为前缀就可以直接使用。但是这样也存在一个缺点，就是在当前文件中如果存在同名变量，那模块中的名字就会被覆盖掉，即访问不到了。接下来再看一个示例：
```python
# moudle.py
name = 'wfj'

def get():
    print(name)

def set():
    global name
    name = 'xxxx'

# test.py
from moudle import name, set, get
set()
print(name)
get()

## 执行结果
wfj
xxxx
```
这里test.py文件中的name指向了moudle.py名称空间name的内存。在执行set后，moudle.py名称空间的name被修改了，但是test.py中还是之前原来字符串的内存地址，因此还是wfj。但是moudle.py名称空间中的name已经指向了xxxx。

from导入还有其他形式：
```python
from moudle import * # 导入所有名字
```
使用这种方式，需要谨慎使用，因为我们不清楚*会导入那些名字，很容易引发名称冲突。 另外说明一点，\*导入的是模块的__all__属性，__all__属性是一个列表，默认包含模块的所有名字。当然from导入的方式也可以起别名。
```python
from moudle import func1 as f1
```

## 六、循环导入
循环导入就是模块b导入了模块a，但是模块a中又导入了模块b。如：
```python
# a.py
from b import y
print(y)
x = 1

# b.py
from a import x
print(x)
y = 2

# main.py
import a

### 执行结果：
报错：ImportError: cannot import name 'x' from 'a' 
```
这是因为运行main，导入了a产生a的名称名称空间。去执行a.py，a产生了b的名称空间，又要去执行b.py。 这里b导入了a，但是a的名称空间已经存在了，但是执行a.py的时候x定义还没有执行到，因此就产生了报错。

解决循环导入有两种方式，第一是把名字提升到导入的前面，即把x和y的定义放到文件的最上边。第二种是那里使用那里再导入，即在函数内部导入。但这两种方式都是屎上雕花。

## 七、模块的查找顺序
模块本身也是一个python文件，当我们导入一个模块时，肯定需要找到这个文件。那查找顺序是什么呢？首先会在内存里找，如过这个模块被导入过在内存已经存在了，其次是在硬盘里找，在硬盘查找的顺序是sys模块的path属性决定的。
```python
>>> import sys
>>> sys.path
['', '/Library/Frameworks/Python.framework/Versions/3.9/lib/python39.zip', '/Library/Frameworks/Python.framework/Versions/3.9/lib/python3.9', '/Library/Frameworks/Python.framework/Versions/3.9/lib/python3.9/lib-dynload', '/Users/wangfujie/Library/Python/3.9/lib/python/site-packages', '/Library/Frameworks/Python.framework/Versions/3.9/lib/python3.9/site-packages']
```
第一个是执行文件所在的文件夹路径，即当前路径，后续都是python一些内置模块，第三方模块的存放路径。 这里需要注意的是，pycharm会增加一个当前项目的路径，这是pycharm自身的优化。
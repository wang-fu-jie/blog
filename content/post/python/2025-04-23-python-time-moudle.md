---
title:       Python基础 - 时间模块
subtitle:    "时间模块"
description: "Python提供了两个时间模块，分别是time模块和datetime模块。可用于对时间进行操作，时间有三种格式，分别为时间戳、格式化时间、结构化时间，可以通过时间模块对时间格式进行转换等。"
excerpt:     "Python提供了两个时间模块，分别是time模块和datetime模块。可用于对时间进行操作，时间有三种格式，分别为时间戳、格式化时间、结构化时间，可以通过时间模块对时间格式进行转换等。"
date:        2025-04-23T10:53:05+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/6d/22/man_soledad_only_landscape_summer_people_silhouettes_sun-830610.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-time-moudle"
categories:  [ "PYTHON" ]
---

## 一、时间模块
时间有三种格式，第一种是时间戳格式就是1970年到现在的秒数，第二种形式是格式化后的字符串形式，第三中形式是结构化时间，可以理解这是一个结构体，时间信息结构体的多个属性。python内置了两个时间模块，一个是time模块，一个是datetime模块。

## 二、time模块介绍
time模块属于python的内置模块，我们先看下利用time模块分别获取三种时间格式。
```python
>>> import time
>>> time.time()
1745377522.91184
>>> time.strftime('%Y-%m-%d %H:%M:%S')
'2025-04-23 11:07:18'
>>> time.localtime()
time.struct_time(tm_year=2025, tm_mon=4, tm_mday=23, tm_hour=11, tm_min=8, tm_sec=36, tm_wday=2, tm_yday=113, tm_isdst=0)
```
如上所示，时间戳格式进常用于进行时间统计，字符串格式用于获取当前时间，结构化格式可以用来单独获取时间的某个属性例如获取年份。虽然字符串可以利用%Y也可以单独获取年份，但是得到的是字符串，而结构化时间得到的是整型。

## 三、datetime模块介绍
在使用time模块时，获取字符串格式时间还需要些一堆字符串，非常的麻烦，于是出现了datetime模块。datetime模块就不需要我们再写格式化字符串了。
```python
>>> import datetime
>>> datetime.datetime.now()
datetime.datetime(2025, 4, 23, 11, 15, 40, 428121)
>>> print(datetime.datetime.now())
2025-04-23 11:16:01.354631
```
datetime模块比较方便的是可以直接进行时间的加减。
```python
>>> print(datetime.datetime.now() + datetime.timedelta(days=7))
2025-04-30 11:19:00.350554
```
如上，我们就得到了7天后的时间。获取7天前的时间把参数days改为-7即可，或者把加法改为减法。时间加减还可以进行月，周，秒等的计算，这里自行实验即可。

## 四、时间格式转换
时间格式转换指的是时间戳、字符串格式、结构化时间之间的相互转换。需要说明的是，时间戳和字符串格式的时间不能直接进行转换，需要使用结构化时间进行过渡。

### 4.1、时间戳->结构化时间->字符串格式时间
我们首先看一下时间戳如何转化为结构化时间。这里调用了time模块的localtime函数。
```python
>>> time.localtime()
time.struct_time(tm_year=2025, tm_mon=4, tm_mday=23, tm_hour=14, tm_min=0, tm_sec=46, tm_wday=2, tm_yday=113, tm_isdst=0)
>>> 
>>> time.localtime(111111)
time.struct_time(tm_year=1970, tm_mon=1, tm_mday=2, tm_hour=14, tm_min=51, tm_sec=51, tm_wday=4, tm_yday=2, tm_isdst=0)
>>> 
>>> time.gmtime(111111)
time.struct_time(tm_year=1970, tm_mon=1, tm_mday=2, tm_hour=6, tm_min=51, tm_sec=51, tm_wday=4, tm_yday=2, tm_isdst=0)
```
如上所示，time.localtime函数默认获取当前时区的结构化时间，我们也可以传参一个时间戳，就获取指定时间戳的结构化时间。另外time.gmtime函数获取的国际化标准时间，因为北京位于东八区，地球自西向东转，因此同一时刻北京时间比国际标准时间快了8小时。接下来我们再把结构化时间转换为字符串格式时间。
```python
>>> time.strftime('%Y-%m-%d %X')
'2025-04-23 14:10:38'
>>> 
>>> time.strftime('%Y-%m-%d %X', time.localtime(111111))
'1970-01-02 14:51:51'
```
如上，使用time.strftime函数即可将结构化时间转换为字符串格式时间，默认是当前时区的时间，也可以手动传入指定时间。

### 4.2、字符串格式时间->结构化时间->时间戳
我们先看如何把字符串时间转换为结构化时间，这里使用time.strptime函数实现。
```python
>>> t = '1995-09-14 21:30:00'
>>> time.strptime(t, '%Y-%m-%d %H:%M:%S')
time.struct_time(tm_year=1995, tm_mon=9, tm_mday=14, tm_hour=21, tm_min=30, tm_sec=0, tm_wday=3, tm_yday=257, tm_isdst=-1)
```
字符串转换为结构化时间的函数time.strptime需要传入两个参数，一个是字符串格式时间，一个是format也就是需要我们说明哪部门是年，哪部分是月。接下来我们利用time.mktime将结构化时间转换为时间戳
```python
>>> t = '1995-09-14 21:30:00'
>>> res = time.strptime(t, '%Y-%m-%d %H:%M:%S')
>>> time.mktime(res)
811085400.0
```

## 五、时间模块补充
时间模块还具备其他的函数，这里进行补充介绍几个，还有一些不常用的，可以自行查资料。
```python
time.sleep()  # 等待一段时间
datatime.datetime.utcnow()  # 获取世界标准时间
datetime.datetime.fromtimestamp(1111)  # 把时间戳转换为字符串格式，但是不能自定义格式。
```
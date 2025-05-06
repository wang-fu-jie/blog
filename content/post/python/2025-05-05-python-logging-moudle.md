---
title:       Python基础 - 日志模块
subtitle:    "日志模块"
description: "日志通常用于跟踪程序运行状态、调试问题以及记录重要事件。Python标准库中内置了logging模块用于记录日志，它提供了灵活的日志记录系统。本文将介绍Python日志模型的使用方法。"
excerpt:     "日志通常用于跟踪程序运行状态、调试问题以及记录重要事件。Python标准库中内置了logging模块用于记录日志，它提供了灵活的日志记录系统。文将介绍Python日志模型的使用方法。"
date:        2025-05-05T09:59:56+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/69/2a/street_road_night_shutter_speed_transport_highway_history_architecture-1071668.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-logging-moudle"
categories:  [ "PYTHON" ]
---

## 一、日志模块介绍
在日常编程中，通过日志模块可以帮助开发者跟踪程序运行状态、调试问题以及记录重要事件。可以根据程序打印的日志进行问题的排查跟踪。Python标准库中内置了logging模块用于记录日志，它提供了灵活的日志记录系统。日志分为五个等级，分别为debug、info、warning、error、critical。
```python
import logging
logging.debug('调试日志')
logging.info('消息日志')
logging.warning('警告日志')
logging.error('错误日志')
logging.critical('严重错误日志')

## 执行结果
WARNING:root:警告日志
ERROR:root:错误日志
CRITICAL:root:严重错误日志
```
这里没有打印debug和info日志，因为默认的日志级别为warning。root是日志的名字，日志默认打印到控制台，也可以配置输出到日志文件。

## 二、日志基本配置
在上文示例中，日志模块默认级别为warning且输出到控制台，这些都是可以利用日志的配置进行修改的。我们先看一个基本配置：
```python
# 日志基本配置
# 日志基本配置
logging.basicConfig(
    # 1、日志级别
    level=30,
    # DEBUG:10
    # INFO:20
    # WARNING:30
    # ERROR:40
    # CRITICAL:50
    
    # 2、日志输出格式
    # format='%(asctime)s %(name)s [%(pathname)s line:%(lineno)d] %(levelname)s %(message)s',
    
    # 3、asctime的时间格式
    # datefmt='%Y-%m-%d %H:%M:%S',
    
    # 4、日志输出位置：终端/文件
    # filename='user.log', # 不指定此配置，默认打印到终端
)
'''
%(name)s Logger的名字(getlogger时指定的名字)
%(levelno)s 数字形式的日志级别
%(levelname)s 文本形式的日志级别
%(pathname)s 调用日志输出日志的完整路径名
%(filename)s 调用日志输出日志的文件名
%(module)s 调用日志输出日志的模块名
%(funcName)s 调用日志输出日志的函数名
%(lineno)d 调用日志输出函数的语句所在的代码行
'''
```
如上所示，如果把level参数设置为10，那就是设置日志级别为debug。format用来控制日志输出格式，可以使用示例中的参数名获取相关的信息，例如%(asctime)s获取当前时间。filename指定日志输出的文件名，如果不指定默认输出到终端。基本配置仍然存在一些问题，例如无法指定编码，在windows默认编码是gbk，这样在输出日志到文件中使用pycharm打开就会乱码，因为pycharm默认是utf-8编码。

## 三、日志配置字典
日志的基本配置有些问题解决不了，例如指定日志文件的编码，同时输出日志到文件和终端等。这时候就需要日志字典了，如下是一个日志字典的配置：
<details>
 <summary style="cursor: pointer; color: #3b82f6; text-decoration: underline;">
    点击查看日志字典配置
</summary>

```python
# 日志配置字典
LOGGING_DIC = {
    'version': 1.0,
    'disable_existing_loggers': False,
    # 日志格式
    'formatters': {
        'standard': {
            'format': '%(asctime)s %(threadName)s:%(thread)d [%(name)s] %(levelname)s [%(pathname)s:%(lineno)d] %(message)s',
            'datefmt': '%Y-%m-%d %H:%M:%S',
        },
        'simple': {
            'format': '%(asctime)s [%(name)s] %(levelname)s  %(message)s',
            'datefmt': '%Y-%m-%d %H:%M:%S',
        },
        'test': {
            'format': '%(asctime)s %(message)s',
        },
    },
    'filters': {},
    # 日志处理器
    'handlers': {
        'console_debug_handler': {
            'level': 'WARNING',  # 日志处理的级别限制
            'class': 'logging.StreamHandler',  # 输出到终端
            'formatter': 'simple'  # 日志格式
        },
        'file_info_handler': {
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',  # 保存到文件,日志轮转
            'filename': 'abc.log',
            'maxBytes': 800,  # 日志大小 10M
            'backupCount': 3,  # 日志文件保存数量限制
            'encoding': 'utf-8',
            'formatter': 'standard',
        },
        'file_debug_handler': {
            'level': 'DEBUG',
            'class': 'logging.FileHandler',  # 保存到文件
            'filename': 'test.log',	 # 日志存放的路径
            'encoding': 'utf-8',	# 日志文件的编码
            'formatter': 'test',
        },
        'file_deal_handler': {
            'level': 'INFO',
            'class': 'logging.FileHandler',  # 保存到文件
            'filename': 'deal.log',  # 日志存放的路径
            'encoding': 'utf-8',  # 日志文件的编码
            'formatter': 'standard',
        },
        'file_operate_handler': {
            'level': 'INFO',
            'class': 'logging.FileHandler',  # 保存到文件
            'filename': 'operate.log',  # 日志存放的路径
            'encoding': 'utf-8',  # 日志文件的编码
            'formatter': 'standard',
        },
    },
    # 日志记录器
    'loggers': {
        'logger1': {  # 导入时logging.getLogger时使用的app_name
            'handlers': ['console_debug_handler'],  # 日志分配到哪个handlers中
            'level': 'DEBUG',  # 日志记录的级别限制
            'propagate': False,  # 默认为True，向上（更高级别的logger）传递，设置为False即可，否则会一份日志向上层层传递
        },
        'logger2': {
            'handlers': ['console_debug_handler', 'file_debug_handler'],
            'level': 'INFO',
            'propagate': False,
        },
        '': {      # 名字匹配不上就用没有名字的这个
            'handlers': ['console_debug_handler', 'file_info_handler'],
            'level': 'INFO',
            'propagate': False,
        },
        '用户操作': {
            'handlers': ['console_debug_handler', 'file_operate_handler'],
            'level': 'INFO',
            'propagate': False,
        },
    }
}
```
</details>

这个日志字典可以大体分为四部分，分别是formatters(日志格式)、filters(过滤器)、handlers(日志处理器)、loggers(日志记录器)。这里filters(过滤器)不用管，基本用不到。

### 3.1、日志格式
日志格式formatters可以定义多个，使用不同的名字，在调用时使用指定的名字即可。例如示例中 simple 格式，输出为 时间 日志名字 日志等级  日志信息，datefmt指定时间格式。当然还可以根据需要配置其他自定义的日志格式，通过内置的变量名即可。

### 3.2、日志处理器
日志处理器和日志记录器是相对的，日志处理器负责处理日志记录器产生的日志。日志是写入到文件还是输出到控制台都是由日志处理器控制的。可以设置多个日志处理器，例如handler1输出到控制到，handler2输出到日志文件。通过示例可以看到，我们可以通过日志处理器配置日志的编码格式、文件名、日志等级、日志文件大小、保留的日志数量等。

### 3.3、日志记录器
日志记录器是用来产生日志的，日志记录器一样可以有多个。日志记录器可以指定使用一个或多个日志处理器，可以配置日志级别。这里的日志级别和日志处理器的日志级别并不冲突，可以视为两次筛选，例如日志记录器是debug，那它会记录所有日志丢给日志处理器，但是日志处理器的级别为info，那日志处理器会再过滤掉debug级别的日志。

这里还有一点需要注意的是，我们有一个没有的名字的日志记录器，它的作用是当使用logging.getLogger获取日志记录器时，如果传入的名字不存在，那就会默认使用这个无名字的记录器。但是%(name)s Logger的名字记录的传入的名字。

### 3.4、使用日志字典
日志字典配置完成后，需要让logging模块加载这个字典。就需要用到logging模块的子包config。这里需要说明的是，logging的__init__.py文件中并没有导入config的名字，因此不能直接用。因此使用config模块需要使用
```python
from logging import config  # 用不了logging的其他功能
或者
import logging.config  # 可以使用logging的其他功能
```
这里具体的导入说明可以参考[包的介绍](/2025/04/18/python-package/)。接下来我们看如何加载并使用日志字典：
```python
import logging.config 
import settings
logging.config.dictConfig(settings.LOGGING_DIC)
log1 = logging.getLogger('logger1')
log1.info('xxx充值了5毛')
```
如上，我们把日志字典存放到配置文件settings.py中，调用logging.getLogger获取到日志记录器的名字，就可以用过日志记录器写入日志，日志记录器内部再把日志传给日志处理器。


## 四、日志配置文件
除了支持日志配置字典外，logging模块还支持ini格式的配置文件，他们的使用方式是类似的。日志配置文件不常用，可参考[日志配置文件]([/2025/04/18/python-package/](https://www.aliyundrive.com/s/XFB9gh3KApd/folder/647b1180de24778561a74d48b0ab56bb8c28668d))
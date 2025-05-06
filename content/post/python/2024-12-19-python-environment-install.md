---
title:       Python基础 - 安装Python
subtitle:    "安装Python环境与编写第一个程序"
description: "Python是当前世界上最热门的编程语言之一, 具有简单易用、应用领域广泛、强大的库支持、跨平台等诸多优点。本文我们从零开始，进行Python解释器的安装以及安装Python的集成开发工具 pycharm。以及编写我们的第一个Python程序"
excerpt:     "Python是当前世界上最热门的编程语言之一, 具有简单易用、应用领域广泛、强大的库支持、跨平台等诸多优点。本文我们从零开始，进行Python解释器的安装以及安装Python的集成开发工具 pycharm。以及编写我们的第一个Python程序"
date:        2024-12-19T14:25:59+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/a3/46/cross_sunset_humility_devotion_silhouette_human_kneeling_knee-607081.jpg!d"
published:   true
tags:
    - Python
    - Python基础
slug:        "python-environment-install"
categories:  [ "Python" ]
---

## 一、关于Python
Python 是一门高级编程语言，它是由Guido van Rossum在1989年圣诞节期间，为了打发无聊的圣诞节而编写的。目前Python在TIOBE榜单上已经排名第一位，成为世界上最受欢迎的编程语言之一。

Python语法简单，接近自然语言，容易理解。Python还为我们提供了非常完善的基础代码库，使用Python很多功能不需要从头编写。Python目前已经广泛应用于Web开发（使用框架如Django、Flask）、数据科学（如Pandas、NumPy）、人工智能（如TensorFlow、PyTorch）、自动化脚本、系统管理、游戏开发、科学计算等众多领域。

既然Python有这么多优点，那它是否有缺点呢？当然有，Python属于解释型语言，在运行的时候CPU需要逐行运行， 不像C语言先进行编译生成机器语言再执行。因此相比于传统的C、C++等语言运行效率降低。

## 二、安装Python
Python是跨平台语言，支持在windows、mac、linux等多个系统运行，即在windows上写的代码可以直接拿到linux执行。要使用Python，首先需要安装Python的运行环境。目前Python有两个大版本，Python2和Python3。这两个版本是不兼容的，当前Python3越来越普及，因此我们主要学习Python3。

### 2.1、在Windows上安装Python
首先根据你的Windows版本(32位还是64位，目前基本都是64位)到Python官网下载对应的[安装包程序](https://www.python.org/downloads/windows/)
![图片加载失败](/post_images/python/{{< filename >}}/2.1-01.png)
然后点击exe程序运行，点击下一步安装即可。注意在安装时候勾选上 Add Python 3.x to PATH。

### 2.2、在Mac上安装Python
Mac系统会自带Python环境，目前Mac自带的版本是Python2。 因此在Mac上使用Python3也是需要单独安装的，在Mac上安装有两种方法:

1. 到Python官网下载[Mac平台的安装程序](https://www.python.org/downloads/macos/), 双击安装即可
2. 如果你的Mac已经安装了Homebrew，直接使用命令 brew install python3 安装即可
   
### 2.3、在Linux上安装Python
在linux上安装Python的方法非常之多。最简单的可以通过命令 yum install python3 来安装。也可以使用 pydev, conda等来安装，这样可以很方便的管理Python多个版本。甚至可以下载源码进行编译安装。如果你是Linux平台开发者，安装对你来说应该非常简单，此处笔者比较推荐使用 conda 环境管理系统。可以下载 [Miniconda安装程序](https://repo.anaconda.com/miniconda/)，它是一个shell程序，直接运行即可完成安装。

## 三、运行Python
运行Python有两种方式，第一种我们可以直接在命令提示符窗口输入Python，即可进入Python的交互式环境。第二种也可以通过文本编辑器编写代码，然后使用 python 文件名 来运行python程序

### 3.1、交互式运行
登录到命令行提示窗口，输入python。如图:
![图片加载失败](/post_images/python/{{< filename >}}/3.1-01.png)
如上所示：在执行完python命令后，我们就进行了交互式界面。在交互式界面输入 print("hello world") 代码后，控制台马上执行并打印了 "hello world"。接口输入 exit() 便退出了python的交互式程序

### 3.2、使用文本编辑器
在交互式环境运行代码，好处是马上就能看到代码的运行结果，但是同样存在缺点，下次运行还得再敲一遍代码。因此我们写程序通常需要用文本编辑器将代码保存到磁盘上，便于下次直接运行。Windows上可以使用notepadd++、UE甚至记事本都是可以程序的编写，Linux、mac等类UNIX系统可以使用vim工具等。

我们打开文本编辑器，写入 print("hello world")，然后保存文件名为 hello_world.py。文件后缀并没有强制要求为.py，但是我们作为一个python代码文件，强烈要求这样命名。这样看到文件名就知道这是一个python程序。
![图片加载失败](/post_images/python/{{< filename >}}/3.2-01.png)
如图所示，我们就成功的运行了第一个python程序

## 四、集成开发工具pycharm
在上文中，我们已经可以通过文本编辑器来编写并运行python程序。但是仍然存在缺点，文本编辑器不能为我们检查语法是否正常，也没有命令提示，编写完成还需要手动通过命令去运行代码。那么有没有更强大的文本编辑器实现这些功能，当然是有的，市面上有很多这类编辑器。这里我们建议使用pycharm，可以去[pycharm官网](https://www.jetbrains.com/pycharm/download/?section=windows)下载pycharm安装包。官方提供社区版和专业版，作为初学者社区版足够了，当然你也可以安装专业版体验更多功能，专业版属于收费版本，建议通过官方渠道购买。

### 4.1、使用pycharm编写代码
在安装完pycharm后，就可以通过pycharm创建项目并编写程序了。
![图片加载失败](/post_images/python/{{< filename >}}/4.1-01.png)
如图所示：我们成功运行了第一个代码。如果你已经成功运行了第一个程序，就会发现在输入 pr 时，pycharm就自动给你提示了print()函数，非常方便。pycharm还有非常多的强大功能，在后续使用过程你都会慢慢体验到

## 五、结尾
如果你有过编程经验，那应该很容易完成python的安装和第一个程序的编写。但是如果你完全没有任何经验，或者非计算机相关专业来学习，安装过程可能也会遇到各种坎坷。毕竟本教程通过文章的形式发布，无法做到像视频那样面面俱到的演示。因此你不能顺利完成安装，可以通过文字下边的微信图标添加笔者微信进行答疑，或者在文章下边进行留言哦。



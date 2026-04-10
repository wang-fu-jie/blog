---
title:       JavaScript - js概述与基础语法
subtitle:    "js概述与基础语法"
description: "JavaScript是网景公司开发的用于动态网页的脚本语言。随着发现，它不仅可以用于浏览器渲染网页，还可以开发桌面软件和服务端。支持运行在不同的环境，本教程中我们主要是在浏览器环境运行js代码。"
excerpt:     "JavaScript是网景公司开发的用于动态网页的脚本语言。随着发现，它不仅可以用于浏览器渲染网页，还可以开发桌面软件和服务端。支持运行在不同的环境，本教程中我们主要是在浏览器环境运行js代码。"
date:        2023-01-02T14:11:43+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/e5/86/laptop_office_home_office_desk_workplace_computer_business_notebook-544885.jpg!d"
published:   true
tags:
    - js
slug:        "js-hello-world"
categories:  [ "JavaScript" ]
---

## 一、JavaScript概述
JavaScipt语言是在1995年由布兰登.艾奇用10天时间开发出来的。它是一种脚本语言，开发初衷是为了让网页变的更加动态，它最初的名字是livescript。由于当时java语言非常流行，他们为了蹭java的热度，更名为JavaScript，简称js。

1996年微软发布的IE3引入了JScript，这是一种与JavaScipt兼容的脚本语言。1997年欧洲计算机制造商协会制定了JavaScipt标准，称为ECMAScript，简称ES。ES是语言标准，它定义了代码怎么写，变量怎么定义，用什么关键字等等。它并不干预js在什么环境中运行。这导致js的应用愈发广泛，不仅可以运行在浏览器中，还可以做手机应用开发，也可以在服务端使用。

js是一种解释型语言，js代码在浏览器中运行的时候，是由浏览器的js引擎来解释执行的。目前比较流行的引擎是谷歌的V8引擎。


## 二、hello world
没当学习一门新语言时，第一行代码都是输出hello world。通用的我们来使用js写一个hello world，如下：
```html
<body>
    <script>
        document.write("<h1>hello world</h1>");
    </script>
</body>
```
如上，我们在html页面的script标签中写了一行代码，它的意思是通过js在页面上添加一盒标题，标题内容为hello world。 用浏览器打开这个html文件即可。

## 三、js的书写位置
js第一种书写位置可以直接写在html页面的`<script`>标签内，上节中的hello world代码就是采用的这种方式。js第二种书写位置是单独写在一个文件内，然后再html文件中引用。
```html
<script src="./script/index.js"></script>
```
当一个`<script`>元素引用了外部的js之后，在这个元素内再写js代码是无效的。`<script`>除了src属性之外，还有一个type属性。除了js代码外还可以指定其他的代码。这个属性可以省略，默认就是js。 另外在node.js的环境里面，js代码就只能写在文件里面。

## 四、js的基础语法
js代码的每一条语句以分号结尾，当然这并不是强制的格式，不加分号也可以。但是在一些情况下不加分号可能导致程序报错，因此建议还是加上分号。js代码是大小写敏感的。

js写注释的方式有两种，注释不参与代码执行。第一种是单行注释，使用 // 表示，一行内在这个符号后的内容都被视为注释。第二种是多行注释，使用/* */，在这个符号中间的内容都属于注释。
```javascript
// 单行注释
/*
多行
注释
*/
```

## 五、输出语句
输出语句可以把代码执行产生的结果输出到界面上，所有的输出语句都不是ES标准。在不同环境下输出内容显示的方式都不太一样，我们这里使用的输出语句是浏览器环境提供的。一旦脱离浏览器就不一定能使用了。下面介绍浏览器环境中的几种输出语句。
* document.write()   用于把数据输出到页面
* alert() 把数据用弹窗的方式显示
* console.log() 把数据输出到控制台。

当代码中使用alert()，页面会弹窗，你会发现页面先弹窗，当你关闭弹窗后页面其他元素才会被渲染，尽管弹窗的代码在下面。这涉及到浏览器的渲染原理和js执行引擎原理，我们后面再说。 这三种方式最常用的是 console.log()。这种方式在页面是不显示的，需要打开开发者工具在控制台查看。
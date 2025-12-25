---
title:       HTML & CSS - 第一个网页
subtitle:    "第一个网页"
description: "HTML和CSS是用来开发前端的语言，使用浏览器进行加载和渲染。开发前端使用vscode，本文我们将通过html语言来写第一个网友hello world。已经在html如何写注释。"
excerpt:     "HTML和CSS是用来开发前端的语言，使用浏览器进行加载和渲染。开发前端使用vscode，本文我们将通过html语言来写第一个网友hello world。已经在html如何写注释。"
date:        2025-08-11T17:21:43+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/b6/66/anonymous_studio_figure_photography_facial_mask-750872.jpg!d"
published:   true
tags:
    - HTML
slug:        "h5c3-first-webpage"
categories:  [ "H5C3" ]
---

## 一、 HTM & CSSL简介
HTML（HyperText Markup Language，超文本标记语言）是用于创建网页的标准标记语言。它定义了网页的结构和内容，并通过标签（tags）描述文本、图片、链接等元素的显示方式。

CSS（Cascading Style Sheets，层叠样式表）是一种用于控制网页 样式和布局 的样式表语言。它定义如何渲染 HTML 元素，包括颜色、字体、间距、动画等，使网页更美观、易读。

HTML和CSS都是w3c组织(万维网联盟)定义的语言标准。


## 二、浏览器内核
我们写好的HTML & CSS是交给浏览器来执行并渲染的。浏览器内核包含js执行引擎，渲染引擎，网络引擎等等。常见的浏览器和内核有以下几种：
* Chrome 最开始使用的内核是 WebKit，现在换成了 Blink。Blink是WebKit的一个分支
* Safari 使用WebKit， WebKit是谷歌和苹果公司联合开发的
* Firfox 火狐浏览器使用的内核是 Gecko
* IE ie浏览器使用的内核是 Trident

## 三、开发环境
前端的开发推荐使用vscode和chrome浏览器，当然pycharm专业版也是可以的。使用vscode这里建议安装以下插件：
* live server: 启动一个具有实时刷新功能的本地开发服务器，用于静态和动态页面
* Auto Rename Tag: 自动重命名配对的HTML/XML标签
* Chinese lorem:  生成中文简体的乱数假文

## 四、第一个网页
配置好开发环境就可以写网页的代码了，新建一个文件，后缀为html或htm结尾，输入一个 ! 号，即可自动生成html的框架。如下：
```html
<!DOCTYPE html>
<html lang="zh-CN">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    hello world
</body>

</html>
```
然后在右键使用浏览器打开即可完成网页的展示。

## 五、注释
学任何一门编程语言，注释都是非常必要的，它可以用来解释我们写的代码。HTML的注释结构如下：
```html
<!--这是一段注释-->
```
当然如果存在多行注释，直接回车换行即可。浏览器会把`<!--  -->`内的所有内容认为是注释。
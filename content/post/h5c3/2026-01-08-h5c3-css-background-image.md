---
title:       HTML & CSS - 背景图&精灵图
subtitle:    "背景图&精灵图"
description: "背景图是给元素设置背景图片，背景图是样式由CSS来控制的，通过背景图可以实现精灵图。精灵图是把多个小图合并为一个大图，这样浏览器在加载图片时就可以只做一次请求。"
excerpt:     "背景图是给元素设置背景图片，背景图是样式由CSS来控制的，通过背景图可以实现精灵图。精灵图是把多个小图合并为一个大图，这样浏览器在加载图片时就可以只做一次请求。"
date:        2026-01-08T13:53:50+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/c5/0b/grapes_vine_sunlight_vineyard_wine_grapevine_harvest_agriculture-1062577.jpg!d"
published:   true
tags:
    - CSS
slug:        "h5c3-css-background-image"
categories:  [ "H5C3" ]
---

## 一、背景图
背景图是给元素设置背景图片，背景图是样式由CSS来控制的。它使用background-image属性控制。示例如下：
```css
div {
    width: 500px;
    height: 300px;
    padding: 20px;
    border: 2px solid red;
    background-image: url(img/my.png);
    background-size: cover
}
```
如上所示： background-image属性通过url来设置背景图片。background-size用于设置图片大小，选项：
* cover 铺满图片，如果长宽不合适的情况下，图片比例会被拉伸
* contain: 平铺图片，不改变图片比例但是要铺满。因此图片可能会重复 
* 还可以设置长度和宽度数值，也可以设置百分比。

图片重复的这个效果，也是通过设置的。通过属性 background-repeat 设置。默认值就是repeat重复，还可以设置 repeat-x, repeat-y, no-repeat等多种取值。 还有一个重要的属性 background-position。它可以控制背景图的位置，常在精灵图中使用。


## 二、精灵图
当一个网页需要很多的小图片的时候，例如搜索的放大镜图片，并不会做成单独的多个小图。而是把这些小图都放到一张大图里。如果用多个小图的话就会有很多文件，这些图片如果存在于服务器，那浏览器渲染页面时就需要进行多次网络请求。这就会降低页面的加载速度。
background-position 有五个取值，分别是top、bottom、left、right、center。这是一种复合写法因此可以同时设置两个值。除了这些预设值之外，还可以设置数字和百分比。


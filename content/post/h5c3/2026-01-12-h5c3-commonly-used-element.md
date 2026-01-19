---
title:       HTML & CSS - 常用元素与样式补充
subtitle:    "常用元素与样式补充"
description: "本文中将补充一些常用搭配的元素、样式和选择器。包括多媒体元素、表格元素、表达元素。以及这些样式常用的css样式。"
excerpt:     "本文中将补充一些常用搭配的元素、样式和选择器。包括多媒体元素、表格元素、表达元素。以及这些样式常用的css样式。"
date:        2026-01-12T16:16:40+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/ed/f6/church_praying_prayer_cathedral_interior_building_room_space-726401.jpg!d"
published:   true
tags:
    - HTML
slug:        "h5c3-commonly-used-element"
categories:  [ "H5C3" ]
---

## 一、媒体元素
html中提供了`<video>`元素和`<audio>`元素分别用于展示视频和音频。还有一个`<iframe>`元素用于嵌入第三方网页。这三个元素和`<img>`元素一样，都是可替换元素。也就是它显示的内容取决于元素属性。本身都是行盒但是效果上类似于行块盒，可以修改尺寸。

### 1.1、视频元素
视频元素默认是没有播放按钮的，可以在页面鼠标右键显示所有控件。可以通过controls属性来显示播放控件，这是一个布尔属性，也可以不写属性值，只有出现这个属性就是真。打开网页视频也不会自动播放，需要设置 autoplay 属性，它也是一个布尔属性。诸多浏览器都是进行自动播放的，因为有些陌生站点的视频可能让用户感到不适，因此想要自动播放还需要加上 muted 属性，它的作用是静音，即允许静音播放。
```html
<video controls="controls" src="aa.mp4"></video>
```
有些浏览器可以带声音自动播放，原因在浏览器自动播放策略里面有媒体参与度。也就是你和某个站点互动的记录浏览器都会统计，当数据达到一定阈值时浏览器就会放开限制了。`<video>`还有一个 loop 属性表示循环播放。

### 1.2、音频元素
音频元素`<audio>`元素，它和视频类型，但是默认是不显示的，因此音频没什么可以显示的，唯一可现实的就是播放控件。因此也需要使用controls属性。

### 1.3、内敛框架元素
网页上除了可以嵌入视频和音频之外，还可以嵌入第三方网页。即内敛框架元素`<iframe>`，它用于把另一个html页面嵌入当前页面中。
```html
<iframe src="https://www.bilibili.com" frameborder="0"></iframe>
```
frameborder属性用来指定是否线上边框，1表示显示0表示不显示，默认值是1。边框属于样式范畴，因此这个属性一般会设置为0。
```html
<a href="https://www.bilibili.com" target="window">B站</a>
<a href="https://www.amap.com" target="window">高德</a>
<iframe name="window" src="https://www.bilibili.com" frameborder="0"></iframe>
```
如上这个示例：两个链接分别跳转到两个外部网站，通过target属性就可以切换这两个网站在`<iframe>`中打开，而不是使用浏览器新的标签页。

## 二、表单元素
表单元素用来收集用户输入的信息。如输入框、下拉列表、多选框、单选框、按钮等。

### 2.1、输入框
输入框`<input>`元素是一个空元素，没有结束标记。
```html
<input type="text" placeholder="请输入账号：" value="123456">
```
type属性表示输入框的类型。它的取值比较多，其中常用的有以下选项：
* text： 表示单行文本输入框，如果复制了带换行符的文本进行，它会自动去掉换行符
* password: 密码输入框，输入的内容会默认被加密
* search: 搜索框，和单行文本输入框类型，它的框右侧有个 x 符号，可以清空内容。
* tel: 电话号码输入框
* number: 数字输入框
* radio: 单选框，当使用单选框时，一般配合属性name使用，name相同的单选框为一组，即只能在一组内选择一个。
* checkbox: 多选框，多选框一样需要分组，因为一个页面中也可以存在多种多选框。
* date: 日期选择框
* color: 颜色选择框
* file: 文件选择框
* range: 滑块选择框，可以使用min和max属性设置滑块的最小值和最大值
* button/submit/reset: 这三个都是按钮，现在基本不会使用`<input>`元素做按钮了

除了这些之外，type属性还有其他选项，可以自行查mdn文档。下边看一个示例：
```html
性别：
<input id="male" name="gender" type="radio">
<label for="male">男</label>
<input id="female" name="gender" type="radio">
<label for="female">女</label>
```
这个例子中`<label>`元素是标签文本，它在这块的作用是优化体验，即选择男这个文字时也会选中单选框，以上这种写法叫显示关联。还有一种隐式关联，即直接把`<input>`元素放到`<label>`元素内部，这样就不需要for属性了。

placeholder属性是占位符，也就是输入框里面没有内容的时候显示的提示文字。 value属性表示输入框的内容，当第一次输入内容后，后续再打开网站发现有上次输入的内容，就是给输入框设置了value。

### 2.2、按钮
前边提到使用`<input>`元素做按钮已经过时了，虽然仍然可以使用。但是现在基本直接时候`<button>`元素，它的type属性取值也是有button/submit/reset这三个。默认是submit
```html
<button type="button">按钮</button>
<button type="button">提交</button>
<button type="button">充值</button>
```
现在这些按钮点击是看不出效果的，因为需要js的配合。

### 2.3、文本域与下拉列表
文本域也叫多行文本输入框，使用`<textarea>`元素
```html
<textarea cols="30" rows="10"></textarea>
```
cols表示文本框的宽度，可以显示多少个字符，指的是英文。rows表示文本框的高度。这两个属性一般使用时通过css控制。

下拉列表选择框使用`select`元素展示，需要配合`<option>`元素来设置可选值
```html
<select>
    <option value="语文"></option>
    <option selected value="数学"></option>
    <option value="英语"></option>
</select>
```
selected属性是用于设置默认被选择的选项。这些选项也可以进行分组，使用`<optgroup>`元素包括起来。下拉列表默认只能选一个，但是如果给`<select>`元素加一个multile属性就可以选多个。

## 三、表单状态
以上介绍的这些表单元素有两个布尔属性表示表单的状态，如下：
* readonly: 只读状态，
* disbaled: 禁用状态，有这个属性，样式默认就会变成灰色 

## 四、form元素
大部分情况下，表单元素会配合`<form>`元素使用，即把页面上用到的表单元素使用`<form>`元素包裹起来。当点击提交按钮时，`<form>`中的表单元素的内容会提交到服务器。当点击重置按钮，所有表单元素会重置为初始值。
```html
<form action="" method="get">
        <div><input name="xxx" type="text"></div>
        <button>提交</button>
</form>
```
action属性是用来设置服务器地址的，即将表单数据提交到的目标服务器。 默认是提交给当前页面。method属性指定数据提交的方式。被提交的数据都需要加name属性告诉服务器提交的是什么数据。点击提交后，浏览的地址栏会变成如下：
```
file:///Users/wangfujie/Documents/code/front/text/index.html?xxx=aaa
```
地址栏中会多个问号，问号后的内容就是提交的表单数据，使用kv格式。他们作为参数一起提交给服务器。

还有一个常用的元素`<fieldset>`元素，它的作用是给表单进行分组。分组可以使用`<legend>`元素来设置个标题，这个就请读者自己写代码来体验了。

## 五、表单样式
表单元素有一些特有的css样式，还有一些特殊的伪类选择器。我们分别介绍：

focus: 表示元素聚焦时的样式。当鼠标选择输入框，会发现他样式多了蓝色的外边框，这就是聚焦时候的样式。通过tab键可以切换被聚焦的元素。
```css
input:foucs {
    outline: 5px soild red;
    color: red; 
}
```
如上所示： 来修改表单元素被聚焦后的样式，是outline。 并不是border，它和边框比较像，但是outline并不占用空间。它的宽度不影响其他元素的位置。一般在开发中会把outline设置为none，因为不同浏览器的默认样式不一样。

checked: 单选、多选框被选中时的样式。但是如果通过checked来设置单选多选框的颜色，背景颜色时会发现没有效果。 这是因为单选框多选框、日期选择框等属于可替换元素，很多样式不能设置。想修改就只能自己做。以下是一个实现方式：既然选择框不能修改，那它对应的说明文字是可以修改的，就可以把它的文字使用一个元素包起来。
```css
input {
    display: none;
}
input:checked+span {
    background-color: red;
}
```
这样，我们隐藏了输入框的样式，就可以任意设置span的样式了。前边也设置了点击文字也可以让选择框被选中。

placeholder-shown: 选择placeholder展示时的状态，前提是设置了placeholder。 还有一个placeholder伪元素选择器，这个选择器可以选中placeholder并设置样式。这块就请自行写代码体验。


## 六、元素和选择器补充
这里我们补充几个元素，大多属于经常可以见到的：

`<meta>`元素是用来添加元数据的，比如设置字符集、移动端适配。当然还有其他用法，如：
```html
<meta name="keywords" conetent="网上购物、服装、电子设备">
```
这里面的关键字会被搜索引擎读取，有利于网站seo优化。除了keywords，还可以写 description、author。

`<link>`元素是用来链接外部资源，除了可以链接css文件，还可以链接图标 

接下来还有一组元素比较常用，那就是表格表格元素。表格使用`<table>`元素来标记，表格标题使用`<caption>`元素，表头使用`<thead>`元素标记，表头可以有一行也可以有多行。`<tr>`元素表示行，`<th>`元素表头的列。表体使用元素`<tbody>`表示，在表体中，表示列使用`<td>`，表示行依然使用`<tr>`元素。还可以有表尾使用`<tfooter>`一般用来展示合计的数据。也是使用`<td>`和`<tr>`分别表示行元素和列元素。

对于表格我们可以实现隔行变色效果，涉及一个新的伪类选择器，就是nth-child，表示第几个子元素。可以传odd表示奇数也可以传even表示偶数。还有一些其他的常用选择器如下：
* :first-child  选择第一个子元素
* :first-of-type 选择一组兄弟元素中第一个同类型的元素 
* :last-child  选择最后一个子元素
* :last-of-type 选择一组兄弟元素中最后同类型的元素 
* :nth-child  选择指定位置的子元素，可以传入odd、even、表达式（3、 3n、 2n+1）等
* :nth-of-type 选择指定位置的同类型的兄弟元素
* ::first-letter  选择元素中的第一个文字 
* ::first-line   选择元素中的第一行文字 
* ::selection   选择用户选中的文字


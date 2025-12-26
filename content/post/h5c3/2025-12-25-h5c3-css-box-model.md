---
title:       HTML & CSS - 盒子模型
subtitle:    "盒子模型"
description: "在网页设计中，每个元素都会生成一个矩形区域，这个矩形区域就会被认为是一个盒子(box)。盒模型就是用来描述这些盒子的方式。盒子类型我们先了解，行盒、块盒和行块盒。"
excerpt:     "在网页设计中，每个元素都会生成一个矩形区域，这个矩形区域就会被认为是一个盒子(box)。盒模型就是用来描述这些盒子的方式。盒子类型我们先了解，行盒、块盒和行块盒。"
date:        2025-12-25T17:28:02+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/07/51/toronto_city_cn_tower_skydome_skyline_architecture_buildings_towers-639098.jpg!d"
published:   true
tags:
    - CSS
slug:        "h5c3-css-box-model"
categories:  [ "H5C3" ]
---

## 一、盒子模型
在网页设计中，每个元素都会生成一个矩形区域，这个矩形区域就会被认为是一个盒子(box)。盒模型就是用来描述这些盒子的方式。盒子类型我们先了解两种，行盒和块盒:
* 行盒：display等于inline的元素，水平方向排列不换行，以前的叫法是行内元素
* 块盒：display等于block的元素，垂直方向排列独占一行，以前的叫法的是块级元素

行盒块盒是CSS属性控制的，因此也是可以修改的，需要修改元素的display属性即可。

### 1.1、盒子组成
盒子由内容、填充、边框、外边距四部分组成。

内容(content): 内容区域有两个属性，width和height，用来设置内容区域的宽高。这部分也可以叫做内容盒（content-box）

填充(padding): 填充也叫做内边距，有四个属性分别是padding-top、padding-bottom、padding-left、padding-right分别是上内边距、下内边距、左内边距、右内边距。这四个属性也可以使用padding属性简写。如下：
```css
/*应用于所有边*/
padding: 10px

/*上边下边 | 左边右边*/
padding: 10px 20px

/*上边 | 左边右边 | 下边*/
padding: 10px 20px 30px

/*上边  | 右边 | 下边  | 左边*/
padding: 10px 20px 30px 40px
```
内容区+填充区就被称为填充盒（padding-box）

边框(border): 边框有border-style、border-width、border-color、border-radius等常用属性， 其中border-radius是CSS3的属性用于设置元素的圆角。 border-width用于设置边框的宽度，border-style设置边框的样式，他内置多种选择，例如虚线、圆点、实线等，默认值时none不显示边框。。border-color设置边框颜色，如果不设置默认和字体颜色一致。这些属性都是简写属性，因此也可以设置多个值分别设置上下左右。 如果四个方向属性都一样，还有一个更简洁的书写方式，如：
```css
border: 5px solod green
```
这样就可以同时设置四个方向的这三种样式。这三个值的顺序是没有要求的。内容区+填充区+边框区被称为边框盒（border-box）

外边距(margin): 外边距指的是边框到其他盒子之间的距离。和padding一样，有margin-top、margin-bottom、margin-left、margin-right四个方向的外边距。也可以使用margin简写来表示。margin还可以设置为auto用于自动调整剩余空间（比如居中）。但是padding不支持auto。

## 二、盒模型扩展
先看如下一个示例代码：我们实现一个盒子，设置左边距50px把内容向右撑开。
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        div {
            width: 220px;
            height: 50px;
            line-height: 50px;
            padding-left: 50px;
        }
        div:hover{
            background-color: #green;
        }
    </style>
</head>
<body>
    <div>数据分析</div>
</body>
</html>
```
用浏览渲染这个网页，效果如下：
<rawhtml>
<img src="/post_images/h5c3/{{< filename >}}/2-01.png" alt="图片加载失败" width="300"/>
</rawhtml>
如图：div这个元素的尺寸变为了270 * 50。也就是现在边框盒的尺寸。这样如果我们再加上其他方向的内边距，再加上边框盒子会变的变更大，如果我们初衷希望盒子的大小就是 220 * 50。这样实现是不行的。接下来我们就解决这个问题。有三种解决方式：

* 不使用外边距，使用文本缩进属性 text-ident。 缩进50像素。但是不建议这么使用，这个属性是用于段落缩进的，这里使用不合适。
* 把宽度属性减去50，但是这样每次都需要计算宽度或高速。
* 使用css3的属性，box-sizing。它有两种取值：content-box和border-box，默认就是content-box表示我们设置的宽高是内容盒的宽高。当改为border-box后就代表边框盒的宽高，这样再设置padding-left后，它根据border和padding的宽高自动设置内容盒的宽高

### 2.1、宽高与背景范围
从上面示例中，我们可以通过box-sizing: border-box 来设置宽高属性为边框盒的宽高。同样如果设置了border后，内容盒的宽度也会自动调整变小。如下：
```css
div {
    width: 220px;
    height: 50px;
    line-height: 50px;
    padding-left: 50px;
    box-sizing: border-box;
    border: 5px dashed red;
}
```
设置了边框5像素，上下左右边框各占用5像素。这样内容盒就被压缩到了 140 * 40。因此line-height也需要调整为40px，否则就不再居中显示了。同时可以看到背景颜色覆盖的是边框盒，边框虚线的部分显示了背景的绿色。这也可以修改，通过background-clip属性来实现，它有三个取值：border-box、content-box、padding-box，默认是border-box覆盖边框。

### 2.1、溢出控制
当元素文本内容比较多时，例如一个`<p>`元素，如果给`<p>`设置了宽度，由于宽度变小原来的内容放不下。 就会自动增加高度。再设置一个高度，高度放不下后，多出的文字就会溢出。这个效果可以通过给`<p>`设置边框属性来演示，可以明显看到文字溢出边框。默认情况下溢出的文字是正常显示的，可以通过一个属性控制溢出边框盒的部分是否显示：
```css
p{
    border: 5px dashed red;
    width: 200px;
    height: 200px;
    overflow: hidden;
}
```
如上：给overflow属性设置为hidden就会隐藏溢出的文字，它默认是visibale可见，还有一个选项是scroll，表示生成滚动条，就可以通过鼠标滚动来查看隐藏的文字。overflow也是个简写属性，它是overflow-x，overflow-y的简写。但是设置了scroll后滚动条一直展示即使没有隐藏的文字，这样不太美观。因此还有一个取值 auto 表示浏览器自动计划是否需要滚动条。

### 2.2、文本溢出
在实际场景中，例如网易的新闻列表对于内容超长的标题，只显示一部分文字然后加三个点。我们可以使用如下样式实现：
```css
li{
    width: 200px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}
```
如上，当我们给列表设置宽度后，如果文字长度超过200，默认会换行，这时white-space就可以设置不换行一行显示。overflow就会隐藏右边溢出的文字。但是最右边的文字可能只显示一般，因为默认是切割的，即text-overflow默认取值是clip. 这是text-overflow就会使用 ... 来显示。

同时这个实验还有个现象，设置了overflow后，`<li>`元素前面的小点不见了，这是因为这个小点是通过伪元素生成的，且位置是由list-type-position属性控制，默认是outside即位于盒子外面。把它改为inside这个小点就会被放置到内容盒里面。


## 三、行盒
当同样的文字分别使用块盒`<p>`元素和行盒`<span>`元素看起来显示一样，但是加上边框，边框效果就变成了如下：
![图片加载失败](/post_images/h5c3/{{< filename >}}/3-01.png)
块盒是一块一块的，但是行盒第一行结束的位置没有边框，意味着内容没有结束，它一直延伸到内容结束。 行盒是跟着内容进行延伸的，无法直接设置宽高，宽高由内容决定。

行盒的内边距、边框、外边距水平方向尺寸有效，垂直方向不占空间。我们拿内边距来说明，设置左右内边距时是有效的它会把内容撑开。但是设置上下内边距时，文字不会动的，上下内边距增加只是背景在增加。因此行盒的内边距是不占空间的，它占的空间不会把内容撑开。

## 四、行块盒
前边我们知道行盒无法设置宽高。但是如果有需求即要求水平排列又要求可以设置上下左右的尺寸，那就是行块盒。行块盒是display属性等于 inline-block 的元素。它是水平排列且所有方向的尺寸有效。行块盒经常会用于网页下放的翻页按钮。
```css
<!DOCTYPE html>
<html lang="en">
<head>
    <style>
        body {
            background-color: #ccc;
        }
        .pagintion a {
            background: #fff;
            display: inline-block;
            width: 30px;
            height: 30px;
            text-align: center;
            line-height: 30px;
            text-decoration: none;
            border-radius: 5px;
        }
        .pagintion .selected {
            background-color: blue;
            color: #fff;
        }
    </style>
</head>
<body>
    <div class="pagintion">
        <a class="selected" href="">1</a>
        <a href="">2</a>
        <a href="">3</a>
        <a href="">4</a>
        <a href="">5</a>
    </div>
</body>
</html>
```
如上所示，我们就实现了翻页按钮的效果。

### 4.1、空白折叠
在以上的实例中，页面的展示效果如下：
<rawhtml>
<img src="/post_images/h5c3/{{< filename >}}/4-1-01.png" alt="图片加载失败" width="300"/>
</rawhtml>
如上所示：每个按钮之间有一小段距离，且鼠标可以选择。这是一个空白字符，它来自于空白折叠。因此空白折叠会发生在行盒内部以及行盒之间，包含行块盒。因此在以上代码中`<a>`之间的换行和空格就会产生空白折叠。有三种解决办法：

* 元素间不要换行直接写一行。但是会降低代码可读性
* 使用注释把多个元素隔开不要使用换行。 同样降低代码可读性
* 这些空白属于父元素`<div>`因此可以把父元素的字体设置为0，空白字符就没有了。但是子元素会继承父元素的字体大小，再把子元素的字体设置为initial不继承。

第三种方式并没有去掉空白折叠，只是把它变小了，目前只能这样做。后续有其他布局方式来解决这个问题。

### 4.2、可替换元素
可替换元素指的是页面上显示的内容，取决于元素属性。在效果上是类似于行块盒的，给它设置的所有尺寸都是有效的。比如`<img>`元素、文本框、按钮、多媒体元素等。这些都是行盒，也是可替换元素。

与之相对的是不可替换元素，页面上显示的内容取决于元素内容。 也就是内容写什么，页面就显示什么。
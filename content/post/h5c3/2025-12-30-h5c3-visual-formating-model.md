---
title:       HTML & CSS - 格式化视觉模型(浮动与定位)
subtitle:    "格式化视觉模型(浮动与定位)"
description: "格式化视觉模型研究的是页面中多个盒子的排列规则，俗称布局规则。它把页面中盒子的排列方式分为三种：普通流、浮动和定位。"
excerpt:     "格式化视觉模型研究的是页面中多个盒子的排列规则，俗称布局规则。它把页面中盒子的排列方式分为三种：普通流、浮动和定位。"
date:        2025-12-30T16:51:37+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/c2/70/dark_dj_turntable_hand_music-1176248.jpg!d"
published:   true
tags:
    - CSS
slug:        "visual-formating-model"
categories:  [ "H5C3" ]
---

## 一、格式化视觉模型
盒模型研究的是单个盒子的规则。 格式化视觉模型研究的是页面中多个盒子的排列规则，俗称布局规则。它把页面中盒子的排列方式分为三种：普通流、浮动和定位。

首先介绍一个概念叫包含块：在大多数情况下：包含块就是父元素的内容盒，它决定了盒子尺寸和排列位置。假设有两个盒子，父元素包含一个子元素，子元素默认排列在父元素的内容盒位置。当父元素增加外边距时父元素进行移动，子元素也会随之移动。
```html
<html lang="en">
<head>
    <style>
        .parent {
            background: #ccc content-box;
            width: 500px;
            height: 500px;
            border: 5px, solid;
        }
        .son {
            background-color: red;
            width: 200px;
            height: 200px;
        }
    </style>
</head>
<body>
    <div class="parent">
        <div class="son"></div>
    </div>
</body>
</html>
```
如上代码所示：子元素会显示在父元素内容盒的左上角。

## 二、普通流
标准流又称为标准流、文档流、常规流等等。所有元素默认情况下，都是属于标准流布局。布局方式就是由盒模型的特点决定的，即块盒独占一行垂直排列，行盒不换行水平排列。


### 2.1、auto的意义
在普通流中，块盒有如下特点：在水平方向上默认情况下，块盒总宽度等于包含块宽度。也就是内容区+填充区+边框+外边距的总宽度等于包含块的宽度， 因为width的默认值是auto(auto的作用是剩余空间全部吃掉)。在上面代码示例中：将子元素的width注释掉或者改为auto，那子元素的宽度也会是500px。margin也是设置为auto，当margin和width都是auto时，外边距会是0。原因是内容区的重要性比外边距高。 当只把margin设置为auto， 而waidth固定时：子元素就在包含块内左右居中了，因此左右margin都是auto各占一半。而margin不设置默认是0，子元素默认在最左边，多余的空间就只能空着。当margin设置为负数时且width为auto，那剩余空间就会变多全部被内容盒吸收，内容盒就会变宽。

接下来看垂直方向：heght属性的默认值就是auto，表示自适应内容宽度。margin默认是auto表示0。

### 2.2、百分比取值
width、hegint、padding、margin都可以使用百分比取值。例如width可以设置为30%，但是他们的含义不同。width、padding、margin相对的是包含块的宽度。hegint属性的百分比包含两种情况：

* 如果包含块的宽度不是auto，百分比就是相对于包含块的高度
* 如果包含块的宽度是auto，百分比无效。这是因为auto表示自适应内容宽度，即自适应子元素高度，子元素设置百分比又依赖父元素包含块的高度，因此互相依赖百分比无效。

### 2.3、margin合并
当两个块盒都设置了margin时，第一个盒子的下margin和第二个盒子的上margin和合并，示例如下：
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        p {
            border: 1px, solid;
            margin: 100px;
        }
    </style>
</head>
<body>
    <p>Lorem ipsum dolor sit amet consectetur adipisicing elit. Nesciunt laudantium sint neque distinctio. Ipsam obcaecati doloribus omnis enim tempora veniam itaque, numquam a impedit, voluptas voluptate. Veritatis odio earum quibusdam.</p>
    <p>Lorem ipsum dolor sit amet consectetur adipisicing elit. Ad magnam adipisci voluptate quidem. Quo, sit non! Nam sed nobis sapiente tempore adipisci, culpa asperiores beatae eos doloremque amet impedit esse!</p>
</body>
</html>
```
如上所示，两个`<p>`元素都设置是上下左右margin为100px，在浏览器调试界面选择两个元素时，会发现第一个元素的下margin是紧挨着第二个元素的border的，选择第二个元素，它的上margin紧贴着第一个元素的border。因此它们共用了一个margin，其他方向都没问题就是上下方向相邻的margin。合并的方式是取最大值。

不止是兄弟元素会margin合并，嵌套元素也会margin合并。例如父元素和子元素都设置了margin，理论上子元素应该全部在包含块内，即距离包含块应该存在外边距的距离。事实上左右是没问题的，确实是在包含块内，距离包含块存在指定的margin距离，但是上下外边距它们就合并了，即子元素和父元素的外边距范围一样。因为它们的外边距是相邻的，因此就合并了。如果不想合并只要把他们分开就可以了，可以给父元素加一个border。如果父元素要求不能有border，给父元素设置padding也可以，子元素就不需要设置margin了，父元素的padding把它撑开就可以了。


## 三、浮动
浮动（float）的核心作用是：让元素脱离正常文档流的一部分，并向左或向右贴边排列，其它内容可以环绕它。无论是行盒还是块盒，浮动后都会变成块盒。通过float属性来设置盒子进行浮动，可选取值如下：
* 默认值 none: 不浮动，即普通流
* left: 左浮动，元素靠上靠左排列
* right: 右浮动，元素靠上靠右排列

如下给一个浮动的示例：
```html
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        .parent {
            width: 500px;
            height: 500px;
            background-color: red;
        }
        .parent div {
            width: 100px;
            height: 100px;
            background-color: blue;
        }
        .left {
            float: left;
        }
        .right {
            float: right;
        }
    </style>
</head>
<body>
    <div class="parent">
        <div class="left"></div>
        <div class="right"></div>
    </div>
</body>
</html>
```
浮动的浮动范围和普通流是一样的，也是在包含块内。浮动和普通流在 auto 上也有区别。
* 当width、hight为auto时，是适应内容宽度，而不再是吃掉剩余空间
* margin为auto时，表示0。

### 3.1、多个盒子浮动
当1个盒子左浮动时，是靠上靠左排列。那多个盒子同时浮动是挨着进行靠上靠左排列，当第一行拍不下了，就会自动开启第二行。这里可以自行测试。

当多个盒子中，同时存在普通流块盒和浮动流盒子时。排列方式存在两种情况：第一种是普通流块盒元素在浮动流盒子之前，那浮动流盒子会绕开普通流盒子。第二种是普通流块盒元素在浮动流盒子之后，普通流块盒会显示在浮动流盒子的下面，即普通流块盒会无视浮动盒子。注意，这里的普通流只是块盒。 这里可以理解为，普通盒子在第一层，浮动盒子会浮动到第二层，当浮动盒子先排列到第二层后，那它下面的第一层是空的，因此普通流块盒会排列在浮动盒子下面。当先排列了普通盒子在第一层后，后面的浮动盒子不能直接覆盖到第二层，因此会绕开普通盒子进行排列。

<rawhtml>
<img src="/post_images/h5c3/{{< filename >}}/3.1-01.png" alt="图片加载失败" width="400"/>
</rawhtml>
我们来看以上一个排列，蓝色为浮动盒子，绿色为普通流盒子。这里第三个蓝色盒子为什么没有居左排列呢。这是因为第三个蓝色盒子之前是绿色盒子，因此它首先应该避开绿色盒子因此排列是在新的一行，那为什么没有居左呢。这是因为前两个蓝色盒子有外边距，它占用了最左边两个位置垂直方向上的几个像素。因为第三个盒子摆放原则是找 找“最上、最靠左”的合法位置。因此显示出来就在中间摆放。

普通流块盒排列会无视浮动盒子， 但是行盒不同，普通流行盒排列时会避开浮动盒子。这个特性可以用于设置文字环绕，当在div中存放文字时，虽然div是块盒，但是内容文字是行盒，或者说浏览器生产了一个匿名行盒对文字进行包裹。

### 3.2、高度坍塌
当一个父元素中所有子元素都浮动，且父元素不设置高度属性时，浏览器渲染后会发现父元素直接不显示了。调试界面可以查看到高度变为0了，这个现象就是高度坍塌。原因是浮动盒子脱离了文档流，因此普通盒子在计算高度时不考虑浮动盒子，普通又没有内容因此高度为0。解决高度坍塌方式如下：

在父元素中放一个普通流盒子来把高度撑开，但是无法给这个普通盒子设置高度，如果直接设置高度那和直接给父元素设置高度就没有区别了，另外浮动的盒子有可能未来增加固定高度不合适。这时候就需要用到另外一个属性 clear:both， 这个普通盒子设置了清除浮动后，他在排列是就会考虑浮动盒子了。通常解决高度坍塌不会在html写一个空元素，虽然这样可行，但这个元素是个空元素没什么意义，因此一般使用伪元素来解决高度坍塌问题。
```css
.clearfix::after {
    content: "";
    display: block;
    clear: both; /*清除浮动*/
}
```
这样就可以把父元素加上 class="clearfix"属性了，这个伪元素选择器就会在父元素末尾添加一个空的伪元素，伪元素默认是行盒因此需要改为块盒。这个伪元素after撑开了父元素的高度。

## 四、定位
普通流和浮动流，都不能精确控制盒子的位置。因此就需要一个新的布局规则，定位。用于定位的css属性是 position， 它可以控制元素在包含块中的精准位置。position属性的默认取值是static，表示没有定位。

### 4.1、相对定位
相对定位是使元素相对于自身原来的位置进行偏移，不会影响其他元素的位置，不会使元素脱离文档流。也就是元素原来是什么样子，定位后还是什么样子，原来是文档流定位后还是文档流，原来是浮动流定位后也还是浮动流。相对定位需要设置元素的position属性为 relative。
```css
p {
    border: 1px, solid, red;
    position: relative;
    top: 10px;
}
```
如上：当设置1个元素为相对定位后，就可以通过 top、bottom、left、right进行上下左右方向的偏移了。

### 4.2、绝对定位
前面我们提到绝大部分情况下，包含块都是父元素的内容盒。但是在绝对定位时，包含块是所属定位元素的填充盒。也就是说找它是属于哪个定位元素，找到最近的定位元素，如果父元素没有定位那就找祖先元素。如果没有已定位的祖先元素，那就回相对于初始包含块进行定位。初始包含块是一个虚拟的矩形可以理解为整个页面。这里说的定位元素指的是不要position不为static都是定位元素，无论哪种定位。当对元素进行绝对定位时，它就会脱离文档流，后面的元素就会无视它。

一般情况下：我们给一个元素进行绝对定位，都是给它的父元素设置相对定位，也就是常说的子绝父相。这样这个元素的包含块就是父元素的填充盒。一个元素需求叠在另一个元素上边就可以使用绝对定位。例如：音乐或电影网站的喜欢按钮叠在海报的图片上。既然重叠在一起就需要控制谁在前谁在后， css提供了一个 z-index属性，只有定位元素设置这个属性才有效。数字越大就越靠前。数字一样就按元素顺序。

### 4.3、固定定位
固定定位的所有特点和绝对定位一样。相对定位需要设置元素的position属性为 fixed。唯一不一样的是包含块不一样，固定定位的包含块是视口，也就是浏览器可是窗口。固定定位会把元素固定到浏览器窗口的某个位置。

### 4.4、初始包含块
初始包含块是和视口大小一样的固定虚拟图形，它会靠在页面左上方，位置不会的，但是大小会跟着视口变化。

### 4.5、粘性定位
粘性定位可以理解为相对定位和固定定位的混合，元素在跨越特定阈值前为相对定位，之后为固定定位。可以理解当一个元素设置为粘性定位后，当进行窗口滚动时，它的边框与视口边框小于等于0的时候，它就变成固定定位了。这个属性非常适合制作导航栏。粘性定位的父元素显示，它就会显示，父元素笑死它就会消失。

---
title:        JavaScript - 数据类型与变量
subtitle:    "数据类型与变量"
description: "js中的数据类型分为两大类，一类是原始类型，属于不可再细分的类型。第二类是引用类型，它可以再细分。一般开发中会使用变量来保存数据，变量需要先定义再赋值。"
excerpt:     "js中的数据类型分为两大类，一类是原始类型，属于不可再细分的类型。第二类是引用类型，它可以再细分。一般开发中会使用变量来保存数据，变量需要先定义再赋值。"
date:        2023-01-04T13:57:06+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/c5/9b/work_computer_laptop_technology_coding-7742.jpg!d"
published:   true
tags:
    - js
slug:        "js-datatype"
categories:  [ "JavaScript" ]
---

## 一、数据类型
js中的数据类型分为两大类，一类是原始类型，属于不可再细分的类型。第二类是引用类型，它可以再细分。

### 1.1、原始类型
js的原始类型中包含以下几类，数字类型(number)、字符串类型(string)、布尔类型(boolean)、undefined类型、null类型。

**数字类型**

js中默认的数字类型是十进制，我们可以直接在浏览器的控制台进行测试：
```js
>console.log(10)
10
>console.log(010)
8
>console.log(0o10)
8
>console.log(0x10)
16
>console.log(0b10)
2
```
如上所示，默认表述十进制，如果在前边加上 0 或 0o 就表示八进制。 但是如果在vs中，使用0开头就会被标记红色波浪线，但是程序依然可以正常运行。这是因为在ES5严格模式以及后来的彼岸准中，这种写法被废弃了，因为它容易导致混淆。但是在非严格模型下浏览器依然兼容这种模型，


**字符串类型**

字符串表示的是一段文本，可以是0个或多个文本。字符串可以使用单引号、双引号、反引号来表示。其中反引号是ES6才支持的语法，叫做模版字符串。
```js
console.log('你好')
console.log("你好")
console.log(`你 

好`)
```
反引号字符串是可以包含空白字符和换行的，但是单引号和双引号不可以。但字符串本身存在单引号或者双引号时，可以结合使用，如 "你'好"。js中的字符串也可以使用转义字符 \，关于转义字符我们这里不过多讲述。

**布尔类型**

布尔类型用来表示真和假两种状态，true表示真，false表示假。
```js
console.log(true)
console.log(false)
```

**undefined类型**

undefined类型只有一个值，那就是 undefined。它表示值未定义
```js
console.log(undefined)
```

**null类型**

null类型的值也是只有一个，null。它表示空


### 1.2、引用类型
引用类型分为两种，分别是对象 object 和函数 function。函数的写法我们先不讲。对象就是容器，用来存放一系列的数据和功能。js中的对象使用 {} 来表示：
```js
console.log({
    name: '小明',
    age: 18,
    score: {"语文": 98, "数学": 78}
})
```
如上所示，对象在形式上来看和字典非常类似，它是可以嵌套的，甚至可以无限的嵌套下去。


## 二、获取数据类型
js中提供了一个功能 typeof，它可以获取数据的类型。我们先看示例：
```js
>console.log(typeof 10)
number
>console.log(typeof '10')
string
console.log(typeof null)
object
```
你可能比较好奇，null为什么是object类型，这是一个历史遗留问题。如果修复这个问题会带来很大的兼容性问题。


## 三、变量
变量是用来保存数据的一块内存空间，变量的使用有两个步骤，即先声明再赋值。声明变量使用关键字 var， 变量名只能由字母、数字、下划线、美元符号组成，且不能以数字开头。只声明了变量但是没有赋值，变量默认存的就是 undefined。
```js
var age;
console.log(age, typeof age);
console.log(typeof abc)
```
以上的示例中，age只定义了没赋值，abc没有定义，因此以上的输出都是 undefined。但是直接打印abc就会报错，因此可以使用 typeof 来判断变量是否被定义。接下来我们看赋值：
```js
var age;
age = 18;
console.log(age, typeof age);
age = "123"
console.log(age, typeof age);
```
如上所示，给age变量赋值后，age变量的数据和类型都可以变化，因此js属于弱类型语言。变量的声明和赋值也是可以通过一条语句完成的。即简写方式：
```js
var age = 18;
var a = 1, b = 2, c = 3, d;
```

变量没有声明直接使用会报错，那我们看以下代码：
```js
console.log(age);
var age = 18;
```
执行会发现这个代码并不会报错，会输出 undefined。这其实涉及到了js中的变量提升概念，因为 var age = 18, 其实是 var age; age = 18的简写语法糖。js执行的时候会把代码中所有的变量声明提升到代码的最顶部。 变成了如下这样：
```js
var age;
console.log(age);
age = 18;
```
变量提升只会发送在当前脚本块，即同一个`<script`>标签中的js代码会变量提升。如果再前面再写一个`<script`>输出age，那这个输出就会报错。


## 四、总结
本节中我们讲述了js的数据类型和变量使用，旨在了解都有哪些数据类型，并没有对这些数据类型的使用做详细解释。不要着急，后续的课程我们会逐个讲解。另外本文中的代码，可以在浏览器的控制台来执行。也可以写在html的`<script`>标签中执行。
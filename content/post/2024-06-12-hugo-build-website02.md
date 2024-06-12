---
title:       从零开始搭建个人博客网站(二)
subtitle:    "将个人博客网站托管到Github Pages上"
description: "GitHub Pages 提供了一项静态网页托管服务，因此可以直接将个人博客网站托管到Github Pages，它可以为个人网站提供免费二级域名供访问。它可以从 GitHub 上的仓库获取 HTML、CSS 和 JavaScript 文件，通过构建过程运行文件，然后发布网站"
excerpt:     "GitHub Pages 提供了一项静态网页托管服务，因此可以直接将个人博客网站托管到Github Pages，它可以为个人网站提供免费二级域名供访问。它可以从 GitHub 上的仓库获取 HTML、CSS 和 JavaScript 文件，通过构建过程运行文件，然后发布网站"
date:        2024-06-12T10:11:26+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/23/dc/sailing_ship_rushing_water_river_ship_vessel_sailboat_shore_transport-1082029.jpg!d"
published:   true
tags:
    - hugo
    - 网站建设
slug:        "hugo-build-website02"
categories:  [ "网站建设" ]
---

## 一、关于 Github Pages
GitHub Pages 是一项静态站点托管服务。它直接从 GitHub 上的仓库获取 HTML、CSS 和 JavaScript 文件，通过构建过程运行文件，然后发布网站。可以在 GitHub 的 github.io 域或自己的自定义域上托管站点。[更多Github Pages介绍](https://docs.github.com/zh/pages/getting-started-with-github-pages/about-github-pages)

### 1.1、创建 Github 账号和代码仓库
登录 [Github 官网首页](https://github.com/), 点击右上角的注册按钮，输入可用邮箱进行验证注册。账号注册完成后登录账号进行仓库创建。如图所示：填写仓库名称为 wang-fu-jie.github.io。 勾选上 Add a README file 复选框 (目的是为了自动创建 main 分支)， 其他保持默认，点击右下角的 Create repositoty 按钮
{{< rawhtml >}}<br><br><span style="font-weight: 700; color: red">注意：这里的仓库命名必须为{your_username}.github.io</span>{{< /rawhtml >}}
![图片加载失败](/post_images/{{< filename >}}/1.1-01.jpg)

然后进入的刚刚创建的仓库， 进入设置页面，点击设置页面的 Pages 按钮。 即可以看到个人的站点域名，如下所示：
![图片加载失败](/post_images/{{< filename >}}/1.1-02.jpg)
点击 Vist site 按钮即可访问，当前目前站点没有任何信息，因为我们还没有降个人博客做托管。 这里如果你不想使用 Github Pages 提供的默认域名，也可以自定义域名，自定义域名不再过多介绍，有兴趣可以参考官方手册

## 二、 托管个人博客站点
在上一篇博客中，已经介绍了如何通过 Hugo 搭建个人博客网站。如果你目前还没有个人 hugo 站点请参考上篇文章进行搭建， 现在我们需要把个人站点托管到 Github Pages，这样才可以在互联网的任何地方访问到我们的网站。

### 2.1、hugo站点和目标文件
在上篇博客中，我们通过一系列的操作在本地创建了站点源文件，写了第一篇博客。然后通过 hugo server 指令在本地发布了网站。可以看到，在执行了 hugo server 指令后，在本地生成了一个 public 文件夹。 这个 public 文件夹就是目标文件，即通过 hugo 指令生成了 HTML 内容，这个内容是可以通过浏览器进行渲染查看的。现在需要做的就是把 public 文件夹推送到刚刚创建的仓库

### 2.2、发布博客到 GitHub Pages
首先进入到 public 目录，执行以下指令
```shell
git init                               # 初始化新仓库
git add .                              # 将所有文件存放在 git 暂存区
git commit -m "Initial commit"         # 文件提交
# 链接到远程仓库
git remote add origin https://github.com/wang-fu-jie/wang-fu-jie.github.io.git  
git pull origin main                   # 拉取代码  
git push -u origin main                # 推送代码
```
这时再次访问 Github 提供的域名，就可以正常访问到个人网站
![图片加载失败](/post_images/{{< filename >}}/2.2-01.jpg)

## 三、 可能遇到的问题
1、国内部分用户如果访问 Github 比较慢， 可以选择国内的替代方案 Gitee Pages。但是目前官方公告：因服务维护调整，Gitee Pages 暂停提供服务，给您带来不便深感抱歉，感谢对 Gitee Pages 服务的支持。并未说永久下线，有需要可实时关注能否恢复访问

2、如按照本文操作存在其他问题欢迎留言
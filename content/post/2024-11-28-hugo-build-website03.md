---
title:       搭建个人博客网站之三、hugo站点自动部署到vercel托管平台
subtitle:    "hugo站点自动部署到vercel托管平台"
description: "Vercel是一个免费的网站托管平台，既可以托管静态网站也可以托管动态网站。Vercel可以关联GitHub，代码提交即可自动部署。并且提供免费域名和证书、无需担心证书到期问题。所以我们可以使用Vercel充当免费的服务器来托管我们通过hugo搭建的个人网站"
excerpt:     "Vercel是一个免费的网站托管平台，既可以托管静态网站也可以托管动态网站。Vercel可以关联GitHub，代码提交即可自动部署。并且提供免费域名和证书、无需担心证书到期问题。所以我们可以使用Vercel充当免费的服务器来托管我们通过hugo搭建的个人网站"
date:        2024-11-28T15:30:00+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/ee/64/landscape_valley_mist_misty_mountain_sunrise_view_rural-1096577.jpg!d"
published:   true
tags:
    - hugo
    - 网站建设
slug:        "hugo-build-website03"
categories:  [ "网站建设" ]
---

## 一、为什么使用 Vercel
看过上篇文章的小伙伴可能会有疑问，既然可以通过 GitHub Pages 托管我们的站点，为什么还要使用 Vercel。这是因为把网站部署到 GitHub Pages 有两个缺陷:

1. 在国内访问 GitHub 比较慢， 部分地区甚至打不开 GitHub
2. GitHub 禁止百度抓取网页，无法被百度收录，即不能通过搜索引擎搜到你的站点

基于以上两个缺陷，我们将站点托管到 Vercel

## 二、关于Vercel
Vercel 是一个面向现代开发者的全球化部署平台，旨在简化部署和托管现代应用程序的流程。Vercel非常适合个人开发者和小型团队使用，其个人版永久免费，每月提供100GB带宽，适合个人项目部署。此外，Vercel内置了持续集成/持续部署（CI/CD）功能，只需将项目导入Vercel，通过命令即可自动部署。
将个人站点部署到 Vercel，有两种部署方式:  

1. 源码托管，Vercel会自动编译成目标文件  
2. 目标文件托管，把Github Pages的文件直接导入到 Vercel

无论使用哪种方式，都需要注册 GitHub 和 Vercel 账号。第一种方式将源代码推送到 GitHub 的仓库即可。 如果使用第二种方式, 需要将编译后的文件先托管到 GitHub Pages。 托管到 GitHub Pages 的操作步骤请参考上一篇文档 [将hugo站点托管到Github Pages上](/2024/06/12/hugo-build-website02/)

### 2.1、创建 Vercel 账号
登录 [Vercel官网](https://vercel.com/), 我们目的是将 GitHub 上的站点托管到 Vercel。因此直接用 GitHub 登录 Vercel 是最方便的。如图所示:
![图片加载失败](/post_images/{{< filename >}}/2.1-01.jpg)

## 三、创建项目并导入仓库
第一步我们需要先创建一个项目
![图片加载失败](/post_images/{{< filename >}}/3.1-01.jpg)
导入 git 仓库，这里导入所有仓库即可
{{< rawhtml >}}<img src="/post_images/{{< filename >}}/3.1-02.jpg" alt="图片加载失败" width="60%" />{{< /rawhtml >}}

## 四、源码托管到 Vercel
这里我们首先介绍源码托管的方式，笔者本人比较推荐这种方式。因为每次将源码提交到 GitHub 后即可自动开始编译。省去了自己编译再托管到 GitHub Pages上的步骤

### 4.1、源码托管注意点
1、首先将源代码提交到 GitHub 上  
2、public目录不能提交，将 public 目录添加到 .gitignore 文件  

### 4.2、导入项目
选择你的源代码仓库，例如我的示例是 blog 仓库，然后点击 Import 按钮
![图片加载失败](/post_images/{{< filename >}}/4.2-01.jpg)

### 4.3、项目设置&部署
项目导入之后，Vercel会自动识别你的项目框架为 Hugo  
Build Command 填hugo  
Output directory填public  
环境变量填写 hugo 的版本号，可以通过命令 hugo version 查看。
{{< rawhtml >}}<img src="/post_images/{{< filename >}}/4.3-01.jpg" alt="图片加载失败" width="60%" />{{< /rawhtml >}}
然后点击 Deploy 即可发布项目

### 4.4、访问站点
在站点发布成功后，Vercel 会自动分配一个域名。在你的项目界面上可以看到，点击 visit 即可访问

## 五、编译文件部署
在上一章节中我们讲述了源码如何部署，这里再补充下编译文件如何部署。前提编译文件已托管到 GitHub Pages, 有疑问可以参考上篇博客 [将hugo站点托管到Github Pages上](/2024/06/12/hugo-build-website02/)  
和源码部署一样，首先我们需要导入项目。这里导入的需要是 wangfujie.github.io (示例名称)。  
FRAMEWORK PRESET 选Other (因为目标文件是没有框架的)  
点击 Deploy 进行部署  
部署成功就会弹出 Congratulations 界面，同样会生成一个域名访问即可  
上边做过了源码部署，编译文件部署比较简单，这里节省篇幅不再逐步贴图。有任何疑问请在文章下方留言


## 六、可能遇到的问题
1、无法访问 Vercel 网址，原因在国内会被墙，可以使用科学上网的方式

2、编译失败，可能原因主题如从github下载下来的，子目录里面会有 .git 文件，发布代码的话，内层目录包含.git的，它会识别成另一个git仓库并忽略上传，只上传一个软链接文件。需要删除themes主题下面的.git文件，否则 Vercel 无法从 github上拉取到主题。

3、如按照本文操作存在其他问题欢迎留言

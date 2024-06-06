---
title:      从零开始搭建个人博客网站(一)
subtitle:    "使用hugo创建个人博客网站"
description: "Hugo 是一个快速且灵活的静态网站生成器，它采用 Go 语言编写，专为构建博客、文档和其他静态网站而设计。它具有高性能、简洁灵活的模板系统、扩展性强、支持多语言、支持MarkDown等诸多优点。本文将演示如何使用hugo搭建一个个人博客网站"
excerpt: "Hugo 是一个快速且灵活的静态网站生成器，它采用 Go 语言编写，专为构建博客、文档和其他静态网站而设计。它具有高性能、简洁灵活的模板系统、扩展性强、支持多语言、支持MarkDown等诸多优点。本文将演示如何使用hugo搭建一个个人博客网站"
date:        2024-06-06T10:10:51+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/58/e8/snow_night_moon_cold_winter_trees_landscape_nature-1083144.jpg!d"
published:   true
tags: 
    - hogo
    - 网站建设
slug:        "hugo-build-website01"
categories:  [ "网站建设" ]
---

## 一、关于Hugo
Hugo 是一个快速且灵活的静态网站生成器，专为构建博客、文档和其他静态网站而设计。Hugo的运行不需要Python、PHP等高级语言以及数据库的依赖。它具备运行速度快、扩展性强、支持多语言、支持MarkDown等诸多优点。通过Hugo构建的网站非常安全和快速、并且可以托管在任何地方如 [Google Cloud Storage](https://cloud.google.com/storage/)、 [GitHub Pages](https://pages.github.com/)、 [Gitee Pages](https://gitee.com/help/articles/4136)、 [Vercel](https://vercel.com/)等

### 1.1、Hugo安装
Hugo支持在任何能运行go编译器的工具链上安装，如macOS、Windows、Linux、BSD等。且支持源码安装、预编译二进制文件安装、包管理器安装等多种安装方式。

#### 1.2、macOS安装
在macOS平台我们建议使用包管理器 Homebrew 安装, 这将直接安装 Hugo 的扩展版:
```shell
brew install hugo
```

#### 1.3、Windows 和 Linux安装
在 Windows 和 Linux 平台推荐使用预编译二进制文件安装Hugo。预编译二进制文件可用于多种操作系统和架构，请访问GitHub上发布的[最新版本](https://github.com/gohugoio/hugo/releases/latest)下载二进制文件，建议下载扩展版。然后按照以下步骤安装:
1. 根据个人的操作系统和架构下载对应版本
2. 解压缩二进制文件到目标目录
3. 添加目标目录到环境变量 PATH 中
4. 验证文件的执行权限
   
以上介绍了常用的操作系统和平台的最优安装方式，更多安装方式请参考[官方安装手册](https://gohugo.io/installation/)、 [中文版安装手册](https://hugo.opendocs.io/installation/)


### 二、网站构建
在完成 Hugo 安装后，即可通过命令来创建站点、添加内容、配置网站、发布网站等动作。

#### 2.1、创建网站
通过 Hugo 执行程序进行网站的创建，命令：
```shell
hugo new site mysite
```
执行完成会在当前目录生成一个名为 mysite 的文件夹。目录结构如下:
![图片加载失败](/post_images/{{< filename >}}/2.1-01.jpg)
<!-- {{< rawhtml >}}
<img src="/images/{{< filename >}}/2.1-01.jpg" alt="图片未加载" style="float: left; margin-right: 10px;">
<div style="clear: both;"></div>
{{< /rawhtml >}} -->

#### 2.2、添加主题
Hugo 官方给我们提供了很多漂亮的主题可以用来直接使用。可以在[官方主题地址](https://themes.gohugo.io/)选择喜欢的进行下载使用。这里以 CLean White 主题为例，点击[Clean White](https://themes.gohugo.io/themes/hugo-theme-cleanwhite/)进入官方主题页面，首页最上方有 Download 按钮，点击可以进入 CLean White 主题的GitHub地址。然后就可以通过命令下载 CLean White 主题。
```shell
cd themes/
git clone https://github.com/zhaohuabing/hugo-theme-cleanwhite.git
```
如上, 将主题下载到站点根目录的 themes 文件夹中即可

#### 2.3、网站配置
将主题提供的示例配置复制到根目录，并覆盖根目录下的 hugo.yaml 文件
```shell
cp themes/hugo-theme-cleanwhite/exampleSite/config.toml hugo.toml
```
修改基础配置，其他更多配置功能请根据实际需要进行调整
```
baseurl = "https://www.wangfujie.site/"
languageCode = "zh-cn"
title = "王富杰的博客"
```

#### 2.4、创建第一个博客
通过 hugo 名称创建第一个博客
```shell
hugo new post/my-first-blog.md
```
创建完成后，会在 content/post 目录生成一个 my-first-blog.md 文件。 可以在文件内随便写入内容，如 "这是我的第一篇博客"

#### 2.5、发布网站
执行 hugo server 命令
```shell
hugo server
```
完成后，根据提示在本地浏览器登录 http://localhost:1313/ 即可访问网站。
演示效果和本站基本一致

### 三、可能出现的问题
1、 下载了喜欢的主题按照本文无法正常启动，可能原因主题有其他依赖，每个主题的官方界面都会有提示如何使用示例。可以按照官方提示重新进行操作  
  
2、如按照本文操作存在其他问题欢迎留言


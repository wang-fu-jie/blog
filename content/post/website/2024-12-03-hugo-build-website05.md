---
title:       搭建个人博客网站之五、让搜索引擎收录个人网站
subtitle:    "让搜索引擎收录个人博客网站"
description: "个人网站部署完成之后，需要让搜索引擎来对我们的网站进行收录，这样他人才能够通过搜索引擎找到我们的站点，同时也利于我们网站的传播。目前主流的一些搜索引擎有谷歌、百度、必应、以及国内的搜狗，这篇文章就是讲述如何让这几大搜索引擎收录我们的网站"
excerpt:     "个人网站部署完成之后，需要让搜索引擎来对我们的网站进行收录，这样他人才能够通过搜索引擎找到我们的站点，同时也利于我们网站的传播。目前主流的一些搜索引擎有谷歌、百度、必应、以及国内的搜狗，这篇文章就是讲述如何让这几大搜索引擎收录我们的网站"
date:        2024-12-03T10:16:09+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/43/6d/clouds_summer_storm_clouds_form_dark_clouds_thunderstorm_summer_clouds_field-1414981.jpg!d"
published:   true
tags:
    - hugo
    - 网站建设
slug:        "hugo-build-website05"
categories:  [ "网站建设" ]
---

## 一、背景
在完成搭建个人博客网站之后，虽然已经可以在公网通过域名访问我们的博客网站，但是他人是不能通过搜索引擎找到我们的博客。那如何才能通过百度、谷歌等搜索引擎找到我们的博客呢，就需要在我们网站上添加搜索引擎的验证，这样搜索引擎就能定期爬取我们的网站，如果搜索引擎认为文章质量还不错，就会进行收录。目前全球比较流行的搜索引擎有谷歌、必应，在国内份额最高的就是百度了，接下来我们来让这三个搜索引擎来对我们的网站进行收录。

### 二、谷歌收录
#### 2.1、收录网站
登录[谷歌收录网站](https://search.google.com/search-console/welcome?utm_source=wmx&utm_medium=deprecation-pane&utm_content=home#utm_source=zh-CN-wmxmsg&utm_medium=wmxmsg&utm_campaign=bm&authuser=0)
然后我们自定义的域名[https://www.wangfujie.cn](https://www.wangfujie.cn)
![图片加载失败](/post_images/website/{{< filename >}}/2.1-01.png)

#### 2.2、网站验证
在上一步点击继续后，Google需要你验证网站的所有权，即验证这个网站是属于你的。比较简单的验证方式是使用html文件验证。按照提示下载html文件，将该html文件放在站点的static目录下，上传到 GitHub 进行验证即可

#### 2.3、配置站点地图
在网站验证成功后，需要配置我们的站点地图。这样谷歌才可以根据地图搜索我们网站的网页。站点地图一般叫做sitemap.xml, 每当我们的网站有更新站点地图就会更新，谷歌会通过定期的去爬取站点地图。
![图片加载失败](/post_images/website/{{< filename >}}/2.3-01.png)

#### 2.4、收录验证
在收录完成后，可以在谷歌搜索栏输入 site:wangfujie.cn 进行验证收录是否成功
![图片加载失败](/post_images/website/{{< filename >}}/2.4-01.png)

### 三、百度收录
#### 3.1、收录网站
首先进入[百度资源搜索平台](https://ziyuan.baidu.com/site/siteadd#/)
输入网站，点击下一步
![图片加载失败](/post_images/website/{{< filename >}}/3.1-01.png)

#### 3.2、网站验证
和谷歌收录一样，需要对网站的所有权进行验证。如图所示，下载html文件放到static文件夹下
![图片加载失败](/post_images/website/{{< filename >}}/3.2-01.png)

#### 3.3、提交站点
百度本来是和谷歌一样支持使用站点地图的，但是在2023年9月后百度限制了资源提交。目前只能手动提交或者通过API提交，需要整理出网站的URL手动粘贴提交即可。
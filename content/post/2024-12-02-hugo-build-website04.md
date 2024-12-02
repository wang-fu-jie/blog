---
title:       搭建个人博客网站之四、将个人网站绑定到自定义域名
subtitle:    "将Vercel托管的站点绑定到自定义域名"
description: "在将个人站点部署到Vercel后，可以申请一个自定义域名绑定到我们的站点。这样首先可以在中国大陆访问到我们网站，因为Vercel默认分配的二级域名中国大陆是被禁止访问的。其次有标示性的域名也更方便通过互联网找到我们的个人站点"
excerpt:     "在将个人站点部署到Vercel后，可以申请一个自定义域名绑定到我们的站点。这样首先可以在中国大陆访问到我们网站，因为Vercel默认分配的二级域名中国大陆是被禁止访问的。其次有标示性的域名也更方便通过互联网找到我们的个人站点"
date:        2024-12-02T11:47:44+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/ca/06/scotland_highlands_and_islands_landscape_highlands_mood_nature_clouds_mountains-499011.jpg!d"
published:   true
tags:
    - hugo
    - 网站建设
slug:        "hugo-build-website04"
categories:  [ "网站建设" ]
---

## 一、为什么绑定自定义域名
在讲个人网站部署到 Vercel 后，Vercel会自动分配一个二级域名给我们免费使用。这个二级域名存在两个缺陷，第一点因为中国网络的特殊原因，在国内无法访问到这个域名。第二是因为这个域名生成可能是一串随机码，没有任何标示性，不方便用于记忆。且不利于在互联网通过搜索引擎找到我们的网站。

## 二、申请个人域名
首先需要去域名注册商那里注册一个顶级域名，可以选择腾讯dnspod、 阿里云、 华为云等云服务厂商基本都提供域名注册服务。这里需要说明域名的注册是收费的，云厂商有活动一般情况第一年可以1元注册，但是续费时候费用就相对较高，一般一年几十块钱。有些国外的厂商也提供免费的域名，但是不建议使用，因为免费的域名不稳定随时可能被回收，一旦网站上线后更换域名会变得非常麻烦。我们这里以阿里云为例讲述下域名的注册流程

### 2.1、注册阿里云域名
首先我们需要注册一个阿里云平台的账号，这个步骤比较简单。因为是阿里是国内的服务商，使用手机号注册即可。进入到 [阿里云域名控制台](https://dc.console.aliyun.com/next/index?spm=5176.28197619.console-base_product-drawer-right.ddomain.3d5a1ad8hGMDFi#/overview)，搜索期望注册的域名，如 wangfujie:
![图片加载失败](/post_images/{{< filename >}}/2.1-01.jpeg)
如图所示：未被占用的域名会显示可注册。 点击右侧立即注册按照阿里云的提示操作即可，注册完成后回到域名控制台再域名列表即可看到我们成功注册的域名

## 三、将域名托管到 Cloudflare
通过国内的云服务商也可以对域名进行解析。这里笔者推荐使用 CLoudFlare，CloudFlare的免费套餐可以提供cdn服务。并且国内外都可以访问。首选我们注册一个 Cloudflare 账号，通过 [Cloudflare注册](https://dash.cloudflare.com/login)，笔者这里直接使用 Google 账号进行注册登录。

### 3.1、托管域名
登录进 CLoudflare 后，进入首页点击添加域
![图片加载失败](/post_images/{{< filename >}}/3.1-01.jpeg)
点击继续，进入到选择套餐界面。这里我们选择免费套餐即可，对于个人网站这足够了。点击继续，可以看到 Cloudflare 提供两个服务器地址。如图所示：
![图片加载失败](/post_images/{{< filename >}}/3.1-02.jpeg)
然后我们回到阿里云的域名控制台，点击我们注册成功的域名，进行域名详情页，修改DNS为 CLoudFlare 的服务器
![图片加载失败](/post_images/{{< filename >}}/3.1-03.jpeg)
至此，就完成了域名的托管

## 四、域名绑定
登录到 Vercel。进入个人网站的详情页，点击 Domains 进行域名添加。输入刚刚注册的域名进行添加即可，可以看到添加完成后，提示无效的配置，如下：
![图片加载失败](/post_images/{{< filename >}}/3.1-03.jpeg)
这是因为我们需要去域名添加记录进行域名解析，按照报错提示到 Cloudflare 添加记录即可，添加后的效果如图所示：
![图片加载失败](/post_images/{{< filename >}}/3.1-02.jpeg)
添加完成后，需要稍等待一会dns传播。可以在 Vercel 上查看，等 Vercel 上绑定的域名为可用状态后就成功了。这时就可以通过个人域名 [www.wangfujie.cn](www.wangfujie.cn) 来访问个人网站

## 五、其他说明
1、本文中粘贴的图片有的用的 www.wangfujie.site ，有的用的 www.wangfujie.cn。 这种由于笔者在写这篇文章时个人网站已经部署成功，无法进行中间过程的截图，因此采用了测试域名的图片
2、如果操作过程中有问题可以留言或文章底部添加笔者微信支持



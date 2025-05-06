---
title:       Redis源码阅读 - 搭建redis调试环境
subtitle:    "搭建redis调试环境"
description: "Redis源码阅读从比较早的redis1.0版本开始，低版本代码量少，易于进行打基础。为了一劳永逸，这里使用docker构建redis的调试镜像，并通过vscode远程调试。也可以根据文章中提供的Dockerfile自定义构建镜像进行redis调试。"
excerpt:     "edis源码阅读从比较早的redis1.0版本开始，低版本代码量少，易于进行打基础。为了一劳永逸，这里使用docker构建redis的调试镜像，并通过vscode远程调试。也可以根据文章中提供的Dockerfile自定义构建镜像进行redis调试。"
date:        2023-01-01T13:54:06+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/13/9f/hand_color_crayon_art_drawing-180149.jpg!d"
published:   true
tags:
    - Redis
slug:        "redis-debug-environment"
categories:  [ "REDIS" ]
---

## 一、源码编译redis
本文开始进行redis的源码学习，我们优先读redis1.0版本的代码，因为早期的代码行数较少更便于阅读，打下基础后，再读取不同的高版本代码学习版本的新增功能，为阅读最新的版本代码打下基础。

学习源码首先需要下载redis的源码进行编译，然后再进行调试。在不同的平台，基于不同的IDE有很多种调试方式。这里可以根据个人习惯，通过gdb进行编译调试，也可以使用vscode、clion、vs等工具进行调试。为了后续方便，我们选用docker+vscode进行调试，当然你可以直接下载笔者构建好的镜像，也可以根据本文的步骤自己构建镜像。

### 1.1、拉取redis_debug镜像
为了方便，我这里将已经构建的镜像上传到了阿里云镜像中心，读者可以自行拉起。当前前提需要安装docker，docker的安装步骤本文就不再细说，网上有大量的文章指导。我们这里直接进行镜像的拉取和容器的创建。
```shell
# 拉取镜像
docker pull registry.cn-beijing.aliyuncs.com/wangfujie-db/redis_debug:1.2.6

# 创建容器
docker run --privileged -it -d -p 2222:22 --name redis126 registry.cn-beijing.aliyuncs.com/wangfujie-db/redis_debug:1.2.6
```
注意，这里容器创建给了privileged权限是为了gdb可以调试。开启端口映射目的是使用vscode远程调试。这里构建的镜像默认是启动的sshd服务和bash，不会启动redis，如果想默认启动redis可以参考接下来的1.2章节修改Dockerfile自定义构建镜像。

### 1.2、构建redis_debug镜像
构建镜像需要自己写Dockerfile，这里贴出我自己构建使用的Dockerfile，当然你也可以进行魔改创建自己的镜像。Dockerfile镜像内容如下:
<details>
 <summary style="cursor: pointer; color: #3b82f6; text-decoration: underline;">
    点击查看Dockerfile文件内容
</summary>

```dockerfile
FROM ubuntu:24.04
LABEL version="1.2.6" description="Redis 1.2.6 debugging environment based on Ubuntu 24.04" auth="wangfujie" email="wfj914@163.com"

WORKDIR /opt

ARG REDIS_PACKAGE=redis-1.2.6.tar.gz

ENV REDIS_HOME=/opt/redis
ENV PATH=$REDIS_HOME/bin:$PATH

RUN apt update && apt install -y vim \
    make \
    gcc \
    gdb \
    openssh-server

ADD https://download.redis.io/releases/${REDIS_PACKAGE} /opt

RUN mkdir /opt/redis_src && \
    tar -zxvf /opt/${REDIS_PACKAGE} -C /opt/redis_src --strip-components=1 && \
    cd /opt/redis_src && \
    mkdir build && \
    cd build && \
    make -f ../Makefile VPATH=.. BUILD_DIR=. CFLAGS='-g -O0' && \
    cp ../redis.conf . && \
    rm /opt/${REDIS_PACKAGE} && \
    mkdir -p /var/run/sshd && \
    ssh-keygen -A && \
    mkdir -p /run/sshd && \
    chmod 0755 /run/sshd && \
    sed -i 's/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    echo "root:1" | chpasswd && \
    mkdir /opt/redis_src/.vscode


RUN cd /opt/redis_src/.vscode && \
    cat <<EOF > /opt/redis_src/.vscode/tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build Redis (debug + copy config)",
            "type": "shell",
            "command": "bash",
            "args": [
                "-c",
                "mkdir -p build; cd build; make -f ../Makefile V=1 VPATH=.. BUILD_DIR=. CFLAGS='-g -O0'; cp ../redis.conf ./"
            ],
            "options": {
                "cwd": "\${workspaceFolder}"
            },
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": [
                "$gcc"
            ]
        }
    ]
}
EOF

RUN cd /opt/redis_src/.vscode && \
    cat <<EOF > /opt/redis_src/.vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug redis-server",
            "type": "cppdbg",
            "request": "launch",
            "program": "\${workspaceFolder}/build/redis-server",
            "args": [
                "redis.conf"
            ],
            "stopAtEntry": true,
            "cwd": "\${workspaceFolder}/build",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        },
        {
            "name": "Debug redis-cli",
            "type": "cppdbg",
            "request": "launch",
            "program": "\${workspaceFolder}/build/redis-cli",
            "args": [
                "-h",
                "127.0.0.1",
                "-p",
                "6379"
            ],
            "stopAtEntry": true,
            "cwd": "\${workspaceFolder}/build",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
EOF

CMD [ "bash", "-c", "/usr/sbin/sshd && exec bash" ]
```
</details>
因为docker的文件内容很长，这里进行了折叠显示，你可以点击展开查看，当然你如果不想自己构建镜像，可以直接跳过该小节。

配置好了dockerfile，直接运行以下命令构建即可:
```python
docker build -t redis_debug_1.2.6 .
```

## 二、vscode远程调试
通过上一节，我们已经创建好了容器。接下来就进行vscode调试，首先自行进行vscode的安装。然后安装Remote-SSH 插件。接着，按 shift + command + p  (win 平台按Ctrl + Shift + p)通过 Remote 插件连接到远程的 Linux 服务器，并打开远程服务器。远程服务器打开容器的/opt/redis_src文件夹即可，笔者的容器已经编译好了redis存放到了这个路径。
![图片加载失败](/post_images/{{< filename >}}/2-01.png)
这里对文件夹的内容进行下简要说明：
* build文件夹是编译好的文件
* .vscode/launch.json是调试文件
* .vscode/tasks.json是任务文件，在魔改redis源码后进行俺shift+option+B可直接重新编译
* 其他的都是redis的源码文件

是不是很人性化，所有的配置都给你写好了呢。接下来，你只需要在vscode远程服务器上安装C/C++ Extension Pack这个插件，按一下F5就可以进行调试了呢。这里自动会在入口函数也就是main函数处断点。

## 三、gdb调试
如果你习惯使用gdb在终端调试，这里容器也给你准备好了呢。示例如下:
```shell
docker exec -it redis126 /bin/bash -c "gdb -args /opt/redis_src/build/redis-server /opt/redis_src/build/redis.conf"
```
只需这一条命令就可以开始调试了。这里附录一些gdb调试过程中常用的命令
```shell
-p 指定attahc 到指定 进程
-args 在命令行增加参数

r 启动程序
n 下一步
s 进入到函数
f 结束函数回到函数调用点

tui en 进入tui模式
tui dis 退出 tui 模式
focus cmd 方向键聚焦到命令行
focus src 方向键聚焦到代码ß
```
再次重申下，笔者这里的容器并未默认启动redis，应该这样方便调试redis的启动过程。当然你也可以微微修改dockerfile默认启动redis，再通过gdb attach到redis进程上进行调试，就看你自己的喜好了。

## 四、参考资料
[细说Redis源码](https://github.com/shencong1992/RedisNotes/blob/main/%E7%BB%86%E8%AF%B4Redis%E6%BA%90%E7%A0%81-%E7%AE%80%E5%8C%96%E7%89%88.pdf)
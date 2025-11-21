---
title:       自制操作系统 - 基于Ubuntu搭建开发环境
subtitle:    "基于Ubuntu搭建开发环境"
description: "开发操作系统首先需要搭建开发环境，基于不同的调试方式可以结合使用bochs和qemu两款虚拟软件。本文将分别介绍在Ubuntu系统上如何搭建和配置bochs和qemu，以及分别调试汇编和C程序。"
excerpt:     "开发操作系统首先需要搭建开发环境，基于不同的调试方式可以结合使用bochs和qemu两款虚拟软件。本文将分别介绍在Ubuntu系统上如何搭建和配置bochs和qemu，以及分别调试汇编和C程序。"
date:        2024-01-01T19:19:11+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/6f/db/owl_hunt_nature_hunter_predator_animal_world_animal_wildlife_photography-774736.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-build-envioonment"
categories:  [ "操作系统" ]
---

## 一、bochs和qemu介绍
Bochs是纯软件模拟的 x86 虚拟机，主要用于操作系统开发、教学和研究，高度可调试（支持单步执行、内存检查等），缺点是运行速度较慢。qemu是动态二进制翻译的快速模拟器，支持硬件加速，支持多种架构，但是调试能力不如Bochs。我们在开发操作系统过程中会结合这两种软件的优势进行调试测试。

## 二、Ubuntu 配置 bochs
笔者这里选择Ubuntu系统，当然也可以选择其他系统，如centos, redhat、mac等，因为centos开源版本已经不再维护，所以我们优先选择Ubuntu。可以安装虚拟机，或者使用ubuntu的桌面版容器。容器总是比较方便，但是容器不是完整的系统，开发过程会受到诸多限制。这里笔者推荐直接使用虚拟机。
```shell
# 如果使用的宿主机也是ubuntu，可以做如下一些初始化配置，方便开发使用。如果是其他linux发行版本，则可以根据实际情况调整

# 手动配置IP地址
IP 192.168.111.11  子网掩码 255.255.255.0  网关 192.168.111.2  DNS 8.8.8.8

# 安装并启动sshd 和 一些依赖包
sudo apt update && sudo apt install -y openssh-server vim nasm
systemctl start ssh

# 安装miniconda
sudo wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
sudo sh Miniconda3-latest-Linux-x86_64.sh
echo 'export PATH=$PATH:/opt/miniconda3/bin' | sudo tee -a /etc/profile
conda create -y -n wfj python=3.12
conda config --set changeps1 false
echo 'conda activate wfj' >> ~/.bashrc

# 配置sudo免密
echo "wfj ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/wfj

# 建议卸载snap，它会占用大量本地回环设备。
sudo apt autoremove --purge snapd
```
我们这里对ubuntu系统做了一系列初始化配置，都是开发过程中需要用到的或者方便调试目的。

### 2.1、安装bochs
在Ubuntu系统可以直接使用apt指令安装bochs。
```shell
sudo apt -y install bochs bochs-x
```
其中最重要的是 bochs-x 这个包，包括了 gui 插件。apt安装的bochs还存在一个问题，就是最新的bochs-bios不可用，这个问题好像持续了很久了，也有人给Ubuntu上报过[bug](https://bugs.launchpad.net/ubuntu/+source/bochs/+bug/2019531)，但是一直没有得到修复。这里我们自己来修复这个问题。第一种方式就是使用老版本的bios。在/usr/share/bochs/文件夹下存放了bios文件。
```shell
ls /usr/share/bochs/
BIOS-bochs-latest  BIOS-bochs-legacy  BIOS-qemu-latest  keymaps  VGABIOS-lgpl-latest
```
如上所示BIOS-bochs-latest为最新的bios，BIOS-bochs-legacy为老版本bios，在配置时使用BIOS-bochs-legacy可以正常运行。但是更推荐修复最新的bios。这里需要自行下载bochs的源码包，可以通过[sourceforge下载bochs](https://sourceforge.net/projects/bochs/files/bochs/)或者github下载。
```shell
sudo wget https://github.com/bochs-emu/Bochs/raw/REL_2_7_FINAL/bochs/bios/BIOS-bochs-latest
sudo mv /usr/share/bochs/BIOS-bochs-latest /usr/share/bochs/BIOS-bochs-latest-bak
sudo mv BIOS-bochs-latest /usr/share/bochs/BIOS-bochs-latest
```

### 2.2、配置bochs
在完成bochs安装后，执行命令：
```shell
bochs
```
* 选择 4. Save options to...
* 然后输入文件名 bochsrc 直接保存
* 选择 7. Quit now 退出 bochs
* 然后创建虚拟磁盘，输入命令
```shell
bximage -q -hd=16 -func=create -sectsize=512 -imgmode=flat master.img

## 命令执行结果
Creating hard disk image 'master.img' with CHS=32/16/63 (sector size = 512)

The following line should appear in your bochsrc:
  ata0-master: type=disk, path="master.img", mode=flat
```
这样就创建了一个16M的硬盘。根据提示，需要将上述最后一行写入到 bochsrc 文件中。
* 将 display_library: x 改成 display_library: x, options="gui_debug" 以支持 GUI 的调试方式。
* 将 boot: floppy 改成 boot: disk，以支持从硬盘启动。
* 将 magic_break: enabled=0 改成 magic_break: enabled=1，以支持bochs的魔术断点。说明在汇编中写 xrgs bx, bx。 bochs将自动在这里打断点

### 2.3、编写代码
配置完bochs之后，我们就可以编写代码进行开发调试了。代码如下：
```asm
[org 0x7c00]

; 设置屏幕模式为文本模式，并清除屏幕
mov ax, 3
int 0x10

; 初始化段寄存器
mov ax, 0
mov ds, ax
mov es, ax
mov ss, ax
mov sp, 0x7c00

; 文本显示器的内存区域
mov ax, 0xb800
mov ds, ax; 将代码段设置为 0xb800

mov byte [0], 'T'; 修改屏幕第一个字符为 T

; 下面阻塞停下来, $代表当前行
jmp $

times 510 - ($ - $$) db 0 ; 用 0 填充满 510 个字节
db 0x55, 0xaa; 主引导扇区最后两个字节必须是 0x55aa
```
以上为在屏幕上显示字母T的汇编代码，接下来需要把代码编译成二进制文件，使用nasm进行编译，并通过dd命令将二进制文件写入主引导扇区，命令如下：
```shell
nasm -f bin boot.asm -o boot.bin
dd if=boot.bin of=master.img bs=512 count=1 conv=notrunc
```
如果系统上没有 nasm 编译器，通过apt进行安装即可。最后执行以下命令就可进行调试：
```shell
bochs -q
```
运行结果如图所示：
![图片加载失败](/post_images/os/{{< filename >}}/2.3-01.png)

## 三、Ubuntu 配置 bochs-gdb
在第二章节中，我们安装了bochs，通过结果演示也可以知道，bochs是调试默认是反汇编的，也就是每次只能调试一行汇编代码。后续我们写操作系统时是用C语言编写，因为期望通过vscode对C进行调试，因此就需要安装bochs-gdb。以下安装步骤：

### 3.1、安装bochs-gdb
1、安装依赖
```shell
sudo apt install -y build-essential gcc-multilib libx11-dev libxrandr-dev libxpm-dev libgtk2.0-dev pkg-config libncurses-dev
```
2、下载源码并解压
```shell
sudo wget -O bochs-2.7.tar.gz http://downloads.sourceforge.net/sourceforge/bochs/bochs-2.7.tar.gz
sudo tar -xvf bochs-2.7.tar.gz
cd bochs-2.7
```
3、编译bochs-gdb
```shell
sudo sed -i 's/2\.6\*|3\.\*)/2.6*|3.*|4.*)/' configure*
sudo ./configure \
    --prefix=/usr/local \
    --without-wx \
    --with-x11 \
    --with-term \
    --disable-docbook \
    --enable-cpu-level=6 \
    --enable-fpu \
    --enable-3dnow \
    --enable-long-phy-address \
    --enable-pcidev \
    --enable-usb \
    --enable-all-optimizations \
    --enable-gdb-stub \
    --with-nogui

sudo sed -i 's/^LIBS = /LIBS = -lpthread/g' Makefile
sudo make -j1
```
4、安装bochs-gdb
```shell
sudo make install
sudo rm -f /usr/local/bin/bochs-gdb-a20 /usr/local/bin/bximage
sudo mv /usr/local/bin/bochs /usr/local/bin/bochs-gdb
```

### 3.2、配置bochs-gdb
首先需要生成bochs-gdb的配置文件。方式和bochs一样，唯一的区别需要添加一下内容
```
gdbstub: enabled=1, port=1234, text_base=0, data_base=0, bss_base=0
```
通过vscode进行内核调试，需要配置一个新的调试器，修改launch.json，添加内容如下：
```
{
            "name": "onix - Build and debug kernel",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/kernel.bin",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerServerAddress": "localhost:1234",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "Set Disassembly Flavor to Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
            "miDebuggerPath": "/usr/bin/gdb"
        }
```

### 3.3、启动调试
```
bochs-gdb -q -f ../bochs/bochsrc.gdb
```
先启动bochs-gdb，它会启动1234端口，然后vscode启动调试即可。这里我们还没有开始写内核的代码，因此无法进行演示，先放在这里即可。

## 四、Ubuntu 配置 qemu
qemu的安装使用比bochs会简单很多，我们直接使用apt安装即可：
```shell
sudo apt install qemu-system
```
安装完成后直接启动即可
```
qemu-system-i386 -m 32M -boot c -hda master.img
```
这个命令和bochs一样在屏幕打印字符T。如果通过vscode调试，需要多加一个参数：
```shell
qemu-system-i386 -s -S -m 32M -boot c -hda master.img
```
执行完后qemu一样启动了1234端口，vscode启动调试即可，和bochs使用一样的调试配置。

### 4.1、vmware启动
vmware使用的是vmdk格式的硬盘，通过qemu可以将硬盘转换为vmdk格式，这样就可以使用硬盘创建虚拟机了。
```
qemu-img convert -O vmdk master.img master.vmdk
```
使用vmware启动的过程这里不再进行演示，有兴趣的自行配置即可。

## 五、参考资料
[Onix操作系统实现](https://github.com/StevenBaby/onix/tree/dev)
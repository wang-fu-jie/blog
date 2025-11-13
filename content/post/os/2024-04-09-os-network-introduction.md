---
title:       自制操作系统 - 通信原理简介
subtitle:    "通信原理简介"
description: ""
excerpt:     ""
date:        2024-04-09T16:19:17+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/2b/da/forest_tree_path_fork_nature-8086.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-network-introduction"
categories:  [ "操作系统" ]
---

## 一、通信的基本原理
通信就是将 信息 从 信源 传输 到 信宿 的过程。以下为通信的几个概念：
* 信息：可以消除不确定性的东西；
* 信源：信息的来源；
* 信宿：信息的归宿；
* 信道：传输信号的媒介；
* 信号：信息的载体；
* 编码：将信息转换为二进制符号；
* 解码：将信号转换为信息；
* 调制：将符号转换为信号；
* 解调：将信号转换为符号；
* 噪声：信道中的不确定因素；
* 冗余：添加在信息中用于纠错或检错的内容

通信还依赖于物理设备，OSI模型中讲网络分为七层。通信介质包括网卡、中继器（重复信号，防止衰减）、集线器（除了接收线路以外，将信号发送到其他线路）、双绞线（网线）、同轴电缆（传输高频信号）、光纤、无线电。

## 二、以太网协议
以太网是一种通信协议、创建的其他通信协议还有串口、wifi、蓝牙等。以太网由四个部分组成：
* 帧：标准化的一组比特，用于在系统中传输数据；
* 媒体访问控制协议(Media Access Control protocol)：由一组嵌入在每个以太网接口中的规则组成，这些规则允许以太网站以半双工或全双工模式访问以太网信道；
* 信号组件：通过以太网信道发送和接收信号的标准化电子设备；
* 物理媒介：用于在连接到网络的计算机之间传输数字以太网信号的物理介质、电缆和其他硬件；

以太网栈结构如下：
![图片加载失败](/post_images/os/{{< filename >}}/2-01.png)
* 前导码：用于 10Mb/s 的硬件识别帧的开始，提示它开始接收数据，好比，设备清了清嗓子，开始说话了。新版的设备运行在更快的速度，实际已经无需前导码，但为了兼容性仍然保留了前导码。
* 目标地址/源地址：物理地址，也叫 MAC(Media Access Control) 地址，供应商为其生产的设备提供唯一的 MAC 地址；
* 类型/长度：大多数情况下，此字段用于识别上层协议的类型，一般是 (IP 0x0800),(ARP 0x0806)，如果值 <= 1500，则描述数据包的长度，若值 >= 1536(0x0600) 则表示类型，中间的空洞未定义；
* 数据：至少 需要 46 字节，最小长度保证了帧信号在网络上停留足够长的时间，使原始 10Mb/s 半双工系统中的每个以太网站点，都能在正确的时间限制内收到到帧。如果数据字段中携带的高级协议数据小于46 字节，则使用填充数据填充数据字段，一般会在后面补 0。
* CRC 校验和：帧的最后 4 字节是 帧校验序列(FCS Frame Check Sequence)，使用循环冗余校验码(CRC Cyclic Redundancy Check) 检测数据的完整性

以太网使用广播机制，意思是每个以太网帧会传输到网络中的所有设备，这看起来确实低效，但是优点是设备实现相当简单。目标设备只需要匹配地址，如果不是传输给自己的数据包就丢弃。这显然有安全隐患（ARP 欺骗，MAC 泛洪攻击），不过这是后话了，先让机器跑起来再说。

## 三、基础网络配置
测试网络需要一个新的网卡，以避免正常的通信中断，测试网卡修改网卡名称为 onix ，修改方法在vmware设置虚拟机添加网卡时有高级选项可以指定新网卡的mac地址，如 5a:5a:5a:5a:5a:20。
```
sudo ip link set ens37 down
sudo ip link set ens37 name onix
sudo ip link set onix up
```
如上为修改网卡名字的步骤，这属于临时生效，重启系统就会恢复原名称。因为笔者使用ubuntu系统，永久生效好像比较麻烦没有去验证。我们选择一个偷懒的方式，把以上命令编写一个脚本在crontab配置开机执行。
```
@reboot /usr/local/bin/rename_iface.sh
```
虚拟机默认已经有一个工作的网卡，这个网卡我们一般在使用中，例如vscode远程调试等，新建网卡避免相互影响。

### 3.1、测试网络拓扑
网络的拓扑如下图所示：
![图片加载失败](/post_images/os/{{< filename >}}/3.1-01.png)
* Host 表示 Windows 主机；即宿主机
* Gateway 是 VNet8 网卡用于虚拟机和主机之间以 NAT 通信；
* arch 表示 ubuntu 默认网卡；
* onix 表示 新添加的网卡；
* br0 是一个网桥 2，可以粗略的理解为虚拟交换机；
* tap0/tap1 是 tap 设备 3 4，可以用于应用程序操作二层数据包；经过配置普通用户也可以访问，无需 root 权限，方便调试；这两个设备是完全相同的，不过，一般情况下 tap0 用于 qemu，tap1 用于调试数据包的发送。

在makefile中通过以下配置来创建网络设备：
```makefile
IFACE:=onix

BR0:=/sys/class/net/br0
TAPS:=	/sys/class/net/tap0 \
		/sys/class/net/tap1 \
		/sys/class/net/tap2 \

.SECONDARY: $(TAPS) $(BR0)

BNAME:=br0     # 网桥 IP 地址
IP0:=192.168.111.22
MAC0:=5a:5a:5a:5a:5a:22
GATEWAY:=192.168.111.2

$(BR0):
	sudo ip link add $(BNAME) type bridge
	sudo ip link set $(BNAME) type bridge ageing_time 0

	sudo ip link set $(IFACE) down
	sudo ip link set $(IFACE) master $(BNAME)
	sudo ip link set $(IFACE) up

	sudo ip link set dev $(BNAME) address $(MAC0)
	sudo ip link set $(BNAME) up

	# sudo iptables -A FORWARD -i $(BNAME) -o $(BNAME) -j ACCEPT
	sudo iptables -F
	sudo iptables -X
	sudo iptables -P FORWARD ACCEPT
	sudo iptables -P INPUT ACCEPT
	sudo iptables -P OUTPUT ACCEPT

	sudo ip addr add $(IP0)/24 brd + dev $(BNAME) metric 10000
	sudo ip route add default via $(GATEWAY) dev $(BNAME) proto static metric 10000
	# sudo dhclient -v -4 br0

br0: $(BR0)
	-

/sys/class/net/tap%: $(BR0)
	sudo ip tuntap add mode tap $(notdir $@)
	sudo ip link set $(notdir $@) master $(BNAME)
	sudo ip link set dev $(notdir $@) up

tap%: /sys/class/net/tap%
	-
```

## 四、网络调试工具
Scapy 7 是一个 Python 程序，可以用于网络包的发送和嗅探。可以很容易的对其编程，以供调试网络之用。Wireshark 是最流行的网络数据包分析工具，可以通过Wireshark 进行抓包。

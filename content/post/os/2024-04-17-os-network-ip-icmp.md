---
title:       自制操作系统 - IP协议与ICMP协议
subtitle:    "IP协议与ICMP协议"
description: "IP是网络层协议，负责在网络中把数据从源主机传递到目标主机。ICMP是IP的上层协议，即传输层协议，ICMP协议一般多用于判断网络是否通常，常见于ping命令，本文将进行IP协议和ICMP协议的实现。"
excerpt:     "IP是网络层协议，负责在网络中把数据从源主机传递到目标主机。ICMP是IP的上层协议，即传输层协议，ICMP协议一般多用于判断网络是否通常，常见于ping命令，本文将进行IP协议和ICMP协议的实现。"
date:        2024-04-17T14:51:10+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/3d/49/spring_hazel_nature-746953.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-network-ip-icmp"
categories:  [ "操作系统" ]
---

## 一、IP协议
IP（Internet Protocol）是网络层协议，负责在网络中把数据从源主机传递到目标主机。它 不保证可靠性、不保证顺序、不保证不丢包，只是一个“最佳努力传输（best-effort）”协议。IP地址中有几类特殊地址：
* 私有地址
| 地址块 | 地址空间 |地址数量 |  
|------|----------|------|  
| 10.0.0.0/8	 | 10.0.0.0 ~ 10.255.255.255	 | 16777216 |  
| 172.16.0.0/12	 | 172.16.0.0 ~ 172.31.255.255	 | 1048576 |  
| 192.168.0.0/16 | 192.168.0.0 ~ 192.168.255.255	 | 65536 |  
* 受限广播地址 255.255.255.255
* 多播地址  多播允许单个主机同时通过互联网向数个主机发送数据流。它通常用于音频和视频流，例如基于 IP 的有线电视网络。
* 环回地址 127.0.0.0/8，一般是 127.0.0.1
* 本机地址 0.0.0.0
* 本网络地址 0.0.0.0/8

### 1.1、IP数据报
IP协议的报文结构如下所示：
<rawhtml>
<img src="/post_images/os/{{< filename >}}/1.1-01.png" alt="图片加载失败" width="600"/>
</rawhtml>
* 版本(Version)：4 bit，该值为 4，表示 IPv4；
* 头长度(Internet Header Length IHL)：4 bit，表示 IP 数据包头（黄色部分）的长度 / 4；如果没有选项，头长度为 20 / 4 = 5；所以 IP 头最长为 15 * 4 = 60；
* 服务类型 (Type of Service TOS)：8bit, 用于指示数据包的延迟，吞吐量、可靠性等；
* 总长度 (Total Length)：16bit，数据包的总长度，包括 IP头 和 数据区域；
* 标识 (Identification)：16bit，分片编号
* 标志(Flags)：3bit：
* 0 bit: 保留
* 1 bit: 0：可能分片，1：不分片
* 2 bit: 0：最后一个分片，1：不是最后一个分片
* 分片偏移 (Fragment Offset)：13bit，表示当前分片所携带的数据在整个 IP 数据报中的相对偏移位置；
* 生存时间(Time to Live TTL)：8bit，用于防止 IP 包出现在循环网络中，IP 数据包每经过一个路由器，该值减一，如果 TTL 为 0，则丢弃，同时可能返回一个 ICMP 消息，表示目标不可达；
* 上层协议: 8bit，1：ICMP 6：TCP 17：UDP
* 校验和：用于检测IP头是否出错，与数据区域无关；
* 选项：0 ~ 40 字节；如果没有对齐到 4 字节，最后需要填充 0 对齐到 3 字节，因为头长度的单位是 4 字节；

IP结构定义如下：
```cpp
typedef struct ip_t
{
    u8 header : 4;  // 头长度
    u8 version : 4; // 版本号
    u8 tos;         // type of service 服务类型
    u16 length;     // 数据包长度
    // 以下用于分片
    u16 id;         // 标识，每发送一个分片该值加 1
    u8 offset0 : 5; // 分片偏移量高 5 位，以 8字节 为单位
    u8 flags : 3;   // 标志位，1：保留，2：不分片，4：不是最后一个分片
    u8 offset1;     // 分片偏移量低 8 位，以 8字节 为单位
    u8 ttl;        // 生存时间 Time To Live，每经过一个路由器该值减一，到 0 则丢弃
    u8 proto;      // 上层协议，1：ICMP 6：TCP 17：UDP
    u16 chksum;    // 校验和
    ip_addr_t src; // 源 IP 地址
    ip_addr_t dst; // 目的 IP 地址
    u8 payload[0]; // 载荷
} _packed ip_t;
```

### 1.2、IP协议实现
IP数据包需要有上层协议，因此我们需先定义上层协议，如下：
```cpp
enum
{
    IP_PROTOCOL_NONE = 0,
    IP_PROTOCOL_ICMP = 1,
    IP_PROTOCOL_TCP = 6,
    IP_PROTOCOL_UDP = 17,
    IP_PROTOCOL_TEST = 254,
};
```
上层协议有ICMP、TCP、UDP。这些协议我们还没有实现，可以使用测试协议来进行IP的测试工作。接下来就是IP包的实现
```cpp
err_t ip_input(netif_t *netif, pbuf_t *pbuf)
{
    ip_t *ip = pbuf->eth->ip;
    if (ip->version != IP_VERSION_4)      // 只支持 IPv4
    {
        return -EPROTO;
    }
    if (!ip->ttl)     // ttl 耗尽
        return -ETIME;
    if (ip->header != sizeof(ip_t) / 4)     // 不支持选项
    {
        return -EOPTION;
    }
    if (!ip_addr_cmp(ip->dst, netif->ipaddr))    // 不是本机 ip
        return -EADDR;

    ip->length = ntohs(ip->length);
    ip->id = ntohs(ip->id);

    switch (ip->proto)    // 判断上层协议类型
    {
    case IP_PROTOCOL_TCP:
        LOGK("IP:TCP received\n");
        break;
    case IP_PROTOCOL_UDP:
        LOGK("IP:UDP received\n");
        break;
    case IP_PROTOCOL_ICMP:
        LOGK("IP:ICMP received\n");
        break;
    default:
        LOGK("IP {%X}: %r -> %r \n", ip->proto, ip->src, ip->dst);
        return -EPROTO;
    }
    return EOK;
}

err_t ip_output(netif_t *netif, pbuf_t *pbuf, ip_addr_t dst, u8 proto, u16 len)
{
    ip_t *ip = pbuf->eth->ip;

    ip->header = 5;
    ip->version = 4;
    ip->flags = 0;
    ip->offset0 = 0;
    ip->offset1 = 0;
    ip->tos = 0;
    ip->id = 0;
    ip->ttl = IP_TTL;
    ip->proto = proto;
    u16 length = sizeof(ip_t) + len;
    ip->length = htons(length);
    ip_addr_copy(ip->dst, dst);     // 一定要先对 dst 赋值，因为 dst 指针可能是下面的 src
    ip_addr_copy(ip->src, netif->ipaddr);

    ip->chksum = 0;
    ip->chksum = ip_chksum(ip, sizeof(ip_t));

    return arp_eth_output(netif, pbuf, ip->dst, ETH_TYPE_IP, length);
}

void ip_init()
{
}
```
如上是发送和接收IP包的实现，以太网帧接收到的包如何为IP包，就可以调用ip_input来进行处理了。

## 二、ICMP协议
ICMP (Internet Control Message Protocol) Internet 控制报文协议，用于在主机之间传递控制消息，比如测试网络连通性，主机是否可达，路由是否可用等，虽然没有传输信息，但是确实网络中很重要的协议。它的数据报文如下所示：
![图片加载失败](/post_images/os/{{< filename >}}/2-01.png)
ICMP数据包嵌在IP协议中，即IP数据包的载体可能是一个ICMP数据包。ICMP 类型有三种：0：Echo Replay，查询应答  3：Destination Unreachable Message，目标不可达  8：Echo，查询

以下为ICMP协议的实现：
```cpp
enum
{
    ICMP_ER = 0,   // Echo reply
    ICMP_DUR = 3,  // Destination unreachable
    ICMP_SQ = 4,   // Source quench
    ICMP_RD = 5,   // Redirect
    ICMP_ECHO = 8, // Echo
    ICMP_TE = 11,  // Time exceeded
    ICMP_PP = 12,  // Parameter problem
    ICMP_TS = 13,  // Timestamp
    ICMP_TSR = 14, // Timestamp reply
    ICMP_IRQ = 15, // Information request
    ICMP_IR = 16,  // Information reply
};

typedef struct icmp_echo_t
{
    u8 type;       // 类型
    u8 code;       // 状态码
    u16 chksum;    // 校验和
    u16 id;        // 标识
    u16 seq;       // 序号
    u8 payload[0]; // 可能是特殊的字符串
} _packed icmp_echo_t;

typedef struct icmp_t
{
    u8 type;       // 类型
    u8 code;       // 状态码
    u16 chksum;    // 校验和
    u32 RESERVED;  // 保留
    u8 payload[0]; // 载荷
} _packed icmp_t;


static err_t icmp_echo_reply(netif_t *netif, pbuf_t *pbuf)  // ICMP 查询响应
{
    ip_t *ip = pbuf->eth->ip;
    if (ip_addr_isbroadcast(ip->dst, netif->netmask) || ip_addr_ismulticast(ip->dst))
        return -EPROTO;
    if (!ip_addr_cmp(ip->dst, netif->ipaddr))
        return -EPROTO;

    icmp_t *icmp = ip->icmp;
    icmp->type = ICMP_ER;
    icmp->chksum = 0;
    u16 len = ip->length - sizeof(ip_t);
    icmp->chksum = ip_chksum(icmp, len);
    LOGK("IP ICMP ECHO REPLY: %r -> %r \n", ip->dst, ip->src);
    pbuf->count++;
    return ip_output(netif, pbuf, ip->src, IP_PROTOCOL_ICMP, len);
}

err_t icmp_input(netif_t *netif, pbuf_t *pbuf)   // icmp协议输入
{
    ip_t *ip = pbuf->eth->ip;
    icmp_t *icmp = ip->icmp;
    switch (icmp->type)
    {
    case ICMP_ER:    // 我们ping对方，对方给我们回复了，做一下打印。
        LOGK("IP ICMP REPLY: %r -> %r \n", ip->src, ip->dst);
        break;
    case ICMP_ECHO:           // 如果别人ping我们，就进行回复
        return icmp_echo_reply(netif, pbuf);
        break;
    default:
        LOGK("IP ICMP other: %r -> %r\n", ip->src, ip->dst);
        break;
    }
    return EOK;
}

err_t icmp_echo(netif_t *netif, pbuf_t *pbuf, ip_addr_t dst)    // ICMP 查询
{
    ip_t *ip = pbuf->eth->ip;
    icmp_echo_t *echo = ip->echo;
    echo->code = 0;
    echo->type = ICMP_ECHO;
    echo->id = 1;
    echo->seq = 1;
    char message[] = "Onix icmp echo 1234567890 asdfghjkl;'";
    strcpy(echo->payload, message);
    u32 len = sizeof(icmp_echo_t) + sizeof(message);
    echo->chksum = 0;
    echo->chksum = ip_chksum(echo, len);
    LOGK("IP ICMP ECHO: %r \n", dst);
    return ip_output(netif, pbuf, dst, IP_PROTOCOL_ICMP, len);
}

void icmp_init()
{
}
```
如上，icmp_echo是请求别人，例如ping其他主机。另外在接收以太网帧时，如果判断为ICMP协议，就可以调用icmp_input进行处理。来决定是响应还是打印，这个协议主要用于ping命令。


## 三、IP校验和
IP校验和的核心作用是：确保IP数据报头部在传输过程中没有发生错误。

发送方计算：
* 在发送IP数据报之前，发送方将IP头部的“校验和”字段置为0。
* 将整个IP头部（通常是20字节的固定部分，不含选项或包含选项）视为一系列16位（2字节）的字。
* 对这些16位的字进行二进制反码求和。
* 将求和结果的二进制反码 填入“校验和”字段。

（二进制反码求和的一个特性是：当将所有16位字，包括校验和本身，相加时，如果传输无误，结果应该是一个全1的二进制数，即16进制的0xFFFF）。

接收方验证：
* 接收方收到IP数据报后，同样将整个IP头部（这次包括发送方填写的校验和字段）视为一系列16位的字。
* 对这些字进行同样的二进制反码求和。
* 如果计算结果是全1（0xFFFF），则认为IP头部在传输过程中没有出错，数据报是有效的。

如果计算结果不是全1，则证明头部在传输中发生了错误，接收方会** silently discard（静默丢弃）** 这个数据包。它不会通知发送方，因为IP协议本身是无连接的、不可靠的。上层协议（如TCP）负责检测数据包的丢失并进行重传。
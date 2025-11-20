---
title:       自制操作系统 - 本地回环地址与UDP协议
subtitle:    "本地回环地址与UDP协议"
description: "回环地址为 127.0.0.1/8，表示本机地址，它是一个虚拟网卡设备，不经过真正的网卡，不发给 e1000。内核只需要“把包转一圈再返回协议栈即可。UDP协议是简单的数据报协议，目的是以最小的代价发送数据。"
excerpt:     "回环地址为 127.0.0.1/8，表示本机地址，它是一个虚拟网卡设备，不经过真正的网卡，不发给 e1000。内核只需要“把包转一圈再返回协议栈即可。UDP协议是简单的数据报协议，目的是以最小的代价发送数据。"
date:        2024-04-21T19:26:00+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/40/4a/sailor_man_delivery_bamboo_basketball_sea_red_sky-778654.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-network-localhost-udp"
categories:  [ "操作系统" ]
---

## 一、ping命令实现
ping命令使用 ICMP 回声包进行 IP 网络诊断和测量。实现如下：
```cpp
#define BUFLEN 2048
static char tx_buf[BUFLEN];
static char rx_buf[BUFLEN];

static int ping_input(ip_t *ip, ip_addr_t addr, size_t bytes)
{
    if (ip->proto != IP_PROTOCOL_ICMP)
        return EOK;
    icmp_echo_t *echo = ip->echo;
    printf("%d bytes from %r: icmp_seq=%d ttl=%d icmp=%d\n",
           bytes, ip->src, echo->seq, ip->ttl, echo->type);
    return EOK;
}

int main(int argc, char const *argv[])
{
    // 省略了错误检查
    fd_t fd = socket(AF_INET, SOCK_RAW, PROTO_ICMP);  // 创建一个原始套接字
    int opt = 2000;
    int ret = setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, &opt, 4);   // 设置超时时间2秒
    ip_t *ip = (ip_t *)tx_buf;
    ip_addr_copy(ip->dst, addr);
    ip->proto = IP_PROTOCOL_ICMP;   // 设置IP包上层协议为ICMP

    icmp_echo_t *echo = ip->echo;
    echo->code = 0;
    echo->type = ICMP_ECHO;
    echo->id = 1;
    echo->seq = 0;

    char message[] = "Onix icmp echo 1234567890 asdfghjkl;'";
    strcpy(echo->payload, message);
    u32 len = sizeof(icmp_echo_t) + sizeof(message);
    int count = 4;
    while (count--)  // 连续ping四次
    {
        echo->seq += 1;
        echo->chksum = 0;
        echo->chksum = ip_chksum(echo, len);
        ret = send(fd, tx_buf, len + sizeof(ip_t), 0);   // 发送ip包
        ret = recv(fd, rx_buf, sizeof(rx_buf), 0);       // 接收响应
        ret = ping_input((ip_t *)rx_buf, addr, ret);     // 处理响应进行打印
        sleep(1000);
    }
}
```
如上所示，我们实现了ping命令，启动操作系统后可以ping目标地址了。


## 二、本地回环地址
回环地址为 127.0.0.1/8，表示本机地址；/8 意思是说 127.x.x.x，都是本机地址，和 127.0.0.1 是一样的，当然，这个设计浪费了很多地址。localhost 是一个主机名，用来表示本机，一般 localhost 会被域名解析为 127.0.0.1。
```cpp
static void send_packet(netif_t *netif, pbuf_t *pbuf)
{
    ip_addr_t addr;
    ip_addr_copy(addr, pbuf->eth->ip->dst);        // 保存原始 dst 地址
    ip_addr_copy(pbuf->eth->ip->dst, pbuf->eth->ip->src);  // 把 dst 改成 src（即回给发送者）
    ip_addr_copy(pbuf->eth->ip->src, addr);  // 把 src 改成原来的 dst
    netif_input(netif, pbuf);  // 将处理后的 pbuf 直接送回协议栈
}

void loopif_init()
{
    loopif = netif_create();   // 创建一个虚拟网卡
    loopif->nic_output = send_packet;
    strcpy(loopif->name, "loopback");
    assert(inet_aton("127.0.0.1", loopif->ipaddr) == EOK);   // 设置ip和网关
    assert(inet_aton("255.0.0.0", loopif->netmask) == EOK);

    assert(eth_addr_isany(loopif->hwaddr));   // 本地回环没有mac地址，设置为0
    assert(ip_addr_isany(loopif->gateway));

    loopif->flags = (NETIF_LOOPBACK |
                     NETIF_IP_RX_CHECKSUM_OFFLOAD |
                     NETIF_IP_TX_CHECKSUM_OFFLOAD |
                     NETIF_UDP_RX_CHECKSUM_OFFLOAD |
                     NETIF_UDP_TX_CHECKSUM_OFFLOAD |
                     NETIF_TCP_RX_CHECKSUM_OFFLOAD |
                     NETIF_TCP_TX_CHECKSUM_OFFLOAD);
}
```
loopback 不经过真正的网卡，不发给 e1000。内核只需要“把包转一圈再返回协议栈即可。为了模拟正常网络行为：回包必须交换 src/dst 地址。


## 三、UDP协议
UDP(User Datagram Protocol) 用户数据报协议，目的是以最小的代价发送数据。因此它的数据报非常的简单：
![图片加载失败](/post_images/os/{{< filename >}}/3-01.png)
进入UDP协议后，就开始引入了端口号的概念。操作系统也需要记录哪些端口号被占用，使用位图结构进行管理。
```cpp
void port_init(port_map_t *map)
{
    map->buf = alloc_kpage(2);
    bitmap_init(&map->map, (char *)map->buf, PAGE_SIZE * 2, 0);
}
```
申请两页内存进行端口号的管理，因此一共支持65536个端口号。在端口号获取和释放直接操作位图即可。以下是UDP的实现：
```cpp
typedef struct udp_pcb_t udp_pcb_t;

typedef struct udp_t
{
    u16 sport;     // 源端口号
    u16 dport;     // 目的端口号
    u16 length;    // 长度
    u16 chksum;    // 校验和
    u8 payload[0]; // 载荷
} _packed udp_t;

typedef struct udp_pcb_t
{
    list_node_t node; // 链表结点

    ip_addr_t laddr; // 本地地址
    ip_addr_t raddr; // 远程地址
    u16 lport;       // 本地端口号
    u16 rport;       // 远程端口号

    u32 flags; // 状态

    list_t rx_pbuf_list;      // 接收缓冲队列
    struct task_t *rx_waiter; // 等待进程
} udp_pcb_t;

static int udp_socket(socket_t *s, int domain, int type, int protocol)
{
    LOGK("udp socket...\n");
    assert(!s->udp);
    s->udp = kmalloc(sizeof(udp_pcb_t));
    memset(s->udp, 0, sizeof(udp_pcb_t));

    udp_pcb_t *pcb = s->udp;
    list_init(&pcb->rx_pbuf_list);
    list_push(&udp_pcb_list, &pcb->node);
    return EOK;
}

static int udp_output(netif_t *netif, udp_pcb_t *pcb, pbuf_t *pbuf, ip_addr_t dst, u16 dport, u16 len)
{
    if (!dst)
        dst = pcb->raddr;
    if (ip_addr_isany(dst))
        return -EADDR;
    if (dport == 0)
        dport = pcb->rport;
    if (dport == 0)
        return -ENOTCONN;

    if (ip_addr_isbroadcast(dst, netif->netmask) && !(pcb->flags & UDP_FLAG_BROADCAST))
        return -EACCES;

    ip_t *ip = pbuf->eth->ip;
    udp_t *udp = ip->udp;

    udp->sport = htons(pcb->lport);
    udp->dport = htons(dport);

    ip_addr_t *src;
    if (ip_addr_isany(pcb->laddr))
        src = &netif->ipaddr;
    else
        src = &pcb->laddr;

    u16 length = len + sizeof(udp_t);
    udp->length = htons(length);
    udp->chksum = 0;

    if (!(netif->flags & NETIF_UDP_TX_CHECKSUM_OFFLOAD))
    {
        udp->chksum = inet_chksum(udp, length, *src, pcb->raddr, IP_PROTOCOL_UDP);
    }

    return ip_output(netif, pbuf, pcb->raddr, IP_PROTOCOL_UDP, length);
}
```
以上为UDP的部分示例， 注意 udp_connect() 做了以下事情：记录服务器 IP (raddr)，记录服务器端口 (rport)，标记 socket 已连接 (UDP_FLAG_CONNECTED)，若未设置本地端口，自动分配一个临时端口 (lport)
返回成功，不建立任何真实的连接（UDP 无连接）。
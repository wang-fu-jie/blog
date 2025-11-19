---
title:       自制操作系统 - 以太网协议与ARP协议
subtitle:    "以太网协议与ARP协议"
description: "有了虚拟网络设备后，就可以进行以太网协议的实现了。以太网接收到以太网帧后，需要根据以太网帧载体携带内容来判断如何解包，例如如果携带的是ARP协议，就需要进行ARP的请求或回复。本文将同时进行ARP协议的实现。"
excerpt:     "有了虚拟网络设备后，就可以进行以太网协议的实现了。以太网接收到以太网帧后，需要根据以太网帧载体携带内容来判断如何解包，例如如果携带的是ARP协议，就需要进行ARP的请求或回复。本文将同时进行ARP协议的实现。"
date:        2024-04-15T17:49:03+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/be/43/kitchen_furniture_interior_cook_house_eat_modern_kitchen-507251.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-network-arp"
categories:  [ "操作系统" ]
---

## 一、以太网协议实现
前面我们已经实现了物理网卡驱动、虚拟网络设备，这里我们继续实现以太网协议，当前网络协议栈如下：
<rawhtml>
<img src="/post_images/os/{{< filename >}}/1-01.png" alt="图片加载失败" width="500"/>
</rawhtml>
网卡收到包时会交给虚拟网卡，虚拟网卡来处理网络包， 网络包处理的过程中首先会交给以太网来处理。以太网协议的实现如下:

```cpp
enum
{
    ETH_TYPE_IP = 0x0800,   // IPV4 协议
    ETH_TYPE_ARP = 0x0806,  // ARP 协议
    ETH_TYPE_IPV6 = 0x86DD, // IPV6 协议
};

err_t eth_input(netif_t *netif, pbuf_t *pbuf)    // 接收以太网帧
{
    eth_t *eth = (eth_t *)pbuf->eth;
    u16 type = ntohs(eth->type);

    switch (type)
    {
    case ETH_TYPE_IP:
        LOGK("ETH %m -> %m IP4, %d\n", eth->src, eth->dst, pbuf->length);
        break;
    case ETH_TYPE_IPV6:
        LOGK("ETH %m -> %m IP6, %d\n", eth->src, eth->dst, pbuf->length);
        break;
    case ETH_TYPE_ARP:
        LOGK("ETH %m -> %m ARP, %d\n", eth->src, eth->dst, pbuf->length);
        break;
    default:
        LOGK("ETH %m -> %m UNKNOWN [%04X], %d\n", type, eth->src, eth->dst, pbuf->length);
        return -EPROTO;
    }
    return EOK;
}

err_t eth_output(netif_t *netif, pbuf_t *pbuf, eth_addr_t dst, u16 type, u32 len)   // 发送以太网帧
{
    pbuf->eth->type = htons(type);
    eth_addr_copy(pbuf->eth->dst, dst);
    eth_addr_copy(pbuf->eth->src, netif->hwaddr);
    pbuf->length = sizeof(eth_t) + len;
    netif_output(netif, pbuf);
    return EOK;
}

// 初始化以太网协议
void eth_init()
{
}
```
以太网帧的类型，我们只会用到IPV4和ARP协议，TPV6用不到，这里定义了以太网帧的接收和发送，在接收，当前只进行了打印操作。在虚拟网卡的接收任务进程中，就可以修改原来的打印操作为 eth_input。意思即虚拟网卡接收到网络包交给以太网协议进行处理。 在发送时依然调用虚拟网卡的发送函数即可。


## 二、ARP协议
ARP(Address Resolution Protocol) 协议用于解析局域网中 IP 地址对应的 MAC 地址。另外，还有 RARP(Reverse Address Resolution Protocol) 协议用于解析 MAC 地址对应的 IP 地址。以下是ARP的报文格式，它被嵌在以太网的数据包中：
<rawhtml>
<img src="/post_images/os/{{< filename >}}/2-01.png" alt="图片加载失败" width="700"/>
</rawhtml>
其中 硬件类型只有 1 表示以太网。 协议类型 0x0800 表示 IPv4。 操作类型有两种，分别是 1：请求、2：回复。 ARP协议的结构如下：

```cpp
typedef struct arp_t
{
    u16 hwtype;       // 硬件类型
    u16 proto;        // 协议类型
    u8 hwlen;         // 硬件地址长度
    u8 protolen;      // 协议地址长度
    u16 opcode;       // 操作类型
    eth_addr_t hwsrc; // 源 MAC 地址
    ip_addr_t ipsrc;  // 源 IP 地址
    eth_addr_t hwdst; // 目的 MAC 地址
    ip_addr_t ipdst;  // 目的 IP 地址
} _packed arp_t;
```
ARP 协议放在 网络层 不太合适，更多的认为它是 数据链路层 的协议，这是有争议的，但这种分类意义不大，它更像是网络层和数据链路层的胶水，使两层能够联系起来。

### 2.1、ARP协议实现
ARP协议分为请求和回复，当收到请求时，如果不是找自己的直接丢弃数据包，如果是找自己的则进行回复，代码如下：
```cpp
static err_t arp_reply(netif_t *netif, pbuf_t *pbuf)      // ARP应答
{
    arp_t *arp = pbuf->eth->arp;
    LOGK("ARP Request from %r\n", arp->ipsrc);

    arp->opcode = htons(ARP_OP_REPLY);    // 修改下目的和源的ip和mac，做一下颠倒
    eth_addr_copy(arp->hwdst, arp->hwsrc);   
    ip_addr_copy(arp->ipdst, arp->ipsrc);
    eth_addr_copy(arp->hwsrc, netif->hwaddr);
    ip_addr_copy(arp->ipsrc, netif->ipaddr);

    pbuf->count++;
    return eth_output(netif, pbuf, arp->hwdst, ETH_TYPE_ARP, sizeof(arp_t));  // 利用虚拟网卡的发包函数，将响应包发出去。
}

err_t arp_input(netif_t *netif, pbuf_t *pbuf)
{
    arp_t *arp = pbuf->eth->arp;
    if (ntohs(arp->hwtype) != ARP_HARDWARE_ETH)   // 只支持 以太网
        return -EPROTO;
    if (ntohs(arp->proto) != ARP_PROTOCOL_IP)      // 只支持 IP
        return -EPROTO;
    if (!ip_addr_cmp(netif->ipaddr, arp->ipdst))    // 如果请求的目的地址不是本机 ip 则忽略
        return -EPROTO;

    u16 type = ntohs(arp->opcode);
    switch (type)
    {
    case ARP_OP_REQUEST:       // 如果是ARP请求，即有人问我们，就进行响应
        return arp_reply(netif, pbuf);
    case ARP_OP_REPLY:        // 如果是别人响应的我们的询问，就打印一下日志。
        LOGK("arp reply %r -> %m\n", arp->ipsrc, arp->hwsrc);
        break;
    default:
        return -EPROTO;
    }
    return EOK;
}

void arp_init()
{
}
```
如上，虚拟网卡接收到的以太网帧判断属于 ARP 协议时，就可以调用arp_input来进行ARP请求的处理。


### 2.2、ARP缓存
在通过IP发送数据包时，需要ARP协议获取到IP和MAC地址的映射关系。但是不能每次发送数据包都通过ARP协议，因此需要做一下ARP缓存。 如果IP在子网中，直接通过以太网进行发送即可。如果不在一个子网中，就需要发送给网关。
```cpp
#define ARP_ENTRY_TIMEOUT 600  // ARP 缓冲失效时间秒
#define ARP_RETRY 5            // ARP 请求重试次数
#define ARP_DELAY 2            // ARP 请求延迟秒
#define ARP_REFRESH_DELAY 1000 // ARP 刷新间隔毫秒

static list_t arp_entry_list;   // ARP 缓存队列
static task_t *arp_task;        // ARP 刷新任务
typedef struct arp_entry_t      // arp 缓存
{
    list_node_t node;  // 链表结点
    eth_addr_t hwaddr; // MAC 地址
    ip_addr_t ipaddr;  // IP 地址
    u32 expires;       // 失效时间
    u32 used;          // 使用次数
    u32 query;         // 查询时间
    u32 retry;         // 重试次数
    list_t pbuf_list;  // 等待队列
    netif_t *netif;    // 虚拟网卡
} arp_entry_t;

static arp_entry_t *arp_entry_get(netif_t *netif, ip_addr_t addr)      // 获取 ARP 缓存
{
    arp_entry_t *entry = (arp_entry_t *)kmalloc(sizeof(arp_entry_t));   // 申请内存
    entry->netif = netif;
    ip_addr_copy(entry->ipaddr, addr);                 // 拷贝ip给 arp_entry_t 结构
    eth_addr_copy(entry->hwaddr, ETH_BROADCAST);       // 拷贝广播地址，即ff:ff:ff:ff:ff:ff
    entry->expires = 0;
    entry->retry = 0;
    entry->used = 1;
    list_init(&entry->pbuf_list);   // 初始化等待队列
    list_insert_sort(             // 将arp_entry_t进行按照失效时间插入排序到链表中
        &arp_entry_list,
        &entry->node,
        element_node_offset(arp_entry_t, node, expires));
    return entry;
}

static void arp_entry_put(arp_entry_t *entry)     // 释放 ARP 缓存
{
    list_t *list = &entry->pbuf_list;      // 获取ARP的等待队列 
    while (!list_empty(list))             // 循环判断，直到等待队列为空
    {
        pbuf_t *pbuf = element_entry(pbuf_t, node, list_popback(list));
        assert(pbuf->count == 1);
        pbuf_put(pbuf);
    }
    list_remove(&entry->node);   // 移除节点并释放内存
    kfree(entry);
}

static arp_entry_t *arp_lookup(netif_t *netif, ip_addr_t addr)   // 查找ARP，辅助函数
{
    ip_addr_t query;
    if (!ip_addr_maskcmp(netif->ipaddr, addr, netif->netmask))  // 如果虚拟网卡的ip 和 传入的ip是否在同一个子网
        ip_addr_copy(query, netif->gateway);  // 如果不在，将虚拟网卡的网关拷贝给query
    else
        ip_addr_copy(query, addr);   // 如果在，将ip直接赋给要查询的query

    list_t *list = &arp_entry_list;
    arp_entry_t *entry = NULL;
    for (list_node_t *node = list->head.next; node != &list->tail; node = node->next)   // 遍历ARP缓存链表
    {
        entry = element_entry(arp_entry_t, node, node);
        if (ip_addr_cmp(entry->ipaddr, query) && (netif == entry->netif))  // 如果找到缓存，就返回
        {
            return entry;
        }
    }
    entry = arp_entry_get(netif, query);    // 如果没找到，就申请一个新的
    return entry;
}

static err_t arp_query(arp_entry_t *entry)     // ARP 查询
{
    if (entry->query + ARP_DELAY > time())    // 判断ARP缓存是否已经失效
    {
        return -ETIME;
    }
    entry->query = time();   // 更新查询时间
    entry->retry++;      // 重试次数+1
   
    if (entry->retry > 2)   // 如果重试次数大于2， 就拷贝广播地址为改ARP的mac地址
    {
        eth_addr_copy(entry->hwaddr, ETH_BROADCAST);
    }

    pbuf_t *pbuf = pbuf_get();
    arp_t *arp = pbuf->eth->arp;

    arp->opcode = htons(ARP_OP_REQUEST);   // 设置ARP包头字段 操作吗
    arp->hwtype = htons(ARP_HARDWARE_ETH);
    arp->hwlen = ARP_HARDWARE_ETH_LEN;
    arp->proto = htons(ARP_PROTOCOL_IP);
    arp->protolen = ARP_PROTOCOL_IP_LEN;

    eth_addr_copy(arp->hwdst, entry->hwaddr);   // 填充地址
    ip_addr_copy(arp->ipdst, entry->ipaddr);
    ip_addr_copy(arp->ipsrc, entry->netif->ipaddr);
    eth_addr_copy(arp->hwsrc, entry->netif->hwaddr);

    eth_output(entry->netif, pbuf, entry->hwaddr, ETH_TYPE_ARP, sizeof(arp_t));   // 发送arp请求
    return EOK;
}

static err_t arp_refresh(netif_t *netif, pbuf_t *pbuf)      // 刷新 ARP 缓存
{
    arp_t *arp = pbuf->eth->arp;
    if (!ip_addr_maskcmp(arp->ipsrc, netif->ipaddr, netif->netmask))  // 判断是否在同一个子网
        return -EADDR;
    arp_entry_t *entry = arp_lookup(netif, arp->ipsrc);      // 查找ARP缓存

    eth_addr_copy(entry->hwaddr, arp->hwsrc);
    entry->expires = time() + ARP_ENTRY_TIMEOUT;   // 更新arp缓存
    entry->retry = 0;
    entry->used = 0;

    list_remove(&entry->node);  // 移除原来的ARP，，并将刷新后的进程插入
    list_insert_sort(
        &arp_entry_list,
        &entry->node,
        element_node_offset(arp_entry_t, node, expires));

    list_t *list = &entry->pbuf_list;
    while (!list_empty(list))
    {
        pbuf_t *pbuf = element_entry(pbuf_t, node, list_popback(list));
        eth_addr_copy(pbuf->eth->dst, entry->hwaddr);
        netif_output(netif, pbuf);
    }
    return EOK;
}
```
如上所示，ARP缓存使用链表结构进行存储。每次访问IP时，通过查询ARP缓存，即可找到IP对应的MAC地址

## 三、ARP缓存的应用
在有了ARP协议后，发送数据包就可以通过IP来进行发送了，不能再直接通过MAC地址直接发送数据包。另外我们需要一个后台线程，每隔一段时间定期进行ARP缓存刷新。
```cpp
err_t arp_eth_output(netif_t *netif, pbuf_t *pbuf, ip_addr_t addr, u16 type, u32 len)  // 发送数据包到对应的 IP 地址
{
    pbuf->eth->type = htons(type);      // 设置以太网类型字段
    eth_addr_copy(pbuf->eth->src, netif->hwaddr);   // 设置源MAC地址（本机网卡地址）
    pbuf->length = sizeof(eth_t) + len;       // 设置数据包总长度
    arp_entry_t *entry = arp_lookup(netif, addr);  // 查找ARP缓存
    if (entry->expires > time())            // 如果ARP缓存有效   
    {
        entry->used += 1;
        eth_addr_copy(pbuf->eth->dst, entry->hwaddr);   // 拷贝目标MAC地址，并直接发送数据包
        netif_output(netif, pbuf);
        return EOK;
    }

    list_push(&entry->pbuf_list, &pbuf->node);   // 如果缓存失效，将数据包加入等待队列。 发送ARP查询请求
    arp_query(entry); 
    return EOK;
}


static void arp_thread()   // ARP 刷新线程
{
    list_t *list = &arp_entry_list;
    while (true)
    {
        task_sleep(ARP_REFRESH_DELAY);   // 死循环，每次休眠指定时间
        for (list_node_t *node = list->tail.prev; node != &list->head;)
        {
            arp_entry_t *entry = element_entry(arp_entry_t, node, node);
            node = node->prev;
            if (entry->expires > time())
                continue;
            if (entry->retry > ARP_RETRY)
            {
                LOGK("ARP retries %d times for %r...\n",
                     entry->retry, entry->ipaddr);
                arp_entry_put(entry);
                continue;
            }
            if (entry->used == 0)
            {
                LOGK("ARP entry %r timeout...\n", entry->ipaddr);
                arp_entry_put(entry);
                continue;
            }
            arp_query(entry);
        }
    }
}

void arp_init()      // 初始化 ARP 协议
{
    LOGK("Address Resolution Protocol init...\n");
    list_init(&arp_entry_list);
    arp_task = task_create(arp_thread, "arp", 5, KERNEL_USER);
}
```

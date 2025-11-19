---
title:       自制操作系统 - 网络协议与虚拟网络设备
subtitle:    "网络协议与虚拟网络设备"
description: "虚拟网络设备指的是 netif_t， 它表示一个网络接口（Network Interface）的抽象结构体，用来表示一个网卡设备，无论是真实的。通过虚拟网络设备，可以实现本地回环地址。"
excerpt:     "虚拟网络设备指的是 netif_t， 它表示一个网络接口（Network Interface）的抽象结构体，用来表示一个网卡设备，无论是真实的。通过虚拟网络设备，可以实现本地回环地址。"
date:        2024-04-13T17:41:10+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/3e/86/country_guitar_girl_music_instrument_musical_acoustic_musician-692362.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-network-device"
categories:  [ "操作系统" ]
---

## 一、CRC校验
CRC(Cyclic Redundancy Check) 循环冗余校验码 1 常用于数字网络和存储设备的检错，来保证数据的完整性。以太网数据包的最后四个字节被认为是帧校验序列(Frame Check Sequence FCS)，通过对前面所有数据的计算得到一个 4 字节的值，发送前写入 FCS 字段，接收方收到后重新进行计算，如果计算结果不匹配（数据计算结果不为零）则认为数据在传输的过程中出错，如果出错则丢弃数据包；计算方法如下：
```cpp
#define CRC_POLY 0xEDB88320

u32 eth_fcs(void *data, int len)
{
    u32 crc = -1;
    u8 *ptr = (u8 *)data;
    for (int i = 0; i < len; i++)
    {
        crc ^= ptr[i];
        for (int j = 0; j < 8; j++)
        {
            if (crc & 1)
                crc = (crc >> 1) ^ CRC_POLY;
            else
                crc >>= 1;
        }
    }
    return ~crc;
}
```
在进行收发包时通过进程校验和的比对，就可以判断包在传输过程中出现错误。


## 二、虚拟网络设备
虚拟网络设备指的是 netif_t， 它表示一个网络接口（Network Interface）的抽象结构体，用来表示一个网卡设备，无论是真实的。通过虚拟网络设备，可以实现本地回环地址。实现如下：
```cpp
typedef struct netif_t
{
    list_node_t node; // 链表节点
    char name[16];    // 名字
    list_t rx_pbuf_list; // 接收缓冲队列
    list_t tx_pbuf_list; // 发送缓冲队列
    eth_addr_t hwaddr; // MAC 地址
    ip_addr_t ipaddr;  // IP 地址
    ip_addr_t netmask; // 子网掩码
    ip_addr_t gateway; // 默认网关
    void *nic; // 设备指针
    void (*nic_output)(struct netif_t *netif, pbuf_t *pbuf);
} netif_t;

static list_t netif_list; // 虚拟网卡列表
static task_t *neti_task; // 接收任务
static task_t *neto_task; // 发送任务

netif_t *netif_setup(void *nic, eth_addr_t hwaddr, void *output)   // 初始化虚拟网卡
{
    netif_t *netif = kmalloc(sizeof(netif_t));   // 申请内存
    list_push(&netif_list, &netif->node);        // 加入链表
    list_init(&netif->rx_pbuf_list);             // 初始化接收缓冲队列
    list_init(&netif->tx_pbuf_list);             // 初始化发送缓冲队列
    eth_addr_copy(netif->hwaddr, hwaddr);       // 拷贝mac地址
    netif->nic = nic;                           // 设备指针
    netif->nic_output = output;                 // 发包函数
    return netif;
}

netif_t *netif_get()    // 获取虚拟网卡
{
    list_t *list = &netif_list;  // 从虚拟网卡链表中获取
    if (list_empty(list))
        return NULL;
    list_node_t *ptr = list->head.next;
    netif_t *netif = element_entry(netif_t, node, ptr);
    return netif;
}

void netif_remove(netif_t *netif)  // 移除虚拟网卡
{
    list_remove(&netif->node);
    kfree(netif);
}

void netif_input(netif_t *netif, pbuf_t *pbuf)   // 网卡接收任务输入
{
    list_push(&netif->rx_pbuf_list, &pbuf->node);
    if (neti_task->state == TASK_WAITING)
    {
        task_unblock(neti_task, EOK);
    }
}

void netif_output(netif_t *netif, pbuf_t *pbuf)   // 网卡发送任务输出
{
    list_push(&netif->tx_pbuf_list, &pbuf->node);
    if (neto_task->state == TASK_WAITING)
    {
        task_unblock(neto_task, EOK);
    }
}

void netif_init()      // 初始化虚拟网卡
{
    list_init(&netif_list);   // 初始化虚拟网卡列表，创建两个进程分别用于接收任务和发送任务
    neti_task = task_create(neti_thread, "neti", 5, KERNEL_USER);
    neto_task = task_create(neto_thread, "neio", 5, KERNEL_USER);
}
```
如上所示，虚拟网卡使用列表进行管理。例如e1000是一个物理网卡，在e1000初始化时也需要通过虚拟网络设备进行管理，即进行虚拟设备初始化。在虚拟设备初始化时，我们创建了两个任务进行测试，接下来看测试代码。

## 三、虚拟网络设备测试
```cpp
e1000->netif = netif_setup(e1000, e1000->mac, send_packet);    // e1000 物理网卡的初始化过程，会初始化出一个虚拟网卡

static void neti_thread()   // 接收任务
{
    list_t *list = &netif_list;
    pbuf_t *pbuf;
    netif_t *netif;
    while (true)
    {
        int count = 0;
        for (list_node_t *ptr = list->head.next; ptr != &list->tail; ptr = ptr->next)  // 遍历虚拟网卡列表
        {
            netif = element_entry(netif_t, node, ptr);    // 获取到虚拟网卡结构

            if (!list_empty(&netif->rx_pbuf_list))    // 如果接收缓冲队列不为空
            {
                pbuf = element_entry(pbuf_t, node, list_popback(&netif->rx_pbuf_list));  // 获取接收缓冲队列
                LOGK("ETH RECV [%04X]: %m -> %m %d\n",ntohs(pbuf->eth->type),pbuf->eth->src,pbuf->eth->dst,pbuf->length);
                pbuf_put(pbuf);
                count++;
            }
        }

        if (!count)     // 没有数据包，阻塞处理线程
        {
            task_t *task = running_task();
            int ret = task_block(task, NULL, TASK_WAITING, TIMELESS);
        }
    }
}

static void neto_thread()        // 发送任务
{
    list_t *list = &netif_list;
    pbuf_t *pbuf;
    netif_t *netif;
    while (true)
    {
        int count = 0;
        for (list_node_t *ptr = list->head.next; ptr != &list->tail; ptr = ptr->next)
        {
            netif = element_entry(netif_t, node, ptr);
            if (!list_empty(&netif->tx_pbuf_list))
            {
                pbuf = element_entry(pbuf_t, node, list_popback(&netif->tx_pbuf_list));
                netif->nic_output(netif, pbuf);
                LOGK("ETH RECV [%04X]: %m -> %m %d\n",ntohs(pbuf->eth->type),pbuf->eth->src,pbuf->eth->dst,pbuf->length);
                count++;
            }
        }

        if (!count)     // 没有数据包，阻塞处理线程
        {
            task_t *task = running_task();
            int ret = task_block(task, NULL, TASK_WAITING, TIMELESS);
            assert(ret == EOK);
        }
    }
}
```
如上所示，在初始化时，创建了两个任务，分别循环遍历发送缓冲区和接收缓冲区，如果有数据，就进行相应的发送和接收。
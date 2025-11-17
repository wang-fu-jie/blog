---
title:      自制操作系统 - 网卡驱动
subtitle:    "网卡驱动"
description: "需要完成网卡驱动，才能使系统能够接收和发送数据包。e1000网卡也属于PCI设备。数据包在内存间进行拷贝非常消耗性能，因为还需要实现数据包的高速缓冲。"
excerpt:     "需要完成网卡驱动，才能使系统能够接收和发送数据包。e1000网卡也属于PCI设备。数据包在内存间进行拷贝非常消耗性能，因为还需要实现数据包的高速缓冲。"
date:        2024-04-11T11:07:14+08:00
author:      "王富杰"
image:       "/img/home-bg-jeep.jpg"
published:   true
tags:
    - 操作系统
slug:        "os-network-driver"
categories:  [ "操作系统" ]
---

## 一、e1000 网卡驱动
首先需要完成网卡驱动，才能使系统能够接收和发送数据包。e1000网卡也属于PCI设备。我们这里直接根据代码来看网卡的工作流程
```cpp
typedef struct eth_t   // 以太网帧
{
    eth_addr_t dst; // 目标地址
    eth_addr_t src; // 源地址
    u16 type;       // 类型
    u8 payload[0];  // 载荷
} _packed eth_t;

#define ETH_ADDR_LEN 6
#define IP_ADDR_LEN 4
typedef u8 eth_addr_t[ETH_ADDR_LEN]; // MAC 地址
typedef u8 ip_addr_t[IP_ADDR_LEN];   // IPV4 地址
```
以上为以太网一个帧格式的定义，接下来看网卡驱动：
```cpp
#define RX_DESC_NR 32 // 接收描述符数量
#define TX_DESC_NR 32 // 传输描述符数量

typedef struct rx_desc_t        // 接收描述符
{
    u64 addr;     // 数据缓冲器地址
    u16 length;   // 长度
    u16 checksum; // 校验和
    u8 status;    // 状态
    u8 error;     // 错误
    u16 special;  // 特殊
} _packed rx_desc_t;

typedef struct tx_desc_t      // 传输描述符
{
    u64 addr;    // 缓冲区地址
    u16 length;  // 包长度
    u8 cso;      // Checksum Offset
    u8 cmd;      // 命令
    u8 status;   // 状态
    u8 css;      // Checksum Start Field
    u16 special; // 特殊
} _packed tx_desc_t;

#define NAME_LEN 16

typedef struct e1000_t
{
    char name[NAME_LEN]; // 名称

    pci_device_t *device; // PCI 设备
    u32 membase;          // 映射内存基地址
    u8 mac[6];   // MAC 地址
    bool link;   // 网络连接状态
    bool eeprom; // 只读存储器可用
    rx_desc_t *rx_desc; // 接收描述符
    u16 rx_cur;         // 接收描述符指针
    tx_desc_t *tx_desc; // 传输描述符
    u16 tx_cur;         // 传输描述符指针
    task_t *tx_waiter;  // 传输等待进程
} e1000_t;

static e1000_t obj;

static void recv_packet(e1000_t *e1000)   // 接收数据包
{
    while (true)
    {
        rx_desc_t *rx = &e1000->rx_desc[e1000->rx_cur];   // 接收描述符
        
        // TODO RECEIVE PACKET
        eth_t *eth = (eth_t *)(u32)(rx->addr & 0xffffffff);   // 获取以太网帧
        LOGK("ETH R [0x%04X]: %m -> %m, %d\n",
             ntohs(eth->type),
             eth->src,
             eth->dst,
             rx->length);

        rx->status = 0;
        moutl(e1000->membase + E1000_RDT, e1000->rx_cur);
        e1000->rx_cur = (e1000->rx_cur + 1) % RX_DESC_NR;
    }
}

void send_packet(eth_t *eth, u16 len)   // 发送数据包
{
    e1000_t *e1000 = &obj; 
    tx_desc_t *tx = &e1000->tx_desc[e1000->tx_cur];  // 发送描述符
    while (tx->status == 0)
    {
        assert(e1000->tx_waiter == NULL);
        e1000->tx_waiter = running_task();
        assert(task_block(e1000->tx_waiter, NULL, TASK_BLOCKED, TIMELESS) == EOK);
    }

    memcpy((void *)(u32)tx->addr, eth, len);
    tx->length = len;
    tx->cmd = TCMD_EOP | TCMD_RS | TCMD_RPS | TCMD_IFCS;
    tx->status = 0;
    e1000->tx_cur = (e1000->tx_cur + 1) % TX_DESC_NR;
    moutl(e1000->membase + E1000_TDT, e1000->tx_cur);
    LOGK("ETH S [0x%04X]: %m -> %m, %d\n",
         ntohs(eth->type),
         eth->src,
         eth->dst,
         len);
}
```
我们这里只粘贴了发送数据包和接收数据包的实现，省略了大部分辅助函数和网卡的初始化。在网卡初始化过程中，首先要查找PCI设备是否有网卡，找到后设置物理内存映射区域和网卡中断处理函数。

接收数据包我们这里写了死循环，它不断从网卡的接收描述符环（RX Descriptor Ring）里取出已经接收到的数据包，然后打印以太网帧的信息。它的职责是：不停地检查网卡的接收描述符队列，看看有没有新包进来。网卡每次接收到包，会触发中断，中断服务例程（ISR）调用 recv_packet()，让它把所有当前队列中已完成的包都取完。

在现代网络中，只有发给自己或广播/多播的帧才会被送到网卡端口，网卡芯片内部有一套地址过滤机制（Address Filtering），比如 Intel e1000 就有这样的硬件逻辑：它会检查每个接收到的以太网帧头部（frame header）：接收广播帧和发给自己MAC地址的帧。如下为日志打印的包结果：
```
[kernel/e1000.c] [263] ETH R [0x0806]: 00:50:56:f2:39:f2 -> ff:ff:ff:ff:ff:ff, 60
[kernel/e1000.c] [263] ETH R [0x0806]: 5a:5a:5a:5a:5a:22 -> 00:50:56:f2:39:f2, 60
[kernel/e1000.c] [263] ETH R [0x0800]: 5a:5a:5a:5a:5a:22 -> 00:50:56:f2:39:f2, 74
[kernel/e1000.c] [263] ETH R [0x0800]: 5a:5a:5a:5a:5a:22 -> 00:50:56:f2:39:f2, 60
[kernel/e1000.c] [263] ETH R [0x0800]: 5a:5a:5a:5a:5a:22 -> 00:50:56:f2:39:f2, 142
[kernel/e1000.c] [263] ETH R [0x0800]: 5a:5a:5a:5a:5a:22 -> 00:50:56:f2:39:f2, 60
[kernel/e1000.c] [263] ETH R [0x0806]: 00:50:56:f2:39:f2 -> ff:ff:ff:ff:ff:ff, 60
[kernel/e1000.c] [263] ETH R [0x0806]: 5a:5a:5a:5a:5a:22 -> 00:50:56:f2:39:f2, 60
[kernel/e1000.c] [263] ETH R [0x0806]: 5a:5a:5a:5a:5a:22 -> 3e:22:fb:78:3d:65, 60
```
如图所示，mac地址22是网桥br0， 3e:22:fb:78:3d:65为宿主机的网卡MAC。我们的网卡可以接受到广播包和局域网内的包。最后我们理一下数据接收流程：
1. 网线收到电/光信号 → 网卡 PHY 解码成比特流 → 网卡 MAC 层组装成以太网帧。
2. 网卡检测到帧目标地址是自己（或者广播/多播） → 决定接收。
3. 网卡通过 DMA把数据包写入 RX 描述符的 addr 指向的缓冲区。
4. 网卡在 RX 描述符里更新： length：写入的数据长度  status：标记数据已经接收完成  checksum / error 等硬件信息
5. CPU 驱动检查描述符的状态位，知道数据包已经到内存，可以处理。

## 二、发送数据包测试
首先需要再makefile中配置qemu网卡，
```
QEMU+= -netdev tap,id=eth0,ifname=tap0,script=no,downscript=no # 网络设备
QEMU+= -device e1000,netdev=eth0,mac=5A:5A:5A:5A:5A:33 # 网卡 e1000
```
指定使用 TAP 网络接口 作为虚拟机的网络后端。给这个网络后端分配一个 ID名字为 eth0。绑定宿主机上已有的 TAP 设备 tap0，也就是说，虚拟机网卡通过 tap0 与宿主机通信。 以下为发包的测试程序：
```cpp
// 发送测试数据包
void test_e1000_send_packet()
{
    e1000_t *e1000 = &obj;    // 获取到网卡
    eth_t *eth = (eth_t *)alloc_kpage(1);   // 申请一页内存，存放以太网帧
    memcpy(eth->src, e1000->mac, 6);        // 将以太网卡的mac地址赋为源
    memcpy(eth->dst, "\xff\xff\xff\xff\xff\xff", 6);  // 将目的 MAC 设置为广播地址 FF:FF:FF:FF:FF:FF。发送到这个地址意味着局域网内所有主机都会接收到这个帧。
    eth->type = 0x0090; // LOOP 0x9000
    int len = 1500;
    memset(eth->payload, 'A', len);
    send_packet(eth, len + sizeof(eth_t));   // 发送数据包A
    free_kpage((u32)eth, 1);
}
```
这个函数是发送一个以太网测试数据包，通过 e1000 网卡驱动发送。
![图片加载失败](/post_images/os/{{< filename >}}/3-01.png)
我们在宿主机上进行抓包，可以看到，网卡发送的广播包会被发给局域网内的所有地址。


## 三、数据包高速缓冲
数据包高速缓冲是一种缓存机制，尽可能避免调用 memcpy 函数。如图所示：
![图片加载失败](/post_images/os/{{< filename >}}/4-01.png)
缓存内容是以太网帧，这里有一个概念叫MTU，MTU为1500表示最大传输单元，单位是字节。如下为缓冲的实现：
```cpp
typedef struct pbuf_t
{
    list_node_t node; // 列表节点
    size_t length;    // 载荷长度
    u32 count;        // 引用计数
    union
    {
        u8 payload[0]; // 载荷
        eth_t eth[0];  // 以太网帧
    };
} pbuf_t;

static list_t free_buf_list;
static size_t pbuf_count = 0;

pbuf_t *pbuf_get()      // 获取空闲缓冲
{
    pbuf_t *pbuf = NULL;
    if (list_empty(&free_buf_list))   // 如果链表为空就申请一页内存
    {
        u32 page = alloc_kpage(1);   
        pbuf = (pbuf_t *)page;
        list_push(&free_buf_list, &pbuf->node);

        page += PAGE_SIZE / 2;   //  一页内存分为两部分，各2048字节，加入到空闲链表
        pbuf = (pbuf_t *)page;
        list_push(&free_buf_list, &pbuf->node);

        pbuf_count += 2;
        LOGK("pbuf count %d\n", pbuf_count);
    }
    pbuf = element_entry(pbuf_t, node, list_popback(&free_buf_list));
    assert(((u32)pbuf & 0x7ff) == 0);       // 应该对齐到 2K

    pbuf->count = 1;
    return pbuf;
}

void pbuf_put(pbuf_t *pbuf)  // 释放缓冲
{
    
    assert(((u32)pbuf & 0x7ff) == 0);   // 应该对齐到 2K

    assert(pbuf->count > 0);
    pbuf->count--;
    if (pbuf->count > 0)
    {
        assert(pbuf->node.next && pbuf->node.prev);
        return;
    }

    list_push(&free_buf_list, &pbuf->node);
}

void pbuf_init()  // 初始化数据包缓冲, 即初始化链表
{
    list_init(&free_buf_list);
}
```
如此，在网卡驱动的实现中就可以申请数据包缓冲区，这个具体实现我们不做过多分析。又了数据包缓冲后，流程就发生了些变化，接收流程：
* 从 pbuf_get() 获取一个 pbuf
* 把 pbuf->payload 的物理地址填入 RX 描述符 (addr)
* 网卡 DMA 把收到的数据写到 pbuf→payload
* 网卡写完更新描述符状态
* CPU 从 pbuf→payload 读数据并传递给协议栈
* 协议栈处理完后 pbuf_put() 回收

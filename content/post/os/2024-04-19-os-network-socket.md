---
title:       自制操作系统 - 套接字
subtitle:    "套接字"
description: "在一切皆文件的理念中，套接字也属于文件系统。我们需要支持新的文件系统，因此需要虚拟文件系统统一管理。套接字是一套统一的编程接口，提供给用户态进行网络编程。在本文我们将实现数据包套接字和原始套接字，它们分别可用于接收发送以太网帧和IP数据局包。"
excerpt:     "在一切皆文件的理念中，套接字也属于文件系统。我们需要支持新的文件系统，因此需要虚拟文件系统统一管理。套接字是一套统一的编程接口，提供给用户态进行网络编程。在本文我们将实现数据包套接字和原始套接字，它们分别可用于接收发送以太网帧和IP数据局包。"
date:        2025-11-19T16:58:58+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/3a/fb/person_sunset_woman_girl_beach-180159.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-network-socket"
categories:  [ "操作系统" ]
---

## 一、虚拟文件系统
在一切皆文件的理念中，套接字也属于文件系统。我们需要支持新的文件系统，因此需要虚拟文件系统统一管理。虚拟文件系统即是把文件系统中重要的系统调用抽象出来，对每种文件系统单独做处理：
```cpp
typedef struct fs_op_t
{
    int (*mkfs)(dev_t dev, int args);

    int (*super)(dev_t dev, super_t *super);

    int (*open)(inode_t *dir, char *name, int flags, int mode, inode_t **result);
    void (*close)(inode_t *inode);

    int (*read)(inode_t *inode, char *data, int len, off_t offset);
    int (*write)(inode_t *inode, char *data, int len, off_t offset);
    int (*truncate)(inode_t *inode);

    int (*stat)(inode_t *inode, stat_t *stat);
    int (*permission)(inode_t *inode, int mask);

    int (*namei)(inode_t *dir, char *name, char **next, inode_t **result);
    int (*mkdir)(inode_t *dir, char *name, int mode);
    int (*rmdir)(inode_t *dir, char *name);
    int (*link)(inode_t *odir, char *oldname, inode_t *ndir, char *newname);
    int (*unlink)(inode_t *dir, char *name);
    int (*mknod)(inode_t *dir, char *name, int mode, int dev);
    int (*readdir)(inode_t *inode, dentry_t *entry, size_t count, off_t offset);
} fs_op_t;

fs_op_t *fs_ops[FS_TYPE_NUM];
```
当前我们已有minix和管道两个文件系统，这里不做具体的展示，例如minix定义函这个函数后，封装为fs_op_t结构，并注册到fs_ops中。即可通过fs_op_t对minix文件系统进行一系列的操作。

## 二、套接字
伯克利套接字(Berkeley sockets) 最初随着 4.2BSD(Berkeley Software Distribution) UNIX 操作系统一同发布，作为一种编程接口。目前大多数其他编程语言都提供类似的接口，通常以基于 C API 的包装器库的形式编写。

### 2.1、套接字协议组定义
套接字作为对用户态提供的网络编程接口，定义了许多协议组，因为它不仅要支持TCP协议，也要支持UDP协议，以及更多其他协议。
```cpp
enum
{
    AF_UNSPEC = 0,   // 未定义
    AF_PACKET,       // 数据包
    AF_INET,         // IPV4
};

enum
{
    SOCK_STREAM = 1,   // 数据流
    SOCK_DGRAM = 2,    // 数据报
    SOCK_RAW = 3,      // 原始套接字
};

enum
{
    PROTO_IP = 0,  
    PROTO_ICMP = 1,
    PROTO_TCP = 6,
    PROTO_UDP = 17,
};

typedef enum socktype_t   // 套接字的类型
{
    SOCK_TYPE_NONE = 0,
    SOCK_TYPE_PKT,    // 数据包
    SOCK_TYPE_RAW,    // 原始套接字
    SOCK_TYPE_TCP,
    SOCK_TYPE_UDP,
    SOCK_TYPE_NUM,
} socktype_t;

typedef struct sockaddr_t   // 套接字地址
{
    u16 family;
    char data[14];
} sockaddr_t;

typedef struct sockaddr_in_t
{
    u16 family;
    u16 port;
    ip_addr_t addr;
    u8 zero[8];
} sockaddr_in_t;

typedef struct iovec_t   // 输入输出向量，实质是缓冲区
{
    size_t size;
    void *base;
} iovec_t;

typedef struct msghdr_t
{
    sockaddr_t *name;
    int namelen;
    iovec_t *iov;
    int iovlen;
} msghdr_t;

typedef struct socket_t   // socket描述符
{
    socktype_t type; // socket 类型
} socket_t;

typedef struct socket_op_t   // 封装为虚拟操作系统
{
    int (*socket)(socket_t *s, int domain, int type, int protocol);
    int (*close)(socket_t *s);
    int (*listen)(socket_t *s, int backlog);
    int (*accept)(socket_t *s, sockaddr_t *addr, int *addrlen, socket_t **ns);
    int (*bind)(socket_t *s, const sockaddr_t *name, int namelen);
    int (*connect)(socket_t *s, const sockaddr_t *name, int namelen);
    int (*shutdown)(socket_t *s, int how);

    int (*getpeername)(socket_t *s, sockaddr_t *name, int *namelen);
    int (*getsockname)(socket_t *s, sockaddr_t *name, int *namelen);
    int (*getsockopt)(socket_t *s, int level, int optname, void *optval, int *optlen);
    int (*setsockopt)(socket_t *s, int level, int optname, const void *optval, int optlen);

    int (*recvmsg)(socket_t *s, msghdr_t *msg, u32 flags);
    int (*sendmsg)(socket_t *s, msghdr_t *msg, u32 flags);
} socket_op_t;
```


### 2.2、套接字的实现
套接字也作为一个虚拟操作系统进行实现，需要定义open，close等函数。
```cpp
#define BACKLOG 5

static socket_op_t *socket_ops[SOCK_TYPE_NUM];

static inode_t *socket_create()
{
    inode_t *inode = get_free_inode();
    // 区别于 EOF 这里是无效的设备，但是被占用了
    inode->dev = -FS_TYPE_SOCKET;
    inode->desc = NULL;
    inode->count = 1;
    inode->type = FS_TYPE_SOCKET;
    inode->op = fs_get_op(FS_TYPE_SOCKET);
    return inode;
}

static socket_t *socket_get(fd_t fd)
{
    task_t *task = running_task();
    file_t *file = task->files[fd];
    if (!file)
        return NULL;

    inode_t *inode = file->inode;
    if (!inode)
        return NULL;

    if (!inode->desc)
        return NULL;

    return (socket_t *)inode->desc;
}

int sys_socket(int domain, int type, int protocol)
{
    socktype_t socktype = SOCK_TYPE_NONE;
    file_t *file;
    fd_t fd = fd_get(&file);
    if (fd < EOK)
        return fd;
    inode_t *inode = socket_open();
    file->inode = inode;
    file->flags = 0;
    file->count = 1;
    file->offset = 0;
    socket_t *s = (socket_t *)inode->desc;
    s->type = socktype;
    if (s->type != SOCK_TYPE_NONE)
    {
        socket_get_op(s->type)->socket(s, domain, type, protocol);
    }
    return fd;
}

int sys_recvmsg(int fd, msghdr_t *msg, u32 flags)
{
    socket_t *s = socket_get(fd);
    if (!s)
        return -EINVAL;
    msghdr_t m;
    memcpy(&m, msg, sizeof(msghdr_t));
    m.iov = iovec_dup(msg->iov, msg->iovlen);
    int ret = iovec_check(m.iov, m.iovlen, true);
    if (ret < EOK)
        return ret;
    ret = socket_get_op(s->type)->recvmsg(s, &m, flags);
    kfree(m.iov);
    return ret;
}

void socket_init()
{
    fs_register_op(FS_TYPE_SOCKET, &socket_op);
}
```
如上所示，在创建一个socket时，也需要申请一个inode进行管理，打开和关闭socket就是打开inode和释放inode。在我们监控链接、监听，收发消息本质是一系列的系统调用。我们展示的代码中只保留了 sys_recvmsg 这一个系统调用，本质还是调用虚拟文件系统的 recvmsg 函数。

## 三、数据包套接字
数据包套接字 用于在设备驱动程序 (OSI第2层) 级别接收或发送原始数据包。它们允许用户在物理层之上的用户空间中实现协议模块。它的主要用途就是用来抓包。通过它可以直接发送以太网数据包：
```cpp
typedef struct pkt_pcb_t
{
    list_node_t node;     // 链表节点，用于连接到全局PCB列表
    list_t rx_pbuf_list;  // 缓冲地址
    eth_addr_t laddr;     // 本地mac地址
    eth_addr_t raddr;     // 远端mac地址
    struct task_t *rx_waiter;    // 等待接收的进程
} pkt_pcb_t;

static list_t pkt_pcb_list;

static int pkt_socket(socket_t *s, int domain, int type, int protocol)  // 创建套接字
{
    if (!s->pkt)
    {
        s->pkt = (pkt_pcb_t *)kmalloc(sizeof(pkt_pcb_t));
        memset(s->pkt, 0, sizeof(pkt_pcb_t));
    }

    pkt_pcb_t *pcb = s->pkt;
    list_init(&pcb->rx_pbuf_list);
    list_push(&pkt_pcb_list, &pcb->node);
    return EOK;
}

static int pkt_bind(socket_t *s, const sockaddr_t *name, int namelen)   // 绑定本地MAC地址
{
    sockaddr_ll_t *sin = (sockaddr_ll_t *)name;
    eth_addr_copy(s->pkt->laddr, sin->addr);
    return EOK;
}

static int pkt_connect(socket_t *s, const sockaddr_t *name, int namelen)   // 连接远端MAC地址
{
    sockaddr_ll_t *sin = (sockaddr_ll_t *)name;
    eth_addr_copy(s->pkt->raddr, sin->addr);
    return EOK;
}

static int pkt_getpeername(socket_t *s, sockaddr_t *name, int *namelen)
{
    sockaddr_ll_t *sin = (sockaddr_ll_t *)name;

    sin->family = AF_PACKET;
    ip_addr_copy(sin->addr, s->pkt->raddr);
    *namelen = sizeof(sockaddr_ll_t);
    return EOK;
}

static int pkt_recvmsg(socket_t *s, msghdr_t *msg, u32 flags)
{
    err_t ret = EOK;
    if (list_empty(&s->pkt->rx_pbuf_list))
    {
        s->pkt->rx_waiter = running_task();
        ret = task_block(s->pkt->rx_waiter, NULL, TASK_WAITING, TIMELESS);
    }
    if (ret != EOK)
        return ret;

    pbuf_t *pbuf = element_entry(pbuf_t, node, list_popback(&s->pkt->rx_pbuf_list));
    ret = iovec_write(msg->iov, msg->iovlen, pbuf->payload, pbuf->length);
    pbuf_put(pbuf);
    return ret;
}

static int pkt_sendmsg(socket_t *s, msghdr_t *msg, u32 flags)
{
    int ret = EOK;
    size_t size = iovec_size(msg->iov, msg->iovlen);
    if (size > ETH_MTU)
        return -EINVAL;
    if (msg->name)
        ret = pkt_connect(s, msg->name, msg->namelen);
    if (ret < EOK)
        return EOK;
    pbuf_t *pbuf = pbuf_get();
    ret = iovec_read(msg->iov, msg->iovlen, pbuf->payload, size);
    if (ret < EOK)
        return ret;
    pbuf->length = size;

    netif_t *netif = netif_get();
    netif_output(netif, pbuf);
    return size;
}

static int pkt_recv(pkt_pcb_t *pcb, pbuf_t *pbuf)
{
    eth_t *eth = pbuf->eth;
    if (!eth_addr_isany(pcb->raddr) && !eth_addr_cmp(pcb->raddr, eth->src))
        return false;
    if (!eth_addr_isany(pcb->laddr) && !eth_addr_cmp(pcb->laddr, eth->dst))
        return false;

    pbuf->count++;
    list_push(&pcb->rx_pbuf_list, &pbuf->node);
    if (pcb->rx_waiter)
    {
        task_unblock(pcb->rx_waiter, EOK);
        pcb->rx_waiter = NULL;
    }
    return true;
}

err_t pkt_input(netif_t *netif, pbuf_t *pbuf)
{
    int eaten = false;
    list_t *list = &pkt_pcb_list;
    for (list_node_t *ptr = list->head.next; ptr != &list->tail; ptr = ptr->next)
    {
        pkt_pcb_t *pcb = element_entry(pkt_pcb_t, node, ptr);
        err_t ret = pkt_recv(pcb, pbuf);
        if (ret < 0)
            return ret;
        if (ret > 0)
        {
            eaten = true;
            break;
        }
    }
    return eaten;
}
```
这是一个内核级套接字实现，提供了对数据链路层的直接访问能力。数据收发直接操作以太网帧。例如pkt_sendmsg 内部直接调用的 netif_output，这是以太网的发送。它不涉及ARP、IP等协议，为应用程序提供对网络硬件的底层访问能力。


## 四、原始套接字
原始套接字允许在用户空间中实现新的 IPv4 协议。原始套接字 接收或发送 不包括 链路层报头的原始数据报。即原始套接字发送IP数据包，它的实现和数据包套接字比较类似。例如原始套接字在 raw_sendmsg中 调用的是 ip_output 函数。我们这里不再详细贴原始套接字的实现，只看一下如何使用测试的。
```cpp
#define BUFLEN 0x1000
static char buf[BUFLEN];

int main(int argc, char const *argv[])
{
    int fd = socket(AF_INET, SOCK_RAW, PROTO_IP);  // 创建一个原始套接字，这里的socket是系统调用
    msghdr_t msg;
    iovec_t iov;

    msg.name = NULL;
    msg.namelen = 0;

    msg.iov = &iov;
    msg.iovlen = 1;

    iov.base = buf;
    iov.size = sizeof(buf);

    printf("receiving...\n");
    int ret = recvmsg(fd, &msg, 0);  // 等到接收数据包
    printf("recvmsg %d\n", ret);
    ip_t *ip = (ip_t *)iov.base;
    printf("recv ip %r -> %r : %s\n", ip->src, ip->dst, ip->payload);

    ip_addr_t addr;
    ip_addr_copy(addr, ip->dst);
    ip_addr_copy(ip->dst, ip->src);
    ip_addr_copy(ip->src, addr);
    memcpy(ip->payload, "this is ack message", 19);
    iov.size = sizeof(ip_t) + 19;

    sendmsg(fd, &msg, 0);  // 发送一个数据包

    close(fd);
    return EOK;
}
```
我们这里拿recvmsg举例，这是一个系统调用。在sys_recvmsg中会调用虚拟文件系统的 sendmsg 方法，因为这里是原始套接字，因此会去调用raw_recvmsg。在raw_recvmsg判断接收队列是否有数据包，如果没有就阻塞进程等待，否则就从接收队列中获取数据包，并将数据写入缓冲区。


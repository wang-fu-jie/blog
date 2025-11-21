---
title:       自制操作系统 - TCP协议
subtitle:    "TCP协议"
description: "TCP提供可靠的字节流而设计的协议,，它有很多机制：连接机制、确认重传、滑动窗口、拥塞控制等，实现也非常复杂，我们本文将大致的介绍TCP协议以及一些实现。"
excerpt:     "TCP提供可靠的字节流而设计的协议,，它有很多机制：连接机制、确认重传、滑动窗口、拥塞控制等，实现也非常复杂，我们本文将大致的介绍TCP协议以及一些实现。"
date:        2024-04-25T10:43:31+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/e0/69/cat_kitten_tree_green_summer_animal_domestic_adorable-498814.jpg!d"
published:   true
tags:
    - 操作系统
slug:        "os-network-tcp01"
categories:  [ "操作系统" ]
---

## 一、TCP协议简介
TCP(Transmission Control Protocol) 传输控制协议，TCP 协议为了在不可靠的 IP 协议上提供可靠的字节流而设计的协议，它有很多机制：连接机制、确认重传、滑动窗口、拥塞控制。它的报文结构如下：
<rawhtml>
<img src="/post_images/os/{{< filename >}}/1-01.png" alt="图片加载失败" width="600"/>
</rawhtml>

* 头长度：TCP 头部长度，单位 4 字节，数据长度 = IP 数据长度 - TCP 头长度；
* 紧急指针：用于紧急数据；
* 选项：MSS(Maximum Segment Size)，最大分段大小，一般为 1460 (IP_MTU(1500) - IP_HDR(20) - TCP_HDR(20))

标志位的说明：
* FIN: Finish，没有更多的数据发送，序列号用于终止连接
* SYN: Synchronize，序列号用于同步建立连接
* RST: Reset，连接重置
* PSH: Push，提交功能
* ACK: Acknowledgement，确认号有效
* URG: Urgent，紧急指针有效

下面两个是后加的 2，用于显式拥塞控制：
* ECE: ECN(Explicit Congestion Notification)-Echo，显式拥塞响应
* CWR: Congestion Window Reduced，拥塞窗口减少

## 二、客户端连接
客户端申请和服务器连接需要经历三次握手，流程如下：
<rawhtml>
<img src="/post_images/os/{{< filename >}}/2-01.png" alt="图片加载失败" width="500"/>
</rawhtml>
客户端申请建立连接，发送SYN包，服务端回复ACK，同时服务端也需要建立和客户端的链接，同样发送SYN，客户端回复ACK。因为服务器回复ACK和发送SYN的两个包是合并为1个发送的，因此建链是三次握手。这个握手期间客户端的状态会发生状态变化。如下：
<rawhtml>
<img src="/post_images/os/{{< filename >}}/2-02.png" alt="图片加载失败" width="400"/>
</rawhtml>
客户端开始处于CLOSE状态，在发送ACK之后进入SYNC_SEND状态。当服务端接收并ACK，客户最终进入连接状态。如下为连接的代码实现

```cpp
static int tcp_connect(socket_t *s, const sockaddr_t *name, int namelen)
{
    sockaddr_in_t *sin = (sockaddr_in_t *)name;
    tcp_pcb_t *pcb = s->tcp;
    if (pcb->state != CLOSED)    // 判断初始化状态是否是 CLOSED
    {
        return -EINVAL;
    }
    if (ip_addr_isany(sin->addr) || ntohs(sin->port) == 0)
        return -EADDR;

    ip_addr_copy(pcb->raddr, sin->addr);
    pcb->rport = ntohs(sin->port);
    if (!pcb->lport)                    // 如果端口是0，则申请一个端口。
        pcb->lport = port_get(&map, 0);

    pcb->rcv_nxt = 0;
    pcb->rcv_mss = TCP_MSS;
    pcb->rcv_wnd = TCP_WINDOW;
    pcb->snd_nxt = tcp_next_isn();   // 获取下一个序列号
    pcb->state = SYN_SENT;           // 将状态改为 SYN_SENT
    list_remove(&pcb->node);
    list_push(&tcp_pcb_active_list, &pcb->node);
    tcp_send_ack(pcb, TCP_SYN);   // 发送SYN包，
    pcb->ac_waiter = running_task();
    int ret = task_block(pcb->ac_waiter, NULL, TASK_WAITING, s->sndtimeo);  // 进程进入阻塞，等待服务端的ACK。
    return ret;
}
```
如上我们可以看到客户端申请建链的过程，发送SYN包后就进入阻塞态。当服务端回包之后，交给tcp_input进行处理，来唤醒进程，回复ACK，回复ACK后，连接状态进入ESTABLISHED。

## 三、TCP选项与重置
TCP 选项是 TCP 头部中 可变长度的扩展字段，用于在建立连接或数据传输时 协商额外功能、优化性能。当 TCP 头长度大于 20 (头长度字段 > 5) 时，表示 TCP 包含选项。TCP 实现 必须： 
* MUST-69：TCP 选项列表结尾必须用 0 填充到四字节对齐；
* MUST-4：必须支持如下选项：

| 类型 | 名称 | 长度 | 描述           |
|------|------|--------|----------------|
| 0    | EOL  | -      | 选项结束       |
| 1    | NOP  | -      | 无操作         |
| 2    | MSS  | 4      | 最大分段尺寸   |

* MUST-5：必须在任何段中都支持接收 TCP 选项；
* MUST-6：必须忽略任何它没有实现的 TCP 选项而不会出错；
* MUST-68：除 EOL 和 NOP 选项外，所有的选项必须有 长度(length) 字段，包括所有未来的选项；
* MUST-7：必须准备处理不合法的选项长度，比如 0，推荐的做法是重置连接并记录日志；

MSS 的作用就是：告诉对方我一次最多能接收多少 TCP 数据，避免 IP 层分片。

### 3.1、TCP重置
作为一般规则，当一个明显不是为当前连接准备的段到达时，就会发送 Reset(RST)，如果不清楚是这种情况，则不能发送重置；
* 如果连接不存在（CLOSED），则发送重置，以响应除 RST 和 SYN 之外的任何传入段。如果传入段设置了 ACK 位，则复位从该段的 ACK 字段中取其序列号；否则，复位序列号为 0, ACK 字段设为传入序列号与数据长度之和；连接保持在 CLOSED 状态；
* 如果连接处于任何非同步状态(LISTEN, SYN-SENT, SYN-RECEIVED)，并且传入段确认尚未发送的内容 (段携带不可接受的 ACK)，或者传入段的安全级别或分区与连接请求的级别和分区不完全匹配，则发送重置；如果传入段有 ACK 字段，则复位从该段的 ACK 字段中取其序列号；否则，复位序列号为 0，ACK 字段设为传入序列号与数据长度之和；连接保持在相同的状态；
* 如果连接处于同步状态 (ESTABLISHED, FIN-WAIT-1, FIN-WAIT-2, CLOSE-WAIT, CLOSING, LAST-ACK, TIME-WAIT)，任何不可接受的段 (窗口外序列号或不可接受的确认号) 都必须用一个空的确认段(没有任何用户数据) 来响应，其中包含当前发送序列号和指示下一个预期接收序列号的确认，并且连接保持在相同的状态；如果传入段的安全级别或分区与连接请求的级别和分区不完全匹配，则发送重置，连接进入 CLOSED 状态。重置从传入段的 ACK 字段中获取序列号

在除 SYN-SENT 之外的所有状态下，所有的重置 RST 段都是通过检查其 SEQ 字段来验证的，如果其序列号在窗口中，则重置是有效的，在 SYN-SENT 状态 (响应初始 SYN 时接收到的 RST)，如果 ACK 字段确认该 SYN，则 RST 是可接受的；

接收方首先对 RST 进行验证，然后改变状态，如果接收方处于 LISTEN 状态，则忽略它，如果接收方处于 SYN-RECEIVED 状态，并且之前处于 LISTEN 状态，则接收方返回到 LISTEN 状态；否则，接收方终止连接并进入 CLOSED 状态，如果接收方处于任何其他状态，则终止连接并通知用户并进入 CLOSED 状态；

TCP 实现应该允许接收到的 RST 段包含数据 SHLD-2。有人建议 RST 段可以包含解释 RST 原因的诊断数据。目前还没有为这类数据建立标准；


## 四、TCP的发送和接收
发送和接收数据报时需要控制发送窗口。如下所示，超过窗口的不能进行发送和接收。
<rawhtml>
<img src="/post_images/os/{{< filename >}}/4-01.png" alt="图片加载失败" width="400"/>
</rawhtml>
所谓的“窗口”都是一个 范围（Range），表示“可以发送 / 可以接收多少字节”，如发送窗口 = 允许发送但尚未确认的数据范围。如下如发送：

```cpp
static int tcp_sendmsg(socket_t *s, msghdr_t *msg, u32 flags)
{
    err_t ret = EOK;
    tcp_pcb_t *pcb = s->tcp;
    size_t size = iovec_size(msg->iov, msg->iovlen);   // 计算发送的数据总长度
    if (size > pcb->snd_wnd)       // 检查是否超过发送窗口
        return -EMSGSIZE;
    size_t left = size;
    iovec_t *iov = msg->iov;
    size_t iovlen = msg->iovlen;
    for (; left > 0 && iovlen > 0; iov++, iovlen--)   // 遍历 iovec，将数据放入发送队列
    {
        if (iov->size <= 0)
            continue;

        int len = left < iov->size ? left : iov->size;
        tcp_enqueue(pcb, iov->base, left, flags);
        left -= len;
    }
    tcp_output(pcb);   // 进行发送
    return size - left;
}
```
以上大致是发送的实现，接收相比于发送要简单一些，这里我们不再一一粘贴，这部分代码比较冗长且复杂。


## 五、TCP定时器
TCP 为每条连接建立了七个定时器。按照它们在一条连接生存期内出现的次序，简要介绍如下：

连接建立(connection establishment)定时器：在发送 SYN 报文段建立一条新连接时启动。如果没有在 75 秒内收到响应，连接建立将中止；

重传(retransmission) 定时器：在 TCP 发送数据时设定。如果定时器已超时而对端的确认还未到达，TCP 将重传数据。重传定时器的值（即 TCP 等待对端确认的时间）是动态计算的，取决于 TCP 为该连接测量的往返时间和该报文段已被重传的次数；

延迟 ACK(delayed ACK) 定时器：在 TCP 收到必须被确认但不需要马上发出确认的数据时设定，TCP 等待 200ms 后发送确认响应。如果，在这 200ms 内，有数据要在该连接上发送，延迟的 ACK 响应就可随着数据一起发送回对端，称为捎带确认；

持续(persist) 定时器：在连接对端通告接收窗口为 0, 阻止 TCP 继续发送数据时设定。由于连接对端发送的窗口通告不可靠（只有数据才会被确认， ACK 不会被确认），允许 TCP 继续发送数据的后续窗口更新有可能丢失。因此，如果 TCP 有数据要发送，但对端通告接收窗口为 0, 则持续定时器启动，超时后向对端发送 1 字节的数据，判定对端接收窗口是否已打开。与重传定时器类似，持续定时器的值也是动态计算的，取决于连接的往返时间，在 5 秒到 60 秒之间取值；

保活(keepalive) 定时器：在应用进程选取了插口的 SO_KEEPALVE 选项时生效。如果连接的连续空闲时间超过 2 小时，保活定时器超时，向对端发送连接探测报文段，强迫对端响应。如果收到了期待的响应， TCP 可确定对端主机工作正常，在该连接再次空闲超过 2 小时之前，TCP 不会再进行保活测试。如果收到的是其他响应，TCP 可确定对端主机已重启。如果连续若干次保活测试都未收到响应，TCP 就假定对端主机已崩溃，尽管它无法区分是主机故障（例如，系统崩溃而尚未重启），还是连接故障（例如，中间的路由器发生故障或电话线断了）；

FIN_WAIT2 定时器：当某个连接从 FIN_WAIT1 状态变迁到 FIN_WAIT2 状态，并且不能再接收任何新数据时（意味着应用进程调用了 close, 而非 shutdown，没有利用 TCP 的半关闭功能），FIN_WAIT2 定时器启动，设为 10 分钟。定时器超时后，重新设为 75 秒，第二次超时后连接被关闭。加入这个定时器的目的是为了避免如果对端一直不发送 FIN，某个连接会永远滞留在 FIN_WAIT2 状态；

TIME_WAIT 定时器， 一般也称为 2MSL 定时器。2MSL 指两倍的 MSL, 最大报文段生存时间。当连接转移到 TIME_WAIT 状态，即连接主动关闭时，定时器启动；连接进入 TIME_WAIT 状态时，定时器设定为 1 分钟，超时后，TCP 控制块 和 PCB 被删除，端口号可重新使用；

TCP包括两个定时器函数：一个函数每 200 ms 调用一次（快速定时器）：一个函数每 500 ms 调用一次（慢速定时器）；
延迟 ACK 定时器与其他 6 个定时器有所不同：如果某个连接上设定了延迟 ACK 定时器，那么下一次 200 ms 定时器超时后，延迟的 ACK 必须被发送 ACK 的延迟时间必须在 0-200 ms 之间。其他的定时器每 500ms 递减一次，计数器减为 0 时，就触发相应的动作。

## 六、总结
TCP后续还需要实现的有断开连接、服务端连接，窗口管理、超时重传和拥塞控制。我们就不一一实现，可以直接看代码。
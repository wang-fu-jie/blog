---
title:       Redis源码阅读 - Redis数据结构双向链表(adlist)
subtitle:    "Redis数据结构双向链表(adlist)"
description: "双向链表是一种通用的链式数据结构，Redis很多模块都有使用到。它的优点是操作前驱后继和头尾节点的时间复杂度都是O(1), 并且不需要连续的内存空间。同样它的缺点访问中间元素的时间复杂度为O(n)，这点是不如列表的。"
excerpt:     "双向链表是一种通用的链式数据结构，Redis很多模块都有使用到。它的优点是操作前驱后继和头尾节点的时间复杂度都是O(1), 并且不需要连续的内存空间。同样它的缺点访问中间元素的时间复杂度为O(n)，这点是不如列表的。"
date:        2026-04-14T16:40:26+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/d3/ae/london_eye_ferris_wheel_london_england_landmark_thames_river_architecture-681901.jpg!d"
published:   true
tags:
    - Redis
slug:        "redis-adlist"
categories:  [ "REDIS" ]
---

## 一、双向链表概述
双向链表是一种链式数据结构，每个节点包含三部分：
* prev：指向前一个节点
* next：指向后一个节点
* value：存储数据

它具有支持双向遍历、插入 / 删除效率高、已知节点指针时，不需要像数组一样移动元素、不支持随机访问、查找某个位置需要遍历等特点。 

## 二、双向链表在Redis中的应用
在Redis早起版本中，双向链表是列表结构的底层实现之一。在新版本中列表类型已经不再使用双向链表。但是双向链表仍然有大量的应用场景，为Redis大量模块所用。如：
1. 事务模块使用双端链表依序保存输入的命令；
2. 服务器模块使用双端链表来保存多个客户端；
3. 订阅/发送模块使用双端链表来保存订阅模式的多个客户端；
4. 事件模块使用双端链表来保存时间事件（time event）

类似的应用还有很多， 在后续的章节中我们将看到， 双端链表在 Redis 中发挥着重要的作用。

## 三、双向链表的实现
双端链表的代码实现位于 adlist.h 和 adlist.c 源文件中，由 listNode 和 list 两个数据结构构成， 下图展示了由这两个结构组成的一个双端链表实例：
![图片加载失败](/post_images/redis/{{< filename >}}/3-01.png)
其中， listNode 是双向链表的节点，而 list 则是双向链表本身。它们的结构如下：
```c
typedef struct listNode {
    struct listNode *prev;   // 前驱节点
    struct listNode *next;   // 后继节点
    void *value;             // 值
} listNode;

typedef struct list {
    listNode *head;          // 头节点
    listNode *tail;          // 尾节点
    void *(*dup)(void *ptr);  // 复制函数
    void (*free)(void *ptr);  // 释放函数
    int (*match)(void *ptr, void *key);  // 比对函数
    unsigned long len;      // 节点数量
} list;
```
注意， listNode 的 value 属性的类型是 void * ，说明这个双向链表对节点所保存的值的类型不做限制。对于不同类型的值，有时候需要不同的函数来处理这些值，因此， list 类型保留了三个函数指针 —— dup 、 free 和 match ，分别用于处理值的复制、释放和对比匹配。在对节点的值进行处理时，如果有给定这些函数，就会调用这些函数。

### 3.1、双向链表的基本操作
首先创建一个双向链表，即申请一个list结构的内存空间，并给属性赋初始值，新建的双向链表没有节点，因此头尾指针和函数都指向NULL，长度为0。接下来我们分析其中一个API，即插入一个数据到指定节点前或指定节点后。
```c
list *listInsertNode(list *list, listNode *old_node, void *value, int after) {   // 插入一个数据到指定节点前或指定节点后
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)   // 申请一个新的链表节点
        return NULL;
    node->value = value;  // 把数据赋给节点的value属性
    if (after) {               // 如果插入到指定节点 old_node 之后
        node->prev = old_node;   // 新节点的前驱节点指向 old_node
        node->next = old_node->next;   // 新节点的后继节点指向 old_node 的原后继节点
        if (list->tail == old_node) {  // 如果 old_node 是尾节点，就把新节点作为尾节点
            list->tail = node;
        }
    } else {                         // 插入到指定节点 old_node 之前
        node->next = old_node;       // 新节点的后继节点指向 old_node
        node->prev = old_node->prev;  // 新节点的前驱节点指向 old_node 的原前驱节点
        if (list->head == old_node) {  // 如果old_node是头节点，就把新节点作为头节点
            list->head = node;
        }
    }
    if (node->prev != NULL) {      // 新节点如果不是头节点，则新节点的前驱节点的下一个节点指向新节点。
        node->prev->next = node;   // 这里使用如插入 old_node 前，因为此时old_node的原前驱节点还执行 old_node，应更新为新节点node
    }
    if (node->next != NULL) {      // 新节点如果不是尾节点，则新节点后继节点的前驱节点指向新节点
        node->next->prev = node;
    }
    list->len++;      // 双向链表元素个数+1
    return list;
}
```
如上所示，是在指定节点前面或者后面插入一个新数据。整个过程基本都是修改指针。对双向链表的操作其实就是对这些前驱指针、后继指针、头尾节点指针进行调整。

### 3.2、双向链表常用API
以下是用于操作双向链表的更多API。
| 函数              | 作用                                                         | 算法复杂度 |
|-------------------|--------------------------------------------------------------|------------|
| listCreate        | 创建新链表                                                   | O(1)       |
| listRelease       | 释放链表，以及该链表所包含的节点                             | O(N)       |
| listDup           | 创建给定链表的副本                                           | O(N)       |
| listRotate        | 取出链表的表尾节点，并插入到表头                             | O(1)       |
| listAddNodeHead   | 将包含给定值的节点添加到链表的表头                           | O(1)       |
| listAddNodeTail   | 将包含给定值的节点添加到链表的表尾                           | O(1)       |
| listInsertNode    | 将包含给定值的节点添加到某个节点的之前或之后                 | O(1)       |
| listDelNode       | 删除给定节点                                                 | O(1)       |
| listSearchKey     | 在链表中查找和给定 key 匹配的节点                            | O(N)       |
| listIndex         | 根据给定索引，返回列表中相应的节点                           | O(N)       |
| listLength        | 返回给定链表的节点数量                                       | O(1)       |
| listFirst         | 返回链表的表头节点                                           | O(1)       |
| listLast          | 返回链表的表尾节点                                           | O(1)       |
| listPrevNode      | 返回给定节点的前一个节点                                     | O(1)       |
| listNextNode      | 返回给定节点的后一个节点                                     | O(1)       |
| listNodeValue     | 返回给定节点的值                                             | O(1)       |

可以看到，双向链表的修改时间复杂度都是O(1)， 但查找时间复杂度是O(N)。

## 四、迭代器
Redis 为双端链表实现了一个迭代器 ， 这个迭代器可以从两个方向对双端链表进行迭代：
* 沿着节点的 next 指针前进，从表头向表尾迭代；
* 沿着节点的 prev 指针前进，从表尾向表头迭代；

以下是迭代器的数据结构定义：
```c
typedef struct listIter {
    listNode *next;   // 下一个节点
    int direction;    // 迭代方向
} listIter;
```
direction 记录迭代应该从那里开始：
* 如果值为 AL_START_HEAD ，那么迭代器执行从表头到表尾的迭代；
* 如果值为 AL_START_TAIL ，那么迭代器执行从表尾到表头的迭代；

这里我们主要看下迭代器是如何进行迭代的：
```c
listNode *listNext(listIter *iter) 
{
    listNode *current = iter->next;   // 获取迭代器当前指向的节点

    if (current != NULL) {
        if (iter->direction == AL_START_HEAD)  // 如果是表头到表尾的方向，迭代器的下一个值指向当前节点的后继节点。
            iter->next = current->next;
        else
            iter->next = current->prev;       // 如果是表尾到表头的方向，迭代器的下一个值指向当前节点的前驱节点。
    }
    return current;
}
```
可以看到当进行迭代是，是吧迭代器指针当前指向的节点返回，然后指针根据方向前进或者后退一步。

以下是迭代器的操作 API ，API 的作用以及算法复杂度：
| 函数                 | 作用                                 | 算法复杂度 |
|----------------------|--------------------------------------|------------|
| listGetIterator      | 创建一个列表迭代器                   | O(1)       |
| listReleaseIterator  | 释放迭代器                           | O(1)       |
| listRewind           | 将迭代器的指针指向表头               | O(1)       |
| listRewindTail       | 将迭代器的指针指向表尾               | O(1)       |
| listNext             | 取出迭代器当前指向的节点             | O(1)       |

 ## 五、总结
 Redis 实现了自己的双向链表结构，作为通用数据结构，被其他功能模块所使用。并且提供了迭代器功能可以遍历双向链表，这里的双向链表遍历并没有保证强一致性。其实这里并不需要，取决于使用场景，例如client list遍历客户端链表时，如果client list没执行完，单线程工作的Reids新连接是无法建链的。或者说有些场景并不需要强一致遍历，后边根据使用的地方可以具体分析。






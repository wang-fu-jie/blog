---
title:       Redis源码阅读 - Redis对象处理机制
subtitle:    "Redis对象处理机制"
description: "redisObject（robj）是 Redis 内部统一的数据对象结构，用来封装所有值类型（如 String、List、Hash、Set、ZSet）。redisObject 相当于 Redis 数据的统一“对象头”，负责描述数据类型、编码方式和生命周期，而真正数据存放在 ptr 指向的结构中。"
excerpt:     "redisObject（robj）是 Redis 内部统一的数据对象结构，用来封装所有值类型（如 String、List、Hash、Set、ZSet）。redisObject 相当于 Redis 数据的统一“对象头”，负责描述数据类型、编码方式和生命周期，而真正数据存放在 ptr 指向的结构中。"
date:        2026-04-28T15:55:14+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/00/d9/baby_teddy_bear_cute_child_small_boy_sweet-910117.jpg!d"
published:   true
tags:
    - Redis
slug:        "redis-redisobject"
categories:  [ "REDIS" ]
---

## 一、对象处理机制
在 Redis 的命令中，用于对键（key）进行处理的命令占了很大一部分， 而对于键所保存的值的类型（后简称“键的类型”），键能执行的命令又各不相同。 比如说， LPUSH 和 LLEN 只能用于列表键， 而 SADD 和 SRANDMEMBER 只能用于集合键， 等等。另外一些命令， 比如 DEL 、 TTL 和 TYPE ， 可以用于任何类型的键， 但是， 要正确实现这些命令， 必须为不同类型的键设置不同的处理方式： 比如说， 删除一个列表键和删除一个字符串键的操作过程就不太一样。

以上的描述说明，Redis 必须让每个键都带有类型信息， 使得程序可以检查键的类型， 并为它选择合适的处理方式。另外，在前面介绍各个底层数据结构时有提到， Redis 的每一种数据类型，比如字符串、列表、有序集， 它们都拥有不只一种底层实现（Redis 内部称之为编码，encoding）， 这说明， 每当对某种数据类型的键进行操作时， 程序都必须根据键所采取的编码， 进行不同的操作。

这说明，操作数据类型的命令除了要对键的类型进行检查之外， 还需要根据数据类型的不同编码进行多态处理。为了解决以上问题， Redis 构建了自己的类型系统， 这个系统的主要功能包括：
* redisObject 对象。
* 基于 redisObject 对象的类型检查。
* 基于 redisObject 对象的显式多态函数。
* 对 redisObject 进行分配、共享和销毁的机制。

以下小节将分别介绍类型系统的这几个方面

## 二、redisObject数据结构
redisObject 是 Redis 类型系统的核心， 数据库中的每个键、值，以及 Redis 本身处理的参数， 都表示为这种数据类型。redisObject 的定义位于 server.h:
```c
struct redisObject {
    unsigned type:4;          // 表示 Redis 数据类型。如OBJ_STRING、OBJ_LIST
    unsigned encoding:4;      // 表示底层实现方式。使用的哪种数据结构，如OBJ_ENCODING_QUICKLIST，OBJ_ENCODING_RAW
    unsigned lru:LRU_BITS;    // 记录对象最近访问时间或 LFU 信息。在 LRU 模式表示最近访问时间，在LFU模式保存访问频率和最近衰减时间
    unsigned iskvobj : 1;     // 标记对象是否用于：key-value 主存储。 1表示数据库中的KV对象 0表示临时对象
    unsigned expirable : 1;   // 标记对象是否允许设置过期时间。
    unsigned refcount : OBJ_REFCOUNT_BITS;  // 引用计数。
    void *ptr;                // 指向真实数据结构。
```
举个例子，如果一个 redisObject 的 type 属性为 OBJ_STRING ， encoding 属性为 REDIS_ENCODING_RAW ，那么这个对象就是一个 Redis 字符串，它的值保存在SDS结构内，而 ptr 指针就指向这个SDS。


## 三、命令的类型检查和多态
有了 redisObject 结构的存在， 在执行处理数据类型的命令时， 进行类型检查和对编码进行多态操作就简单得多了。当执行一个处理数据类型的命令时， Redis 执行以下步骤：

* 根据给定 key ，在数据库字典中查找和它相对应的 redisObject ，如果没找到，就返回 NULL 。
* 检查 redisObject 的 type 属性和执行命令所需的类型是否相符，如果不相符，返回类型错误。
* 根据 redisObject 的 encoding 属性所指定的编码，选择合适的操作函数来处理底层的数据结构。
* 返回数据结构的操作结果作为命令的返回值

## 四、共享对象
有一些对象在 Redis 中非常常见， 比如命令的返回值 OK 、 ERROR 、 WRONGTYPE 等字符， 另外，一些小范围的整数，比如个位、十位、百位的整数都非常常见。为了利用这种常见情况， Redis 在内部使用了一个 Flyweight 模式 ： 通过预分配一些常见的值对象， 并在多个数据结构之间共享这些对象， 程序避免了重复分配的麻烦， 也节约了一些 CPU 时间。

Redis 预分配的值对象有如下这些：
* 各种命令的返回值，比如执行成功时返回的 OK ，执行错误时返回的 ERROR ，类型错误时返回的 WRONGTYPE ，命令入队事务时返回的 QUEUED ，等等。
* 包括 0 在内，小于 redis.h/REDIS_SHARED_INTEGERS 的所有整数（REDIS_SHARED_INTEGERS 的默认值为 10000）

共享对象在服务启动时进行初始化会创建，执行了createSharedObjects函数。
```c
void createSharedObjects(void) {
    int j;
    shared.ok = createObject(OBJ_STRING,sdsnew("+OK\r\n"));
    shared.emptybulk = createObject(OBJ_STRING,sdsnew("$0\r\n\r\n"));
    shared.czero = createObject(OBJ_STRING,sdsnew(":0\r\n"));
    ......
}
```
除了启动时创建的共享对象，还可以把一个普通的对象创建为共享对象，调用makeObjectShared函数即可实现。实现如下：
```c
robj *makeObjectShared(robj *o) {
    serverAssert(o->refcount == 1);
    o->refcount = OBJ_SHARED_REFCOUNT;
    return o;
}
```
一个普通对象编程共享对象，即永远不再释放共享对象。

## 五、引用计数和对象销毁
当将 redisObject 用作数据库的键或者值， 而不是用来储存参数时， 对象的生命期是非常长的， 因为 C 语言本身没有自动释放内存的相关机制， 如果只依靠程序员的记忆来对对象进行追踪和销毁， 基本是不太可能的。

另一方面，正如前面提到的，一个共享对象可能被多个数据结构所引用， 这时像是“这个对象被引用了多少次？”之类的问题就会出现。

为了解决以上两个问题， Redis 的对象系统使用了引用计数技术来负责维持和销毁对象， 它的运作机制如下：

* 每个 redisObject 结构都带有一个 refcount 属性，指示这个对象被引用了多少次。
* 当新创建一个对象时，它的 refcount 属性被设置为 1 。
* 当对一个对象进行共享时，Redis 将这个对象的 refcount 增一。
* 当使用完一个对象之后，或者取消对共享对象的引用之后，程序将对象的 refcount 减一。
* 当对象的 refcount 降至 0 时，这个 redisObject 结构，以及它所引用的数据结构的内存，都会被释放。

## 六、redisObject的实现
redisObject 本身的创建、编码、优化、释放等在object.c源文件中实现。我们这里只分析创建函数，创建redisObject的函数非常多，因为不同数据类型都封装了各自的创建函数，首先我们看一个通用的创建函数：
```c
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));          // 申请内存
    o->type = type;                         // 设置类型
    o->encoding = OBJ_ENCODING_RAW;         // RAW 是最通用、最安全的默认编码。
    o->ptr = ptr;                           
    o->refcount = 1;                        
    o->lru = 0;
    o->iskvobj = 0;
    o->expirable = 0;
    return o;
}
```
如上所示，为一个通用的创建函数。针对不同的类型，例如字符串类型，可以看到使用 createStringObject 函数实现。
```c
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```
首先，对于 createRawStringObject 函数来说，它在创建 String 类型的值的时候，会调用 createObject 函数。 你可能有一个疑惑的点，当字符串长度小于44时，使用了嵌入式字符串的创建方法，大于44时才使用了sds。对于不超过 44 字节的字符串来说，就可以避免内存碎片和两次内存分配的开销。至于为什么44字节，我们在讲字符串类型时再具体分析。


## 七、补充
我们这里在介绍redisObject的实现时，仅仅介绍了字符串类型的创建。除字符串之外，还有list、set、hash等各种数据类型的创建，编码、销毁等操作。但是数据类型的命令实现大部分集中在 `t_`开头的一些源文件中，如t_string中的setCommand函数，在创建一个新键值对会调用这个函数，然后调用了object.c中的 tryObjectEncoding 对字符串进行编码。 再如t_list实现了对列表的命令实现。

但是接下来并不准备单独介绍数据类型，因为在数据类型的命令实现中，存在大量的其他模块的操作，如网络通信、db操作、过期判断、resp协议等等。这些内容目前对我们来说都还比较陌生，很容易阻塞到一个函数中进行不下去，等了解完其他相关模块，我们再回头来看数据类型的操作命令是如何实现的。

在object.c中，对robj操作的功能非常多，我们无法做到逐个函数进行分析，通过本篇文章，你只需要了解redisObject是什么，它解决了什么问题。当你想了解具体某个功能时，例如LRU是怎么实现的，可以再去看具体的代码。




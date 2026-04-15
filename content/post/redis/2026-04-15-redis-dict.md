---
title:       Redis源码阅读 - Redis数据结构字典(dict)
subtitle:    "Redis数据结构字典(dict)"
description: "字典是一种抽象数据结构，由键值对（key-value pairs）组成，在Redis中的应用非常广泛。Redis使用拉链法解决哈希冲突问题，并且实现了渐进式rehash避免了传统rehash阻塞正常请求。"
excerpt:     "字典是一种抽象数据结构，由键值对（key-value pairs）组成，在Redis中的应用非常广泛。Redis使用拉链法解决哈希冲突问题，并且实现了渐进式rehash避免了传统rehash阻塞正常请求。"
date:        2026-04-15T10:19:56+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/9f/29/wildfire_forest_fire_blaze_smoke_trees_heat_burning-834878.jpg!d"
published:   true
tags:
    - Redis
slug:        "redis-dict"
categories:  [ "REDIS" ]
---

## 一、字典概述
字典是一种抽象数据结构，由键值对（key-value pairs）组成，在Redis中的应用非常广泛。字典通过 Hash 函数的计算，就能定位数据在表中的位置，紧接着可以对数据进行操作，理论上来说，它能以 O(1) 的复杂度快速查询数据。随着数据量不断增加，就经常会出现哈希冲突，Redis使用拉链法来解决哈希冲突问题。

## 二、字典的应用
字典在 Redis 中的应用非常广泛，基本上各个功能模块都有用到字典的地方。主要用途有以下两个：实现数据库键空间（key space）和 用作 Hash 类型键的底层实现之一。在使用Redis时我们日常所说的库，如0号库1号库，每个数据库都有一个字典来存储键值对，这个字典就被称做键空间。Redis 的 Hash 类型的底层实现就使用了字典，在创建新 Hash 键时， 默认使用listpack作为底层实现， 当满足一定条件时， 程序才会将底层实现转换到字典。

## 三、字典的实现
在讲字典实现之前，我们先介绍一下rehash的概念，因为字典操作中会经常涉及到这个概念。

### 3.1、什么是rehash
在新创建一个字典时，字典中元素比较少，少量桶就足够了。例如初始化了4个桶，但是随着数据增多，例如已经写入了32个元素，拉链会变的很长。这时就需要增加桶的数量。当将桶扩充到32之后，哈希算法就需要对32取模，重新计算这些数据应该落在哪个桶，这个过程就是rehash。

传统的rehash有一个缺点，那就是假设已经有一亿个桶，再进行rehash时间会很长，rehash过程影响Redis正常提供服务。因此redis实现了渐进式rehash，即一次rehash少量桶，记录rehash的进度，这样在有请求时需判断key是否已经被rehash，在两个哈希表中查找key信息。具体如何实现的我们后文再说，这里先了解下rehash这个概念即可。

### 3.2、字典的代码实现
字典的实现在源文件 dict.h和dict.c 中，它的结构定义如下：
```c
struct dict {
    dictType *type;               // 字典类型，定义字典的行为函数（函数指针集合），如指定hash算法
    dictEntry **ht_table[2];      // 定义两个哈希表数组，用于rehash，一个迁出数据一个迁入
    unsigned long ht_used[2];     // 记录当前哈希表中“元素（entry）总数”
    long rehashidx;                // 记录 rehash 进度的标志，值为 -1 表示 rehash 未进行
    unsigned pauserehash;         // 用于定义rehash是否暂停，如何该值>0 则暂停rehash
    /* Keep small vars at end for optimal (minimal) struct padding */
    signed char ht_size_exp[2];   //  使用指数记录哈希表大小，哈希表大小始终是 2 的幂，表示桶的总数量
    signed pauseAutoResize: 15;   // 控制是否允许自动扩容/缩容，大于0表示禁止自动 resize， 等于0表示正常， 小于0表示异常状态（错误）
    unsigned useStoredKeyApi: 1;  // 控制 key 的处理方式, 1 表示使用“已存储 key”的 API， 0 表示使用普通 key API
    void *metadata[];             // 可扩展元数据（柔性数组）,存储额外信息。
};
```
dict 类型使用了两个指针，分别指向两个哈希表。其中， 0 号哈希表（ht_table[0]）是字典主要使用的哈希表， 而 1 号哈希表（ht_table[1]）则只有在程序对 0 号哈希表进行 rehash 时才使用。接下来我们看一下字典类型、和哈希表的定义。
```c
struct dictEntry {     // 定义字典元素
    struct dictEntry *next;  /* Must be first 指向下一个字段数据 */
    void *key;               /* Must be second */
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
};

typedef struct dictEntryNoValue {     // 没有值，只有key的字典
    dictEntry *next; /* Must be first */
    void *key;       /* Must be second */
} dictEntryNoValue;
```
如上所示，dictEntry表示一个哈希条目，即一个元素。它具有一个后继指针，用于解决哈希冲突。另外还定义了dictEntryNoValue类型用于只存储key不存储value的字典类型。现在我们再来看字典的定义：dictEntry **ht_table[2]。它是一个数组，元素类型是dictEntry **，ht_table[0] 是一个数组的起始地址，数组里的每个元素是 dictEntry *。即桶数组本身是连续内存。如下图展示了一个dict结构：
<rawhtml>
<img src="/post_images/redis/{{< filename >}}/3-01.png" alt="图片加载失败" width="700"/>
</rawhtml>
在上图的字典示例中， 字典虽然创建了两个哈希表， 但正在使用的只有 0 号哈希表， 这说明字典未进行 rehash 状态。

### 3.3、哈希算法
Redis实现的字典是一个通过的数据结构，可以自行指定哈希算法，通过hashtype来定义。同样的字典也有默认的哈希算法。如下所示：
```c
uint64_t dictGenHashFunction(const void *key, size_t len) {
    return siphash(key,len,dict_hash_function_seed);
}
```
siphash函数的实现位于 spihash.c 源文件中，SipHash 是一种高速、带密钥的哈希函数（keyed hash function），主要用于防御哈希碰撞攻击。我们这里不做过多介绍。



## 四、字典的基本操作
字典的操作还是比较复杂的，尤其是字典的增删改查，因为还要涉及是否正在rehash的判断。接下来我们来逐个解析。

### 4.1、字典的创建
首先我们看一下字典的创建，它调用的是 dictCreate 函数，在该函数中申请内存然后调用 _dictInit 进行初始化。
```c
int _dictInit(dict *d, dictType *type)
{
    _dictReset(d, 0);   // 把字典中的第一个和第二个哈希表都做重置
    _dictReset(d, 1);
    d->type = type;
    d->rehashidx = -1;
    d->pauserehash = 0;
    d->pauseAutoResize = 0;
    d->useStoredKeyApi = 0;
    return DICT_OK;
}
static void _dictReset(dict *d, int htidx)  // 重置已使用 _dictInit() 初始化的哈希表参数
{
    d->ht_table[htidx] = NULL;
    d->ht_size_exp[htidx] = -1;
    d->ht_used[htidx] = 0;
}
```
如上所示，新创建的字典中都两个哈希表都指向了NULL，并没有分配任何空间。这是因为0号哈希表的空间分配将在第一次往字典添加键值对时进行，1号哈希表的空间分配将在 rehash 开始时进行。


### 4.2、添加键值对
根据字典所处的状态， 将给定的键值对添加到字典可能会引起一系列复杂的操作：
* 如果字典为未初始化（即字典的 0 号哈希表的 table 属性为空），则程序需要对 0 号哈希表进行初始化；
* 如果在插入时发生了键碰撞，则程序需要处理碰撞；
* 如果插入新元素，使得字典满足了 rehash 条件，则需要启动相应的 rehash 程序；

添加键值对位于dictAdd函数中，底层调用了dictAddRaw函数，我们这里先看下dictAddRaw的实现：
```c
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)   // 更低级的添加元素
{
    void *position = dictFindLinkForInsert(d, key, existing);  // 如果已经存在，position会返回NULL，插入也返回NULL
    if (!position) return NULL;
    if (d->type->keyDup) key = d->type->keyDup(d, key);
    return dictInsertKeyAtLink(d, key, position);  // 在查找到的位置插入key
}
```
如上，在插入一个键值对时，会先判断key是否存在，如果存在直接返回NULL，如果不存在就在已经找到应该插入的位置进行插入。dictInsertKeyAtLink是真正的插入函数。在这个函数中会首先判断是否正在rehash以及字典是否无value，针对没有value的字典类型是单独的分子，这里我们不做过多解析。接下来我们分别探讨以上三种情况是如何插入键值对的。这三种情况其实是在 dictFindLinkForInsert 中判断的，在确定应该的插入的位置时就需要确认了。实现如下：
```c
dictEntryLink dictFindLinkForInsert(dict *d, const void *key, dictEntry **existing) {    // 查找key应该被插入的位置
    ......
    uint64_t hash = dictHashKey(d, key, d->useStoredKeyApi);    // 计算hash值
    if (existing) *existing = NULL;
    idx = hash & DICTHT_SIZE_MASK(d->ht_size_exp[0]);   // 计算索引
    _dictRehashStepIfNeeded(d,idx);  // 判断是否需要rehash
    _dictExpandIfNeeded(d);  // 判断是否需要扩容哈希表
    ......
}
```
_dictRehashStepIfNeeded 即进行了第三情况所说的判断，如果满足rehash的条件，就启动渐进式rehash。 _dictExpandIfNeededy 会判断是否需要扩容，如果是空表，那就是需要扩容。这两个函数的代码可自行阅读源文件。这里我们分情况进行讨论：

#### 4.2.1、需要 rehash + 需要扩容
这种情况发生的条件是：当前没有在 rehash，且负载因子过高。这种情况过程如下：
```c
_dictRehashStepIfNeeded → 不做事（还没开始 rehash）

_dictExpandIfNeeded → 触发扩容
                   → 分配 ht[1]
                   → 设置 rehashidx = 0（开始 rehash）
```
从这一刻开始，进入渐进式rehash状态。

#### 4.2.2、不 rehash，只扩容
这种情况实际并不存在，因为只要扩容就比如会rehash。在对空表进行插入元素时，其实是初始化。
```c
int dictExpandIfNeeded(dict *d) {           //如果需要对字典进行扩容
    if (dictIsRehashing(d)) return DICT_OK;
    if (DICTHT_SIZE(d->ht_size_exp[0]) == 0) {
        dictExpand(d, DICT_HT_INITIAL_SIZE);
        return DICT_OK;
    }
    ......
}
```
如上所示，这种情况会直接扩容到初始化大小，扩容的函数中会进行判断。如果d->ht_table[0] 指向NULL或者元素个数为0，会直接设置完成rehash。

#### 4.2.3、只需要 rehash（不扩容）
发生条件，已经在 rehash（rehashidx != -1），调用渐进式rehash，每次操作推进一点 rehash。此时一般不会触发扩容（因为正在 rehash）。

#### 4.2.4、两者都不需要
这种情况是最常见的，没有在 rehash，且负载因子是正常的。直接在 ht[0] 插入数据即可。

从添加键值对的过程中可以看到，每次添加会调用_dictRehashStepIfNeeded，即如果正在rehash，在插入过程中进行一个桶的迁移。并且会将这个新元素插入到1号哈希表中，等到全部rehash完成1号哈希表赋予到0号，再把1号表置NULL。


## 五、渐进式rehash
从插入一个元素我们可以看到，Redis的rehash是一步一步逐渐完成的，并不是一次性全部迁移后再响应其他请求。我们先看Redis实现rehash的过程：

### 5.1、rehash的详细过程
字典的 rehash 操作实际上就是执行以下任务：

* 创建一个比 ht_table[0] 更大的 ht_table[1]
* 将 ht_table[0] 中的所有键值对迁移到 ht_table[1]
* 将原有 ht_table[0] 的数据清空，并将 ht_table[1] 替换为新的 ht_table[0]

经过以上步骤之后， 程序就在不改变原有键值对数据的基础上， 增大了哈希表的大小。rehashidx标志rehash的进度。rehashidx等于0标志开始，大于0如等于2时标记已经迁移了2个桶。当rehash完成时。rehashidx重新设置为-1标志着rehash的结束。

### 5.2、渐进式rehash的实现
前边我们提到了如果哈希表已经非常大的时候，rehash 过程必须将所有键值对迁移完毕之后才将结果返回给用户，这是非常不友好且性能很差的。因此Redis实现了渐进式rehash，渐进式 rehash 主要由 _dictRehashStepIfNeeded 和 dictRehashMilliseconds 两个函数进行：
* _dictRehashStepIfNeeded 用于对数据库字典、以及哈希键的字典进行被动 rehash ；
* dictRehashMicrosecond 则由 Redis 服务器常规任务程序（server cron job）执行，用于对数据库字典进行主动 rehash 

当每次进行增删改查时，都会去判断是否需要做一次渐进rehash，每次指向会迁移其中一个桶到新的哈希表。因为一个桶上的元素并不会很多，对单个索引上的节点进行迁移， 几乎不会对响应时间造成影响。

dictRehashMicrosecond 可以在指定的毫秒数内， 对字典进行 rehash。当 Redis 的服务器常规任务执行时， dictRehashMilliseconds 会被执行， 在规定的时间内， 尽可能地对数据库字典中那些需要 rehash 的字典进行 rehash ， 从而加速数据库字典的 rehash 进程（progress）。代码如下：
```c
int dictRehashMicroseconds(dict *d, uint64_t us) {   // 按时间片做渐进式 rehash
    if (d->pauserehash > 0) return 0;   // 判断rehash是否已经被暂停
    monotime timer;
    elapsedStart(&timer);  // 开始计时
    int rehashes = 0;
    while(dictRehash(d,100)) {   // rehash100个桶
        rehashes += 100;
        if (elapsedUs(timer) >= us) break; // 到达指定时间，结束循环，返回本次rehash的桶数量
    }
    return rehashes;
}
```
从代码可以看到，按照时间进行rehash，一次会迁移100个桶，如果没有达到时间片，会持续循环进行迁移。

### 5.3、其他措施
在哈希表进行 rehash 时， 字典还会采取一些特别的措施， 确保 rehash 顺利、正确地进行：

* 因为在 rehash 时，字典会同时使用两个哈希表，所以在这期间的所有查找、删除等操作，除了在 ht[0] 上进行，还需要在 ht[1] 上进行。
* 在执行添加操作时，新的节点会直接添加到 ht[1] 而不是 ht[0] ，这样保证 ht[0] 的节点数量在整个 rehash 过程中都只减不增。


## 六、字典的收缩
当字典数据越来越多时，需要进行扩展，同样的当字段删除了许多数据，也会进行收缩。收缩过程也是通过渐进式rehash来完成的。和扩展 rehash 的操作几乎一样。
```c
int dictShrink(dict *d, unsigned long size) {
    if (dictIsRehashing(d) || d->ht_used[0] > size || DICTHT_SIZE(d->ht_size_exp[0]) <= size)
        return DICT_ERR;
    return _dictResize(d, size, NULL);
}
```
如上，字典收缩底层调用的也是 _dictResize 方法，和扩展底层调用的是同一个函数。


## 七、字典迭代器
字典带有自己的迭代器实现 —— 对字典进行迭代实际上就是对字典所使用的哈希表进行迭代：

* 迭代器首先迭代字典的第一个哈希表，然后，如果 rehash 正在进行的话，就继续对第二个哈希表进行迭代。
* 当迭代哈希表时，找到第一个不为空的索引，然后迭代这个索引上的所有节点。
* 当这个索引迭代完了，继续查找下一个不为空的索引，如此反覆，直到整个哈希表都迭代完为止。

迭代器的结构如下：
```c
typedef struct dictIterator {    // 字典迭代器
    dict *d;                     // 正在迭代的字典
    long index;                  // 正在迭代的哈希表的号码（0 或者 1）
    int table,                   // 正在迭代的哈希表数组的索引
        safe;                    // 是否安全
    dictEntry *entry, *nextEntry; // 当前哈希节点和它的后继节点
    unsigned long long fingerprint; // 
} dictIterator;
```
我们重点介绍一下安全迭代器和非安全迭代器。安全迭代器：在迭代进行过程中，可以对字典进行修改。不安全迭代器：在迭代进行过程中，不对字典进行修改。 如果正在进行rehash，使用安全迭代器进行迭代时，会暂停rehash，迭代完成后再恢复继续进行rehash。


## 八、字典总结
在本文中，我们介绍了字典是如何实现的，已经rehash是如何进行的。 这已经包含了字典的核心模块。但是字典的实现源文件有两千多行代码，我们不可能把每个细节都讲到。例如什么时候触发 rehash？rehash 扩容扩多大？暂停rehash是如何实现的？等等，这些疑问都可以在源代码中找到答案。感兴趣的读者可以在评论区分享你的问题和答案。







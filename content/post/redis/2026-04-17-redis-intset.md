---
title:       Redis源码阅读 - Redis数据结构整数集合(intset)
subtitle:    "Redis数据结构整数集合(intset)"
description: "整数集合（intset）用于有序、无重复地保存多个整数值， 根据元素的值， 自动选择该用什么长度的整数类型来保存元素，它是按照从小到大的顺序排列的，是set类型的底层实现之一。"
excerpt:     "整数集合（intset）用于有序、无重复地保存多个整数值， 根据元素的值， 自动选择该用什么长度的整数类型来保存元素，它是按照从小到大的顺序排列的，它是set类型的底层实现之一。"
date:        2026-04-17T10:26:50+08:00
author:      "王富杰"
image:       "https://c.pxhere.com/photos/8f/f6/firework_fountain_night_light_fire_sparks_celebration_explosion-712373.jpg!d"
published:   true
tags:
    - Redis
slug:        "redis-intset"
categories:  [ "REDIS" ]
---

## 一、整数集合概述
整数集合（intset）用于有序、无重复地保存多个整数值， 根据元素的值， 自动选择该用什么长度的整数类型来保存元素，它是set类型的底层实现之一。

在一个 intset 里面， 最长的元素可以用 int16_t 类型来保存， 那么这个 intset 的所有元素都以 int16_t 类型来保存。 如果有一个新元素要加入到这个 intset ， 并且这个元素不能用 int16_t 类型来保存 —— 比如说， 新元素的长度为 int32_t ， 那么这个 intset 就会自动进行“升级”： 先将集合中现有的所有元素从 int16_t 类型转换为 int32_t 类型， 接着再将新元素加入到集合中。根据需要， intset 可以自动从 int16_t 升级到 int32_t 或 int64_t ， 或者从 int32_t 升级到 int64_t 。

## 二、整数集合的结构
整数集合的实现定义在 intset.h 和 intset.c 源文件中，以下是整数集合的结构定义：
```c
typedef struct intset {
    uint32_t encoding;   // 保存元素所使用的类型的长度
    uint32_t length;     // 元素个数
    int8_t contents[];   // 保存元素的数组
} intset;

// 定义encoding属性的可能取值
#define INTSET_ENC_INT16 (sizeof(int16_t))   // 16位存储的整数
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```
contents 数组是实际保存元素的地方，数组中的元素有以下两个特性：元素不重复，元素在数组中由小到大排列。

这里你可能为奇怪，contents数组为什么类型声明为 int8_t。实际上， intset 并不使用 int8_t 类型来保存任何元素，结构中的这个类型声明只是作为一个占位符使用：在对 contents 中的元素进行读取或者写入时，程序并不是直接使用 contents 来对元素进行索引，而是根据 encoding 的值，对 contents 进行类型转换和指针运算，计算出元素在内存中的正确位置。在添加新元素，进行内存分配时，分配的空间也是由 encoding 的值决定。


## 三、整数集合的操作
我们从一个intset的常见到添加元素，来了解了intset是如何工作的。

### 3.1、创建intset
intset.c/intsetNew 函数创建一个新的 intset ，并设置初始化值：
```c
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    is->length = 0;
    return is;
}
```
可以看到创建一个新的 intset 还是比较简单的， encoding 使用 INTSET_ENC_INT16 作为初始值。intrev32ifbe函数用于字节序的转换，它的作用是如果当前系统是大端字节序，则将数据反转（转为小端）；否则不做任何操作。 这个函数用于处理在不同平台的兼容性。 大多数情况下，我们日常使用的系统都是小端序，即这个函数什么也不做。如果你不了解大小端可以直接忽略这个函数，并不影响继续阅读代码。


### 3.2、添加新元素到 intset
添加新元素到 intset 的工作由 intsetAdd 函数完成，需要处理以下三种情况：
* 元素已存在于集合，不做动作；
* 元素不存在于集合，并且添加新元素并不需要升级；
* 元素不存在于集合，但是要在升级之后，才能添加新元素；

并且插入过程需要确保数组中的元素从小到大排序。我们来看具体实现：
```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {   // 在整数集中插入一条数据
    uint8_t valenc = _intsetValueEncoding(value);   // 计算value需要的编码
    uint32_t pos;
    if (success) *success = 1;

    if (valenc > intrev32ifbe(is->encoding)) {  // 如果需要的编码大于当前编码，那就需要更新整数集的编码类型并插入
        return intsetUpgradeAndAdd(is,value);
    } else {
        if (intsetSearch(is,value,&pos)) {   // 如果值已经存在于集合了，直接返回。记录插入失败, 不存在也计算出需要插入的位置pos
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1);  // 给集合大小扩容1个长度
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    _intsetSet(is,pos,value);  // 插入值并更新长度
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```
如上所示，从这段代码中，可以了解到插入一条数据的整体过程。
1.  通过_intsetValueEncoding计算新插入的值，需要使用哪种编码。如当大小在int16_t可表示的范围内，那就用int16_t，否则继续判断int32_t。
2.  判断新值的编码是否大于当前编码，如果是就需要升级，调用 intsetUpgradeAndAdd。如果不需要升级就继续判断元素是否已存在
3.  如果元素存在记录插入失败，并返回。不存在计算出需要插入的位置。
4.  调整整数集合的大小，扩容1个元素的长度、根据插入位置进行插入，如果插入位置小于总长度，就把插入位置往后的元素向后移动。否则就是插入到尾部
5.  调用_intsetSet进行插入。

### 3.3、升级插入
在插入的新数据超过当前intset所能表示的范围时，就需要升级后再插入，实现如下
```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);   // 计算存储新值需要的编码
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;    // 根据value的正负判断是前插还是后插， 负数就前插，正数后插，因为从小到大排序。

    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));   // 遍历原来的内容从最后一个元素开始倒序，更新到新的长度，注意这里不会覆盖因为新的内存大于旧的

    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```
如上所示，当扩容到更大的内存是，会桶原来的数据进行倒序遍历，然后进行更新，这样不会覆盖原来的内存中的内容。另外注意点是这个函数中没有计算新数据应该插入的位置，其实也很简单，新数据肯定是最大的，应该插入到尾部。因为原来的数据都能使用更小的数据类型表示。

### 3.4、数据查找
在插入数据过程中，我们看到需要通过数据查找先找到应该插入的位置，这里我们就看下查找是如何实现的：
```c
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {     // 在整数集合中，查找一个值，如果设置了出参pos。记录该值在集合中的位置
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    if (intrev32ifbe(is->length) == 0) {  // 当集合为空时，查找不到直接返回0 且出参pos也置0 
        if (pos) *pos = 0;
        return 0;
    } else {
        if (value > _intsetGet(is,max)) {      // 如果查找的值大于集合中的最大值或者小于最小值，直接返回0, 说明不存在
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }

    while(max >= min) {       // 使用二分查找法
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {       // 找到了就赶回value的位置
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;   // 如果没找到，就返回小于value的那个值的位置
        return 0;
    }
}
```
如上所示，首先判断集合是否为空，或者被查找的数据是否在当前整型的范围内，如果不在，直接报不存在。 如果集合可能存在就利用二分法查找。

### 3.5、intset的其他操作
删除单个元素的工作由 intsetRemove 操作， 它先调用 intsetSearch 找到需要被删除的元素在 contents 数组中的索引， 然后使用内存移位操作，将目标元素从内存中抹去， 最后，通过内存重分配，对 contents 数组的长度进行调整

Intset 不支持降级操作。它被定位为一种受限的中间表示， 只能保存整数值， 而且元素的个数也会受到限制，当元素过多时会切换为使用其他的数据结构，因此没有必要进行太复杂的操作，


## 四、总结
Intset 用于有序、无重复地保存多个整数值，会根据元素的值，自动选择该用什么长度的整数类型来保存元素。当一个位长度更长的整数值添加到 intset 时，需要对 intset 进行升级，新 intset 中每个元素的位长度，会等于新添加值的位长度，但原有元素的值不变。升级会引起整个 intset 进行内存重分配，并移动集合中的所有元素，这个操作的复杂度为 O(N)，Intset 是有序的，程序使用二分查找算法来实现查找操作，复杂度为 O(lgN)。





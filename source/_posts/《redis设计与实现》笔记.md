---
title: 《redis设计与实现》笔记
date: 2018-11-05 13:19:29
tags:
    - redis
    - 数据库
---

记录阅读这本书时觉得有用的东西，也许很零散。(第一部分)

## 字符串

redis的字符串底层实现为简单动态字符串`simple-dynamic-string`。
```c
struct sdshdr{
    int len; 字符个数，不包括‘/0’
    int free; 没有使用的字符个数
    char buf[]; 字符数组
}
```

<!--more-->

### 空间预分配

空间预分配用于优化SDS的字符串增长操作。
1. 如果对sds修改后，len小于1MB，那么分配与len同样大小的未使用空间。buf的长度变为`len+buf+1`
2. 如果修改后，lend大于等于1MB,那么分配1MB的未使用空间。buf的长度变为`len+1MB+1byte`

### 惰性空间释放

用于优化SDS的字符串缩短操作
1. 进行缩短操作时，不减少buf的长度，将缩短的部分加入free。
2. 同时提供了真正释放未使用空间的API，不用担心内存浪费

### C字符串与SDS对比

| C字符串 | SDS |
| ------ | ------ |
| 获取字符串长度复杂度O(n) | 复杂度O(1) |
| API不安全，可能缓冲区溢出 |  API安全     |
| 修改字符串长度N次必然需要执行N次内存重分配 | 修改字符串长度N次最多需要N次内存重分配 |
| 只能保存文本数据 | 可以保存文本或者二进制数据 |
| 可以使用所有<string.h>库中的函数 | 可以使用部分 |

## 链表

链表在redis中应用广泛，当一个列表键包含了数量较多的元素，又或者列表包含的元素都是比较长的字符串时，redis就会用链表作为列表键的底层实现。

### 特性

1. 双向， 无环
1. 有表头，表尾指针
1. 带链表长度计数器
1. 多态，可以保存不同类型的值

## 字典

应用广泛。整个数据库就是一个kv数据库，其次哈希键这个数据结构的底层也是字典。

哈希表结构
```c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值，总是等于size-1
    unsigned long sizemask;

    // 该哈希表已有节点数量
    unsigned long used;
} dictht;
```

哈希表节点结构
```c
typedef struct dictEntry {
    void *key;

    union(
        void *val;
        uint64_tu64;
        int64_ts64;
    ) v;

    struvt dictEntry *next;
} dictEntry;
```

字典结构
```c
typedef struct dict {
    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash索引， 当rehash不在进行时，值为-1
    int trehashidx;
} dict;
```

字典保存在ht[0]中，ht[1]用来对哈希表进行rehash。使用MurmueHash算法。数组+链表的方式解决哈希冲突。

### rehash优化-渐进式rehash

如果字典表过大，一次完成rehash会造成数据库在一段时间停止服务。步骤如下：

1. 为ht[1]分配空间
1. 在字典中维持一个索引计数器变量rehashidx，并将值置为0
1. 在rehash期间，每次对字典进行添加删除，查找或者更新操作时，顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，完成后rehashidx+1
1. 最终全部rehash到ht[1]， 将rehashidx设为-1表示完成。把ht[0]的哈希表指向ht[1]的哈希表，ht[1]指向null。

## 压缩列表

压缩列表是列表和哈希的底层实现之一，当一个列表键只包含少量列表项，并且是小整数值或短字符串。同时满足以下两点，使用压缩链表。

1. 列表对象保存的所有字符串长度都小于64字节
1. 列表对象保存的元素数量小于512个

## 跳跃表

redis使用跳跃表作为有序集合的底层实现之一，同时也用在集群节点中用作内部数据结构。可以参考网上跳表数据结构分析。（个人感觉与二分的思想是一致的，有序情况下降低复杂度）

## set

集合数据结构靠字典实现，集合的值为字典的键，对应的值为null。
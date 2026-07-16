---
title: Redis系列文章之 List
author: Walter
pubDatetime: 2026-07-16T03:00:00Z
featured: false
draft: false
tags:
  - backend
  - redis
description: 本文介绍了 Redis 中列表（List）的底层实现：ziplist 与 quicklist 的结构、编码转换条件，以及常用命令的时间复杂度。
---

> 参考：[Redis List底层数据结构解析：原理与实现细节](https://www.trae.cn/article/706616834)

## Redis List 数据结构概述

Redis 的 List 类型是一个双向链表结构，支持在列表两端进行高效的 push/pop 操作。但在底层实现上，Redis 并没有简单地使用传统的双向链表，而是采用了更加内存友好的设计方案。

Redis List 的底层实现经历了重要的演进过程：

- Redis 3.2 之前：主要使用 **ziplist（压缩列表）** 和 **linkedlist（双向链表）** 两种编码
- Redis 3.2 起：引入 **quicklist（快速列表）** 作为 List 的主要底层实现，结合了 ziplist 的内存效率和双向链表的操作灵活性

## ziplist 压缩列表

### ziplist 的结构设计

ziplist 是 Redis 为节约内存设计的一种特殊编码结构。它将多个小的数据项紧凑地存储在一块连续的内存空间中：

```c
// ziplist 的内存布局
// [zlbytes][zltail][zllen][entry1][entry2]...[entryN][zlend]

struct ziplist {
    uint32_t zlbytes;        // 整个 ziplist 占用的字节数
    uint32_t zltail;         // 最后一个 entry 的偏移量
    uint16_t zllen;          // entry 的数量
    unsigned char[] entries; // 实际的 entry 数据
    unsigned char zlend;     // 结束标记，恒为 0xFF
};
```

每个 entry（条目）内部又包含三个部分：

```c
// entry 的内部结构
// [prevlen][encoding][data]

struct ziplist_entry {
    unsigned char[] prevlen;   // 前一个 entry 的长度
    unsigned char[] encoding;  // 当前 entry 的编码方式
    unsigned char[] data;      // 实际的数据内容
};
```

### 变长编码

ziplist 采用变长编码存储长度信息，根据数据大小动态调整存储空间：

- **prevlen 字段**：前一个 entry 长度小于 254 字节时用 1 字节；否则用 5 字节（1 字节标记 + 4 字节长度）
- **encoding 字段**：按内容类型和长度采用不同编码
  - 整数：使用特殊编码标识
  - 字符串：按长度使用 1 字节或 5 字节编码

这种机制使 ziplist 在处理小数据时能大幅节省内存。例如，存储短字符串列表时，相比传统双向链表节点，可节省约 40% 的内存。

### ziplist 的操作特点

ziplist 支持双向遍历，通过 prevlen 可快速定位前一个 entry。但插入和删除相对复杂：

1. **插入**：需要重新分配内存，并将插入点后的所有数据向后移动
2. **删除**：需要重新分配内存，并将删除点后的所有数据向前移动
3. **级联更新**：当插入或删除导致后续 entry 的 prevlen 字段长度变化时，可能触发级联更新

因此 ziplist 适合数据量较小、操作不频繁的场景；大规模数据操作时性能会明显下降。

### ziplist 的性能瓶颈

1. **级联更新**：后续 entry 的 prevlen 需要扩容时会连锁更新，最坏情况需更新整个 ziplist
2. **内存重分配频繁**：每次插入/删除都可能重新分配内存，大列表开销大
3. **查找效率低**：只能顺序遍历，时间复杂度为 O(N)

## quicklist 快速列表

### quicklist 的设计思想

为解决 ziplist 的性能瓶颈，Redis 3.2 引入了 quicklist。它本质上是由多个 ziplist 组成的双向链表，在保持内存紧凑的同时，提升大规模数据操作的性能：

```c
typedef struct quicklist {
    quicklistNode *head;              // 头节点
    quicklistNode *tail;              // 尾节点
    unsigned long count;              // 所有 ziplist 中的元素总数
    unsigned long len;                // quicklistNode 的数量
    int fill : QL_FILL_BITS;          // 单个 ziplist 的最大容量
    unsigned int compress : QL_COMP_BITS; // 压缩深度
} quicklist;

typedef struct quicklistNode {
    struct quicklistNode *prev; // 前一个节点
    struct quicklistNode *next; // 后一个节点
    unsigned char *zl;          // 指向 ziplist 的指针
    unsigned int sz;            // ziplist 的大小
    unsigned int count : 16;    // ziplist 中的元素数量
    unsigned int encoding : 2;  // 编码方式：RAW==1 or LZF==2
    unsigned int container : 2; // 容器类型：NONE==1 or ZIPLIST==2
    unsigned int recompress : 1;
    unsigned int attempted_compress : 1;
    unsigned int extra : 10;
} quicklistNode;
```

### 分片策略

quicklist 的核心优化是**分片存储**：将大量元素分散到多个较小的 ziplist 中，而不是集中在一个巨大的 ziplist 里：

1. **缩小级联更新影响范围**：限制在单个节点内
2. **降低内存重分配成本**：插入/删除只需重分配单个 ziplist
3. **提升查找效率**：先定位到具体 quicklistNode，再在较小的 ziplist 中查找

### 压缩深度

quicklist 引入了**压缩深度**机制：

```c
#define QUICKLIST_NOCOMPRESS 0    // 不压缩
#define QUICKLIST_COMPRESS_MAX 16 // 最大压缩深度
```

工作原理：链表两端节点保持不压缩，中间节点可进行 LZF 压缩。这基于一个观察——**列表两端元素的访问频率通常高于中间元素**。

例如压缩深度为 2 时，头尾各 2 个节点不压缩，中间节点会被压缩存储，从而在内存占用和访问性能之间取得平衡。

### quicklist 的操作优化

**插入优化**：

- 优先在当前节点插入，避免创建新节点
- 节点满时采用**分裂策略**：将当前节点分裂成两个，再在新节点中插入
- 支持向前/向后插入，按插入位置选择更优方案

**删除优化**：

- 节点元素过少时触发**合并策略**：合并相邻节点
- 合并会考虑内存效率，避免过度合并导致后续插入频繁分裂

**查找优化**：

- 先在 quicklistNode 层面定位，再在 ziplist 中查找
- 索引访问时维护累计计数器加速定位

## 编码与配置参数

Redis 通过配置参数控制 List 底层行为（可在 `redis.conf` 中配置）：

```conf
list-max-ziplist-size -2    # 单个 ziplist 的最大容量
list-compress-depth 0       # quicklist 压缩深度
```

**list-max-ziplist-size** 决定单个 ziplist 的容量上限：

- **正值**（如 5）：ziplist 最多包含 5 个元素
- **负值**：表示 ziplist 的最大内存大小
  - `-1`：最大 4KB
  - `-2`：最大 8KB（默认）
  - `-3`：最大 16KB
  - `-4`：最大 32KB
  - `-5`：最大 64KB

## List 操作命令的时间复杂度

| 命令 | 时间复杂度 | 底层实现说明 |
| --- | --- | --- |
| LPUSH/RPUSH | O(1) | 在 quicklist 头尾节点操作，通常不需要遍历 |
| LPOP/RPOP | O(1) | 直接从 quicklist 头尾节点移除元素 |
| LLEN | O(1) | quicklist 维护总元素计数，无需遍历 |
| LINDEX | O(N) | 需遍历 quicklist 找到目标节点，再在 ziplist 中定位 |
| LINSERT | O(N) | 需找到插入位置，可能触发节点分裂/合并 |
| LREM | O(N) | 需遍历元素进行匹配删除 |
| LSET | O(N) | 需定位到具体位置进行修改 |
| LRANGE | O(S+N) | S 是 start 偏移量，N 是返回元素数量 |

### LPUSH/RPUSH 优化

当在列表头部 push 时，quicklist 会优先尝试写入当前头节点：

```c
void quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head;

    // 检查是否可以在当前头节点插入
    if (orig_head && orig_head->count < fill_limit &&
        orig_head->sz + sz < ZIPLIST_HEADROOM) {
        quicklist->head->zl = ziplistPush(quicklist->head->zl, value, sz, ZIPLIST_HEAD);
        quicklist->head->count++;
        quicklist->head->sz += sz;
    } else {
        // 创建新的 quicklistNode
        quicklistNode *node = quicklistCreateNode();
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);
        node->count = 1;
        node->sz = sz;
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }
    quicklist->count++;
}
```

这使得 LPUSH/RPUSH 在大多数情况下能保持 O(1)。

### LINDEX 的双阶段查找

```c
unsigned char *quicklistIndex(quicklist *quicklist, long long idx) {
    quicklistNode *node;
    unsigned char *p;
    unsigned long accum = 0;

    // 阶段1：定位到目标 quicklistNode
    if (idx >= 0) {
        node = quicklist->head;
        while (node && accum + node->count <= idx) {
            accum += node->count;
            node = node->next;
        }
    } else {
        idx = (-idx) - 1;
        node = quicklist->tail;
        while (node && accum + node->count <= idx) {
            accum += node->count;
            node = node->prev;
        }
    }

    // 阶段2：在目标 ziplist 中精确定位
    if (node) {
        long long offset = idx - accum;
        p = ziplistIndex(node->zl, offset);
        return p;
    }
    return NULL;
}
```

## 总结

1. **内存与性能的平衡**：ziplist 极致省内存，quicklist 提升操作性能
2. **分片存储**：多个小 ziplist 组成双向链表，限制级联更新和重分配开销
3. **合理配置**：按数据规模调整 `list-max-ziplist-size` 和 `list-compress-depth`
4. **访问模式**：优先使用 O(1) 的头尾操作，批量操作用 pipeline，避免频繁的中间位置 O(N) 操作

## 参考文献

1. [Redis 官方文档 - Lists](https://redis.io/docs/data-types/lists/)
2. Redis 源码：https://github.com/redis/redis
3. 《Redis设计与实现》- 黄健宏
4. [quicklist.c](https://github.com/redis/redis/blob/unstable/src/quicklist.c)
5. [ziplist.c](https://github.com/redis/redis/blob/unstable/src/ziplist.c)

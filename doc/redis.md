# redis #

```
   https://www.cnblogs.com/ysocean/p/9080942.html
   1. 动态字符串(all)
   2. 双向链表(list)
   3. 跳表(zset)：https://www.cnblogs.com/aspirant/p/11475295.html
   4. 字典(hash)
```

## 基本数据结构 ##

| 数据结构 | 名称     | desc                |
| -------- | -------- | ------------------- |
| string   | kv       | 键值对              |
| list     | 列表     | 一个有序数据列表    |
| set      | set      | 一个无序的数据集合  |
| hash     | 字典     | 类似于java的HashMap |
| zset     | 排序集合 | 维护排序的数据集合  |

## 底层数据结构 ##

### 动态字符串 ###

```c++
struct sdshdr{
     //记录buf数组中已使用字节的数量，等于 SDS 保存字符串的长度
     int len;
     //记录 buf 数组中未使用字节的数量
     int free;
     //字节数组，用于保存字符串，也是以‘/0’结束
     char buf[];
}
```

<img src="https://images2018.cnblogs.com/blog/1120165/201805/1120165-20180528075607627-218845583.png" style="zoom:67%;" />

优点：

1. 常数复杂度获取字符串长度
2. 使用 len 检查是否满足需求，能够避免缓冲区溢出
3. 因为存在 len 和 free 减少字符串内存空间分配次数
   1. 空间预分配，在申请的时候，比实际申请的多申请一些。
   2. 惰性释放，对字符串进行缩短操作时，程序不立即使用内存重新分配来回收缩短后多余的字节，而是使用 free 属性将这些字节的数量记录下来，等待后续使用。
4. 使用 len 来判断是否结束，二进制安全。
5. 但是依然以 '\0' 结束，可以复用 c 的语言哭库。

### 双向链表 ###

```c++
typedef  struct listNode{
       //前置节点
       struct listNode *prev;
       //后置节点
       struct listNode *next;
       //节点的值
       void *value;  
}listNode;
// --- -- - -- - - - - -- - - - - - - --  --- -- - -- - - - - -- - - - - - - -- 
typedef struct list{
     //表头节点
     listNode *head;
     //表尾节点
     listNode *tail;
     //链表所包含的节点数量
     unsigned long len;
     //节点值复制函数
     void (*free) (void *ptr);
     //节点值释放函数
     void (*free) (void *ptr);
     //节点值对比函数
     int (*match) (void *ptr,void *key);
}list;
```

![](https://images2018.cnblogs.com/blog/1120165/201805/1120165-20180528074403440-111834793.png)

redis底层使用的是双向链表：

1. 双端：链表具有前置节点和后置节点的引用，获取这两个节点时间复杂度都为O(1)。
2. 无环：表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL,对链表的访问都是以 NULL 结束。　　
3. 长度计数器：通过 len 属性获取链表长度的时间复杂度为 O(1)。
4. 多态：链表节点使用 void* 指针来保存节点值，可以保存各种不同类型的值。

### 跳表 ###

<img src="https://images2018.cnblogs.com/blog/1120165/201805/1120165-20180528210921601-949409375.png" style="zoom:75%;" />

1. 多层结构，每层都是有序链表。
2. 链表节点数量上层少，下层多。最底层的链表包含了所有的元素。
3. 如果一个元素出现在某一层的链表中，那么在该层之下的链表节点的值均为该值。

跳表，使用的是空间换时间的思想，通过构建多级索引来提高查询效率，实现基于链表的“二分查找”，跳表是一种动态的数据结构，支持快速的查找、插入和删除操作，时间复杂度是 O(logn)。

其中，插入、删除、查找以及迭代输出有序序列这几个操作，红黑树也可以完成，时间复杂度和跳表是一样的。**但是，按照区间查找数据这个操作，红黑树的效率没有跳表高。跳表可以在 O(logn)时间复杂度定位区间的起点，然后在原始链表中顺序向后查询就可以了，这样非常高效。**

### 压缩列表 ###

压缩列表（ziplist）是Redis为了节省内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构，一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

**压缩列表的优点在于能够一定程度地节约内存。**

压缩列表的原理：压缩列表并不是对数据利用某种算法进行压缩，而是将数据按照一定规则编码在一块连续的内存区域，目的是节省内存。

<img src="https://images2018.cnblogs.com/blog/1120165/201805/1120165-20180528215852732-1088896020.png" style="zoom:80%;" />

压缩列表的各个组成部分：

1. zlbytes：当前压缩列表的总的字节数
2. zltail：指向尾节点的地址
3. zllen：当前压缩列表的节点数
4. entryX：列表的节点
5. zlend：列表的尾端，特殊值0xFF

<img src="https://images2018.cnblogs.com/blog/1120165/201805/1120165-20180528223605060-899108663.png" style="zoom:67%;" />

1. previous_entry_ength：记录压缩列表前一个字节的长度。即当前节点位置减去上一个节点的长度即得到上一个节点的起始位置，压缩列表可以从尾部向头部遍历。这么做很有效地减少了内存的浪费。
2. encoding：节点的encoding保存的是节点的content的内容类型以及长度，encoding类型一共有两种，一种字节数组一种是整数，encoding区域长度为1字节、2字节或者5字节长。
3. content：content区域用于保存节点的内容，节点内容类型和长度由encoding决定。

### 字典 ###

```c++
typedef struct dictht{
     //哈希表数组
     dictEntry **table;
     //哈希表大小
     unsigned long size;
     //哈希表大小掩码，用于计算索引值
     //总是等于 size-1
     unsigned long sizemask;
     //该哈希表已有节点的数量
     unsigned long used;
 
}dictht
```

```c++
typedef struct dictEntry{
     //键
     void *key;
     //值
     union{
          void *val;
          uint64_tu64;
          int64_ts64;
     }v;
 
     //指向下一个哈希表节点，形成链表
     struct dictEntry *next;
}dictEntry
```

1. hash冲突：使用拉链法，来解决hash冲突。

2. 计算桶的位置：计算元素的hash值，然后hash值和当前的 size - 1 相与操作。然后得出这个元素所在的数组的位置。

   index = hash & (size - 1)，和Java中的HashMap一个思路。

3. 扩容：当到达指定的负载因子，那么进行扩容操作。扩容，将当前数组的长度扩展到原来的两倍。

   + 如果没有进行持久化操作，负载因子为1，如果进行持久化，负载因子为5。
   + 服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子阈值为 1 。
   + 服务器目前正在执行BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子阈值 5 。

**和 Java1.7 的 HashMap 可以说是非常相似了。**



## 分布式锁 ##

#### setnx ####

下面是一个分布式锁的demo：

```java
package com.xuzhongjian;

import java.util.Random;
import java.util.UUID;

/**
 * @author zjxu97 at 3/2/21 11:09 AM
 */
public class Solution {

    private RedisOperation redis = new RedisOperation();
    private static final String KEY = "redis-lock-key";

    public static void main(String[] args) {
        Solution solution = new Solution();
        UUID uuid = UUID.randomUUID();
        String token = uuid.toString();
        try {
            if (solution.getRedisLock(token, 1000)) {
                System.out.println("hello world!");
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            solution.releaseLock(token);
        }

    }

    /**
     * 获取锁
     *
     * @param token   每个线程唯一的token
     * @param timeout 最大等待时间
     * @return 获取锁成功与否
     * @throws InterruptedException 中断异常
     */
    public boolean getRedisLock(String token, long timeout) throws InterruptedException {
        boolean success = false;
        long ts = System.currentTimeMillis();
        while (!success && System.currentTimeMillis() - ts < timeout) {
            success = redis.setnx(KEY, token);
            Thread.sleep(2);
        }
        return success;
    }

    /**
     * 释放锁
     *
     * @param token 和获取锁的时候一样，需要一个唯一的token来解锁，只有当唯一的token匹配上了，才能解锁。
     */
    public void releaseLock(String token) {
        String s = redis.get(KEY);
        if (token.equals(s)) {
            redis.del(KEY);
        }
    }
}
```

#### redisson ####

##### Redlock实现 #####

https://zhuanlan.zhihu.com/p/266933567

##### 可重入 #####

https://segmentfault.com/a/1190000021199037

## Hash环 ##

https://blog.csdn.net/kevinxxw/article/details/105908101
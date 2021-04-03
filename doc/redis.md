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

### setnx ###

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

### redisson ###

#### 加锁 ####

**lua脚本**

```java
		// 没有加锁，直接加锁
		if (!redis.exist(key)) {
            redis.hincrby(key, clientId, 1);
            redis.pexpire(key, timeout);
            return null;
        }
		// 加的锁就是当前这个线程的锁，可重入，hash结构 value ++
        if (redis.hexist(key, clientId)) {
            redis.hincrby(key, clientId, 1);
            redis.pexpire(key, timeout);
            return null;
        }
		// 如果互斥，返回当前持有锁的线程的过期时间
        return redis.pttl(key);
```

```java
	@Override
    public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
        long time = unit.toMillis(waitTime);
        long current = System.currentTimeMillis();
        long threadId = Thread.currentThread().getId();
        // 1.尝试获取锁
        Long ttl = tryAcquire(leaseTime, unit, threadId);
        // lock acquired
        if (ttl == null) {
            return true;
        }

        // 申请锁的耗时如果大于等于最大等待时间，则申请锁失败.
        time -= System.currentTimeMillis() - current;
        if (time <= 0) {
            acquireFailed(threadId);
            return false;
        }

        current = System.currentTimeMillis();

        /**
         * 2.订阅锁释放事件，并通过 await 方法阻塞等待锁释放，有效的解决了无效的锁申请浪费资源的问题：
         * 基于信息量，当锁被其它资源占用时，当前线程通过 Redis 的 channel 订阅锁的释放事件，一旦锁释放会发消息通知待等待的线程进行竞争.
         *
         * 当 this.await 返回 false，说明等待时间已经超出获取锁最大等待时间，取消订阅并返回获取锁失败.
         * 当 this.await 返回 true，进入循环尝试获取锁.
         */
        RFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
        // await 方法内部是用 CountDownLatch 来实现阻塞，获取 subscribe 异步执行的结果（应用了 Netty 的 Future）
        if (!subscribeFuture.await(time, TimeUnit.MILLISECONDS)) {
            if (!subscribeFuture.cancel(false)) {
                subscribeFuture.onComplete((res, e) -> {
                    if (e == null) {
                        unsubscribe(subscribeFuture, threadId);
                    }
                });
            }
            acquireFailed(threadId);
            return false;
        }

        try {
            // 计算获取锁的总耗时，如果大于等于最大等待时间，则获取锁失败.
            time -= System.currentTimeMillis() - current;
            if (time <= 0) {
                acquireFailed(threadId);
                return false;
            }

            /**
             * 3.收到锁释放的信号后，在最大等待时间之内，循环一次接着一次的尝试获取锁
             * 获取锁成功，则立马返回 true，
             * 若在最大等待时间之内还没获取到锁，则认为获取锁失败，返回 false 结束循环
             */
            while (true) {
                long currentTime = System.currentTimeMillis();

                // 再次尝试获取锁
                ttl = tryAcquire(leaseTime, unit, threadId);
                // lock acquired
                if (ttl == null) {
                    return true;
                }
                // 超过最大等待时间则返回 false 结束循环，获取锁失败
                time -= System.currentTimeMillis() - currentTime;
                if (time <= 0) {
                    acquireFailed(threadId);
                    return false;
                }

                /**
                 * 6.阻塞等待锁（通过信号量(共享锁)阻塞,等待解锁消息）：
                 */
                currentTime = System.currentTimeMillis();
                // ！！！ 下面的信号量怎么获得呢？释放锁，当前的锁被释放，就会产生信号量
                if (ttl >= 0 && ttl < time) {
                    //如果剩余时间(ttl)小于wait time ,就在 ttl 时间内，从Entry的信号量获取一个许可(除非被中断或者一直没有可用的许可)。
                    getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } else {
                    //则就在wait time 时间范围内等待可以通过信号量
                    getEntry(threadId).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
                }

                // 更新剩余的等待时间(最大等待时间-已经消耗的阻塞时间)
                time -= System.currentTimeMillis() - currentTime;
                if (time <= 0) {
                    acquireFailed(threadId);
                    return false;
                }
            }
        } finally {
            // 7.无论是否获得锁,都要取消订阅解锁消息
            unsubscribe(subscribeFuture, threadId);
        }
        // return get(tryLockAsync(waitTime, leaseTime, unit));
    }
```

流程分析：

1. 尝试使用lua脚本获取锁，如果返回null，那么说明加锁成功。如果返回一个数值，那么说明加锁失败，这个数值是当前的持有这个锁的线程的剩余时间。
2. 如果获取锁失败，那么使用当前这个客户端的 clientId 来订阅锁的释放事件。一旦发现锁释放，就会通知线程开启线程竞争。超过了等待时间，锁没有被释放，那么获得锁失败。
3. 如果释放了，就会循环尝试获取锁。通过redis的订阅和发布 + 信号量等待解锁的消息。在循环等待的过程中，一旦等待的时间超过了获取锁的timeout，那么获取锁失败。

#### 看门狗 ####

客户端加锁的默认生存的时间是30s，如果超过了30s，客户端需要继续持有这把锁。需要使用到看门狗这个机制：

一旦获取锁成功，就会开启一个续期的看门狗机制。

看门狗机制其实就是一个后台定时任务线程，获取锁成功之后，会将持有锁的线程放入到一个redis更新过期时间的map，里面，然后每隔 10 秒检查一下，还持有锁 key（判断客户端是否还持有 key，其实就是遍历 map 里面线程 id 然后根据线程 id 去 Redis 中查，如果存在就会延长 key 的时间），那么就会不断的延长锁 key 的生存时间。

#### 解锁 ####

| 参数    | 示例值                                 | 含义                          |
| ------- | -------------------------------------- | ----------------------------- |
| KEYS[1] | my_lock_name                           | 锁名                          |
| KEYS[2] | redisson_lock__channel:{my_lock_name}  | **解锁消息PubSub频道**        |
| ARGV[1] | 0                                      | **redisson定义0表示解锁消息** |
| ARGV[2] | 30000                                  | 设置锁的过期时间；默认值30秒  |
| ARGV[3] | 58c62432-bb74-4d14-8a00-9908cc8b828f:1 | 唯一标识；同加锁流程          |

```lua

-- 若锁不存在：则直接广播解锁消息，并返回1
if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('publish', KEYS[2], ARGV[1]);
    return 1; 
end;
 
-- 若锁存在，但唯一标识不匹配：则表明锁被其他线程占用，当前线程不允许解锁其他线程持有的锁
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
    return nil;
end; 
 
-- 若锁存在，且唯一标识匹配：则先将锁重入计数减1
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); 
if (counter > 0) then 
    -- 锁重入计数减1后还大于0：表明当前线程持有的锁还有重入，不能进行锁删除操作，但可以帮忙更新过期时间
    redis.call('pexpire', KEYS[1], ARGV[2]); 
    return 0; 
else 
    -- 锁重入计数已为0：间接表明锁已释放了。直接删除掉锁，并广播解锁消息，去唤醒那些争抢过锁但还处于阻塞中的线程
    redis.call('del', KEYS[1]); 
    redis.call('publish', KEYS[2], ARGV[1]); 
    return 1;
end;
 
return nil;
```



#### Redlock思路 ####

1. 在Redis的分布式环境中，我们假设有N个Redis master。这些节点完全互相独立，不存在主从复制或者其他集群协调机制。
2. 我们确保将在N个实例上使用与在Redis单实例下相同方法获取和释放锁。现在我们假设有5个Redis master节点，同时我们需要在5台服务器上面运行这些Redis实例，这样保证他们不会同时都宕掉。
3. 为了取到锁，客户端应该执行以下操作:
   + 获取当前Unix时间，以毫秒为单位。
   + 依次尝试从5个实例，使用相同的key和具有唯一性的value(例如UUID)获取锁。
   + 当向Redis请求获取锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。
     例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。
   + 如果服务器端没有在规定时间内响应，客户端应该尽快尝试去另外一个Redis实例请求获取锁。
   + 客户端使用当前时间减去开始获取锁时间(步骤1记录的时间)就得到获取锁使用的时间。
     当且仅当从大多数（ > N / 2）的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。
   + 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间(步骤3计算的结果)。
     如果因为某些原因获取锁失败，客户端应该在所有的Redis实例上进行解锁。（即便某些Redis实例根本就没有加锁成功，防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁。）

#### 可重入 ####

用的 hash 参数

key - client  --> 1（2、3、4）

**在上面的 lua 加锁，解锁脚本中均有体现。**

## 缓存 ##

正常情况下，请求到达服务器，然后，服务器需要获取数据，如果存在redis缓存，那么先去redis缓存中取得数据，如果redis中没有需要的数据，就再去DB（DB一般为速度较慢的硬盘存储）获取。

![img](https://img-blog.csdn.net/20180919143214712?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tvbmd0aWFvNQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 缓存穿透 ###

所需要的数据在数据库中不存在，所以在缓存中也不存在。

然后这样的请求每次到达，都需要访问一次数据库和一次缓存。进行了两次无效的查询，而且查数据库，效率也低，对服务器也造成了压力。

解决办法：

1. 使用布隆过滤器，将数据的 key 存储在一个足够大的bitmap中，用 index - bit 来存储这个数据是否存在在数据库中。这样就拦截了对数据库无效的查询。
2. 对查询的空数据进行缓存，将其 value 设置成空，这样下次同样的查询到达之后，就不会去查数据库。

### 缓存击穿 ###

指的是一个缓存 key 非常的热点，一直有大量的数据在对这个key进行大量的访问。这个key在失效的瞬间，大量的请求打到 DB 上，然后 DB 压力过大甚至死机。

解决方法：

1. 互斥锁

   从缓存中获取不到数据。加上互斥锁，从数据库中查询，然后将数据写到缓存中，最后解开互斥锁。

2. 热点数据永不过期

### 缓存雪崩 ###

大量缓存在同一时刻到期，然后数据请求到达缓存后，没有缓存，然查询DB，造成DB压力过大。

解决方法：

1. 过期时间避免设置在同一时刻过期
2. 分布式缓存，将缓存放置到不同的节点中
3. 热点数据永不过期

## Hash环  ##

参考 https://blog.csdn.net/kevinxxw/article/details/105908101

### 使用hash ###

#### 思路 ####

在多个主库的时候，可以使用下面的方式来获得master服务器的位置：

```java
        int hash = hash(key);
        int masterIndex = hash % MASTER_SIZE;
```

这样提高了性能。在master服务器的数量不变的时候，可以快速的找到这个key对应的数据库位于的服务器的位置。

#### 问题 ####

当master扩容，或者master需要摘掉的时候，MASTER_SIZE的大小将改变。造成原有的key对应的masterIndex改变：

```java
		hash % (MASTER_SIZE + 1) != hash % MASTER_SIZE
```

这样就会导致在一瞬间所有的缓存都失效，造成缓存雪崩。

### 一致性hash ###

#### 思路 ####

从对key取模，变成了对master服务器取模。对 2<sup>32 </sup>取模，所以取出来的index的位置可以的范围是[ 0 , 2<sup>31</sup>-1 ] ，这中间的值刚好是 2<sup>32 </sup> 个取值。相当于是 2<sup>32 </sup> 点，这些点用来放置我们的机器。

如下图所示，从0开始，顺时针自增，直到 2<sup>31</sup>-1 ，一共 2<sup>32 </sup> 个点。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waWMxLnpoaW1nLmNvbS84MC92Mi1mZDQ0YWI3MWM4MzRmM2ZlNDU4YTZmNzZmMzk5N2Y5OF83MjB3LmpwZw?x-oss-process=image/format,png)

然后选择对 master redis机器进行hash，将其hash到这个圆上。（可以使用机器的 ip 来进行 hash ）

<img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waWMxLnpoaW1nLmNvbS84MC92Mi01MDk5OTNhNDlkNDQ3YjM3ODI3M2U0NTVhMDk1ZGUzY183MjB3LmpwZw?x-oss-process=image/format,png" alt="img" style="zoom:67%;" />

如果机器的 ip 出来的结果是均匀的，那么应该是均匀分布在这个圆上面的。

当有数据插入的时候，对数据的 key 进行 hash，然后他们会落在这个圆上的某个位置，将他们沿着这个圆顺时针移动，然后遇到的第一个 master 就是这个 key 对应的 master 节点。

<img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waWM0LnpoaW1nLmNvbS84MC92Mi00ZmFiNjA3MzVkZmFlMGJmNTExNzA5ZTlkMzM3Nzg5Yl83MjB3LmpwZw?x-oss-process=image/format,png" alt="img" style="zoom:67%;" />

#### 容错性 ####

如果 master nodeC 死机了，被摘下了。然后受影响只有 nodeB 顺时针到 nodeC 之间的这些数据，这些数据会被重定向到 nodeD。只会影响一小部分数据，其他的并不会受影响。

<img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waWMxLnpoaW1nLmNvbS84MC92Mi00ZWJjYjhjMjNiYjY0YTYwODk2YmRlODdkZDU0NjIxNF83MjB3LmpwZw?x-oss-process=image/format,png" alt="img" style="zoom:67%;" />

#### 可扩展性 ####

如果需要在 nodeB 顺时针到 nodeC 之间插入一台新的机器 nodeX，那么受影响的只是 nodeB 顺时针到 nodeX 之间的数据，他们会被重定向到 nodeX上。同样也只会影响一部分数据，其他的数据不会收到影响。

<img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waWMyLnpoaW1nLmNvbS84MC92Mi05Y2RiMWFkYzM3ZWIxYTU0YzExNDIzMjEyMGRhMTQ4NV83MjB3LmpwZw?x-oss-process=image/format,png" alt="img" style="zoom:67%;" />

#### 数据倾斜 ####

hash环会存在的问题是，master 机器在环上分布不均匀。下图，80%以上的数据会被定位到 nodeA 上，而 nodeB 只有不到 20% 的数据。 

<img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waWMzLnpoaW1nLmNvbS84MC92Mi0wMzY4ODQxZTUwMjBkZDA3ZjFlNjdmNDQ5YjQ5YTFiYV83MjB3LmpwZw?x-oss-process=image/format,png" alt="img" style="zoom:60%;" />

出现这个问题的解决方案是，对于数据量较少的节点，可以构虚拟节点，将其放置到环上，这样可以尽量做到节点所对应的数据量做到均衡。
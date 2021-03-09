# redis #

## 基本数据结构 ##

```txt
   https://www.cnblogs.com/ysocean/p/9080942.html
   1. 动态字符串(all)
   2. 双向链表(list)
   3. 跳表(zset)：https://www.cnblogs.com/aspirant/p/11475295.html
   4. 字典(hash)
```





sql 优化：

https://www.cnblogs.com/xiangpeng/p/11032047.html

1. where 和 order 上建立索引
2. 避免 is null 判断
3. 避免使用 != 和 <> 
4. 避免使用 or
5. 避免使用 in 和 not in
6. 避免使用前缀%
7. 不要对索引列进行计算
8. 
## redis：

参考网址：  
http://www.runoob.com/redis/redis-keys.html 这个上面介绍的命令链接可以点击获取详细操作演示。

1.**常用命令：**
- **string**:
    - set name wangxi  
        - key为name,value为wangxi
    - get name 
        - 获取name的值
- **hash**
    -  hmset wxi01 name wangxi sex 男 age 23  
        - 批量设置key-value。wxi01代表Map在缓存的名字
    - hgetall wxi01 
        - 列出所有key-value对。在redis中age的value="23"
    - hset wxi01 age 24  
        - 设置单个字段的值
    - hget wxi01 age    
        - 获取单个字段的值
    - 与HashMap一样，插入相同的key，值就会被覆盖，除非内部的hash冲突，才会有value为list的情况，自己插入的时候是得不到value为list的，在Java的Map<String, List<String>>取出来，在list.add()可以，在redis中没有。
- **list**
    - lpush wxi01 1 2 3 aa 
        - 相当于栈结构,此时lindex wxi01 0 => aa  (lindex通过索引获取列表中的值)
    - rpush wxi02 1 2 3
        - 相当于队列,此时lindex wxi02 0 => 1  
    - llen wxi01 
        - 获取列表的长度
    - lrange wxi01 0 -1
         - 获取列表中所有的元素
- **set**
    - Redis 的 Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。Redis 中集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。
    - sadd wxi01 aa bb aa
        - 添加元素
    - 
- **sorted set**
    - Redis 有序集合和集合一样也是string类型元素的集合,且不允许重复的成员。不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。有序集合的成员是唯一的,但分数(score)却可以重复。集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)
2.**redis的5种数据结构的应用场景：**
- http://www.cleey.com/blog/single/id/808.html

3.

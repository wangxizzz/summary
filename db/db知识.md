1.**mysql的复合索引与单列索引的区别：**
- https://blog.csdn.net/Abysscarry/article/details/80792876 
    - 这篇文章讲述了 复合索引 的前缀匹配规则，利用大量的例子证明，并且讲述了复合索引本质是是建立几个索引
    - 讲述了 复合索引 or 索引失效的原因
    - 讲述了 单列索引 or可以走索引，并且介绍了多个单列索引的匹配规则与存储方式

2.**批量多条件查询：**
- https://segmentfault.com/q/1010000011658327 
具体写法如下：(需要看下执行计划)
```sql
select date_format(create_Time,'%Y-%m-%d %H:%i:%s') as createTime ,reason,operator,remarks,operation from refund_order_log
        where (order_no,channel) in
        <foreach collection="list" item="item" open="(" close=")" separator=",">
            (#{item.orderNo},#{item.channel})
        </foreach>
        and status =1 order by create_time  desc

-- 相当于下面的sql
select * from user where (user_id,type) in ((568,6),(569,6),(600,8));
```

3.**group by 的写法**
- 如果需要使用group by，那么select 的字段，一定需要先group by

4.**update by select 写法**
```sql
update table01 set to_search_id = tmp.package_id from
 (select package_id from table01, table02 where 
 table02.config_id = table01.config_id
and table02.config_info->>'config_type'='daren' and to_search_id ~ '_') as tmp
where tmp.package_id = table01.package_id;

-- 最外层的where tmp.package_id = table01.package_id, 限制了update 的语句范围，防止全表更新
```

5.
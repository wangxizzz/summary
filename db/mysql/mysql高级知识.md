# 1、mysql深分页问题：
假如表中数据有500w
```sql
select * from orders_history where type=8 limit 100,100;
select * from orders_history where type=8 limit 1000,100;
select * from orders_history where type=8 limit 10000,100;
select * from orders_history where type=8 limit 100000,100;
select * from orders_history where type=8 limit 1000000,100;
```
三次查询时间如下：
- 查询100偏移：25ms 24ms 24ms
- 查询1000偏移：78ms 76ms 77ms
- 查询10000偏移：3092ms 3212ms 3128ms
- 查询100000偏移：3878ms 3812ms 3798ms
- 查询1000000偏移：14608ms 14062ms 14700ms

```随着查询偏移的增大，尤其查询偏移大于10万以后，查询时间急剧增加。这种分页查询方式会从数据库第一条记录开始扫描，所以越往后，查询速度越慢，而且查询的数据越多，也会拖慢总查询速度。```

# 解决办法：
## 使用子查询优化：
```这种方式先定位偏移位置的 id，然后往后查询，这种方式适用于 id 递增的情况```
```sql
select * from orders_history where type=8 and
id >= (select id from orders_history where type=8 limit 100000,1)
limit 100;
```
查询时间：1327ms  
这种方式相较于原始一般的查询方法，将会增快数倍。

## 使用 id 限定优化

这种方式假设数据表的id是连续递增的，则我们根据查询的页数和查询的记录数可以算出查询的id的范围，可以使用 id between and 来查询：
```sql
select * from orders_history where type=2
and id between 1000000 and 1000100 limit 100;
```
查询时间：15ms 12ms 9ms  

这种方式需要依赖于前端把偏移量与limit由前端计算好，带到后端。

**提升性能，本质是利用字段索引的查找，然后再查找其他字段。**

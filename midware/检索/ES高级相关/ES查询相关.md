# 关于ES的数据更新删除：
- 如果是删除操作，commit的时候会生成一个.del文件，里面将某个doc标识为deleted状态，那么搜索的时候根据.del文件就知道这个doc被删除了，无法搜索到
- 如果是更新操作，就是将原来的doc标识为deleted状态，然后新写入一条数据。
## 最佳实践
如果ES频繁进行更新操作：

不做更新操作，每次做插入操作。在数据插入时在db保存一个version字段，每次插入更新把最新的version字段更新到es对应的列，查询时根据最新的version即可查询出最新的数据。后续起定时任务删除旧version的数据。

如果直接做更新操作，会影响性能。

# 多字段聚合参考 官网API：  
https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/_structuring_aggregations.html

# ES坑相关：
- group by 操作，如果不设置大小，那么如果group by的字段过多，就会丢弃其他字段的分组统计操作。


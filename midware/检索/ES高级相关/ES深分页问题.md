# ES的深分页问题：
- 官网： https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-body.html#request-body-search-scroll

# 深度分页问题大致可以分为两类
- 随机深度分页：随机跳转页面
- 滚动深度分页：只能一页一页往下查询

## 传统from+size
### 用法：
```bash
GET /_search?size=5&from=10
```

### 存在的问题：
对于大量的数据而言，我们尽量避免使用from+size这种方法。这里的原因是index.max_result_window的默认值是10K，也就是说from+size的最大值是1万。搜索请求占用堆内存和时间与from+size成比例。因为这种方式需要每次把所有数据都查出来，然后取对应的偏移量数据。  

ES是分布式存储的，查询结果一般都是跨多个分片的(spans multiple shards)，每个shard产生自己的排序结果，最后协调节点(coordinating node)对所有分片返回结果进行汇总排序截断返回给客户端。所以深度分页（分页深度）或者一次请求很多结果（分页大小）都会极大降低ES的查询性能。所以ES就默认限制最多只能访问前1w个文档。这是通过index.max_result_window控制的。

假设某个index有 5 个primary shard。我们要获取第1000页的内容，页面大小是10，那么需要排序的文档数是 10001~10010 (pageSize * pageNumber)。那么每个分片需要本地排序产生前10010条记录，然后协调节点/聚合层(Coordinating node/Aggregator)需要对5个主分片返回的 10010 * 5 = 50050 （docs * Shards）条记录进行排序)，最后只返回10条数据给客户端。其他的50040条全部都扔掉了。

排序的原因：让ES的数据保证一定顺序，这样每次分页查询的数据不会重复（可以联想MySQL的分页查询），会有默认的排序字段。

也就是说为了返回第 pageNumber 页的数据，一共需要对 pageSize * pageNumber * shardNumber 个文档进行排序。最后只返回 pageSize 条数据，性价比真心不高。

## 服务端缓存——Scan and scroll API

### 描述：

从上面的分析我们可以看出，为了返回某一页记录，其实我们抛弃了其他的大部分已经排好序的结果。那么简单点就是把这个结果缓存起来，下次就可以用上了。根据这个思路，ES提供了<a href='https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-body.html#request-body-search-scroll'>Scroll API</a>。它概念上有点像传统数据库的游标(cursor)。

scroll调用本质上是实时创建了一个快照(snapshot)，然后保持这个快照一个指定的时间，这样，下次请求的时候就不需要重新排序了。从这个方面上来说，scroll就是一个服务端的缓存。既然是缓存，就会有下面的问题：
- 一致性问题。ES的快照就是产生时刻的样子了，在过期之前的所有修改它都视而不见。
- 服务端开销。ES这里会为每一个scroll操作保留一个查询上下文(Search context)。ES默认会合并多个小的索引段(segment)成大的索引段来提供索引速度，在这个时候小的索引段就会被删除。但是在scroll的时候，如果ES发现有索引段正处于使用中，那么就不会对它们进行合并。这意味着需要更多的文件描述符以及比较慢的索引速度。
- 每一个 scroll_id 不仅会占用大量的资源（特别是排序的请求），而且是生成的历史快照，对于数据的变更不会反映到快照上。

其实这里还有第三个问题，但是它不是缓存的问题，而是因为ES采用的游标机制导致的。就是你只能顺序的扫描，不能随意的跳页。而且还要求客户每次请求都要带上”游标”。

### scroll接口使用方式如下：
```bash
GET /old_index/_search?scroll=1m  // => 1. Keep the scroll window open for 1 minute.
{
    "query": { "match_all": {}},
    "sort" : ["_doc"],  // => 2. _doc is the most efficient sort order.
    "size":  1000 // => 3. Although we specified a size of 1,000, we get back many more documents. 
}
```
#### 说明
1、scroll=1m表示保持这个快照的存活时间，这里是1分钟。  
2、深度分页其实最大的开销还是排序，_doc字段排序其实就是自然顺序。  
3、虽然这里我们指定了size=1000，但是实际上这个请求是分发给每一个分片的，所以我们每次获取 n <= size * number_of_primary_shards个文档。  

服务端会返回一个_scroll_id字段，这是一个Base-64编码的字符串。下一次请求就可以把这个 _scroll_id 带上请求 _search/scroll来获取下一个批次的数据：
```bash
GET /_search/scroll // => 1. should not include the index or type name — these are specified in the original search request instead.
{
    "scroll": "1m", // => 2. The scroll parameter tells Elasticsearch to keep the search context open for another 1m.
    "scroll_id" : "cXVlcnlUaGVuRmV0Y2g7NTsxMDk5NDpkUmpiR2FjOFNhNnlCM1ZDMWpWYnRROzEwOTk1OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MTA5OTM6ZFJqYkdhYzhTYTZ5QjNWQzFqVmJ0UTsxMTE5MDpBVUtwN2lxc1FLZV8yRGVjWlI2QUVBOzEwOTk2OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MDs="
}
```
#### 说明
1. 服务端将返回下一个批次数据，并且会返回一个新的_score_id，下次请求把新的_score_id带上就可以请求下一个批次了。
2. scroll=1m 只要确保 >= 每一个批次的消费时间即可，因为每一次scroll请求都会设置下一个过期时间。

当scroll过期后，查询上下文(Search Context)会自动被删除。因为保持查询上下文是很昂贵的，所以如果已经处理完某个scroll返回的数据，可以使用 clear-scroll API显式删除它:
```bash
DELETE /_search/scroll
{
    "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAD4WYm9laVYtZndUQlNsdDcwakFMNjU1QQ=="
}
```

可以支持批量删除:
```bash
DELETE /_search/scroll
{
    "scroll_id" : [
      "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAD4WYm9laVYtZndUQlNsdDcwakFMNjU1QQ==",
      "DnF1ZXJ5VGhlbkZldGNoBQAAAAAAAAABFmtSWWRRWUJrU2o2ZExpSGJCVmQxYUEAAAAAAAAAAxZrUllkUVlCa1NqNmRMaUhiQlZkMWFBAAAAAAAAAAIWa1JZZFFZQmtTajZkTGlIYkJWZDFhQQAAAAAAAAAFFmtSWWRRWUJrU2o2ZExpSGJCVmQxYUEAAAAAAAAABBZrUllkUVlCa1NqNmRMaUhiQlZkMWFB"
    ]
}
```

也可以一次性删除所有:
```bash
DELETE /_search/scroll/_all
```

## Sliced Scroll

Scoll每次会返回一个批次，如果你觉得这个批次还太大，那么你可以对这个批次进行分片——称之为Sliced Scroll:
```java
GET /twitter/tweet/_search?scroll=1m
{
    "slice": {
        "id": 0,  	// 1. => The id of the slice
        "max": 2 	// 2. => The maximum number of slices

    },
    "query": {
        "match" : {
            "title" : "elasticsearch"
        }
    }
}
GET /twitter/tweet/_search?scroll=1m
{
    "slice": {
        "id": 1, 	// 1. => The id of the slice
        "max": 2	// 2. => The maximum number of slices
    },
    "query": {
        "match" : {
            "title" : "elasticsearch"
        }
    }
}
```
说明: the maximum number of slices allowed per scroll is limited to 1024 and can be updated using the index.max_slices_per_scroll index setting to bypass this limit.

#### TIPS
scroll特别适合做全表扫描，ES的reindex接口内部就是使用scroll机制实现的。

#### 使用注意：
1. 为了避免过度使得我们的cluster繁忙，通常Scroll接口被推荐作为深层次的scrolling，但是因为维护scroll上下文也是非常昂贵的。每生成下一个scrollId需要 2~ 3 秒
2. scroll的查询时间是稳定的，基于原始的快照，因此数据不是最新的，所以这种方法不推荐作为实时用户请求
3. scroll不能单页返回，只能在一次方法调用 分页获取所有的数据。当然你可以在一页数据查询到内存，然后处理，释放掉内存。
4. 因scroll方式给第三方系统提供分页查询时，当一次scroll完成之后返回，scroll快照被删除，再次使用过期的scrollId 查询时  会生成新的scrollId，如果调用方不去检查scroll是否完毕，会出现不断创建scroll快照的问题。
## Search After

Scroll API相对于from+size方式当然是性能好很多，但是也有如下问题：  
1、Search context开销不小。  
2、是一个临时快照，并不是```实时```的分页结果。

针对这些问题，ES 5.0 开始推出了 Search After 机制可以提供了更实时的游标(live cursor)。它的思想是利用上一页的分页结果来提高下一页的分页请求。

### 思考：那search After 为什么不用产生临时快照之类的存储信息，就能保证滚动顺序读取数据呢？
每个文档具有一个唯一值的字段应该用作排序规范的仲裁器。否则，具有相同排序值的文档的排序顺序将是未定义的。建议的方法是使用字段_id，它肯定包含每个文档的一个唯一值。

假设请求第一页的请求如下：
```bash
GET twitter/tweet/_search
{
    "size": 10,
    "query": {
        "match" : {
            "title" : "elasticsearch"
        }
    },
    "sort": [
        {"date": "asc"},
        {"_id": "desc"}
    ]
}
```
```注意，这里为了避免sort字段相同值的导致排序不确定，这里增加了 _id 字段。```

返回的结果会包含每个文档的sort字段的sort value。这个就是上面所说的 “live cursor”。

使用```最后一个文档```的sort value作为search after请求值，我们就可以这样子请求下一页结果了：

```bash
GET twitter/tweet/_search
{
    "size": 10,
    "query": {
        "match" : {
            "title" : "elasticsearch"
        }
    },
    "search_after": [1463538857, "654323"],
    "sort": [
        {"date": "asc"},
        {"_id": "desc"}
    ]
}
```
注意到from变成了search_after了。现在是通过search_after来确定分页的开始位置。

search_after使用方式上跟scroll很像，但是相对于scroll它是无状态的(stateless)，没有search context开销；而且它是每次请求都实时计算的，所以也没有一致性问题（相反，有索引变化的话，每次排序顺序会变化呢）。```但是比起from+size方式，还是有同样的问题没法解决：就是只能顺序的翻页，不能随意跳页。```

### 应用场景：
**```这个排序分页方案其实在app分页中大量使用。```**  

由于app端的分页比较特殊，比如后台数据会近实时的发生变化，所以采用常规的分页算法 [(totalRecord + pageSize - 1) / pageSize] 是会有问题的。如果仍采用这种算法，当上推刷新时，就有可能加载到上一页已经看过的数据，比如用户当前正在看第2页的历史数据，如果此时后台数据源新增了一条数据，那么当用户继续上推操作查看第3页的历史数据时，就会把第2页的最后一条数据获取，并且会把该条数据作为第3页的第一条数据进行展示，这样是有问题的。所以在数据表设计时，需要在表中增加一个自增的orderId字段参与分页，然后分页时，需要将第一页的最后一条数据的orderId回传到后台，后台拿着这个orderId进行条件判断查询并且集合上面的分页算法就可以避免上面的问题（在新闻类的app中，经常使用createdTime作为orderId）。另一方便，app的滑动翻页其实就是顺序翻页，所以特别适合这种分页方式。

### 使用注意：
注意：当我们使用search_after时，from值必须设置为0或者-1。  
search_after缺点是不能够随机跳转分页，只能是一页一页的向后翻，并且需要至少指定一个唯一不重复字段来排序。它与滚动API非常相似，但与它不同，search_after参数是无状态的，它始终针对最新版本的搜索器进行解析。因此，排序顺序可能会在步行期间发生变化，具体取决于索引的更新和删除。

## Redis的分页：
在redis中可以使用Sorted Set来实现。具体的分页命令是 ZREVRANGEBYSCORE：
```bash
ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]

返回有序集 key 中， score 值介于 max 和 min 之间(默认包括等于 max 或 min )的所有的成员。有序集成员按 score 值递减(从大到小)的次序排列。

具有相同 score 值的成员按字典序的逆序(reverse lexicographical order )排列。

除了成员按 score 值递减的次序排列这一点外， ZREVRANGEBYSCORE 命令的其他方面和 ZRANGEBYSCORE 命令一样。

可用版本：
>= 2.2.0
时间复杂度:
O(log(N)+M)， N 为有序集的基数， M 为结果集的基数。
返回值:
指定区间内，带有 score 值(可选)的有序集成员的列表。
```

## 参考
- http://arganzheng.life/deep-pagination-in-elasticsearch.html
- ES嵌套属性查询：http://arganzheng.life/elasticsearch-nested-object-indexing-and-searching.html
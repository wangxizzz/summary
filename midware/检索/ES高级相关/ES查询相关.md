## ES 7的聚合查询问题
1、只有 long、integer、long类型的数组 等其他数值类型及数值类型数组 和 keyword类型的 字段才可以做聚合查询。比如根据某个字段的值做group by操作。如果强行利用text字段做聚合，会报Exception:
```java
Exception in thread "main" ElasticsearchStatusException[Elasticsearch exception [type=search_phase_execution_exception, reason=all shards failed]]; nested: ElasticsearchException[Elasticsearch exception [type=illegal_argument_exception, reason=Fielddata is disabled on text fields by default. Set fielddata=true on [content_type] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead.]]; nested: ElasticsearchException[Elasticsearch exception [type=illegal_argument_exception, reason=Fielddata is disabled on text fields by default. Set fielddata=true on [content_type] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead.]];
```

2、利用springData创建的es mapping结构：
```java
@Id
private Long id;

@Field(type = FieldType.Keyword, analyzer = "ik_max_word")
private List<String> likes;

// 注意：数字类型查询不加 .keyword
@Field(type = FieldType.Keyword)
private List<Integer> ages;
```
上述创建的mapping映射有如下的坑：
likes与ages字段并不是keyword类型的，仍然是text类型，所以根据这likes个字段聚合查询，就会报错,但是可以ages字段聚合，因为ages是long类型的数组。利用 curl -XGET 'http://localhost:9200/item/_mapping?pretty=true' 可以查看

```json
{
  "peopleindex" : {
    "mappings" : {
      "properties" : {
        "ages" : {
          "type" : "long"
        },
        "id" : {
          "type" : "long"
        },
        "likes" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "peopleName" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
  }
}
```
但是这种mapping的创建方式可以使用如下的方式变为keyword查询：
```java
AggregationBuilder aggregationBuilder = AggregationBuilders.terms("AGG").field("likes.keyword");
		searchSourceBuilder.aggregation(aggregationBuilder);
```

如果使用的是这种方式创建的mapping,是无法 fieldName.keyword聚合的，不会报错，但是聚合结果为空:
```json
{
  "properties": {
    "crowdId": {
      "type": "integer"
    },
    "userId": {
      "type": "integer"
    },
    "name": {
      "type": "text",
      "analyzer": "ik_max_word",
      "search_analyzer": "ik_smart"
    },
    "city": {
      "type": "keyword"
    },
    "userSex": {
      "type": "keyword"
    }
  }
}
```

3、创建mapping的方式：  
不要使用springData的API，利用原生ES的api传入 mapping的json字符串进行创建。

4、多字段聚合参考 官网API：  
https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/_structuring_aggregations.html


## ES的深分页问题：

- 传统from+size
  - 对于大量的数据而言，我们尽量避免使用from+size这种方法。这里的原因是index.max_result_window的默认值是10K，也就是说from+size的最大值是1万。搜索请求占用堆内存和时间与from+size成比例。因为这种方式需要每次把所有数据都查出来，然后取对应的偏移量数据。
- 大数据量的快照分页（scroll）
  - 为了避免过度使得我们的cluster繁忙，通常Scroll接口被推荐作为深层次的scrolling，但是因为维护scroll上下文也是非常昂贵的。每生成下一个scrollId需要 2~ 3 秒
  - scroll的查询时间是稳定的，基于原始的快照，因此数据不是最新的，所以这种方法不推荐作为实时用户请求
  - scroll不能单页返回，只能在一次方法调用 分页获取所有的数据。当然你可以在一页数据查询到内存，然后处理，释放掉内存。
- search_after
  - search_after可以一页一页的返回，适合对外提供dubbo 
  - search_after参数通过提供实时cursor来解决此问题,它与scroll API非常相似，但与它不同，search_after参数是无状态的，它始终针对最新版本的搜索器进行解析。 因此，排序顺序可能会在步行期间发生变化。
  - search_after 会随着数据量的加大，查询时间急剧增加。500w数据大概是 1s

## 关于ES的数据更新：
Es本身其实并不支持删除与update，底层只是把旧的数据不可见，查询时默认返回最新值了而且这个步骤是非常耗损es性能。因此我们如果需要对es已存在数据根据主键或唯一标识字段更新时，可以采用:
在数据插入时在db保存一个version字段，每次插入更新把最新的version字段更新到es对应的列，查询时根据最新的version即可查询出最新的数据。




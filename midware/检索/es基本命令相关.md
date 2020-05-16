# 基本命令相关

## 查询相关：

```bash
curl 'http://localhost:9200/?pretty'  
```
查看es版本信息，curl默认是get请求,pretty参数表示以格式化结构输出，或者是pretty=true。
```bash
 curl -X GET 'http://localhost:9200/_cat/indices?v'
```
获取整个集群的索引  
加上 -i 参数，可以获取响应头信息

```bash
curl -XGET 'http://localhost:9200/_mapping?pretty=true'
```
查看所有index下的所有type

```bash
curl -XGET 'http://localhost:9200/item/_mapping?pretty=true'
```
查看index名为item下的所有type

```bash
curl -X GET 'http://localhost:9200/megacorp/employee/_search'
```
查询 megacorp/employee 该index该type下的所有doc记录

```bash
curl -XGET 'http://localhost:9200/_count?pretty' -H 'content-Type:application/json' -d '
{
    "query": {
        "match_all": {}
    }
}
'
```
获取整个集群doc的数量

```bash
curl -X GET 'http://localhost:9200/item/docs/1?pretty' -H  'content-Type:application/json'
```
我们只要执行HTTP GET请求并指出文档的“地址”——索引、类型和ID既可。根据这三部分信息，我们就可以返回原始JSON文档

```bash
curl -X GET 'http://localhost:9200/megacorp/employee/1'
```
根据id=1查询单条doc的记录

```bash
curl -XGET 'http://localhost:9200/blog/_mapping/article?pretty'
```
可以查看blog索引 article类型的doc字段的映射关系(简单来说就是doc字段的结构)


```bash
curl -X GET 'http://localhost:9200/megacorp/employee/_search?q=first_name:周二珂'
```
简单利用关键字搜索（注意：如果没设置字段的类型，默认无法搜索中文）

```bash
curl -X GET 'http://localhost:9200/term_test/docs/_search?pretty' -H  'content-Type:application/json' -d \ '
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "title": "love"
          }
        },
        {
          "term": {
            "title": "china"
          }
        }
      ]
    }
  }
}'

# term是整个value精确匹配。上面例子会找出既包含love，也包含china的记录

curl -X GET 'http://localhost:9200/term_test/docs/_search?pretty' -H  'content-Type:application/json' -d \ '
{
    "query":{  
      "match":{  
         "title":"love HuBei"
      }
   }
}'

# match是经过
```
对字段的值进行查询搜索。  
先看看term的定义，term是代表完全匹配，也就是精确查询，搜索前不会再对搜索词进行分词拆解。(一般会与 should、must、must_not连用)      
match在匹配时会对所查找的关键词进行分词，然后按分词匹配查找，而term会直接对关键词进行查找。一般模糊查找的时候，多用match，而精确查找时可以使用term。  
参考：
- https://www.jianshu.com/p/d5583dff4157
- https://blog.csdn.net/tclzsn7456/article/details/79956625

## 写相关命令：
```bash
curl -X PUT 'http://localhost:9200/blog'
```
 创建一个名为blog索引

```bash
curl -X DELETE 'localhost:9200/weather'
```
删除一个index

```bash
curl -X POST http://localhost:9200/blog/article -H 'content-Type:application/json' -d '{ "author": "zhengkai.blog.csdn.net", "createtime": 1563689639575, "id": 2, "text": "Elasticsearch是一个开源的分布式、高扩展、高实时的RESTful 搜索和分析引擎，基于Lucene......", "title": "SpringBoot整合ElasticSearch" }'
```
在bolg索引 新建类别为article，并且创建一条doc记录
>> 注意：curl如果报错{"error":"Content-Type header [application/x-www-form-urlencoded] is not supported","status":406}：**需要设置在curl中header信息：-H 'content-Type:application/json'**

```bash
curl -XPUT 'http://localhost:9200/megacorp/employee/1' -H  'content-Type:application/json' -d \ '
{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         35,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}'
```
直接创建index（名为megacorp）、type（名为employee）、doc、field. 多次PUT会覆盖字段数据

```bash
curl -XPOST 'http://localhost:9200/item/docs/10/_update' -H  'content-Type:application/json' -d \ '
{
    "doc":{
        "desc":"python基础"
    }
}'
```
更新具体字段的值
```bash
curl -XPOST 'http://localhost:9200/item/docs/9/_update' -H  'content-Type:application/json' -d \ '
{
    "doc":{
        "desc":"python基础"
    },
    "upsert":{
        "test":"我是test"
    }
}'
```
如果更新的文档不存在，那么利用upsert来添加一个初始化文档用于索引。


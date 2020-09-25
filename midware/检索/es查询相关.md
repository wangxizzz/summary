# 性能相关：
1.利用term过滤器，可以加速查询效率。因为会缓存过滤结果，并且只会在已经过滤出来的docs做其他操作。

# match、term、terms 、 multi_match
1、term query会去倒排索引中寻找确切的term，它并不知道分词器的存在(不分词，全词匹配)，这种查询适合keyword、numeric、date等明确值的。  
term：查询某个字段里含有某个关键词的文档
```
GET /customer/doc/_search/
{
  "query": {
    "term": {
      "title":   "blog"
    }
  }
}
```
2、terms：查询某个字段里含有多个关键词的文档(查询数组)
```
GET /customer/doc/_search/
{
  "query": {
    "terms": {
      "title":  [ "blog","first"]
    }
  }
}
```
3、match query 知道分词器的存在，会对field进行分词操作，然后再查询（分词匹配）。
```
GET /customer/doc/_search/
{
  "query": {
    "match": {
      "title":  "my ss"   #它和term区别可以理解为term是精确查询，这边match模糊查询；match会对my ss分词为两个单词，然后term对认为这是一个单词
    }
  }
}
```
4、multi_match：可以指定多个字段
```
GET /customer/doc/_search/
{
  "query": {
    "multi_match": {
      "query" : "blog",
      "fields":  ["name","title"]   #只要里面一个字段包含值 blog 既可以
    }
  }
}
```

# ES的query string用法：
```
简单入门
条件查询
如 name 等于 guozheng
name:guozheng
name 等于 guozheng 或者 等于 zhangsan
name:(guozheng OR zhangsan)
范围查询
age 大于等于10 小于等于20
age:[10 TO 20]
age 大于等于10 小于20
age:[10 TO 20}

时间范围查询
time:["2020-06-30 11:10:10" TO "2020-07-02 10:10:10"]

组合查询（AND OR NOT）
例如 性别是男 且 年龄是10~20 且 name 不是guozheng  也不是 zhangsan
gender:(男) AND age:[10 TO 20] NOT name:(guozheng OR zhangsan)
```

> query string与 DSL可以一起使用。参考：https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html

# ES嵌套索引建立：
https://blog.csdn.net/dufufd/article/details/88578312


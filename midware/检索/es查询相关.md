# 关于es查询与查询性能相关：

## 性能相关：
1.利用term过滤器，可以加速查询效率。因为会缓存过滤结果，并且只会在已经过滤出来的docs做其他操作。

2.**match query VS match_phrase query**：  
注意其差别：  
match query：会对查询语句进行分词，分词后查询语句中的```任何一个词```项被匹配，文档就会被搜索到。如果想查询匹配所有关键词的文档，可以用and操作符连接；  
match_phrase query：满足下面两个条件才会被搜索到
- （1）分词后所有词项都要出现在该字段中
- （2）字段中的词项顺序要一致

## ES的query string用法：
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

## es的与或非运算  
<img src="../../imgs/es的与或非.png">

## es查询案例
<img src="../../imgs/es查询案例.png">

## ES嵌套索引建立：
https://blog.csdn.net/dufufd/article/details/88578312


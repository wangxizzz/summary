## 安装启动相关
1.**mac安装报错，Failed to create native process factories for Machine Learning**

在安装目录 config/elasticsearch.yml 增加配置```xpack.ml.enabled: false```，关闭机器学习相关东西。

2.**后台启动：**
> ./bin/elasticsearch -d 

3.**kibana启动：**
> 进入kibana目录  ./kibana  即可
访问地址：http://localhost:5601

3.**es ik分词器相关**
- 安装：
    - 进入bin目录，输入命令（可以上github上看版本,版本需要和es对应）安装完毕，es需要重启，否则是不生效的：elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.4.0/elasticsearch-analysis-ik-7.4.0.zip
- IK分词器有两种分词模式：ik_max_word和ik_smart模式
    - ik_max_word：会将文本做最细粒度的拆分，比如会将“中华人民共和国人民大会堂”拆分为“中华人民共和国、中华人民、中华、华人、人民共和国、人民、共和国、大会堂、大会、会堂等词语。
    - 会做最粗粒度的拆分，比如会将“中华人民共和国人民大会堂”拆分为中华人民共和国、人民大会堂。
    - **两种分词器使用的最佳实践是**：索引时用ik_max_word，在搜索时用ik_smart。
即：索引时最大化的将文章内容分词，搜索时更精确的搜索到想要的结果。


## 基本概念：
- es主要分为：索引，type,doc,field
- 其中index相当于db, type相当于table, doc相当于 单个记录
```
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices   -> Types  -> Documents -> Fields
```

2、es是不允许修改一个已被索引字段的类型的，比如把text变为long，这会报错。解决办法是：重新建立一个字段，然后重新index.

3.**为什么类型(type)被移除了？**  
起初，我们说"索引"和关系数据库的“库”是相似的，“类型”和“表”是对等的。
这是一个不正确的对比，导致了不正确的假设。在关系型数据库里,"表"是相互独立的,一个“表”里的列和另外一个“表”的同名列没有关系，互不影响。但在类型里字段不是这样的。

在一个Elasticsearch索引里，所有不同类型的同名字段内部使用的是同一个lucene字段存储。也就是说，上面例子中，user类型的user_name字段和tweet类型的user_name字段是存储在一个字段里的，两个类型里的user_name必须有一样的字段定义。

这可能导致一些问题，例如你希望同一个索引中"deleted"字段在一个类型里是存储日期值，在另外一个类型里存储布尔值。

最后,在同一个索引中，存储仅有小部分字段相同或者全部字段都不相同的文档，会导致数据稀疏，影响Lucene有效压缩数据的能力。

因为这些原因，我们决定从Elasticsearch中移除类型的概念。


## **elasticsearch Mapping、字段类型Field type详解:**  
<a href="./es字段类型与mapping介绍.md">es字段类型与mapping介绍</a>

## 基本命令相关
<a href="./es基本命令相关.md">es基本命令相关</a>







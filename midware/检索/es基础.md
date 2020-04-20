## 安装启动相关
1.**mac安装报错，Failed to create native process factories for Machine Learning**

在安装目录 config/elasticsearch.yml 增加配置```xpack.ml.enabled: false```，关闭机器学习相关东西。

2.**后台启动：**
> ./bin/elasticsearch -d 

## 基本概念：
- es主要分为：索引，type,doc,field
- 其中index相当于db, type相当于table, doc相当于 单个记录
```
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices   -> Types  -> Documents -> Fields
```

## 基本命令相关
<a href="./es基本命令相关.md">es基本命令相关</a>







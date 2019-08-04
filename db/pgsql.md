数据库加一个更新字段。

1.extroinf字段
2.insert返回主键，自增
3.各种sql片段

****
## PGSQL的学习：
- hstore的操作：main_order表有hstore字段的操作方式，在MainOrderMapper.xml中存在。
    - https://blog.csdn.net/u013027996/article/details/40115563
    - 也可以直接搜 pgsql的hstore字段
- jsonb与json,也介绍了相关操作函数
    - https://blog.csdn.net/u012129558/article/details/81453640
    - 直接搜  pgsql 创建json

**spring单元测试：**
- https://juejin.im/post/5b95dbe46fb9a05cd7772503

mybatis:
- https://my.oschina.net/zjllovecode/blog/1818716
- https://blog.csdn.net/sinat_28978689/article/details/79406832
- https://blog.csdn.net/yalishadaa/article/details/72580680
- https://blog.csdn.net/ask_rent/article/details/6320326
- https://blog.csdn.net/u012702547/article/details/55105400

**restTemplate使用**
- https://segmentfault.com/a/1190000011093597?utm_source=tag-newest
- https://blog.csdn.net/itguangit/article/details/78825505

**spring 的Cachable注解**
- https://blog.csdn.net/wjacketcn/article/details/50945887
https://blog.csdn.net/wjacketcn/article/details/50945887

foreach简介
foreach的主要用在构建in条件中，它可以在SQL语句中进行迭代一个集合。
foreach元素的属性主要有 item，index，collection，open，separator，close。
item表示集合中每一个元素进行迭代时的别名，index指定一个名字，用于表示在迭代过程中，每次迭代到的位置，open表示该语句以什么开始，separator表示在每次进行迭代之间以什么符号作为分隔 符，close表示以什么结束，在使用foreach的时候最关键的也是最容易出错的就是collection属性，该属性是必须指定的，但是在不同情况 下，该属性的值是不一样的，主要有一下3种情况：

1.如果传入的是单参数且参数类型是一个List的时候，collection属性值为list
2.如果传入的是单参数且参数类型是一个array数组的时候，collection的属性值为array
3.如果传入的参数是多个的时候，我们就需要把它们封装成一个Map了，当然单参数也可以封装成map
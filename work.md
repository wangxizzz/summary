1.lombok:https://www.cnblogs.com/daimajun/p/7136078.html  

2.调用dubbo接口的坑：group,registry,version需要provider与comsumer端对应好，否则就会报No provider异常。在provider那端的id应为该Service bean的name. 然后在comsumer端ref = id来调用

3.jsonp 
- https://www.cnblogs.com/ishenghuo/p/4630419.html  
- https://www.jianshu.com/p/ef0ac078d9bd
- https://blog.csdn.net/justnow_/article/details/53916835

4.jsonNode:
https://blog.csdn.net/mst1010/article/details/78589059

5.pgsql的分页语句：
- select * from user limit  param1 ;    limit + 参数    要获取的数据条数 (从第一条数据开始)
- select * from user limit param1  offset param2 ;  从（param2+1）条数据开始，取 param1条数据

**错误总结：**
- 03-07：
    - product-service工程发布测试环境，在checkurl那一步卡住了，后来差日志知道是mybatis报错引起的。通过到机器上看日志 less ?Exception 匹配最后一个，n键调到上一个。
    - 自己在api工程写的javabean没有实现序列化，导致本地调试touch时，调用服务一直卡在那里。
    - 在调用别人的dubbo接口时，报没有provider,说明reference配置有错误。zk的group和服务本身的group不一样。上次引用机票的服务，就是没有服务本身的group,只给了zk的group.

- 03-08:

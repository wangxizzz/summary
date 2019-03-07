1.lombok:https://www.cnblogs.com/daimajun/p/7136078.html  

2.调用dubbo接口的坑：group,registry,version需要provider与comsumer端对应好，否则就会报No provider异常。在provider那端的id应为该Service bean的name. 然后在comsumer端ref = id来调用

3.jsonp 
- https://www.cnblogs.com/ishenghuo/p/4630419.html  
- https://www.jianshu.com/p/ef0ac078d9bd
- https://blog.csdn.net/justnow_/article/details/53916835

4.jsonNode:

https://blog.csdn.net/mst1010/article/details/78589059

5.
1. select * from user limit  param1 ;    limit + 参数    要获取的数据条数 (从第一条数据开始)

2 .select * from user limit param1  offset param2 ;  从（param2+1）条数据开始，取 param1条数据

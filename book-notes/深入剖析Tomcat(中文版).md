1、servlet容器是怎么工作的？  
对于每一个请求，servlet容器都会为其完成以下3个操作：
- 创建一个request对象，来接收用户的请求；
- 创建一个response来响应用户的请求；
- 调用Servlet中的service方法，传入request和response对象，Servlet从request中读取数据，然后通过response返回给客户端。

2、

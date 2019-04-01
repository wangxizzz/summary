1、servlet容器是怎么工作的？  
对于每一个请求，servlet容器都会为其完成以下3个操作：
- 创建一个request对象，来接收用户的请求；
- 创建一个response来响应用户的请求；
- 调用Servlet中的service方法，传入request和response对象，Servlet从request中读取数据，然后通过response返回给客户端。

2、Servlet的接口方法：
```java
    // 实例化某个Servlet，就会调用此方法，且只会调用一次，可以重写此方法。
    public void init(ServletConfig config);   

    public ServletConfig getServletConfig();

    // 此方法可以调用多次
    public void service(ServletRequest req, ServletResponse res);

    public String getServletInfo();

    // 将servlet从服务器中移除，调用。
    public void destroy();

```

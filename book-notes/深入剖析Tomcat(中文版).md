1、servlet容器是怎么工作的？  
对于每一个请求，servlet容器都会为其完成以下3个操作：
- 创建一个request对象，来接收用户的请求；
- 创建一个response来响应用户的请求；
- 调用Servlet中的service方法，传入request和response对象，Servlet从request中读取数据，然后通过response返回给客户端。

2、文件句柄：在文件I/O中，要从一个文件读取数据，应用程序首先要调用操作系统函数并传送文件名，并选一个到该文件的路径来打开文件。该函数取回一个顺序号，即文件句柄（file handle）。linux内核对对于外设的操作都看作是文件操作。

3.在servlet容器中，每次请求servlet都会载入响应的serevlet类。

4、Servlet的接口方法：
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

5.Tomcat中门面类模式：  
- 样例代码参照：https://github.com/wangxizzz/Design-pattern  
- 在Tomcat中的应用：  

**贴出HttpServletRequest源码**
```java
/**
 * Facade class that wraps a Coyote request object.
 * All methods are delegated to the wrapped request.
 *
 * @author Craig R. McClanahan
 * @author Remy Maucherat
 */
@SuppressWarnings("deprecation")  // 注解表示消除警告
public class RequestFacade implements HttpServletRequest {

public interface HttpServletRequest extends ServletRequest

ServletRequest是底层接口
```
HttpServletResponse与HttpServletRequest的外观模式相同。

6.
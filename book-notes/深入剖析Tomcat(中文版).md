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

ServletRequest是顶层接口
```
HttpServletResponse与HttpServletRequest的外观模式相同。

6.在兼容servlet2.3和2.4的规范中，连接器必须负责创建HttpServletRequest和HttpServletResponse实例，然后通过参数传递到service().

7.Tomcat中的单例模式：  
**StringManager**：Tomcat把错误消息存储在properties文件中，而每个properties文件都是用一个StringManager的实例来处理。  
StringManager单例代码如下：
```java
// 提供static方法，在外面获取StringManager实例
public static final synchronized StringManager getManager(String packageName) {
    StringManager mgr = managers.get(packageName);
    if (mgr == null) {
        mgr = new StringManager(packageName);
        managers.put(packageName, mgr);
    }
    return mgr;
}

// 构造函数private
private StringManager(String packageName) {
    String bundleName = packageName + ".LocalStrings";
    ResourceBundle tempBundle = null;
    try {
        tempBundle = ResourceBundle.getBundle(bundleName, Locale.getDefault());
    } catch( MissingResourceException ex ) {
        // Try from the current loader (that's the case for trusted apps)
        // Should only be required if using a TC5 style classloader structure
        // where common != shared != server
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        if( cl != null ) {
            try {
                tempBundle = ResourceBundle.getBundle(
                        bundleName, Locale.getDefault(), cl);
            } catch(MissingResourceException ex2) {
                // Ignore
            }
        }
    }
    // Get the actual locale, which may be different from the requested one
    if (tempBundle != null) {
        locale = tempBundle.getLocale();
    } else {
        locale = null;
    }
    bundle = tempBundle;
}
```

8.

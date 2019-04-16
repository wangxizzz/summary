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

5.**Tomcat中门面类模式：**  
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
// Request和RequestFacade实现相同的接口，因此Request的门面类是RequestFacade.
public class Request implements HttpServletRequest {

public interface HttpServletRequest extends ServletRequest

ServletRequest是顶层接口
```
```在RequestFacade中，被包装的类是Request```
```java
/**
* Construct a wrapper for the specified request.
*
* @param request The request to be wrapped
*/
public RequestFacade(Request request) {

    this.request = request;

}
```

HttpServletResponse与HttpServletRequest的外观模式相同。

6.在兼容servlet2.3和2.4的规范中，连接器必须负责创建HttpServletRequest和HttpServletResponse实例，然后通过参数传递到service().

7.**Tomcat中的单例模式：**  
**StringManager**：Tomcat把错误消息存储在properties文件中，而每个properties文件都是用一个StringManager的实例来处理。  
StringManager单例代码如下：
```java
// 提供static方法，在外面获取StringManager实例
// 单例模式的写法：直接在方法上加synchronized.
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

8.**Tomcat中的不可变Map的应用:**

ParameterMap : 存储请求参数的键值对(name-value)。可以通过request.getParameter(String key)获取参数值，在Tomcat的设计中，请求参数是不能被servlet程序员更改的，所以使用了特殊的Map.  

ParameterMap源码如下：位于org.apache.catalina.util.ParameterMap
```java
/**
* Construct a new, empty map with the default initial capacity and
* load factor.
*/
public ParameterMap() {
    delegatedMap = new LinkedHashMap<>();
    unmodifiableDelegatedMap = Collections.unmodifiableMap(delegatedMap);
}

// Collections.unmodifiableMap()
public static <K,V> Map<K,V> unmodifiableMap(Map<? extends K, ? extends V> m) {
    return new UnmodifiableMap<>(m);
}

// UnmodifiableMap 是private类，只有在类内部调用，在外部是看不到的。
private static class UnmodifiableMap<K,V> implements Map<K,V>, Serializable {
```
UnmodifiableMap分析：位于JDK中：java.util.Collections.UnmodifiableMap
```java
private static class UnmodifiableMap<K,V> implements Map<K,V>, Serializable {
    /**
     * Tomcat的不可变Map实现，把所有改变Map里的值的方法，直接抛异常。
     */
    public V put(K key, V value) {
        throw new UnsupportedOperationException();
    }
    public V remove(Object key) {
        throw new UnsupportedOperationException();
    }
    public void putAll(Map<? extends K, ? extends V> m) {
        throw new UnsupportedOperationException();
    }
    public void clear() {
        throw new UnsupportedOperationException();
    }
}
```
**当然ParameterMap里面的值并不是绝对不可变的：**
```java
public final class ParameterMap<K,V> implements Map<K,V>, Serializable {

    private static final long serialVersionUID = 2L;

    // 创建的代理Map
    private final Map<K,V> delegatedMap;

    // 可能保存了一份不可变数据的副本。
    private final Map<K,V> unmodifiableDelegatedMap;

    /**
     * Construct a new, empty map with the default initial capacity and
     * load factor.
     */
    public ParameterMap() {
        delegatedMap = new LinkedHashMap<>();
        unmodifiableDelegatedMap = Collections.unmodifiableMap(delegatedMap);
    }

    private void checkLocked() {
        if (locked) {
            throw new IllegalStateException(sm.getString("parameterMap.locked"));
        }
    }
    public void clear() {
        // 检查锁，其实是一个标志变量。
        checkLocked();
        delegatedMap.clear();
    }
    public V put(K key, V value) {
        checkLocked();
        return delegatedMap.put(key, value);
    }
    public V remove(Object key) {
        checkLocked();
        return delegatedMap.remove(key);
    }
    
}
```

9.**Servlet的生命周期：**

10.**Tomcat中的连接器必须满足以下要求**：
- 实现了org.apache.catalina.Connector接口；
- 创建实现了HttpServletRequest接口的Request对象；
- 创建实现了HttpServleResponse接口的Response对象.

11.**content-length字段解析：**   
在Request和Response中都可以设置该字段的值，在HTTP1.1中，默认是长连接，因此服务端可以连续的向客户端发送字节流，因此接收方需要content-length来如何解析这些字段信息。（这是解决HTTP粘包的一种解决方式）

12.**Transfer-Encoding: chunked字段解析：**  
参考网址：https://blog.csdn.net/lblblblblzdx/article/details/80386693  
分块传输-chunked：  
当返回的数据比较大时，如果等待生成完数据再传输，这样效率比较低下。相比而言，服务器更希望边生成数据边传输。可以在响应头加上以下字段标识分块传输：  
Transfer-Encoding: chunked  
1）当选择分块传输时，响应头中可以不包含Content-Length。

（2）服务器会先回复一个不带数据的报文（只有响应行和响应头和\r\n），然后开始传输若干个数据块。 
每个数据块格式如下：

400                      -------------------->该块数据的大小，16进制，这里表示1024Bytes  
adfaddaffasd...          -------------------->数据  
                         -------------------->换行   

（3）当传输完若干个数据块后，需要再传输一个空的数据块

30                      -------------------->数据块大小为0  
                        -------------------->大小为0时也需要一个换行\r\n  
                        -------------------->换行  

（4） 当客户端收到空的数据块时，则客户端知道数据接收完毕。

13.**状态码：100解析：**  
这个状态码实际上是对如下场景的一种优化：客户端有一个较大的文件需要上传并保存，但是客户端不知道服务器是否愿意接受这个文件，所以希望在消耗网络资源进行传输之前，先询问一下服务器的意愿。实际操作为客户端发送一条特殊的请求报文，报文的头部应包含

Expect: 100-continue
此时，如果服务器愿意接受，就会返回 100 Continue 状态码，反之则返回 417 Expectation Failed 状态码。对于客户端而言，如果客户端没有发送实际请求的打算，则不应该发送包含 100 Continue Expect 的报文，因为这样会让服务器误以为客户端将要发送一个请求。

之前提到过，并不是所有的HTTP应用都支持 100 Continue 这个状态码（例如HTTP/1.0及之前的版本的代理或服务器）所以客户端不应该在发送 100 Continue Expect 后一直等待服务器的响应，在一定时间后，客户端应当直接发送计划发送的内容。

而对于服务器而言，也不应当把 100 Continue 当作一个严格的判断方法。服务器有可能在发送回应之前就受到了客户端发来的主体报文。此时服务器就不需要再发送 100 Continue 作为回应了。但仍然需要在接受完成后返回适当的状态码。理论上，当服务器收到一个 100 Continue Expect 请求时，应当进行响应。但服务器永远也不应向没有发送 100 Continue Expect 请求的客户端发送100 Continue 状态码作为回应。这里提到的应当进行响应是指：假设服务器不打算接收客户端将要发送的主体报文，也应当做适当的响应（例如发送 417 Expectation Failed）而不是单纯的关闭连接，这样会对客户端在网络层面上产生影响。  
参考网址：https://www.cnblogs.com/nangcr/p/informational-responses-status-code-100-in-http.html

14.**JDK中工厂方法模式的应用：**  
抽象接口：(抽象工厂)
```java
public interface ServerSocketFactory {
// 抽象工厂提供3个创造ServerSocket的重载函数(根据不同的参数)
    public ServerSocket createSocket(int port) throws IOException, KeyStoreException,                       NoSuchAlgorithmException,
                CertificateException, UnrecoverableKeyException,
                KeyManagementException;

    public ServerSocket createSocket(int port, int backlog) throws IOException, KeyStoreException,              NoSuchAlgorithmException,
                    CertificateException, UnrecoverableKeyException,
                    KeyManagementException;

    public ServerSocket createSocket(int port, int backlog,
                                     InetAddress ifAddress) throws IOException, KeyStoreException, NoSuchAlgorithmException,
           CertificateException, UnrecoverableKeyException,
           KeyManagementException;
}

```
具体工厂类：
```java
public final class DefaultServerSocketFactory implements ServerSocketFactory {

    public ServerSocket createSocket (int port)
    throws IOException, KeyStoreException, NoSuchAlgorithmException,
           CertificateException, UnrecoverableKeyException,
           KeyManagementException {

        return (new ServerSocket(port));

    }

    public ServerSocket createSocket (int port, int backlog)
    throws IOException, KeyStoreException, NoSuchAlgorithmException,
           CertificateException, UnrecoverableKeyException,
           KeyManagementException {

        return (new ServerSocket(port, backlog));

    }

    public ServerSocket createSocket (int port, int backlog,
                                      InetAddress ifAddress)
    throws IOException, KeyStoreException, NoSuchAlgorithmException,
           CertificateException, UnrecoverableKeyException,
           KeyManagementException {

        return (new ServerSocket(port, backlog, ifAddress));

    }
}
```
客户端调用(摘录的Tomcat低版本的源码)：
```java
// 在该类方法中调用open()
public final class HttpConnector
    implements Connector, Lifecycle, Runnable {
```
```java
// 通过调用open函数，传入不同参数，就可以在工厂中获取不同的ServerSocket实例。
private ServerSocket open()
    throws IOException, KeyStoreException, NoSuchAlgorithmException,
           CertificateException, UnrecoverableKeyException,
           KeyManagementException
    {

        // Acquire the server socket factory for this Connector
        ServerSocketFactory factory = getFactory();

        // If no address is specified, open a connection on all addresses
        if (address == null) {
            log(sm.getString("httpConnector.allAddresses"));
            try {
                return (factory.createSocket(port, acceptCount));
            } catch (BindException be) {
                throw new BindException(be.getMessage() + ":" + port);
            }
        }

        // Open a server socket on the specified address
        try {
            InetAddress is = InetAddress.getByName(address);
            log(sm.getString("httpConnector.anAddress", address));
            try {
                return (factory.createSocket(port, acceptCount, is));
            } catch (BindException be) {
                throw new BindException(be.getMessage() + ":" + address +
                                        ":" + port);
            }
        } catch (Exception e) {
            log(sm.getString("httpConnector.noAddress", address));
            try {
                return (factory.createSocket(port, acceptCount));
            } catch (BindException be) {
                throw new BindException(be.getMessage() + ":" + port);
            }
        }

    }

// getFactory()分析：
/**
 * 此方法这样写的应用场景：
 *  此方法所在的类为final类型，并且在调用方创建的类对象应该是static类型，然后才可以得到
 *  factory的单例。
 */
public ServerSocketFactory getFactory() {
    // 创建出factory的单例模式
    if (this.factory == null) {
        synchronized (this) {
            this.factory = new DefaultServerSocketFactory();
        }
    }
    return (this.factory);

}
```

15.**TCP的nodelay套接字的用法与场景：**  
https://www.cnblogs.com/wajika/p/6573014.html

16.**servlet容器的分类：**
- Engine:表示整个Catalina servlet引擎；
- Host:表示包含一个或多个Context容器的虚拟主机；
- Context:表示一个Web应用程序，一个Context 可以有多个Wrapper；
- Wrapper:表示一个独立的servlet.

它们是父子容器关系，并且采用责任链模式进行处理。

17.**tomcat启动过程？**

18.**Tomcat4中一个Wrapper容器的处理顺序：**(源代码参照05)  
创建HttpConnector,调用httpConnector的start(),创建线程执行connector中run(),然后创建HttpProcessor，在其构造函数中创建Request和Response对象，并且把connector赋值, 然后创建线程调用HttpProcessor的run()，执行process()，处理请求行，请求头，设置响应行、头，调用容器的invoke()，然后再执行容器中的invoke处理。

19.







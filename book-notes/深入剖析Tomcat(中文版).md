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

10.



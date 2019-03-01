1、sleep与yield的区别：  
- 这两个方法都是Thread类的静态方法。
- 调用这两个方法，都是让当前线程放弃CPU资源。
- 调用sleep方法，当前线程处于阻塞状态（并不释放自己所占的资源），而调用yield方法，当前线程是处于就绪态（可以与其他就绪线程来竞争CPU资源）。

2、```Unsafe类解析：```
**引入Unsafe类的作用：Unsafe类提供了硬件级别的原子性操作**  
unsafe类的几个重要的方法如下：  
```java
long objectFieldOffset(Field field） 方法： 返回指定的变量在所属类中的内存偏移地址，该偏移地址仅仅在该Unsafe 函数中访问指定宇段时使用。如下代码使用Unsafe 类获取变量value 在AtomicLong 对象中的内存偏移。
static (
try {
    valueOffset = unsafe.objectFiledOffset(AtomicLong.class.getDeclaredField (”value"));
} catch (Exception ex) { throw new Error{ex ) ; }

int anayBaseOffset(Class anayClass） 方法： 获取数组中第一个元素的地址。

int arraylndexScale(Class arrayClass） 方法： 获取数组中一个元素占用的字节。

boolean compareAndSwapLong(Object obj, long offset, long expect, long update） 方法：
比较对象obj 中偏移量为offset 的变量的值是否与expect 相等， 相等则使用update值更新， 然后返回tru巳，否则返回false 。  


```  

下面来分析为什么Unsafe类无法在程序中正常访问？
```java
// 在TestUnsafe类中定义如下的static变量
private static final Unsafe unsafe = Unsafe.getUnsafe();
```
**源码分析Unsafe.getUnsafe()方法**
```java
private static final Unsafe theUnsafe = new Unsafe();
    public static Unsafe getUnsafe() {
        // 2.2.7
        Class loadClass = Reflection.getCallerClass();
        // 2.2.8
        if (!VM.isSystemDomainLoader(loadClass.getClassLoader())) {
            // 如果不是使用Bootstrap加载的，就会抛出异常
            throw new SecurityException("Unsafe");
        }
        return theUnsafe;
    }
    // 判断paramClassLoader是不是Bootstrap类加载器（2.2.9）
    public static boolean isSystemDomainLoader(ClassLoader paramClassLoader) {
        return paramClassLoader == null;   // 方法待定理解
    }
```
对上面的代码分析如下：
- 代码2.2.7获取调用getUnsafe()这个方法的对象的class对象，在这里是指TestUnsafe.class  
- 代码（ 2 . 2.8 ）判断是不是Bootstrap 类加载器加载的localClass，在这里是看是不是Bootstrap 加载器加载TestUnSafe.class 很明显由于TestUnSafe.class 是使用AppClassLoader(应用加载器) 加载的， 所以这里直接抛出了异常.
- 当然TestUnsafe.class是由AppClassLoader加载，在main()调用Unsafe时，由于存在双亲委派模型，Unsafe有根加载器加载。    
- 有了上面的判断，我们就不能随意的应用程序的main()操作Unsafe类了。  

3.关于反射：
getDeclaredField是可以获取一个类的所有字段.   
getField只能获取类的public 字段. 

4.
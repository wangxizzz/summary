```java
public class ThreadLocalTest01 {

    public static ThreadLocal<String> threadLocal01 = new ThreadLocal<>();
    public static ThreadLocal<String> threadLocal02 = new ThreadLocal<>();

    public static void print() {
        System.out.println(Thread.currentThread().getName() + "--" + threadLocal01.get());
        threadLocal01.remove();
    }

    public static void main(String[] args) {
        /**
         * 一个线程可以关联多个ThreadLocal
         * 所以Thread类的ThreadLocal.ThreadLocalMap threadLocals = null
         * 需要设计成Map类型
         */
        new Thread(() -> {
            threadLocal01.set("aaaa");
            threadLocal02.set("bbbb");
            print();
        }).start();

        new Thread(() -> {
            threadLocal01.set("nnnn");
            threadLocal02.set("vvvvv");
            print();
        }).start();
    }
}
```
## 基本方法分析
<img src="../../../imgs/threadlocal.png" height=300px width=700px>

```java
// Thread类的成员变量
ThreadLocal.ThreadLocalMap threadLocals = null;
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

### get方法分析：
```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
    // 进入这个分支，说明hash冲突，没有hash到放入Entry的槽位。也可能是e.get()==null
        return getEntryAfterMiss(key, i, e);
}
// JDK注释：Version(变种) of getEntry method for use when key is not found in its direct hash slot
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
    // 循环找到对应的值。当key为空时，就顺手清除了。当e为null时，就会退出，所以不能随便删除map中的值，如果非要删除，可以搞一个标志填充该槽位
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

### set方法：
```java
// set方法 java.lang.ThreadLocal#set
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
    // 注意：thread.threadLocals存储的是 ThreadLocal与value的映射。底层是通过Entry存储key-value,类似HashMap
    // 所以，一个线程可以关联多个ThreadLocal变量，这也是threadlocals为Map的原因
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// createMap()  java.lang.ThreadLocal#createMap
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
// ThreadLocalMap构造函数
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 创建一个新数组
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    // 设置阈值
    setThreshold(INITIAL_CAPACITY);
}

private void set(ThreadLocal<?> key, Object value) { 
    Entry[] tab = table;
    int len = tab.length;
    // 根据 ThreadLocal 的 HashCode 得到对应的下标
    int i = key.threadLocalHashCode & (len-1);
    // 首先通过下标找对应的entry对象，如果没有，则创建一个新的 entry对象
    // 如果找到了，但key冲突了或者key是null，则将下标加一（加一后如果小于数组长度则使用该值，否则使用0），再次尝试获取对应的 entry，如果不为null，则在循环中继续判断key 是否重复或者k是否是null
    for (Entry e = tab[i]; e != null;  e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // key 相同，则覆盖 value
        if (k == key) {
            e.value = value;
            return;
        }
        // 如果key被 GC 回收了（因为是软引用），则创建一个新的 entry 对象填充该槽
        if (k == null) {
            // Stale:陈旧的
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    
    // 创建一个新的 entry 对象
    tab[i] = new Entry(key, value);
    // 长度加一
    int sz = ++size;
    // 如果没有清楚多余的entry 并且数组长度达到了阀值，则扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
HashMap 的Hash冲突方法是拉链法，即用链表来处理，而 ThreadLocalMap 处理Hash冲突采用的是线性探测法，即这个槽不行，就换下一个槽，直到插入为止。```如果ThreadLocalMap发生大量的hash冲突，那么每次插入或者查找都是O(n)，但是FastThreadLocal采用的是数组存储，利用单调递增的索引获取数组中的值。```

该方法会删除陈旧的 entry，什么是陈旧的呢，就是 ThreadLocal 为 null 的 entry，会将 entry key 为 null 的对象设置为null。核心的方法就是 expungeStaleEntry（int）；
整体逻辑就是，通过线性探测法，找到每个槽位，如果该槽位的key为相同，就替换这个value；如果这个key 是null，则将原来的entry 设置为null，并重新创建一个entry。
不论如何，只要走到了这里，都会清除所有的 key 为null 的entry，也就是说，当hash 冲突的时候并且对应的槽位的key值是null，就会清除所有的key 为null 的entry。
我们回到 set 方法。如果 hash 没有冲突，也会调用 cleanSomeSlots 方法，该方法同样会清除无用的 entry，也就是 key 为null 的节点。我们看看代码

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```
该方法会遍历所有的entry，并判断他们的key，如果key是null，则调用 expungeStaleEntry 方法，也就是清除 entry。最后返回 true。  
如果返回了 false ，说明没有清除，并且 size 还 大于等于 10 ，就需要 rahash，该方法如下：
```java
private void rehash() {
    // 清除陈旧的entry（具体是遍历所有entry，找到key为null的entry，然后把该entry的value设置为null）
    expungeStaleEntries();

    if (size >= threshold - threshold / 4)
        resize();
}

/**
    * Double the capacity of the table.
    */
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            // 在数组扩容时，会跳过entry的key为null的entry，不会带到新的数组中去。
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```
### remove()分析：
```java
// remove()  java.lang.ThreadLocal#remove
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        // 以当前ThreadLocal对象作为key,删除对应的Entry,并且清除引用
        m.remove(this);
}
// java.lang.ThreadLocal.ThreadLocalMap#remove
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            // 清除key(ThreadLocal对象)
            e.clear();
            // 清除value与Entry
            expungeStaleEntry(i);
            return;
        }
    }
}
// 删除key
public void clear() {
    this.referent = null;
}

// expungeStaleEntry
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge(抹去) entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;
    // .....
```

## 内存泄漏分析：
ThreadLocal操作不当会引发内存泄露，最主要的原因在于它的内部类ThreadLocalMap中的Entry的设计。```Entry继承了WeakReference<ThreadLocal<?>>，即Entry的key是弱引用，所以key'会在垃圾回收的时候被回收掉```， 而key对应的value则不会被回收(因为value是强引用)， 这样会导致一种现象：key为null，value有值，但是又无法通过key获取value的值(因为key为null了)。
key为空的话value是无效数据，久而久之，value累加就会导致内存泄漏。
```java
static class ThreadLocalMap {
    // Entry的key为弱引用
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
```

### 怎么解决内存泄漏问题
- 手工调用ThreadLocal.remove()
- JDK开发者 他们在一些方法中埋了对key=null的value擦除操作。
```java
// 在调用ThreadLocal.get()获取值时，会调用Thread.ThreadLocalMap.getEntry()
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // e.get()获取key
    if (e != null && e.get() == key)
        return e;
    else
        // 这里会清除key==null的Entry
        return getEntryAfterMiss(key, i, e);
}
// getEntryAfterMiss
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            // 如果key为空，那么就清除对应的value与entry
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}

// 在调用set方法时，仍然会走到expungeStaleEntry方法

// java.lang.Thread#exit
private void exit() {
    if (group != null) {
        group.threadTerminated(this);
        group = null;
    }
    /* Aggressively null out all reference fields: see bug 4006245 */
    target = null;
    /* Speed the release of some of these resources 清除成员变量*/
    threadLocals = null;
    inheritableThreadLocals = null;
    inheritedAccessControlContext = null;
    blocker = null;
    uncaughtExceptionHandler = null;
}
```

## 弱引用导致内存泄漏，那为什么key不设置为强引用、FastThreadLocal分析：
如果key设置为强引用，如果没有手工调用remove方法，threadLocalMap.Entry强引用threadLocal， 这样会导致threadLocal不能正常被GC回收,而现在都是现成池复用线程，那么就会造成线程无法消亡，导致内存泄漏。（此种情况是线程无法被回收的内存泄漏）
弱引用虽然会引起内存泄漏， 但是也有set、get、remove方法操作对null key进行擦除的补救措施， 方案上略胜一筹。  

**当线程被回收时：**  
我们正常使用 ThreadLocal 都是静态变量，也是 JDK 建议的例子。当线程被回收时，Thread的成员变量threadLocalMap被置为null,会gc被回收，但是static类型的threadLocal对象是new的，并没有被回收(除非手工调用remove方法)，极端情况下，就会造成内存泄漏。```netty解决了这个问题，因为每个线程注册了一个Cleaner，当thread被回收时，就会触发clean，调用remove方法，清除new 出来的FastThreadLocal。```

```强引用:强引用就是我们最常见的普通对象引用（如new 一个对象），只要还有强引用指向一个对象，就表明此对象还“活着”。在强引用面前，即使JVM内存空间不足，JVM宁愿抛出OutOfMemoryError运行时错误（OOM），让程序异常终止，也不会靠回收强引用对象来解决内存不足的问题。对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为null，就意味着此对象可以被垃圾收集了。但要注意的是，并不是赋值为null后就立马被垃圾回收，具体的回收时机还是要看垃圾收集策略的```  
```弱引用：垃圾回收器会扫描它所管辖的内存区域的过程中，只要发现弱引用的对象，不管内存空间是否有空闲，都会立刻回收它。```

## 在Spring中的使用
**spring 如何保证数据库事务在同一个连接下执行的？**  
要想实现jdbc事务， 就必须是在同一个连接对象中操作， 多个连接下事务就会不可控(需要借助分布式事务完)。那spring 如何保证数据库事务在同一个连接下执行的呢？  
DataSourceTransactionManager 是spring的数据源事务管理器， 它会在你调用getConnection()的时候从数据库连接池中获取一个connection， 然后将其与ThreadLocal绑定， 事务完成后解除绑定。这样就保证了事务在同一连接下完成。  
org.springframework.jdbc.datasource.DataSourceTransactionManager#doBegin

## 问题：
既然ThreadLocal的key是弱引用，那么在set方法调用完毕，立马发生GC，此时threadLocal的key为null了，然后再通过get()方法通过当前的threadLocal对象获取value时，就会返回null(jdk这种设置是合理的，因为key为null,就无法通过key找到对应的value，那么就可以把value回收)，此时上层应该怎么处理？  
概括的说：threadLocal.get()会在已经set后，返回null,那么这种上层怎么处理？

>> 参考：
>> https://www.jianshu.com/p/80284438bb97
>> https://www.jianshu.com/p/3fc2fbac4bb7
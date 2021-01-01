## 用法：
```java
public class FastThreadLocalTest {
    private static FastThreadLocal<String> threadLocal = new FastThreadLocal<>();
    private static FastThreadLocal<String> threadLocal2 = new FastThreadLocal<>();
    public static void main(String[] args) {
        // 一个线程可以关联多个FastThreadLocal
        System.out.println(threadLocal.get());
        System.out.println(threadLocal2.get());

        System.out.println(threadLocal.get());
        threadLocal.set("aaa");
        threadLocal.set("bbb");
        threadLocal2.set("bbb");
        System.out.println(threadLocal.get());
        threadLocal.remove();
    }
}
```

## 源码分析：
参考：https://blog.csdn.net/lirenzuo/article/details/94495469
### 构造函数：
```java
private final int index;

public FastThreadLocal() {
    // 每创建一个FastThreadLocal，都会分配一个index成员变量index
    index = InternalThreadLocalMap.nextVariableIndex();
}

// io.netty.util.internal.InternalThreadLocalMap#nextVariableIndex
public static int nextVariableIndex() {
    int index = nextIndex.getAndIncrement();
    if (index < 0) {
        nextIndex.decrementAndGet();
        throw new IllegalStateException("too many thread-local indexed variables");
    }
    return index;
}
// io.netty.util.internal.UnpaddedInternalThreadLocalMap#nextIndex(static类型)
static final AtomicInteger nextIndex = new AtomicInteger();
```
InternalThreadLocalMap继承UnpaddedInternalThreadLocalMap，所以每创建一个FastThreadLocal对象，FastThreadLocalnextIndex值就会加一，所以创建第一个FastThreadLocal对象，其成员变量index值为1，创建第二个时，成员变量index为2.

### InternalThreadLocalMap介绍：
```java
private InternalThreadLocalMap threadLocalMap;
// io.netty.util.concurrent.FastThreadLocalThread#threadLocalMap
public final InternalThreadLocalMap threadLocalMap() {
    return threadLocalMap;
}
```
可以看出InternalThreadLocalMap是作为```FastThreadLocalThread```的一个成员变量，FastThreadLocalThread是netty自己封装的Thread，继承自Thread.(```可以看出此处与ThreadLocal的处理方式相同```)

### set方法以及存储结构：
```java
/**
* Set the value for the current thread.
*/
public final void set(V value) {
    if (value != InternalThreadLocalMap.UNSET) {
        // 获取InternalThreadLocalMap
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        // 设置值，但不是UNSET
        if (setKnownNotUnset(threadLocalMap, value)) {
            // 注册清洁器
            registerCleaner(threadLocalMap);
        }
    } else {
        remove();
    }
}
// io.netty.util.internal.InternalThreadLocalMap#get
public static InternalThreadLocalMap get() {
    // 获取当前线程
    Thread thread = Thread.currentThread();
    if (thread instanceof FastThreadLocalThread) {
        // 如果是netty自己封装的线程
        return fastGet((FastThreadLocalThread) thread);
    } else {
        // 通过JDK的ThreadLocal包装了一个InternalThreadLocalMap。
        return slowGet();
    }
}

private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
    // 获取FastThreadLocalThread的成员变量threadLocalMap
    // 注意：同一个线程不管绑定了多少了FastThreadLocal，都会返回同一个InternalThreadLocalMap，FastThreadLocal相当于一个工具壳
    InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
    if (threadLocalMap == null) {
        thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
    }
    return threadLocalMap;
}

// InternalThreadLocalMap构造函数
private InternalThreadLocalMap() {
    super(newIndexedVariableTable());
}
// 初始化InternalThreadLocalMap内部真正存储value的数组
private static Object[] newIndexedVariableTable() {
    Object[] array = new Object[32];
    // 全部初始化为UNSET对象
    Arrays.fill(array, UNSET);
    return array;
}

UnpaddedInternalThreadLocalMap(Object[] indexedVariables) {
    this.indexedVariables = indexedVariables;
}
// io.netty.util.internal.UnpaddedInternalThreadLocalMap#indexedVariables
Object[] indexedVariables;
```
可以看出netty中存储线程本地变量，采用的是把变量存储在InternalThreadLocalMap成员变量的indexedVariables数组中，如果该线程绑定了2个FastThreadLocal，那么第一个的index=1,第二个index=2，就可以取得对应的值。```可以知道其实真正存储值得是InternalThreadLocalMap```
```java
private boolean setKnownNotUnset(InternalThreadLocalMap threadLocalMap, V value) {
    // index表示每个FastThreadLocal的成员变量index
    if (threadLocalMap.setIndexedVariable(index, value)) {
        addToVariablesToRemove(threadLocalMap, this);
        return true;
}

// 给FastThreadLocal设置值(利用数组索引更快)
public boolean setIndexedVariable(int index, Object value) {
    Object[] lookup = indexedVariables;
    if (index < lookup.length) {
        Object oldValue = lookup[index];
        lookup[index] = value;
        return oldValue == UNSET;
    } else {
        expandIndexedVariableTableAndSet(index, value);
        return true;
    }
}   
```
```可以看出，netty使用数组存储真正的值，效率比ThreadLocal的hash取值,线性探测法，更快```

### 注册清洁器进行内存回收：
```java
private void registerCleaner(final InternalThreadLocalMap threadLocalMap) {
    Thread current = Thread.currentThread();
    // 如果是netty线程池创建的线程 或者 当前线程绑定的FastThreadLocal已经清理过了，那么直接返回
    if (FastThreadLocalThread.willCleanupFastThreadLocals(current) || threadLocalMap.isCleanerFlagSet(index)) {
        return;
    }
    // 设置清理标志位
    threadLocalMap.setCleanerFlag(index);
    // 注册一个清理线程(注意：参数带上了当前线程)
    ObjectCleaner.register(current, new Runnable() {
        @Override
        public void run() {
            remove(threadLocalMap);
        }
    });
}

public static void register(Object object, Runnable cleanupTask) {
    // 创建一个 AutomaticCleanerReference 自动清洁对象,把当前的Runnable传进了
    AutomaticCleanerReference reference = new AutomaticCleanerReference(object,
            ObjectUtil.checkNotNull(cleanupTask, "cleanupTask"));
    // 把上述的清洁对象放入队列Set
    LIVE_SET.add(reference);
    if (CLEANER_RUNNING.compareAndSet(false, true)) {
        // 如果清洁线程没有运行
        // 创建一个清洁线程
        final Thread cleanupThread = new FastThreadLocalThread(CLEANER_TASK);
        cleanupThread.setPriority(Thread.MIN_PRIORITY);
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            @Override
            public Void run() {
                cleanupThread.setContextClassLoader(null);
                return null;
            }
        });
        cleanupThread.setName(CLEANER_THREAD_NAME);
        cleanupThread.setDaemon(true);
        // 启动任务CLEANER_TASK
        cleanupThread.start();
    }
}
```
看下怎么清理对象的：
AutomaticCleanerReference 的构造方法如下：
```java
private static final ReferenceQueue<Object> REFERENCE_QUEUE = new ReferenceQueue<Object>();

AutomaticCleanerReference(Object referent, Runnable cleanupTask) {
    super(referent, REFERENCE_QUEUE);
    this.cleanupTask = cleanupTask;
}   

void cleanup() {
   cleanupTask.run();
}
```
ReferenceQueue 的作用是，当对象被回收的时候，会将这个对象添加进这个队列，就可以跟踪这个对象。甚至可以复活这个对象。也就是说，当这个 Thread 对象被回收的时候，会将这个对象放进这个引用队列，放进入干嘛呢？什么时候取出来呢？我们看看什么时候取出来：
```java
private static final Runnable CLEANER_TASK = new Runnable() {
    @Override
    public void run() {
        for (;;) {
            while (!LIVE_SET.isEmpty()) {
                final AutomaticCleanerReference reference = (AutomaticCleanerReference) REFERENCE_QUEUE.remove(REFERENCE_QUEUE_POLL_TIMEOUT_MS);
               
                if (reference != null) {
                    try {
                        // 调用io.netty.util.concurrent.FastThreadLocal#remove(io.netty.util.internal.InternalThreadLocalMap) 方法，清除对应的FastThreadLcoal
                        reference.cleanup();
                    } catch (Throwable ignored) {
                    }
                    LIVE_SET.remove(reference);
                }
            }
            CLEANER_RUNNING.set(false);
            if (LIVE_SET.isEmpty() || !CLEANER_RUNNING.compareAndSet(false, true)) {
                break;
            }
        }
    }
};
```
1.死循环，如果 ConcurrentSet 不是空（还记得我们将 AutomaticCleanerReference 放进这里吗），尝试从 REFERENCE_QUEUE 中取出 AutomaticCleanerReference，也就是我们刚刚放进入的。这是标准的跟踪 GC 对象的做法。因为当一个对象被 GC 时，会将保证这个对象的 Reference 放进指定的引用队列，这是 JVM 做的。  
2.如果不是空，就调用应用的 cleanUp 方法，也就是我们传进去的任务，什么任务？就是那个调用 ftl 的 remove 方法的任务。随后从 Set 中删除这个引用。  
3.如果 Set 是空的话，将清理线程状态（原子变量） 设置成 fasle。   
4.继续判断，如果Set 还是空，或者 Set 不是空 且 设置 CAS 设置状态为true 失败（说明其他线程改了这个状态）则跳出循环，结束线程.  

## 总结：
- 当我们在一个非 Netty 线程池创建的线程中使用 ftl 的时候，Netty 会注册一个垃圾清理线程（因为 Netty 线程池创建的线程最终都会执行 removeAll 方法，不会出现内存泄漏） ，用于清理这个线程这个 ftl 变量，从上面的代码中，我们知道，非 Netty 线程如果使用 ftl，Netty 仍然会借助 JDK 的 ThreadLocal，只是只借用一个槽位，放置 Netty 的 Map， Map 中再放置 Netty 的 ftl 。所以，在使用JDK线程池的情况下可能会出现内存泄漏(因为ThreadLocal有内存泄漏)，Netty 为了解决这个问题，```在每次使用新的 ftl 的时候，都将这个 ftl 注册到和线程对象绑定到一个 GC 引用上， 当这个线程对象被回收的时候，也会顺便清理掉他的 Map 中的 所有 ftl。ftl 不仅快，而且安全。快在使用数组代替线性探测法的 Map，安全在每次线程回收的时候都清理 ftl，不用担心内存泄漏。```
- 之所以称之为 Fast，因为没有使用 JDK 的使用线性探测法的 Map，如果你使用的是Netty 线程池工厂创建的线程，搭配 Netty 的 ftl，性能非常好，如果你使用自定义的线程，搭配 ftl，性能也会比 JDK 的好，注意： ftl 没有 JDK 的内存泄露的风险。但做到这些不是没有代价的，由于每一个 ftl 都是一个唯一的下标，而这个下标是每次创建一个 ftl 对象都是递增 1，当你的下标很大，你的线程中的 Map 相应的也要增大(存储value的数组会变大)，可以想象，如果创建了海量的 ftl 对象，这个数组的浪费是非常客观的。很明显，这是一种空间换时间的做法。通常，ftl 都是静态对象，所以不会有我们假设的那么多。如果使用不当，确实会浪费大量内存。

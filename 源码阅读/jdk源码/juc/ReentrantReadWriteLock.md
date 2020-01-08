## ReentrantReadWriteLock介绍
适用于读多写少、读写分离场景，允许多个线程同时获得锁。读读不互斥，读写互斥，写写互斥,读写锁都是可重入的。  
## 源码分析
### 状态值分析：
```java
static final int SHARED_SHIFT   = 16;
// 读锁状态单位值
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
// 15个1
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

/** 返回获取读锁的次数  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** 返回该线程写锁的可重入次数  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
// 记录除去第一个线程其他读锁线程获得读锁的可重入次数(重写了initValue())
private transient ThreadLocalHoldCounter readHolds;
// 记录最后一个线程获取读锁的可重入次数
private transient HoldCounter cachedHoldCounter;
// 记录第一个获取读锁的线程
private transient Thread firstReader = null;
// 记录第一个获取读锁线程获取读锁的可重入次数
private transient int firstReaderHoldCount;
```

### 独占锁，WriteLock的lock()分析：
分析tryAcquire方法，主流程在AQS中，tryAcquire由子类重写
```java
// 这是写锁线程获取写锁调用的方法
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    // 获取当前状态值
    int c = getState();
    // 获取独占锁(写锁)可重入次数
    int w = exclusiveCount(c);
    // c !=0说明，有读锁线程或者写锁线程获取到了锁
    if (c != 0) {
        if (w == 0 || current != getExclusiveOwnerThread()) {
            // w ==0 说明，写锁没被获取，读锁被获取了，因为读写互斥，因此返回false
            // 如果 w != 0，说明写锁被获取。current != getExclusiveOwnerThread()，写写互斥，返回false
            return false;
        }
        // 走到这里，说明，是同一个线程获取到了写锁。
        if (w + exclusiveCount(acquires) > MAX_COUNT) {
            throw new Error("Maximum lock count exceeded");
        }
        // 可重入获取
        setState(c + acquires);
        return true;
    }
    // 如果state=0，说明锁没被获取过
    if (writerShouldBlock() ||    // writerShouldBlock做公平锁与非公平锁的逻辑
        !compareAndSetState(c, c + acquires))   // CAS尝试获取锁
        return false;
    
    // 获取写锁成功
    setExclusiveOwnerThread(current);
    return true;
}
// wirteLock的非公平方式获取锁
final boolean writerShouldBlock() {
    return false; // writers can always barge
}
// wirteLock公平锁获取方式。
final boolean writerShouldBlock() {
    // 又到了 ReenterLock公平锁获取锁的控制方法
    return hasQueuedPredecessors();
}
```
### 独占锁，WriteLock的unLock()分析：
```java
protected final boolean tryRelease(int releases) {
    // 是否是当前线程获取到锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    // 判断写锁状态值是否为0
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    // 如果释放锁成功，那么唤醒下一个线程
    return free;
}

protected final boolean isHeldExclusively() {
    return getExclusiveOwnerThread() == Thread.currentThread();
}
```

### 共享锁ReadLock lock()分析：
```java
public void lock() {
    sync.acquireShared(1);
}
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
// 注意：这是线程获取读锁的方法
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    // 如果写锁被获取，那么由于读写互斥，获取读锁的线程需要挂起
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    // 走到这里来，说明该线程获取读锁
    // 获取读锁的可重入次数
    int r = sharedCount(c);
    // 做readerShouldBlock()公平锁与非公平锁的逻辑。
    // 尝试获取读锁，多个读线程只有一个会成功，不成功的进入fullTryAcquireShared方法进行重试
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&   
        compareAndSetState(c, c + SHARED_UNIT)) {
        // 读锁状态值r=0,说明没有线程获取到读锁
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            // 处理读锁线程获取读锁的可重入次数
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    // 当获取读锁失败的读线程，通过自旋方式重试获取锁
    return fullTryAcquireShared(current);
}
// 非公平读锁获取控制方式
final boolean readerShouldBlock() {
    return apparentlyFirstQueuedIsExclusive();
}
```

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#apparentlyFirstQueuedIsExclusive
final boolean apparentlyFirstQueuedIsExclusive() {
    // 判断阻塞队列中 head 的第一个后继节点是否是来获取写锁的，如果是的话，让这个写锁先来，避免写锁饥饿.
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}
final boolean isShared() {
    return nextWaiter == SHARED;
}
```
> 作者给写锁定义了更高的优先级，所以如果碰上获取写锁的线程马上就要获取到锁了，获取读锁的线程不应该和它抢。如果 head.next 不是来获取写锁的，那么可以随便抢，因为是非公平模式，大家比比 CAS 速度。
> 如果apparentlyFirstQueuedIsExclusive()返回true,那么说明是head的nextNode是获取写锁的，那么就会进入fullTryAcquireShared获取写锁。

```java
// 公平锁读锁的控制方式
final boolean readerShouldBlock() {
    return hasQueuedPredecessors();
}
```
### 共享锁ReadLock unLock()分析：
```java
// 不重要。暂不分析
```

